# Moving BHL to InvenioRDM — feasibility assessment

Could BHL move from its current stack (C# / ASP.NET / SQL Server, centred on `bhl-us`) to InvenioRDM? The answer depends on what "move to InvenioRDM" means — but some architectural decisions apply regardless.

Based on the data-flow diagrams in this repo, a code-level walkthrough of `bhl-us`, and BHL's published task schedule. The Invenio side is drawn from general knowledge of the framework and its Celery-based infrastructure. Treat as an opinion grounded in what we know about BHL; the Invenio side is worth sanity-checking with InvenioRDM maintainers.

*Last updated: April 2026.*

## Architectural decisions that apply to any InvenioRDM adoption

Before choosing how much of BHL to move onto InvenioRDM, three cross-cutting decisions need making. These don't change between approaches — get them wrong and neither approach works.

### 1. Internet Archive must be restructured

BHL currently depends on IA in three roles: as a **source** (harvesters pull content), as a **destination** (METS and name-file XML pushed back to IA S3), and as a **live image host** (the IA BookReader iframe serves page images at read time). Macaw also uploads directly to IA.

Moving to InvenioRDM means restructuring this relationship. Two options:

- **Decouple from IA.** InvenioRDM's `invenio-files-rest` manages content (images, OCR, DJVU) in S3-compatible object storage. IA becomes *one possible source* of content (via harvest tasks) rather than the only place content lives. BHL serves page images from its own storage instead of embedding IA BookReader. This is the cleanest architecture but requires BHL to take on image hosting (storage, bandwidth, CDN).
- **Treat IA as another S3-style data store.** Keep IA as the primary content repository but access it through the same storage abstractions InvenioRDM uses for its own files. IA BookReader embedding can continue unchanged until BHL is ready to replace it. More incremental — IA stays in the loop but behind an abstraction.

Either way, the IA Harvest pipeline (discovery → orchestration → per-item worker) needs reimplementing as Celery tasks regardless of how files are stored.

### 2. Search should be a separate Elasticsearch instance

InvenioRDM ships with built-in OpenSearch, but for BHL a **separate Elasticsearch instance** gives more flexibility: full control over index schema, query tuning, and upgrade schedule, with no coupling to InvenioRDM's internal search layer. The custom frontend queries Elasticsearch directly; a Celery task handles indexing when records change. This mirrors BHL's current architecture (the `SearchElastic/` library is already the cleanest seam in the codebase) and preserves the option to evolve search independently.

### 3. The public-facing UI is custom, not InvenioRDM's

Zenodo's UI reflects InvenioRDM's record model and doesn't fit BHL's bibliographic-hierarchy / page-reader experience. Under any InvenioRDM adoption, the public-facing UI would be a **custom frontend** querying the separate Elasticsearch instance for search and browse, and InvenioRDM's REST API for record data and files. This sidesteps the UI-mismatch concern entirely — but it means the frontend is a separate build project.

---

## Two approaches

With those decisions made, the question is how much of BHL's backend InvenioRDM replaces.

| | **A: Headless backend** | **B: Complete replacement** |
|---|---|---|
| **What InvenioRDM replaces** | Data management, file storage, task scheduling, REST API, OAI-PMH, DOI | Everything: backend, frontend, UI, scheduling, the lot |
| **Public-facing UI** | Custom frontend (per decision 3 above) | InvenioRDM's own UI (Zenodo-style) |
| **Page addressability** | Needs a custom extension (bounded work) | Forced into record-with-files model (poor fit) |
| **Assessment** | **Plausible** — if the page model can be proven | **Strategically expensive** — most of what makes BHL distinctive would need rebuilding |
| **De-risking** | Prototype one item with 500 pages (weeks) | No easy way to test without large commitment |

---

## Approach A: InvenioRDM as headless backend

The more promising framing. InvenioRDM replaces the **backend infrastructure** (data management, file storage, task scheduling, REST API), with the custom frontend and Elasticsearch instance sitting alongside it.

### The architecture

```
┌─────────────────────────────────────────────────────┐
│  Custom public frontend                              │
│  Queries Elasticsearch for search + browse            │
│  Renders page reader, names, citations               │
└──────────┬──────────────────────┬───────────────────┘
           │ REST API             │ search queries
┌──────────▼──────────┐   ┌──────▼──────────────────┐
│  InvenioRDM          │   │  Elasticsearch           │
│  (headless backend)  │   │  (separate instance,     │
│  ┌────────────────┐  │   │   not InvenioRDM's       │
│  │ Record mgmt    │  │   │   built-in search)        │
│  │ (PostgreSQL)   │  │   └─────────────────────────┘
│  └────────────────┘  │
│  ┌────────────────┐  │
│  │ File storage   │  │
│  │ (S3-compat)    │  │
│  └────────────────┘  │
│  ┌────────────────┐  │
│  │ Celery +       │  │
│  │ Celery Beat    │  │
│  └────────────────┘  │
│  OAI-PMH · DOI · REST│
└──────────────────────┘
           │ Celery tasks
┌──────────▼──────────────────────────────────────────┐
│  BHL-specific Celery tasks                           │
│  Harvesters · gnfinder names · PDF generation        │
│  Search indexing · AWS/Figshare export                │
└─────────────────────────────────────────────────────┘
```

### What this sidesteps

With the three cross-cutting decisions above (IA restructured, separate Elasticsearch, custom frontend), the remaining obstacles specific to Approach A shrink to the page-addressability question (see below). The UI mismatch, IA coupling, and search-layer concerns are all handled at the shared level.

### Component mapping

| BHL component | Current | On InvenioRDM |
|--------------|---------|---------------|
| BHL DB (SQL Server) | Central schema, `BHLProvider` / `BHLCoreDAL` | InvenioRDM's PostgreSQL + custom record types |
| Static Files (NAS) | Shared filesystem with path-based access | `invenio-files-rest` + S3-compatible object storage |
| Elasticsearch | Written by Search Indexer, read via `SearchElastic/` | Separate Elasticsearch instance; indexed by a Celery task; queried directly by the custom frontend |
| RabbitMQ | Async message passing | Celery (uses RabbitMQ or Redis as broker under the hood) |
| Windows Task Scheduler | Triggers batch .exe apps | Celery Beat crontab entries |
| BHL Services Private API | Aspirational write gateway (7 real data callers) | InvenioRDM REST API + custom endpoints |
| BHLImport DB (staging) | Separate SQL Server database | Invenio deposit/draft workflow (records in draft state before publishing) |
| OAI-PMH server | Custom `OAI2` library in `BHLUSWeb2` | `invenio-oaiserver` (built-in) |
| DOI minting | Custom `BHLDOIService` → CrossRef | `invenio-rdm-records` DOI integration (DataCite native; CrossRef possible) |
| Search Index Queue Load | Batch job that detects DB changes → MQ | Celery task that indexes records into the separate Elasticsearch instance |

### The page addressability question

This is the crux. InvenioRDM's record model doesn't natively have sub-record entities like "pages". Three approaches:

1. **Pages as files with rich metadata.** Each page = a file object in the record, with custom metadata (sequence number, OCR text hash, name list). InvenioRDM's `invenio-files-rest` supports per-file metadata. Page images, OCR text files, and DJVU files are all file objects attached to the item record. The custom frontend resolves `/page/{id}` by querying the record's file list. **Trade-off:** addressing a single page means fetching the record and filtering. For items with thousands of pages, this needs efficient indexing.

2. **Pages as nested metadata.** The item record's metadata JSON contains a `pages` array with per-page entries (sequence, OCR text reference, names). Page images are linked by file key. **Trade-off:** large metadata documents for big items, but Elasticsearch handles nested objects well for search. This is essentially how IIIF manifests work — a single JSON document describing all pages.

3. **Pages as child records.** Each page is its own InvenioRDM record, linked to the parent item via Invenio's relation system. Each page is independently addressable, searchable, and can carry its own files and metadata. **Trade-off:** record count explodes (BHL has millions of pages), but this gives the cleanest addressability and maps well to how BHL currently treats pages (each page has its own ID, URL, and metadata).

Approach (3) is the most natural fit for BHL's page-centric model but needs scale testing. Approach (2) is the most pragmatic starting point. A hybrid (item metadata has lightweight page stubs; OCR text and names live in the search index only, not in the record) might be the sweet spot.

### What you'd gain

- **Modern Python stack** with an active community, replacing aging C# / ASP.NET Web Forms.
- **PostgreSQL** replacing SQL Server (open-source, better tooling ecosystem, no licensing cost).
- **Unified task infrastructure** (Celery) replacing the Windows Task Scheduler + RabbitMQ split.
- **Built-in OAI-PMH, DOI, REST API, file management** — no longer hand-rolled.
- **S3-compatible file storage** — makes mirroring to AWS Open Data and Figshare a natural extension of the file layer rather than an external script.
- **InvenioRDM's deposit workflow** could replace the `BHLImport DB` staging pattern — records start as drafts, go through validation, and get published. That's essentially what staging → production promotion does today.

### What you'd still have to build

- **The custom frontend.** But BHL would need a new frontend regardless of backend — the current Web Forms / MVC hybrid is aging.
- **Harvesters.** The IA trio, Biostor, OAI, Flickr, Wikidata harvesters need reimplementing as Celery tasks that create/update InvenioRDM records. The logic is BHL-specific; the scheduling and execution framework is Celery.
- **gnfinder integration.** A Celery task that reads OCR from the record's files, calls gnfinder, and writes names back as record metadata. Straightforward.
- **PDF generation.** Port the iTextSharp logic to Python (e.g. reportlab, PyMuPDF, or pikepdf). The two-trigger-path lifecycle (user-requested + segment-based) maps onto Celery tasks cleanly.
- **Legacy API compatibility.** `/api2` and `/api3` consumers would need a compatibility shim or a migration path.
- **The page addressability extension.** Whichever approach is chosen, it's custom Invenio work.
- **Outbound publishing.** AWS Open Data (images + OCR text) and Figshare sync as Celery tasks — the packaging/conversion logic is BHL-specific.

### Assessment

**Plausible if you're willing to invest in the page model.** The key contingencies:

1. **Page addressability must be proven.** Build a prototype: one InvenioRDM instance, one BHL item with 500 pages, confirm that pages are addressable, searchable, and performant. This is a bounded experiment (weeks, not months) that would de-risk the whole approach.
2. **IA decoupling is optional.** InvenioRDM can manage its own files AND still embed IA BookReader for page display — the two aren't mutually exclusive. Start with InvenioRDM managing metadata + OCR + names, keep IA for images initially, and migrate image hosting later if/when BHL wants to.
3. **The frontend is a separate project.** Don't let the backend choice constrain the frontend. A custom frontend on Elasticsearch works regardless of whether the backend is InvenioRDM, the current .NET stack, or something else.
4. **Migration is incremental, not big-bang.** Start with one harvester (e.g. Biostor, which is the simplest), one record type (items), and one enrichment task (name extraction). Prove the pipeline end-to-end before committing to a full migration.

---

## Approach B: Complete replacement

Replace the whole BHL platform — backend, frontend, UI, task scheduling — with InvenioRDM. This is the approach that the Zenodo comparison invites, and it's the less promising one.

### Model mismatch is the headline

Invenio's atomic unit is a **record** — a citable object with attached files. BHL's atomic unit is a **page**: items contain pages contain segments (articles/parts), users browse and search at page granularity, with per-page taxonomic names, per-page OCR, deep-linkable page URLs, and a reader (IA BookReader) that's central to the experience.

The reason Zenodo's UI doesn't fit BHL isn't a skin problem — it's that Zenodo's UI honestly reflects Invenio's record model. Forcing BHL's hierarchy into records-with-files means either fat custom record metadata, custom relation graphs, or a bespoke data layer on top of Invenio's. At which point the framework has been kept but the meaningful parts have been rewritten.

### The expensive bits aren't the database migration

Looking at the [integration-seams view](../diagrams/integration-seams.md), the cheap migrations are the things Invenio already does well:

- OAI-PMH server (`invenio-oaiserver`)
- DOI minting (Invenio supports DataCite natively; CrossRef less directly)
- REST API surface
- Search (separate Elasticsearch instance)
- Basic citation downloads
- Task scheduling (Celery + Celery Beat)

The expensive bits are everything that makes BHL *BHL*:

- The IA-coupled ingest pipeline — the three-stage IA trio (`IA Analysis Harvest → IA Harvest Async orchestrator → IA Harvest workers`), the `BHLImport DB` staging layer, and the daily `Publish To Production` promotion step.
- The per-page taxonomic name pipeline (`BHLPageNameRefresh` + `gnfinder`), including the separate external `bhlindex` Global Names tool (which writes to its own PostgreSQL database that nothing in BHL reads back).
- The page-level reader integration (IA BookReader iframe; the IIIF code in `BHLUSWeb2` is wired but returns 404 in production).
- The OpenURL resolver.
- The legacy `/api2` and `/api3` surfaces that external tools still consume — breaking these is a commitment to every downstream consumer migrating too.
- The bibliographic-hierarchy browsing UI (title → item → page → reader), which is how users actually consume BHL.
- The **PDF generation lifecycle** — two trigger paths (user-requested via web form, and segment PDFs auto-generated when a part/article is created or modified), a shared iTextSharp-based generator producing searchable PDFs from DJVU + OCR, a 45-day retention cycle, and direct SMTP notification to users.
- The **outbound publishing pipeline** — BHL publishes images and OCR text to [AWS Open Data](https://registry.opendata.aws/bhl-open-data/), data dumps to Figshare (Smithsonian instance), plus METS XML and name-index files back to Internet Archive S3.

None of those come for free. All would need rebuilding on top of Invenio's primitives. That's most of the codebase.

### The Private API seam is narrower than it looks

A code audit of every REST `Client` class shows only **7 components** have genuine data dependencies on the BHL Services Private API (Flickr Thumb Grab + 6 Process-tier jobs). The remaining 10 callers use it exclusively for operational logging and email notification — they write their actual data directly to the databases via shared .NET libraries (`BHLImportServer`, `BHLProvider`). See the [integration-seams view](../diagrams/integration-seams.md) for the three-cluster breakdown.

This means the "write gateway" pattern is aspirational, not enforced. A platform migration wouldn't be migrating off a clean gateway — it would be migrating off direct library-level DB coupling in many components.

### Assessment

**Strategically expensive — probably not the right shape of bet.** A like-for-like swap means rebuilding most of what makes BHL distinctive on top of Invenio's primitives, while getting the least benefit from Invenio's strengths (since BHL's needs don't align with the record model).

If the goal is to modernise the .NET stack without changing the architecture, a more pragmatic alternative is to use the [integration-seams analysis](../diagrams/integration-seams.md) as a roadmap: the strongest seams (RabbitMQ, `SearchElastic/`, per-harvester isolation) are already swap-friendly, and the weakest (direct DB writers, file-storage abstraction, email routing) are bounded refactors.

Even under this framing, Invenio could legitimately help as **niche components alongside BHL** — a research-data deposits service for derived datasets (OCR corpora, name indexes, bulk exports), or a DOI minting service, run as a separate instance rather than a platform replacement.

---

## Shared considerations

### Task scheduling

BHL runs ~12 scheduled harvester/processor tasks (see `tasks.csv` and the [README task tables](../README.md#tasks)) plus ~15 DB maintenance jobs. Frequencies range from continuous (Search Indexer, PDF Generator as MQ consumers) through daily (most enrichment) to weekly (IA harvest on Saturdays).

BHL currently uses a two-layer approach: **Windows Task Scheduler** invokes batch console apps on a schedule (most harvesters and processors are run-once-and-exit executables), and **RabbitMQ** handles async message passing between components. RabbitMQ is the async backbone but doesn't handle scheduling — it doesn't decide *when* things run.

**InvenioRDM handles this well.** Celery + Celery Beat would unify both layers:

| BHL pattern | Invenio equivalent |
|-------------|-------------------|
| Weekly Saturday harvest | Celery Beat crontab: `crontab(hour=X, day_of_week='saturday')` |
| Daily batch jobs | Celery Beat crontab: `crontab(hour=X)` |
| Continuous MQ consumers | Persistent Celery workers consuming from dedicated queues |
| On-demand (user-requested PDF) | Celery task triggered by web request |

Invenio also ships with task-ready modules for OAI harvesting (`invenio-oaiharvester`), indexing (`invenio-indexer`), file integrity checks (`invenio-files-rest`), and DOI registration. Task scheduling is a genuine positive for both approaches — one of the few areas where InvenioRDM would give BHL something better than what it has today.

### Outbound publishing

BHL's outbound publishing is more substantial than just serving content to users:

- **AWS Open Data** — BHL publishes **images and OCR text** to the [BHL Open Data dataset on AWS](https://registry.opendata.aws/bhl-open-data/). This is content derived from the IA collection, converted and deposited for open reuse.
- **Figshare (Smithsonian instance)** — BHL deposits data dumps (OCR text corpora) here.
- **Internet Archive S3** — METS XML and name-index XML are uploaded back to IA daily.

Neither the AWS nor Figshare sync pipeline lives in the `bhl-us` C# codebase — they're external processes. Under Approach A, InvenioRDM's S3-compatible file storage would make these sync targets more natural (content already lives in S3; publishing to AWS Open Data or Figshare becomes a Celery task that reads from the same storage layer). Under Approach B, the situation is the same — the conversion/packaging logic is BHL-specific either way.

## Caveats

- Invenio's most recent extension capabilities may be more flexible than this note assumes — worth checking with InvenioRDM maintainers before ruling out a more ambitious integration.
- CERN's CDS (CERN Document Server) runs on Invenio in a library-shaped configuration; if anyone has built a page-level reading platform on Invenio, CDS is where to look. I suspect the answer is "no", but a yes would strengthen the case for Approach A.
- The data-flow diagrams in this repo are documentation, not production audits. The BHL source code may have since evolved, and direct conversation with BHL engineers would surface constraints this note doesn't.
- The task-scheduling assessment is based on general knowledge of Celery/Celery Beat, not a direct test with BHL's task definitions. The mapping looks clean on paper but operational details (error handling, retry logic, monitoring, Windows-to-Linux migration for task hosts) would need validating.
- Scale: BHL has millions of pages. If pages become InvenioRDM records (Approach A, option 3), the record count is large. Elasticsearch handles this but InvenioRDM's record-management layer needs benchmarking at that scale.
