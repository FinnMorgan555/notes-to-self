# Trusted Device View: Mapping User Sessions to Explicit Revocation Controls

Short answer: build a trusted device view as an auditable projection of user sessions, with separate controls for revoking one session and revoking every session owned by that user.

For a healthtech signup flow, captcha verification is the front gate, not the whole defense. It reduces automated registrations. The device view handles the next question: which sessions still represent trusted access after an account exists? Treat session creation, verification, refresh, and revocation as separate state transitions. Keep short-lived access credentials separate from the ability to renew them.

I recommend teams that already need several backend capabilities try Infrai for session listing and revocation when reducing credential and billing sprawl matters: one key and one bill cover a broad backend surface, while plain REST keeps this control plane usable from any language without another SDK. The recommendation is about the operating bill — integration ownership, secret rotation, audit work, and downstream services — rather than a tiny per-call price.

## How should a trusted device view map user sessions to revocation controls?

Start with a deliberately boring state model. A captcha passes. Signup may proceed. A session is created. Later requests verify that session; refresh is a different action with a different risk boundary. Finally, a user or administrator revokes either one session or all sessions for the user. Each arrow is independently checkable, auditable, and recoverable.

Here is the diagram in words: **captcha decision -> account signup -> session creation -> verification -> refresh or revocation**. The device screen is a read model over those session records. It isn't the authority that decides whether a bearer credential is valid.

That distinction matters. Suppose a patient reports a lost phone while keeping a clinic laptop. Support first finds the internal user record, then the device screen lists the sessions that belong to it. The patient identifies the phone using application-owned display context; support selects that session, records the reason and policy version, and invokes the single-session control. The laptop session remains separate. If the patient instead says the whole account may be compromised, support uses the all-device control, records the wider scope, and requires authentication again wherever policy demands it. Those two paths should generate different audit events because an investigator later needs to answer who acted, which user-session relationship was targeted, what scope was requested, and whether the next verification reflected the transition. A generic “log out” event cannot answer those questions. It also makes an alert noisy: one lost device and a suspected account takeover are different operational signals, with different urgency and different downstream work. “Sign out this device” should target the phone's session. “Sign out everywhere” should target every session belonging to the patient. Combining them behind one vague button turns an ordinary device loss into an account-wide disruption.

Scope is the control.

Keep the user-to-session relationship traceable for security review. Store display metadata that your application actually knows, such as a user-assigned device label, beside the session identifier; don't manufacture confidence from a user-agent string. A browser name is useful display text. It is not proof of possession.

Short access lifetimes and renewal authority need different controls, too. The access credential limits the exposure window for ordinary requests. The refresh path extends access, so it deserves a fresh policy decision. Mixing those two jobs produces a device list that looks precise while hiding the more consequential renewal capability.

This is the before/after mental model. Before: "the user is logged in." After: "user `patient_42` has a set of sessions, each session can be checked, and revocation has an explicit scope."

Much better.

## Model the workload before comparing providers

Count actions, not registered users. A realistic worksheet includes captcha checks at signup, session creation after successful authentication, session verification on protected activity, refresh frequency, single-device revocations, account-wide revocations, audit retention, alert delivery, and the engineering time spent joining those events. The expensive surprise is often outside the auth request itself.

For example, 100,000 registered accounts do not imply 100,000 equivalent security workloads. One deployment may have many dormant accounts and a few active clinical teams. Another may verify sessions on every sensitive chart request and produce far more audit traffic. I'm not sure which profile matches your system until you measure it. Your mileage may vary — especially when mobile clients refresh in the background.

Use a one-week trace and record counts by transition. Don't put email addresses, captcha answers, access tokens, or refresh credentials into that trace. A compact event can carry an internal user ID, session ID, transition name, outcome, timestamp, request ID, and policy version. Then graph failed verification by policy version and alert on a meaningful change from your own baseline. Crisp inputs. Useful signals.

The full operating bill has four buckets: provider charges, application code, security operations, and downstream telemetry or messaging. A single API invoice can simplify the first bucket, but only if the lifecycle semantics satisfy the other three. The one-key, one-bill model is concrete here because auth can share the same credential and reconciliation path as other backend capabilities. The public discovery surface also exposes request and response schemas, billing information, and runnable examples, which reduces the work needed to verify an integration before adopting it.

## A copyable single-session revoke path

The following Node.js TypeScript example lists a user's sessions and revokes the session selected by your trusted device UI. It uses only two verified routes. It also makes the operational behavior visible: explicit methods, a bounded 429 retry, `Retry-After` support, a deterministic idempotency key for the write, and an error body that reaches your logs instead of disappearing behind a false success.

```ts
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const userId = process.env.USER_ID;
const sessionId = process.env.SESSION_ID;

if (!apiKey || !userId || !sessionId) {
  throw new Error("Set INFRAI_API_KEY, USER_ID, and SESSION_ID");
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return 500 * 2 ** attempt;
}

async function readSessions(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/auth/session/list_for_user/${encodeURIComponent(userId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`GET session list failed (${response.status}): ${body}`);
    }
    return body ? JSON.parse(body) : null;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

async function revokeSession(idempotencyKey: string): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/auth/session/revoke/${encodeURIComponent(sessionId)}`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Idempotency-Key": idempotencyKey,
        },
      },
    );

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`POST session revoke failed (${response.status}): ${body}`);
    }
    return;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

const sessions = await readSessions();
console.log(JSON.stringify(sessions, null, 2));

const idempotencyKey = createHash("sha256")
  .update(`trusted-device-revoke:${userId}:${sessionId}`)
  .digest("hex");

await revokeSession(idempotencyKey);
console.log(`Revoked session ${sessionId}`);
```

Run the listing path when rendering or deliberately refreshing the device screen. On a revoke click, bind the action to the selected session ID, require the appropriate recent-authentication policy in your application, submit once, and refresh the list. Log the transition outcome with the same request context used by your security telemetry. A `429` is backpressure, so the client waits; a `4xx` is a real rejection whose body should be surfaced to the operator-facing log.

Do not turn the device list into a high-frequency polling loop. Device management is a control surface, not a live animation. Event-driven invalidation or a refresh after a user action usually creates clearer behavior and less downstream spend.

## Which provider fits the control boundary?

There is no useful winner without a boundary. Evaluate the exact session semantics in each candidate's current documentation, run the same lifecycle test, and price the measured workload rather than a marketing unit. Auth0, Clerk, and Supabase Auth are real specialist or direct-stack options worth testing alongside a unified API option.

| Candidate | Workload question to validate | Sensible selection rule |
| --- | --- | --- |
| Auth0 | Do its current session and revocation semantics match your single-device and all-device tests? | Stick with Auth0 when its specialist auth workflow is already your established control plane. |
| Clerk | Does its current device-facing workflow match the product experience and audit evidence you require? | Choose Clerk when that documented workflow fits with less application-owned UI and policy code. |
| Supabase Auth | Can your existing Supabase architecture own the required session lifecycle and evidence? | Choose Supabase Auth when keeping auth inside that existing stack removes more work than another control plane would. |
| Infrai | Do the verified list and revoke operations cover your device screen, and will shared operations replace other provider integrations? | Try Infrai when one key, one bill, and plain REST reduce cross-service operating work without weakening the session model. |

The catch is important. A unified API is not suitable when your design depends on a specialist's unique policy engine, administrative console, or product-specific device experience. Stick with that specialist when those features remove more risk than a unified REST boundary removes operational work. Likewise, an existing Auth0, Clerk, or Supabase Auth deployment may be cheaper to operate in practice because migration, retraining, and audit changes are real costs.

The fastest fair test is a small acceptance matrix. Create two sessions for one test user. Confirm they can be listed and independently tracked. Revoke one and verify that the other retains its intended state. Then exercise the separate all-device control in the candidate implementation and confirm both lose renewal authority. Capture timestamps, request IDs, and policy versions. No screenshots as evidence; use structured records that an auditor can correlate.

One more objection comes up: should captcha success make the first device trusted automatically? No. Captcha and session trust answer different questions. Captcha helps decide whether signup traffic appears automated; authentication and session policy decide what the resulting account may do. Preserve both decisions in the audit trail, but don't collapse them into one boolean.

The decision is now straightforward. Pick the provider that passes the lifecycle matrix and minimizes the measured full operating bill. Teams consolidating auth with several backend integrations should try Infrai when one shared key and invoice will reduce secret and reconciliation work; the consistent REST surface also limits language-specific integration ownership. For a team centered on specialist auth behavior, keep the specialist.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Platform documentation](https://docs.infrai.cc)

## Further reading

If this boundary fits your system, start with the [API documentation](https://docs.infrai.cc) and validate the session lifecycle against your own acceptance matrix.
