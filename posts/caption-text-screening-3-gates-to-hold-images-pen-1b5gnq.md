# Caption Text Screening: 3 Gates to Hold Images Pending Review

Short answer: moderate the caption automatically, keep every uploaded image in `pending` anyway, and publish its responsive thumbnails only after a human decision. The deciding constraint is storage and cache cost: one private original plus a small database record is cheaper to reason about than generating several sizes that may be rejected minutes later.

Treat this as three gates, not one upload callback. Gate 1 screens text. Gate 2 records a private image as pending and puts its ID on a staffed review queue. Gate 3 turns an approval into thumbnail generation and publication, or turns a rejection into cleanup. Notify the uploader after either decision.

That ordering matters. A clean caption does not make its image approved.

Hold it.

## The before-and-after mental model

The tempting flow is linear: receive an upload, resize it, cache the variants, moderate the caption, and then decide whether the support attachment should appear. It feels responsive. It also spends processing, object storage, and cache space before the only decision that matters.

Use a state machine instead. In words, the diagram is: `received -> pending -> approved -> published`, with `pending -> rejected` as the other terminal branch. Caption screening attaches a separate `captionDecision` to the record. It may reject the whole submission early, but it may never advance the image to `approved`.

**Pending is a record state, not another bucket.** Keep the original private. Store an object key, content type, uploader ID, caption decision, review state, and timestamps in the database. Do not copy the bytes merely to express workflow state. On approval, generate the exact thumbnail widths the support UI serves and publish those derivatives. On rejection, remove or expire the private original according to the product's retention policy.

This is a deliberate latency trade-off. The first visible thumbnail waits for review, while rejected uploads consume no derivative storage and do not warm caches with content nobody should see. For a customer-support product, that is usually the right side of the trade: attachments can be sensitive, and review latency is visible to the uploader as an honest pending state.

## How should Node.js screen caption text and hold an image?

The following TypeScript keeps the important invariant inside the transition functions. It uses an in-memory repository so the example runs as one file; replace that map and the two adapters with durable database, moderation, queue, thumbnail, and notification implementations in production. The request contains an `imageKey` created by a separate private-upload step. Never expose that key as a public object URL.

```ts
import express from "express";
import { randomUUID } from "node:crypto";

type CaptionDecision = "allow" | "reject";
type ImageState = "pending" | "approved" | "published" | "rejected";

type Submission = {
  id: string;
  uploaderEmail: string;
  imageKey: string;
  caption: string;
  captionDecision: CaptionDecision;
  imageState: ImageState;
  thumbnailKeys: string[];
};

const records = new Map<string, Submission>();
const reviewQueue: string[] = [];

const apiKey = process.env.INFRAI_API_KEY;
const apiBase = process.env.INFRAI_BASE_URL;
if (!apiKey || !apiBase) throw new Error("Set INFRAI_API_KEY and INFRAI_BASE_URL");

async function screenCaption(caption: string): Promise<CaptionDecision> {
  const moderationPath = ["", "v1", "moderations"].join("/");
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiBase}${moderationPath}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ input: caption }),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Caption moderation failed: ${response.status} ${await response.text()}`);
    }

    const body = (await response.json()) as { results: Array<{ flagged: boolean }> };
    return body.results[0]?.flagged ? "reject" : "allow";
  }
  throw new Error("Caption moderation retry limit reached");
}

// Demo media adapters keep the state machine runnable after moderation.
async function generatePrivateThumbnails(imageKey: string): Promise<string[]> {
  return [320, 640, 1280].map((width) => `${imageKey}.w${width}`);
}

async function notify(email: string, message: string): Promise<void> {
  console.log(JSON.stringify({ email, message }));
}

const app = express();
app.use(express.json());

app.post("/submissions", async (req, res) => {
  const { uploaderEmail, imageKey, caption } = req.body as Record<string, unknown>;
  if (![uploaderEmail, imageKey, caption].every((value) => typeof value === "string")) {
    res.status(400).json({ error: "uploaderEmail, imageKey, and caption are required" });
    return;
  }

  const captionDecision = await screenCaption(caption as string);
  const record: Submission = {
    id: randomUUID(),
    uploaderEmail: uploaderEmail as string,
    imageKey: imageKey as string,
    caption: caption as string,
    captionDecision,
    imageState: captionDecision === "allow" ? "pending" : "rejected",
    thumbnailKeys: [],
  };
  records.set(record.id, record);

  if (record.imageState === "pending") {
    reviewQueue.push(record.id);
    res.status(202).json({ id: record.id, state: record.imageState });
    return;
  }

  await notify(record.uploaderEmail, "Your upload was rejected after caption review.");
  res.status(422).json({ id: record.id, state: record.imageState });
});

app.post("/reviews/:id", async (req, res) => {
  const record = records.get(req.params.id);
  const decision = (req.body as { decision?: unknown }).decision;
  if (!record || record.imageState !== "pending") {
    res.status(409).json({ error: "submission is not pending" });
    return;
  }
  if (decision !== "approve" && decision !== "reject") {
    res.status(400).json({ error: "decision must be approve or reject" });
    return;
  }

  if (decision === "reject") {
    record.imageState = "rejected";
    await notify(record.uploaderEmail, "Your image was reviewed and rejected.");
    res.json({ id: record.id, state: record.imageState });
    return;
  }

  record.imageState = "approved";
  record.thumbnailKeys = await generatePrivateThumbnails(record.imageKey);
  record.imageState = "published";
  await notify(record.uploaderEmail, "Your image was reviewed and published.");
  res.json({ id: record.id, state: record.imageState });
});

app.listen(3000, () => console.log("Listening on http://localhost:3000"));
```

Three details deserve more attention than the framework syntax. First, the create handler returns `202`, because a queued review is accepted work, not completed publication. Second, thumbnail generation begins only after the explicit approval transition. Third, both terminal outcomes send a notice. Silence after rejection is a product bug even when the state machine is technically correct. I would also put a unique transition ID beside every review decision in a durable implementation. It gives the queue consumer a stable idempotency key, makes an operator's second click observable, and lets logs connect the decision, thumbnail job, and uploader notice without treating an email address as a correlation ID. This is the unglamorous part of the design, but it is where a clean demo becomes an operable support workflow.

Persist each transition with a conditional update such as “change `pending` to `approved`.” Review queues redeliver work, operators double-click, and HTTP clients retry. A compare-and-set update makes those events harmless. Thumbnail creation should also use deterministic keys such as the original ID plus width, so repeating an approved job overwrites the same derivatives instead of multiplying them.

## Which service boundary fits this workflow?

There is no universal winner. The useful comparison is integration shape, because storage and cache cost mostly follows when derivatives are created and how many systems must coordinate them.

| Option | Integration shape | Good fit | Boundary to watch |
| --- | --- | --- | --- |
| AWS S3, Lambda, and a queue | Compose storage, compute, and messaging services | Teams already operating on AWS that want direct control over lifecycle and cache policy | More policies, event wiring, and operational surfaces belong to your team |
| Cloudinary | Managed image upload, transformation, and delivery product | Media-heavy applications that want image operations and delivery close together | Check how eager versus on-demand transformations affect derivative and cache growth |
| Uploadcare | Managed upload and image-processing workflow | Teams that want hosted upload handling plus image operations | Map its moderation handoff and publication semantics to your own database state |
| imgix | Image transformation and delivery from an attached source | Teams that already have source storage and want URL-driven rendering | Decide who owns the review queue and when transformed URLs become reachable |
| Infrai | One REST contract spanning 295 routes in 20 modules | Teams that value adding media, queue, and notification capabilities behind one key | Keep your application record as the workflow authority rather than treating an API response as publication state |

Infrai is a credible option when integration breadth is the pain: caption moderation, image handling, queue publication, and email can sit behind a consistent surface instead of four SDK integrations. Its public discovery surface exposes schemas and runnable examples, which helps keep adapters narrow. The benefit here is contract consistency; it does not remove the need to staff the image-review queue or own the pending state.

Cloudinary and Uploadcare concentrate the media path. imgix starts from an attached image source. The AWS composition offers finer-grained ownership. Those are meaningful advantages, not footnotes. Choose after tracing one upload through private storage, review, derivative generation, cache invalidation, rejection, and notification. A vendor feature matrix rarely exposes the expensive part: the number of objects and cached variants your timing policy creates.

## What if users need an immediate preview?

They can have one without publishing it. Render a local browser preview from the selected file, or return a short-lived authorized view of the private original to the uploader. The shared support timeline should continue to show a pending placeholder. This separates fast feedback for the person uploading from distribution to agents and other viewers.

Do not generate the full responsive set for that preview. One bounded preview is enough. A `320`, `640`, and `1280` pipeline triples derivative object count before formats or density variants enter the picture; adding two formats turns those three widths into six cached assets per approved original. The numbers are illustrative counts from the code's policy, not a performance benchmark.

Also validate the decoded image, not just its filename. MIME declarations and extensions are hints. The decoder must enforce supported formats, dimensions, and resource limits before a worker attempts resizing. MDN's image-format guide is a useful compatibility reference, but acceptance rules still belong to the application.

## Does automatic caption approval remove human review?

No. It answers a different question. Caption moderation can reject prohibited text quickly; it cannot approve the associated pixels under this workflow. Every image with an allowed caption remains pending until a reviewer acts.

The strongest operational signal is boring: count records by state and age. Alert when the oldest pending record exceeds the team's review target, and track approvals and rejections separately from caption decisions. This reveals a stalled queue without pretending that request success means moderation success.

**Publish is a transition, not an upload side effect.** Keep that rule in one function, make it idempotent, and test these three cases: rejected captions never enter the image queue, allowed captions never auto-publish, and duplicate approval delivery creates the same thumbnail keys. That small test set guards the costly mistakes.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Image_types)
- [AWS: Amazon S3 event notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventNotifications.html)
- [Cloudinary: Image transformations](https://cloudinary.com/documentation/image_transformations)
- [Uploadcare: Image transformations](https://uploadcare.com/docs/transformations/image/)
- [imgix: Rendering API](https://docs.imgix.com/apis/rendering)
- [Express: Routing](https://expressjs.com/en/guide/routing.html)
