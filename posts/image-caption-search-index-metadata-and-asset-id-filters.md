# Image Caption Search Index — Metadata and Asset ID Filters

Put searchable meaning in one document per original image, but keep generated crop binaries and verbose analysis out of that document. **Short answer:** index the caption, normalized merchandising metadata, asset ID, crop status, and compact derivative descriptors; filter on the exact asset ID before scoring text; return URLs for only the aspect ratios the current surface needs. This preserves useful caption search without turning every result into a bandwidth-heavy manifest.

| Choice | Search quality | Response bandwidth | Operational cost |
| --- | --- | --- | --- |
| Index every crop as a document | Duplicate matches can distort ranking | High unless fields are aggressively projected | More index writes and cleanup |
| Index one original plus crop descriptors | One score per catalog image | Low and predictable | Requires a stable derivative contract |
| Store only an original and crop on request | Search stays simple | Small search response, expensive image path | Repeats compute and makes latency variable |

The practical default is the middle row. A search hit represents the original asset; its derivative map says which prepared shapes exist. That choice protects the revenue-per-hour equation: search code stays boring, storefront responses stay small, and crop generation remains an asynchronous job that can improve without rewriting the query layer.

Start there.

## How should Node.js index image captions and metadata for search?

A single product photo may feed square tiles, 4:5 cards, 16:9 banners, and narrow mobile promos. Those files are delivery variants, not four distinct pieces of catalog meaning. Indexing each one repeats the same caption and product metadata. A query for "red linen shirt" can then return several records for the same source image, forcing grouping logic into every consumer.

It also couples ranking to presentation churn. Add a new storefront slot and the apparent corpus grows even though no new product became searchable. Remove a crop and stale records can survive unless deletion is perfectly coordinated.

This is the trap.

One source document gives asset identity a single home. Keep `assetId` as an exact-match field, and treat the caption as analyzed text. Category, locale, moderation state, and product identifiers should also be explicit fields rather than tokens stuffed into the caption. An asset ID filter answers identity; full-text ranking answers relevance. Mixing those jobs is a common source of surprising matches.

The indexed crop data can stay compact:

```ts
type Ratio = "1:1" | "4:5" | "16:9";

type CropDescriptor = {
  ratio: Ratio;
  width: number;
  height: number;
  objectKey: string;
  focusX: number; // Normalized 0..1 coordinates on the original.
  focusY: number;
};

type ImageDocument = {
  assetId: string;
  caption: string;
  category: string;
  locale: string;
  cropRevision: number;
  crops: CropDescriptor[];
};
```

Do not index the image bytes, OCR dumps, model traces, or large region arrays alongside this record. Store those in object storage or an analysis store and retain only the fields needed to filter, rank, render, or invalidate. Less payload means less network time and less accidental coupling.

## Spend bandwidth where shoppers can see quality

Smart cropping is a quality decision made under a delivery constraint. A center crop is cheap to describe but can cut off a face, a shoe, or packaging text. A content-aware crop can preserve the selling subject, yet the storefront still needs fixed dimensions so layout does not jump.

The search API should therefore return crop dimensions and a resolvable path, not all pixels and not every available rendition. The client already knows its slot. A grid can request `1:1`; a campaign module can request `16:9`. Returning three or more URLs per hit "just in case" spends bandwidth on options the page will discard.

Quality needs an offline gate. Build a review set that covers edge-positioned products, multiple objects, people, transparent backgrounds, and text-heavy packaging. For each supported ratio, verify that the intended subject remains inside the crop and that required text is not clipped. Do this before rollout, then sample new failures from production feedback. There is no honest universal threshold in the available evidence, so the acceptance rule must come from the catalog's own visual requirements.

Consider one concrete upload path. A merchant supplies a portrait product photo, the catalog record gets one asset ID, and workers prepare `1:1`, `4:5`, and `16:9` outputs against the same original. The caption and category are written once. If the wide crop loses the product label, that derivative stays unavailable while the square and portrait crops can still publish; the index never pretends the missing ratio exists. A later crop revision can add the corrected wide file under a new deterministic key. Search ranking does not change merely because pixels were reframed, while the delivery layer can reject a missing ratio instead of silently stretching another one. This split is deliberate: the index answers what the picture means, and the derivative record answers what the storefront can render.

Format selection belongs on the delivery side of the boundary. Browser support and image capabilities differ by format; the source guide in the references documents those differences. Search should expose intrinsic dimensions and stable derivative identity. Content negotiation or the rendering layer can select a suitable encoded file.

## A focused query and projection contract

The endpoint needs two modes: an asset lookup for deterministic retrieval and caption search for discovery. Keep the result projection identical. That makes the response easy to cache and prevents internal analysis fields from leaking into clients.

```ts
import express, { Request, Response } from "express";

type SearchHit = {
  assetId: string;
  caption: string;
  category: string;
  cropRevision: number;
  crops: CropDescriptor[];
};

interface ImageIndex {
  findByAssetId(assetId: string): Promise<SearchHit | null>;
  searchCaption(query: string, limit: number): Promise<SearchHit[]>;
}

export function createApp(index: ImageIndex) {
  const app = express();

  app.get("/images", async (req: Request, res: Response) => {
    const assetId = typeof req.query.assetId === "string"
      ? req.query.assetId.trim()
      : "";
    const query = typeof req.query.q === "string" ? req.query.q.trim() : "";

    if (assetId) {
      const hit = await index.findByAssetId(assetId);
      return res.status(200).json({ items: hit ? [hit] : [] });
    }

    if (!query) {
      return res.status(400).json({ error: "q or assetId is required" });
    }

    const items = await index.searchCaption(query, 24);
    return res.status(200).json({ items });
  });

  return app;
}
```

The adapter behind `ImageIndex` owns field mappings and query syntax. The HTTP layer owns input shape and response projection. This separation is small, but useful: a one-person SaaS can test the contract with an in-memory adapter and outsource undifferentiated indexing without letting a backend-specific query language spread through the app.

Keep it dull.

Notice the precedence rule. If `assetId` is present, the handler performs an exact lookup and ignores caption text. That avoids ambiguous requests whose filter and query disagree. Empty results still return `200`; malformed requests return `400`. Unexpected storage failures should reach centralized error handling, be logged with a request correlation value, and produce a generic server response without exposing object keys or index internals.

## Ship the index and cropper as separate revisions

Uploads should first establish an immutable asset ID and original object. Captioning, metadata normalization, and crop generation can then run asynchronously. Publish the searchable document only when its required fields are ready; optional ratios can appear later by incrementing `cropRevision`.

Make updates idempotent. Reprocessing the same asset and crop revision should replace the same derivative keys, not append another set. Deletion has to remove the source document and schedule its objects for cleanup. A periodic reconciliation job can compare index references with stored derivatives and report both missing files and unreferenced files.

Observe the boundaries that affect shoppers: caption-job age, crop-job failures by ratio, index-to-object mismatches, empty-result rate, response payload size, and derivative cache misses. Do not log full user queries by default if they may contain personal data. Aggregate what answers an operational question.

Ship weekly, but stage the risky part. A new captioner can write into a shadow field for evaluation. A new cropper can create a higher revision while the previous derivatives remain addressable. Promote the revision after visual review, and roll back by switching the active revision rather than regenerating the catalog during an incident.

## When is the runner-up design better?

Indexing each derivative separately is defensible when variants carry genuinely different meaning. A publisher may have independently edited crops with different captions, rights, or moderation states. In that case they are editorial assets, not mechanical renditions, and separate documents make those distinctions searchable.

On-request cropping can also win for a small, private catalog with unpredictable ratios and low request volume. It avoids precomputing files nobody requests. The trade is variable latency and repeated work, so cache the result under a key derived from the original identity, crop parameters, and algorithm revision.

The one-document approach has a real limitation: it is not a good fit when each crop needs independent rights, copy, approval, or ranking. Its other trade-off is update coordination. Clients must understand `cropRevision`, and the index can briefly point at a derivative that storage cannot serve if publication is not ordered carefully. Teams unable to enforce that contract should choose separate variant records or on-request generation, accepting duplicate metadata or variable latency in exchange for simpler ownership.

For a storefront with known slots, neither exception is the usual case. **Keep semantic identity singular and delivery variants plural.** The result is easier to rank, cheaper to move over the wire, and safer to evolve. More importantly, the expensive quality work stays concentrated on the pixels shoppers actually receive.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://expressjs.com/en/4x/api.html
- https://www.rfc-editor.org/rfc/rfc9110.html
- https://www.w3.org/TR/WCAG22/#images-of-text
