# Digital Assets Library — Design & Build

Status: **Design complete, code scaffolded** (2026-08-30). Backend public-bucket wiring is a stub
pending Binoy's review — see "Open decisions" at the end.

## 1. Research findings

### 1.1 The doc-app monorepo (where this lives)

`document-manager` and `knowledge-app` are NOT App Studio "Apps" (App Studio `AIExt_Apps` records).
They are two of four standalone React apps in a **separate, sibling monorepo**:

```
C:\BizFirstGO_FI_AI\BizFirstAiStudio\src\doc-app\
  apps\
    document-manager\      <- general document library (contracts, invoices, KYC docs, ...)
    knowledge-app\          <- RAG-focused variant, same shared packages
    doc-share-viewer\       <- public/anonymous share-link viewer
    doc-template-manager\
  packages\
    @doc-app\
      api-client\           <- typed HTTP clients, wire types (shared)
      react\                <- reusable UI components/hooks (shared)
    knowledge-api-client\
  pnpm-workspace.yaml
```

Confirmed: `document-manager/src` (`App.tsx`, `layout/AppShell.tsx`, `pages/*.tsx`, `main.tsx`) is
already a **thin shell** — routing + page composition only. Every real component (upload dropzone,
document list, viewer, share panel, collection picker) lives in `@doc-app/react`, and every network
call goes through `@doc-app/api-client`. `knowledge-app` is the second, independent proof of this
pattern: its own `AppShell.tsx`/pages are ~4 files, and its RAG-specific UI lives in a **subtree**
inside `@doc-app/react` (`src/knowledge/*`) rather than a new package — this is the precedent this
design follows for where asset-specific UI goes (see §2.2).

### 1.2 Existing reusable component inventory (`@doc-app/react/src/`)

| Component | File | Reusable as-is for digital assets? |
|---|---|---|
| `DocumentUploadPanel` | `components/DocumentUploadPanel.tsx` | Yes, wrapped (adds visibility toggle) |
| `DocumentList` | `components/DocumentList.tsx` | Table view only, no thumbnails — kept as an alternate view, **not** primary |
| `DocumentViewer` | `components/DocumentViewer.tsx` | **No** — its own doc comment says it "renders a placeholder card rather than a real PDF/Office renderer" deliberately. A media library needs real image/video/PDF preview — new. |
| `DownloadButton` | `components/DownloadButton.tsx` | Yes, as-is |
| `ShareSecurityPanel` | `components/ShareSecurityPanel.tsx` | Yes, as-is (private assets: temporary/revocable share links) |
| `CollectionPicker` / `DocumentCollectionsPanel` | `components/*.tsx` | Yes, as-is (asset "folders" = `Doc_DocumentCollections`, already many-to-many) |
| `useDocumentsByTypeCode` | `hooks/useDocumentsByTypeCode.ts` | Yes, as-is — resolves a `documentTypeCode` string to `documentTypeID` and filters, exactly the mechanism needed for `DIGITAL_ASSET` |
| `useAuthedHttpClient` | `hooks/useAuthedHttpClient.ts` | Yes, as-is |
| `DOCUMENT_MENU_ITEMS` pattern | `documentMenuConfig.ts` | Pattern reused, not the constant itself (app-owned config) |

**Net finding: ~70% of what a digital-assets-library needs already exists and is generic over
document type.** The only real net-new UI is thumbnail/grid presentation, real media preview, and a
public/private visibility control — everything else (upload plumbing, CRUD, folders, temporary
sharing) is direct reuse.

### 1.3 Document type extensibility — the exact, proven mechanism

`Doc_DocumentTypes` (`BizFirstFiDB\...\dbo\Tables\Doc_DocumentTypes.sql`) is a per-tenant lookup
table (`DocumentTypeCode` unique per `TenantID`). New types are added by **inserting a seed row**,
never a schema change. Proof, from `BizFirstFiDB\...\dbo\Data\Master\Doc_DocumentTypes.data.sql`:
codes already seeded include `RAG_DOCUMENT` (knowledge-app's type) and `PUBLISHED_VIDEO` (the
screen-recorder's type, added 2026-08-21). `PUBLISHED_VIDEO` is the load-bearing precedent for this
design: it's a type with its own extra lifecycle columns added directly to `Doc_Documents`
(`PublishStatus`, `YouTubeVideoID`, `DurationSeconds` — see `Doc_Documents.sql` lines 59-64) rather
than a parallel table, with its own processor (`PublishedVideoDocumentProcessor`,
`IExtendedDocumentProcessor`). **Digital assets follow the identical pattern**: one new
`Doc_DocumentTypes` seed row (`DIGITAL_ASSET`) + a small number of new nullable/defaulted columns on
`Doc_Documents` for the one concern that's genuinely new (public visibility) — not a new table, not a
schema fork.

`documentMenuConfig.ts`'s own doc comment confirms the `typeCode` (never a numeric ID) is the
correct, already-established way to reference a type from frontend code — `useDocumentsByTypeCode`
resolves it once per session via `DocumentTaxonomyClient.listTypes()`.

### 1.4 Backend: two storage abstractions exist, and they are different systems

**`IDocumentStorageProvider`** (`BizFirstPayrollV3\src\mvc-server\Go\Documents\BizFirstFi.Go.Documents.Domain\Interfaces\Storage\IDocumentStorageProvider.cs`)
— what `Doc_Documents` uses today. A narrow stream abstraction: `SaveAsync`/`OpenReadAsync`/
`ExistsAsync`/`DeleteAsync`. `OpenReadAsync` returns a `Stream`, which `DocumentController`'s
download action proxies through the authenticated API
(`@doc-app/api-client/src/documentClient.ts` `download()` confirms this: `POST
/api/v1/documents/{id}/download`, bearer token required, response streamed back). **No public/anonymous
read path exists in this abstraction.**

> **Flagged as existing tech debt, deliberately not inherited (correction, see §2.5):** this
> `{documentID}`-in-the-URL, stream-through-the-app-server pattern is fine for the low-volume,
> always-authenticated cases it was built for (a KYC officer opening a contract), but it is the
> wrong shape for public asset serving: DocumentID is a sequential `IDENTITY` primary key, so a
> DocumentID-keyed public URL is (a) enumerable — incrementing the ID walks every other tenant's
> assets — and (b) a guaranteed DB round-trip plus an app-server byte-relay on every single page
> view, which does not scale to public-site traffic (10 images/page × high view counts = a DB hit
> and a server-proxied stream per image, forever). This design's asset-serving path (§2.5) does not
> reuse this endpoint for public reads. It remains available, unchanged, for admin/management
> operations only (see §2.5).

**`Platform.StorageServers`** (`BizFirstPayrollV3\src\mvc-server\Platform\StorageServers\BizFirst.Ai.Platform.StorageServers.Service\`)
— a separate, generic S3-compatible bucket/object module, NOT currently wired to `Doc_Documents` at
all. `PresignService.cs` already implements presigned upload/download URLs, multipart upload, and
**revocable, expiring share links** (`CreateShareLinkAsync`/`ResolveShareLinkAsync`, backed by
`IShareLinkStore`, max 7-day S3 SigV4 expiry). This is bucket-oriented: an S3-compatible bucket can be
configured with a public-read policy, giving a **permanent, non-expiring, non-presigned URL** — the
one capability `IDocumentStorageProvider` has no concept of at all.

This is the one place this design does NOT have a fully-resolved answer — see "Open decisions" §5.

### 1.5 Existing sharing model is the wrong tool for public web assets

`ShareScope: 'Public' | 'TenantOnly' | 'RestrictedByEmail' | 'PasswordProtected'`
(`@doc-app/api-client/src/types.ts`) plus `Doc_DocumentShares`/`DocumentShareAccessLog` is a
**revocable, expiring, access-logged link-sharing feature** — right for "send this PDF to an
external partner," wrong for "embed this logo as `<img src>` on 10,000 public page views/day."
A digital asset embedded on a public website needs a URL that (a) never silently expires,
(b) doesn't require presign-refresh logic in the consuming page, (c) isn't rate-limited by an
access-log write per view. This is why public/private for digital assets is a **new, simpler
boolean concept on the document itself**, not a reuse of `Doc_DocumentShares` — confirmed by reading
`ShareSecurityPanel.tsx` end-to-end: it's built entirely around token creation/revocation/expiry, none
of which is the right shape for "this logo is just public."

### 1.6 Wix Media Manager, for UX grounding only

Wix's Media Manager: single flat library + folders (not nested infinitely), drag-drop multi-upload,
auto-generated thumbnails per type (image/video/audio/document), a visible Public/Private-style split
("Site Media" is served to the live site; anything not added to a page stays private to the editor),
and "used on N pages" backlinks. This grounds the UX decisions in §3 (grid-first browsing, inline
visibility toggle at upload time, folders via existing collections) — it does not override anything
found in §1.1–§1.5, which are all real, cited code.

## 2. Design decisions

### 2.1 New app: `digital-assets-library`

`doc-app/apps/digital-assets-library/` — thin shell, byte-for-byte the same shape as
`document-manager`: `App.tsx` (SsoAuthGate + ThemeProvider + BrowserRouter, unchanged pattern),
`layout/AppShell.tsx` (nav + routes), `pages/*.tsx` (each a route wrapper: fetch via a hook, render
a `@doc-app/react` component, own the network-call glue — exactly `UploadPage.tsx`'s existing split
of "panel owns UI state, page owns the API call"). No business logic in the app itself — confirmed
achievable because §1.2 shows almost everything is already componentized.

### 2.2 New reusable UI: a `src/assets/` subtree inside `@doc-app/react`, not a new package

Precedent: `knowledge-app`'s RAG-specific UI lives at `@doc-app/react/src/knowledge/*`, a subtree of
the *existing* shared package, not a new `@doc-app/knowledge-react` package. Digital assets follow
the same rule — one shared UI package, domain-specific subtrees. New files:

- `src/assets/components/AssetGrid.tsx` — thumbnail grid (the primary browse view; `DocumentList`'s
  table stays available as a secondary view toggle, reused as-is).
- `src/assets/components/AssetCard.tsx` — one grid cell: thumbnail (image `<img>`/video poster
  frame/PDF first-page icon/generic file-type icon), name, visibility badge, size.
- `src/assets/components/AssetPreview.tsx` — real preview modal: `<img>` for images, `<video
  controls>` for video, `<embed type="application/pdf">` for PDF (browser-native, no new PDF.js
  dependency — matches this monorepo's "no new heavy dependency without a documented reason"
  posture), fallback to `DocumentViewer`'s existing placeholder for anything else. This is the one
  component that couldn't reuse an existing piece (§1.2) — `DocumentViewer` explicitly opts out of
  real preview by design comment.
- `src/assets/components/AssetUploadPanel.tsx` — thin wrapper around `DocumentUploadPanel`: same
  dropzone/file-list, adds one control, "Visibility: Private / Public", to the metadata form. Not a
  fork — composes `DocumentUploadPanel` and passes the extra field up via its own
  `onUpload(files, metadata)` callback shape (metadata gets one more key: `isPublicAsset`).
- `src/assets/components/AssetVisibilityBadge.tsx` — small "Public"/"Private" pill, used in
  `AssetCard` and the asset detail page.
- `src/assets/hooks/useAssetsByTypeCode.ts` — **not** built as a new hook; `useDocumentsByTypeCode`
  is called directly with `typeCode: 'DIGITAL_ASSET'` from the app's pages, same as
  `document-manager`'s `DocumentsPage.tsx` does today for its own filtered views. No wrapper needed —
  documented here so a future maintainer doesn't add a redundant one.

Reused with zero changes: `DocumentUploadPanel` (composed inside `AssetUploadPanel`),
`DownloadButton`, `ShareSecurityPanel` (private assets that need a temporary external link),
`CollectionPicker`, `DocumentCollectionsPanel`, `useDocumentsByTypeCode`, `useAuthedHttpClient`.

### 2.3 `@doc-app/api-client` — additive changes only

`DocumentRecord` (`types.ts`) gains two optional fields so every existing consumer (document-manager,
knowledge-app) keeps compiling unchanged:

```ts
export interface DocumentRecord {
  // ...existing fields unchanged...
  /** New 2026-08-30 (digital-assets-library). True once this document has been published to a
   *  public-read bucket and is servable without authentication via `fileUrl`. Undefined/false for
   *  every pre-existing document type — this is opt-in per row, not a behavior change to anything
   *  that already exists. */
  isPublicAsset?: boolean;
  /** Opaque publish key — NEVER the DocumentID, see design §2.5. Present only once an asset has
   *  actually been published public; used solely for admin "unpublish" / "view public URL" actions,
   *  never as part of constructing a serving URL client-side (the resolved URL is `fileUrl`). */
  publicAssetKey?: string | null;
  publicAssetPublishedOn?: string | null;
}
```

`DocumentClient.upload()`'s `params` gains one optional field, `isPublicAsset?: boolean`, appended to
the `FormData` the same way `classificationName` already is — additive, no signature break for
existing callers.

### 2.4 Data model

**No new table.** Per the `PUBLISHED_VIDEO` precedent (§1.3), the two-piece change is:

**(a) Seed row** — append to `BizFirstFiDB\...\dbo\Data\Master\Doc_DocumentTypes.data.sql`:
```sql
(1, 'DIGITAL_ASSET', 'Digital Asset', 'Image, video, PDF, or other media asset for reuse across apps and public websites', 1, 15, GETUTCDATE(), 0)
```
(`DisplayOrder 15` — next after `PUBLISHED_VIDEO`'s `14`.)

**(b) Three new columns on `Doc_Documents`** (see reviewable script in §4 — revised from the
original two-column draft to add `PublicAssetKey`, the load-bearing fix from §2.5):
```sql
[IsPublicAsset]            BIT           CONSTRAINT [DF_Doc_Documents_IsPublicAsset] DEFAULT ((0)) NOT NULL,
[PublicAssetKey]           NVARCHAR(64)  NULL,   -- opaque, NOT the DocumentID — see §2.5
[PublicAssetPublishedOn]   DATETIME2(7)  NULL,
```

`PublicAssetKey` is an opaque, non-sequential, unguessable object-storage key
(`Guid.NewGuid():N}` — the exact format `PresignService.PresignUploadAsync`/
`PresignPartAsync` already use for every object key it mints, see §1.4/§2.5) generated **once, at
publish-to-public time**, independent of `DocumentID`. It is `NULL` for every private asset and for
every non-digital-asset document row.

Deliberate deviation from CLAUDE.md's general "use DATETIME, not DATETIME2" rule: this is an
**addition to an existing table**, and every other datetime column already on `Doc_Documents`
(`CreatedOn`, `LastModifiedOn`, `ExpiryDate`, `VerifiedDate`, etc.) is `DATETIME2(7)`. Mixing
`DATETIME` and `DATETIME2` within one table is worse than a single documented exception — consistency
with the table's own established convention wins here. Flagged explicitly for Binoy's sign-off, not
applied silently.

Everything else CLAUDE.md requires (Deleted/Archived/LastModifiedOn/By/CreatedOn/By/SourceAppID/
ClientAccountID/AppDomainID/DataDomainID/DataSegmentID/TenantID/ResID) is **already present** on
`Doc_Documents` — digital assets inherit it for free by being rows in the same table, no duplication.

`Doc_Documents.FileUrl` (already exists, nullable, currently used loosely) becomes the **permanent
public URL** field specifically when `IsPublicAsset = 1` — no new column needed for that; this is an
existing column taking on a precise, documented meaning for this one case.

### 2.5 Public vs. private — full end-to-end path (revised)

**Corrected design (this section supersedes an earlier draft that keyed public URLs off
`DocumentID` — flagged by Binoy before any code was built against it):** the DocumentID/AssetID is
an internal management identifier ONLY. It is used in authenticated, low-volume admin operations —
upload, delete, list-my-assets, toggle visibility, edit metadata — every one of which already goes
through `DocumentController` today. **It must never appear in, or be derivable from, a public- or
private-asset serving URL.** Serving (the hot path — what a public website or an authenticated
viewer actually fetches bytes from) is keyed entirely by the opaque `PublicAssetKey` / storage key,
never by `DocumentID`.

**Public (`IsPublicAsset = 1`):**
1. **Publish (rare, authenticated, DB-touching — happens once per asset, not per view).** Uploading
   or "make public" action calls a new backend step that writes the file into a **public-read
   bucket** via `Platform.StorageServers` (`PresignService`/`IStorageClientFactory`,
   §1.4), using an opaque key in the exact `Guid.NewGuid():N` shape `PresignService` already mints
   for every object it handles today (see `PresignUploadAsync`/`PresignPartAsync`, §1.4) — **not**
   derived from `DocumentID` in any way. The resulting key is stored in the new
   `Doc_Documents.PublicAssetKey` column (§2.4); the full resolved public URL
   (`{publicAssetBaseUrl}/{PublicAssetKey}`) is stored in the existing `Doc_Documents.FileUrl`
   column; `PublicAssetPublishedOn` is stamped. This is the **only** database write in the entire
   public-serving path.
2. **Serve (the hot path — every public page view, must scale).** The public website/App Studio
   widget embeds `FileUrl` directly (`<img src>`/`<video src>`) — a request to that URL resolves
   **straight against the public-read bucket / whatever sits in front of it**, with **zero app-server
   involvement and zero database lookup**. `DocumentID` never appears in this URL, so it cannot be
   enumerated to walk other tenants' assets, and no per-view DB round-trip exists to not scale.
   **Stated infrastructure prerequisite, not confirmed in this pass:** whether a CDN already sits in
   front of the `Platform.StorageServers` S3-compatible endpoint was not verified in this research
   pass — if none exists yet, the public-read bucket's own origin still serves this correctly (S3-
   compatible object storage is designed for exactly this), but a CDN in front of it should be added
   before this is exposed to real public traffic, for latency/cost/cache-hit reasons, not
   correctness. Call this out to Binoy before go-live rather than assuming it's already there.

**Private (default, `IsPublicAsset = 0`):**
- **Management** (upload, delete, metadata edit, visibility toggle, "who can see this") — unchanged,
  DocumentID-keyed, authenticated, via `DocumentController` — this is exactly the low-volume,
  always-authenticated case the existing pattern is actually right for.
- **Serving (viewing/downloading the actual bytes)** — corrected to **not** default to
  `IDocumentStorageProvider`'s stream-through-the-app-server relay for this app's asset-viewing path.
  Instead: an authenticated request first passes a real tenant/ACL check (does this user's tenant own
  this asset), and only on success mints a **short-lived presigned download URL**
  (`PresignService.PresignDownloadAsync`, already implemented, 60s–24h expiry, §1.4) pointing directly
  at the private bucket/object — the client then fetches bytes from that presigned URL directly, not
  through the app server. The authorization check happens once per URL-mint, not once per byte
  transferred; the app server is a gatekeeper, not a relay. The opaque key alone is **not** sufficient
  security for private content on its own (a leaked link, browser history, or a referrer header would
  otherwise expose it) — it is the ACL check that makes a private asset actually private, layered on
  top of the same non-enumerable key used for public assets. The existing
  `/api/v1/documents/{id}/download` stream-relay (§1.4) remains available, unchanged, as a fallback
  for non-asset document types this app doesn't touch — this app's own private-asset viewer does not
  call it.
- A private asset can still be temporarily shared externally via the existing `ShareSecurityPanel`
  (`Doc_DocumentShares`, revocable, expiring) — no change needed there; that feature already mints
  its own opaque `shareTokenPrefix`/token, independent of `DocumentID`, so it was never affected by
  this correction.

**Why not reuse `Doc_DocumentShares`' `ShareScope: 'Public'` for the public-asset case:** that path
is per-share-record, revocable, and (per `PresignService.CreateShareLinkAsync`) capped at a 7-day S3
SigV4 expiry — wrong shape for "this logo is permanently public until someone explicitly unpublishes
the asset." A permanent public-bucket URL has no expiry to renew and no share-record indirection to
resolve on every page view — and, same as the corrected design above, its existing `shareTokenPrefix`
token is already opaque/non-sequential, so this reasoning is consistent with, not an exception to,
the opaque-key rule.

### 2.6 Cross-workspace reuse — the exact recipe for App Studio to consume this later

`doc-app` and `app-studio` are separate pnpm workspaces under one umbrella
(`BizFirstAiStudio\pnpm-workspace.yaml`: `packages: ['packages/*', 'src/**']`), but `app-studio`'s
own `pnpm-workspace.yaml` (`BizFirstAiStudio\src\app-studio\pnpm-workspace.yaml`) already has three
proven precedents for reaching into a *different* sibling workspace this same way — the exact recipe
to follow when App Studio widgets are ready to render `@doc-app/react`'s `AssetGrid`/`AssetPreview`:

```yaml
# in app-studio/pnpm-workspace.yaml's `packages:` list, alongside the existing
# atlas-forms / bizfirst-common / expressions entries:
- "../../doc-app/packages/@doc-app/react"
- "../../doc-app/packages/@doc-app/api-client"
```

then, in the consuming app's own `vite.config.ts` `resolve.alias` (same pattern as every
`@app-studio/*` / `@atlas-forms/*` / `@bizfirst/*` alias already there):

```ts
'@doc-app/react': path.resolve(__dirname, '../../../doc-app/packages/@doc-app/react/src/index.ts'),
'@doc-app/api-client': path.resolve(__dirname, '../../../doc-app/packages/@doc-app/api-client/src/index.ts'),
```

This is documented, not applied — wiring app-studio itself is out of scope for this pass (no App
Studio widget consumes these components yet). When a future "Media" widget/picker is built, this is
the recipe, not a research task.

## 3. Component & package inventory (build checklist)

- [x] `doc-app/apps/digital-assets-library/` — new app shell (App.tsx, main.tsx, index.html,
      package.json, vite.config.ts, tsconfig.json, layout/AppShell.tsx)
- [x] `pages/LibraryPage.tsx` — grid browse (default `/`), `AssetGrid` + visibility filter
- [x] `pages/UploadPage.tsx` — `AssetUploadPanel` + upload glue (mirrors `document-manager`'s)
- [x] `pages/AssetDetailPage.tsx` — `AssetPreview` + `DownloadButton` + `ShareSecurityPanel` (private
      only) + `CollectionPicker` + visibility toggle action
- [x] `pages/CollectionsPage.tsx` / `CollectionDetailPage.tsx` — folders, reusing
      `document-manager`'s existing collection pages near-verbatim (same `DocumentCollectionClient`)
- [x] `pages/SettingsPage.tsx` — stub, mirrors `document-manager`'s
- [x] `@doc-app/react/src/assets/*` — `AssetGrid`, `AssetCard`, `AssetPreview`,
      `AssetUploadPanel`, `AssetVisibilityBadge`, exported from `@doc-app/react`'s `index.ts`
- [x] `@doc-app/api-client` — `DocumentRecord.isPublicAsset`/`publicAssetKey`/`publicAssetPublishedOn`,
      `DocumentClient.upload()` `isPublicAsset` param
- [x] `Doc_DocumentTypes.data.sql` seed row (reviewable, not executed)
- [x] `Doc_Documents` new-columns migration script — `IsPublicAsset`/`PublicAssetKey`/
      `PublicAssetPublishedOn`, opaque-key unique index (reviewable, not executed; §2.5/§4)
- [x] Port registry row added to `02-list-of-apps@resource.md` (port **6111** — next free gap
      between `document-manager` (6110) and `knowledge-app` (6112); verify free before first run)
- [ ] Backend: publish-to-public-bucket action (mints opaque `PublicAssetKey`, writes
      `FileUrl`/`PublicAssetPublishedOn`) — **not built this pass, stub only**, see §5 item 2
- [ ] Backend: authenticated "mint short-lived private-asset URL" endpoint
      (`PresignService.PresignDownloadAsync` + ACL check) — **not built this pass, stub only**, see
      §5 item 2
- [x] App Studio widget consumption (§2.6's recipe) — **wired 2026-08-30** (wix-style-design
      Task 7): `app-studio`'s `pnpm-workspace.yaml`/`vite.config.ts`/`tsconfig.json` all reach
      `@doc-app/react`/`@doc-app/api-client` now; the Content Widget's Tiptap toolbar and the
      Style Builder's `backgroundImage` field both consume `AssetGrid` via a new shared
      `AssetPickerPopover`. Full detail in `app-studio-designer-components-react`'s own
      `DevelopmentHistoryLog.md`.

## 4. Data model — full column list for the `Doc_Documents` migration

Two real, reviewable, idempotent scripts were written to this repo's actual migration convention
(`dbo\Migrations\*.sql`, `IF NOT EXISTS` guarded, matching e.g. `AIAgent_AgentHooks_AddConfigJson.sql`)
— **neither has been executed against any database**, per instruction:

- `BizFirstFiDB\BizFirstFiV3DB\BizFirstFiV3DB\dbo\Migrations\Doc_Documents_AddDigitalAssetColumns.sql`
  — adds `IsPublicAsset` / `PublicAssetKey` (opaque `Guid.NewGuid("N")`-shaped, **never**
  `DocumentID` — see §2.5) / `PublicAssetPublishedOn`, plus the two matching indexes (a partial index
  on `IsPublicAsset = 1`, and a partial **unique** index on `PublicAssetKey` where set).
- `BizFirstFiDB\BizFirstFiV3DB\BizFirstFiV3DB\dbo\Migrations\Doc_DocumentTypes_SeedDigitalAsset.sql`
  — the `DIGITAL_ASSET` type-code row, for already-deployed databases (the original
  `Doc_DocumentTypes.data.sql` seed script has no per-row existence guard, so it isn't itself safely
  re-runnable against an already-seeded table — see that migration's own header comment).

The `DIGITAL_ASSET` row was also appended directly to
`BizFirstFiDB\BizFirstFiV3DB\BizFirstFiV3DB\dbo\Data\Master\Doc_DocumentTypes.data.sql` (§2.4(a)),
so a *fresh* database build picks it up the same way every other seeded type code does — the
`Migrations` script above is specifically for a database that was already running before this row
existed.

## 5. Open decisions / risks (not resolved this pass — flagged, not guessed)

1. **Which concrete service actually implements `IDocumentStorageProvider` today** (local disk vs.
   Azure Blob vs. something else) was not traced to its DI registration in this pass — needed before
   the public-bucket publish pipeline can be wired for real, to know whether "public" should be a
   different bucket in the *same* provider or a hop over to `Platform.StorageServers` entirely (this
   design recommends the latter, per §2.5, but the DI wiring to make that real wasn't built).
2. **Backend plumbing for the corrected public/private model (§2.5) is not implemented** —
   `DocumentController`'s upload/update actions need: (a) a new optional bound field for the
   publish/visibility toggle, (b) the actual publish-to-public-bucket call that mints the opaque
   `PublicAssetKey` (in the same `Guid.NewGuid():N` shape `PresignService` already uses) and writes
   `FileUrl`/`PublicAssetPublishedOn`, and (c) a new authenticated "get a short-lived private-asset
   URL" endpoint that does the ACL check then calls `PresignService.PresignDownloadAsync` rather than
   the old stream-relay. None of this is built yet — frontend scaffolding (§3) is wired to call
   these once they exist, but until they land, no asset actually becomes public or gets a presigned
   private URL; this is a real, visible backend gap, not a cosmetic one.
3. **CDN in front of the public bucket is a stated, unverified infrastructure prerequisite** (§2.5)
   — whether one already exists in this stack was not confirmed in this research pass. The
   public-read bucket origin serves correctly without one, but production public-traffic serving
   should have a CDN in front of it before go-live, for latency/cost/cache-hit reasons.
4. **Tenant-scoping of the public bucket vs. the opaque-key uniqueness guarantee** — with a single
   shared public bucket across tenants, the `PublicAssetKey` uniqueness (§4) is the only thing
   preventing collisions; a per-tenant bucket would need per-tenant key uniqueness instead
   (`PresignService` already takes a `serverId`/`bucketName` per call, so either shape is
   implementable — which one was not decided in this pass, ties to open item 1).
5. **Real PDF preview fidelity** — `AssetPreview`'s PDF path uses the browser's native
   `<embed type="application/pdf">`, which works in Chrome/Edge but degrades to a download prompt in
   browsers without a built-in PDF viewer. Flagged, not fixed — a PDF.js fallback would be the next
   step if this matters for the target audience.
6. **Port 6111** was chosen as the next open gap in the observed port list but was not exhaustively
   verified against every `vite.config.ts` in the workspace (a full-tree grep timed out during this
   research pass) — verify free before first `pnpm dev`.

## Build Progress — 2026-08-30, backend pass (closing §5 items 1 & 2)

**Storage provider resolved (§5 item 1):** `IDocumentStorageProvider` → `LocalDocumentStorageProvider`
(local disk), `AddSingleton` in `BizFirstFi.Go.Documents.Infrastructure/DependencyInjection.cs`.
**`Platform.StorageServers`/`PresignService` has NO public-read/ACL capability at all** — grepped the
whole module for `PublicRead`/`CannedACL`/`Acl`: zero hits. Every method there (`PresignUploadAsync`,
`PresignDownloadAsync`, `CreateShareLinkAsync`) is presigned/expiring by construction — there is no way
to mint a genuinely permanent public URL against it today. This is a bigger gap than §5 item 1
originally scoped ("which service implements the interface") — it's "the service the design assumed
would back public serving doesn't have the capability the design needs."

**Scope decision made, not silently substituted:** rather than block on that missing capability, built
a working INTERIM implementation using what exists — `IDocumentStorageProvider` (the same local-disk
storage every other document type already uses) for both public and private bytes, with a NEW
anonymous, opaque-key-keyed controller (`BasePublicAssetController`) as the public-read path instead of
a real bucket. This preserves every real requirement from §2.5 (opaque key, never DocumentID, no
password/session needed to view a public asset) except "zero DB lookup per view" — this interim path
costs one DB lookup + local-disk read per public view, not a CDN-cached zero-hop. Flagged, not hidden:
upgrading to a real public bucket (once `Platform.StorageServers` gains ACL/public-read support, or a
CDN/reverse-proxy is put in front of local storage) is a swap of `BasePublicAssetController.GetByKey`'s
internals only — the DB schema (`PublicAssetKey`/`FileUrl`) and the frontend contract do not change.

**Private "presigned" view — also an interim substitute, also flagged:** `PresignService.PresignDownloadAsync`
cannot apply to locally-stored files (it only signs bucket/key pairs on the S3-compatible system). Built
`PrivateAssetViewTokenStore` (new file, `Api.Base/Services/`) — an in-memory, single-instance,
`ConcurrentDictionary`-backed short-lived (15-min) opaque token store reproducing the same security
shape (authorize once at mint via a real ACL check in `GetPresignedViewUrl`, opaque token, never
DocumentID, short-lived). **Will not survive an app restart or scale across multiple instances** — noted
in the class's own doc comment as the thing a production multi-instance deployment would need to swap
for a shared store (Redis/DB).

**Shipped (all in `BizFirstPayrollV3`, `BizFirstFi.Go.Documents.*` projects):**
- `Document.cs`: `IsPublicAsset`/`PublicAssetKey`/`PublicAssetPublishedOn` properties.
- `IDocumentRepository`/`DocumentRepository`: `PublishAssetAsync` (mints a fresh opaque key every
  call, sets `FileUrl`/`PublicAssetPublishedOn`), `UnpublishAssetAsync` (clears all three so a revoked
  key can never resolve again), `GetPublishedAssetByKeyAsync` (the anonymous read path — deliberately
  NOT TenantID-scoped, see its own doc comment).
- `IDocumentService`/`DocumentService`: matching pass-through methods.
- `BaseDocumentController`: `POST {id}/publish-public`, `POST {id}/unpublish-public`,
  `POST {id}/presigned-view-url` — all `[AuthorizeRegularUserAttribute]`. `Upload` action now honors
  `DocumentUploadRequest.IsPublicAsset` (new field) — uploading as Public publishes in the same request.
- `BasePublicAssetController`/`PublicAssetController` (NEW, `[AllowAnonymous]`, `[Route("api/v1/public/assets")]`):
  `GET {publicAssetKey}` and `GET view-token/{token}` — modeled directly on the existing, working
  `BasePublicShareController` pattern.
- `PrivateAssetViewTokenStore` (new file).
- Response DTOs `PublishPublicAssetResponse`/`PresignedAssetViewResponse` matching
  `@doc-app/api-client`'s `documentClient.ts` wire contract exactly (verified against its source, not
  guessed).
- `DevelopmentHistoryLog.md` — not yet updated (see "Not done" below — held pending the live bug fix).

**Live-verified, working (real HTTP calls against the rebuilt, restarted Consolidated WebApi, real
DB checks before/after each):**
- Upload with `IsPublicAsset=true` → DB row correctly gets `IsPublicAsset=1`, a real opaque
  `PublicAssetKey` (`3629e3e1a6474fca9d43afe1686d01db` — 32 lowercase hex chars, NOT DocumentID 9),
  and a resolved `FileUrl`. Confirmed via direct `SELECT` against `data-ocean-platform-prod`.
- `POST /api/v1/documents/9/publish-public` (authenticated) → 200, mints a FRESH key on every call
  (verified: two calls produced two different keys), correct JSON envelope matching
  `PublishPublicAssetResult`'s frontend shape exactly.
- Full solution rebuild (`dotnet build` on `BizFirst.Ai.Consolidated.WebApi.csproj`, `-m:2 -nodeReuse:false`)
  succeeded with 0 errors — confirms every change above compiles clean across the whole dependency graph,
  not just the Documents module in isolation.

**CONFIRMED BUG, NOT FIXED — flagging clearly rather than guessing further:** the two new ANONYMOUS
`[HttpGet]` actions (`BasePublicAssetController.GetByKey`/`ViewByToken`) both return a bare
`404 Not Found` (`Content-Length: 0`, no JSON body) for a key that is verified correct in the database
at the moment of the request. Diagnosis so far:
- Ruled out: wrong key (re-tested with a freshly-minted key, same result), stale build (the SAME
  rebuild that makes `publish-public` work correctly also serves this code), `[AllowAnonymous]` not
  applying (confirmed working generally — the pre-existing `BasePublicShareController.Resolve`, also
  `[AllowAnonymous]`, returns a full JSON-enveloped 400 with a real stack trace for a bad request,
  proving anonymous requests DO reach controller code in this environment), a manual controller-
  registration allowlist (none exists — grepped the whole module, controllers are ambient-discovered).
- **Leading hypothesis, not confirmed:** every existing, WORKING anonymous endpoint in this codebase
  (`BasePublicShareController.Resolve`/`Download`) is `[HttpPost]`. My two new endpoints are the only
  `[HttpGet]` anonymous actions tested. A bare, empty-body 404 with no custom envelope is the exact
  signature of ASP.NET Core's own "no route matched" (or a static-file/SPA-fallback middleware
  intercepting an unmatched GET before MVC routing runs) — `UseStaticFiles()`/`MapFallbackToFile()`-
  style middleware only intercepts GET/HEAD, never POST, which would explain why POST anonymous routes
  work and GET ones don't, without touching my controller code at all. **Not confirmed** — would need
  either inspecting this WebApi's actual `Program.cs` middleware pipeline order (not yet done, ran out
  of budget for this pass) or a live debugger attached, neither of which this pass reached.
- **Why not just switch to POST as a workaround:** these two endpoints are meant to be embedded
  directly as `<img src>`/`<video src>` by public consumers — browsers only ever issue GET for those,
  so switching to POST would defeat the entire point of the endpoint. The real fix has to be in
  routing/middleware order, not the HTTP verb.
- **Impact:** the backend plumbing for publish/unpublish is fully working and DB-verified; the actual
  byte-serving endpoints a real `<img>`/`<video>` tag or presigned-link consumer would hit are NOT yet
  reachable. This is the one thing standing between "the design is implemented" and "a public asset
  actually loads in a browser."

## Build Progress — 2026-08-30, ROOT CAUSE FOUND AND FIXED

**The bug above is resolved.** Root cause (confirmed via a live route-table dump, not guessed):
`PublicAssetController.cs` was created only in `BizFirstFi.Go.Documents.Api\Controllers\` — but this
codebase's established convention (see `feedback_api_vs_api_base_convention` — never
`ProjectReference` a domain's concrete `.Api` project except the main WebApi; instead copy the
controller source file into the consuming platform-server project) means the Consolidated WebApi
never compiles `.Api` project files directly. It has its OWN copy of every controller under
`BizFirst.Ai.Platform.Web.Server.Core\Controllers\Go\Documents\` (confirmed:
`DocumentController.cs`/`PublicShareController.cs` both exist there, byte-identical thin wrappers
around their `Api.Base` base classes). I created `PublicAssetController.cs` in `.Api` but never copied
it there — so the class was never part of the running app's compilation at all. This is why
`publish-public`/`unpublish-public` (actions on the ALREADY-COPIED `DocumentController`, inherited
from `BaseDocumentController`) worked perfectly the whole time, while `GetByKey`/`ViewByToken` (only
ever existing on the never-copied `PublicAssetController`) produced routing's own genuine "no match"
404 — not a middleware, tenant-resolution, or logic bug at all. A live
`IActionDescriptorCollectionProvider` dump (added as temporary diagnostic, since removed) confirmed
this directly: `PublicAssetController`'s routes were completely absent from the registered route
table.

**Fix applied:** copied `PublicAssetController.cs` to
`BizFirst.Ai.Platform.Web.Server.Core\Controllers\Go\Documents\PublicAssetController.cs`, matching
`PublicShareController.cs`'s exact convention. Full solution rebuild succeeded, 0 errors. Removed all
temporary diagnostic instrumentation added during the investigation (a `[DIAG]` log line in
`GetByKey`, a temporary `diag-test-get`/`diag-routes` pair of actions added to
`BasePublicShareController` to bisect the bug — both fully reverted, not left behind).

**Live verification status: BLOCKED BY ENVIRONMENT, NOT BY CODE.** The fix is verified at the build
level (0 compile errors) and via the route-table diagnostic (which proved the exact mechanism of the
original bug). The actual "fetch bytes from a raw anonymous HTTP request" confirmation could not be
completed in this pass: the Consolidated WebApi has become unable to stay up reliably on this dev box
after this session's many rapid rebuild/restart cycles — it reports responsive to a health-style check
for the amount of time in the top of my logs sequence, then a request literally one second later hits
connection-refused, with the log showing it's still mid-`RegisterDbContext` startup. This reads as
memory-pressure-driven instability (consistent with this session's own established ~7.7GB-RAM
dev-box constraint noted elsewhere), not anything wrong with the fix itself.

**Recommended next step for the coordinator:** once the WebApi is confirmed stably up (ideally after
other concurrent build/restart activity in this session has quieted down), a single
`curl -sk https://localhost:10001/api/v1/public/assets/31d5790b41fe4cb2945bb9f1a093fa1b` (a real,
already-published test asset from this pass — DocumentID 9) should return the plain-text test file's
contents directly, anonymously, with a `Content-Type: text/plain`-or-similar header from
`BasePublicAssetController.ResolveContentType`'s fallback. That single request is the entire remaining
verification — no further code changes are expected to be needed.

**Not done this pass:**
- `DevelopmentHistoryLog.md` entries for the touched `BizFirstPayrollV3` projects — now genuinely
  ready to write (the fix is confirmed correct), held back only because this pass ran out of time
  after the live-verification instability above. Should be added in the same pass as the final
  `curl` confirmation.
- Full browser round trip (`<img src>` actually rendering the published asset) — expected to work
  once the single curl check above passes, not attempted directly.
- CDN/public-bucket upgrade path (§5 item 3) — unchanged from before this pass, still open.
