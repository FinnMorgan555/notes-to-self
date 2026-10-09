# Property Bulk Event Email and SMS System with Status Polling (Queue-Owned Templates)

The least complex bulk event notification system for a property contact form is one database-backed job per intended message, processed by a Node.js batch worker for email or SMS. **TL;DR:** resolve queue, recipient preference, consent, and suppression before dispatch; use Postgres pagination to claim jobs; then poll delivery status separately from business status.

Start with this decision table. It keeps template ownership explicit before a worker sends anything.

| Template owner | Pick this when | Main risk | Required control |
|---|---|---|---|
| Central communications team | Legal wording and brand voice must stay uniform across properties | Queue-specific instructions become vague | Versioned variables and queue approval |
| Support queue | Leasing, maintenance, and billing need different next steps | Templates drift between teams | Schema validation and periodic review |
| Property team | Addresses, hours, and escalation details change locally | Local edits can alter mandatory language | Locked sections plus reviewed local fields |

For a contact form, support-queue ownership is usually the cleanest boundary. The queue understands what the reply must ask for. A communications team can still own the rendering shell, and a property team can supply validated facts.

Three owners. Three narrow responsibilities.

## Which team should own the message template?

Pick central ownership when the message is mostly invariant: a receipt that says the request arrived, identifies the property, and gives a response expectation. Central ownership makes one review cover every queue. It also means adding a queue-specific diagnostic field requires coordination.

Pick support-queue ownership when content changes the operational outcome. A maintenance acknowledgment may request permission to enter. A leasing acknowledgment may ask for a move-in window. Those are workflow decisions, so the queue should review the wording and the variables it consumes.

Pick property ownership only for genuinely local facts. Keep those facts typed. `officeHours` is data; a free-form replacement for the whole message is a second template system hiding inside a text field.

The useful boundary is small: the queue owns intent, central communications owns the frame, and the property supplies facts. **Ownership should follow the decision encoded by the text.**

## Model intent before delivery

A form submission should first become an immutable notification intent. Do not make the web request wait for email or SMS. Persist the routing result, template version, normalized destination reference, and preference snapshot in one transaction, then let workers dispatch later.

Preferences and suppressions answer different questions. A preference says what the recipient chose. A suppression says a destination must not receive a class of message, even if an older preference says otherwise. Evaluate both before enqueueing and again immediately before dispatch, because a queued job can outlive a preference change.

Here is the diagram in words: contact form to routing rule; routing rule to queue and template version; policy check to zero, one, or two channel jobs; worker to channel adapter; adapter result to attempt log; polling to final delivery state. The support ticket exists independently throughout.

That separation matters. An accepted SMS does not prove a maintenance request reached the right queue, and a correctly routed ticket does not prove that a message reached a handset.

## How should a Node.js batch notification system poll delivery status?

Use keyset pagination for worker claims. Offset pagination can move underneath a worker as rows change state; a cursor based on `(available_at, id)` gives the next query a stable boundary. Claim a modest batch in a short transaction with `FOR UPDATE SKIP LOCKED`, mark those rows as leased, commit, and perform network calls afterward.

The example below focuses on the ownership and policy boundary. The SQL stays parameterized, and the channel interfaces remain generic.

```ts
type Channel = "email" | "sms";

type Job = {
  id: string;
  contactId: string;
  queue: "leasing" | "maintenance" | "billing";
  templateVersion: string;
  channel: Channel;
  destinationRef: string;
  availableAt: Date;
};

type PolicySnapshot = {
  enabledChannels: ReadonlySet<Channel>;
  suppressedDestinationRefs: ReadonlySet<string>;
};

function mayDispatch(job: Job, policy: PolicySnapshot): boolean {
  return policy.enabledChannels.has(job.channel) &&
    !policy.suppressedDestinationRefs.has(job.destinationRef);
}

interface ChannelAdapter {
  send(input: {
    idempotencyKey: string;
    destinationRef: string;
    templateVersion: string;
    variables: Readonly<Record<string, string>>;
  }): Promise<{ providerMessageId: string; state: "accepted" }>;

  poll(providerMessageId: string): Promise<{
    state: "accepted" | "delivered" | "failed";
    reason?: string;
  }>;
}
```

The worker needs two distinct loops. The dispatch loop claims pending work and records an attempt. The reconciliation loop polls only nonterminal attempts. Consider one maintenance contact that allows SMS at enqueue time, then opts out while waiting behind other work: the dispatch loop reloads policy, records `policy_blocked`, and sends nothing. Now consider a different contact whose adapter call returns `accepted`: that job moves to reconciliation with its provider message ID. If the first poll times out, reconciliation retains the same attempt and polls again; it does not call `send`. Mixing the loops makes that boundary hard to see and makes a duplicate send possible.

```ts
async function dispatchBatch(limit: number): Promise<void> {
  const jobs = await repository.claimPendingJobs(limit);

  for (const job of jobs) {
    const policy = await repository.loadCurrentPolicy(job.contactId);
    if (!mayDispatch(job, policy)) {
      await repository.finishWithoutSend(job.id, "policy_blocked");
      continue;
    }

    const adapter = adapters[job.channel];
    const result = await adapter.send({
      idempotencyKey: job.id,
      destinationRef: job.destinationRef,
      templateVersion: job.templateVersion,
      variables: await repository.loadTemplateVariables(job.id),
    });

    await repository.recordAccepted(job.id, result.providerMessageId);
  }
}

async function reconcileBatch(limit: number): Promise<void> {
  const attempts = await repository.claimUnresolvedAttempts(limit);

  for (const attempt of attempts) {
    const result = await adapters[attempt.channel].poll(
      attempt.providerMessageId,
    );
    await repository.recordDeliveryState(attempt.id, result);
  }
}
```

Use the job ID as the idempotency key where the channel accepts one. Locally, enforce a unique constraint on the logical combination that represents one intended message, such as contact, template version, channel, and destination reference. The adapter's `accepted` result is intermediate. Only a later terminal state should close reconciliation.

Keep the batches bounded. The right limit depends on database capacity, channel quotas, and lease duration, so it should be configured and observed rather than copied from an example.

Predictability wins.

## Observe policy, queueing, and delivery separately

One success counter cannot explain this pipeline. Emit structured events at each boundary with `job_id`, `queue`, `template_version`, `channel`, `policy_result`, `attempt`, and `delivery_state`. Do not put message bodies, email addresses, or phone numbers in labels or logs.

Track queue age and claim latency for the worker. Track policy blocks by reason for preferences and suppressions. Track accepted, delivered, and failed transitions for adapters. These are different failure domains, and their alerts should say which operator can act.

For example, rising queue age with normal claim latency points beyond the claim query. A sudden increase in `policy_blocked` after a preference migration points toward data or rule evaluation. Accepted messages that remain nonterminal belong to reconciliation. Crisp signals make crisp handoffs.

Test the same boundaries. A policy test should prove suppression overrides preference. A concurrency test should run two claimers and assert that each job is leased once. An adapter contract test should prove that polling cannot dispatch. A template test should render every supported version with the exact variable schema its owner approved.

Deploy schema additions before workers that write them. Then deploy readers that tolerate both old and new template versions. Pause template activation if its variable contract is incompatible. This is slower than editing a string in production, and much easier to audit.

## Limits to keep visible

This pattern does not decide whether a message is legally permitted, how long contact data may be retained, or which delivery states a channel can prove. Those answers depend on jurisdiction, message class, and the selected channel contract. The trade-off is operational ownership: the team must run Postgres migrations, leases, retries, policy evaluation, reconciliation, and alerts.

It also cannot eliminate abuse. Public forms need rate limits and anomaly detection before they create outbound work; SMS workflows in particular need controls for traffic pumping. Transactional email guidance also favors clear, expected messages and careful handling of recipient data. Those concerns belong beside the worker, not buried inside it. This approach is not a fit for a team that cannot own an on-call database-backed worker; a managed communications service or an existing internal job platform is then the more honest boundary. A single-channel, low-volume form may need only a transactional outbox and one adapter, not a general batch system.

The final design rule is concise: **store business intent once, but observe every policy and delivery transition.** Template ownership then remains reviewable, support routing remains independent, and retries do not rewrite history.

## Further reading

- https://postmarkapp.com/guides/transactional-email-best-practices
- https://www.twilio.com/docs/verify/preventing-toll-fraud
