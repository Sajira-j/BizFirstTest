# App Studio RAG Content — Upload Notes

Meta note on how/when the `rag/v1/` content in this folder gets into a real, queryable Knowledge Base
— NOT a re-derivation of the ingestion mechanism itself. That mechanism is already fully designed and
partially proven end-to-end in the sibling project
`Documentation\Employees\agentic-coding\atlas-form-automation-project\lessons\README.md` and
`Documentation\Employees\atlas-forms\atlas-forms-rag\agent\ragUploadDesign.md` — **read those first,
reuse the same process, don't reinvent it.**

## What's real and proven, as of the atlas-forms precedent (reuse this, don't re-derive)

- **Upload mechanism**: the real `document-manager` app's Knowledge Collections UI, batch multi-file
  upload with shared metadata (Type = "RAG Document"), confirmed to actually work for 74 real files in
  one submission.
- **Retrieval**: `knowledge_retrieval` (a `IFunctionCallback`-implementing plugin,
  `BizFirst.Ai.Octopus.Plugin.KnowledgeRetrieval`) calling the real, live `IFlowRagKnowledgeService
  .SearchKnowledgeAsync` against Postgres/pgvector — proven end-to-end with real content and real
  cosine-similarity scores through a real agent, not simulated.
- **Known gotchas already solved once** (see the atlas-forms lessons file for full detail — don't
  rediscover these): the default `AIAgent_KnowledgeBases.SimilarityThreshold` (0.7) was too strict for
  real documentation-vs-question cosine scores and needed lowering to ~0.5; the target
  `PrimaryCollectionName` needs to actually match the real collection name real documents were
  uploaded into, not a leftover test-collection name; Qdrant-backed collections and Postgres-backed
  collections are two separate, non-interoperating vector stores in this codebase — know which one a
  given collection actually uses before debugging "why isn't retrieval finding anything."

## What's specific to this project's content

Unlike the atlas-forms spec (single-purpose: teach an agent to build/edit Atlas Forms), this
`rag/v1/` content set teaches an agent the App/AppPage/AppSection/AppWidget schema, the structured
Style Builder system, and the real per-widget-type config shapes — i.e., how to generate/modify App
Studio app content correctly. Suggested collection name: `app-studio-spec-v1` (not yet created —
this is a naming proposal, matching the atlas-forms precedent's own `atlas-forms-automation`
collection-per-project convention, not a confirmed live collection).

## Honesty flags carried over from this content set's own authoring pass (2026-08-30)

Several `widgetTypes/*.md` files in this same `rag/v1/` folder are marked **OPEN QUESTION** for their
`configuration` field shapes — `chat-panel`, `workflow-template-category`, `hil-inbox`, `signin`,
`notifications`, `site-branding`, `page-navigation` were written from `WidgetRecord.ts`'s own summary
doc comment and confirmed package-directory existence, NOT from independently reading each handler
package's own config type source. `form` and `content` ARE fully verified against real source.
**Before this content is uploaded for real retrieval use, re-verify the OPEN QUESTION files against
each handler package's actual config type** (`packages/widget-handlers-{type}-widget/src/
*WidgetConfig.ts`) — uploading unverified field-name guesses into a RAG store that an agent will
then trust as ground truth would be worse than leaving those widget types undocumented.

## Suggested next step, mirroring the atlas-forms project's own Task 1

1. Confirm/create the target collection (`document-manager`'s Knowledge Collections UI).
2. Close the OPEN QUESTION gaps above by reading the real handler source.
3. Batch-upload `rag/v1/`'s `.md` files (this file and `widgetTypes/`'s own per-type files; exclude
   nothing — unlike atlas-forms-rag there are no `worked-examples/*.json` files in this set to
   exclude).
4. Wire a `knowledge_retrieval`-style function for whichever agent(s) are meant to build/edit App
   Studio content — reuse the exact mechanism already proven for Agent 23 in the atlas-forms project,
   don't build a second one.
