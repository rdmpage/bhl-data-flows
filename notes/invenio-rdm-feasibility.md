# Moving BHL to InvenioRDM — feasibility assessment

An assessment of whether BHL could move from its current stack (C# / ASP.NET / SQL Server, centred on `bhl-us`) to InvenioRDM as its underlying platform. Based on the data-flow diagrams in this repo, a code-level walkthrough of `bhl-us`, and BHL's published task schedule. The Invenio side is drawn from general knowledge of the framework and its Celery-based task infrastructure. Treat as an opinion grounded in what we know about BHL; the Invenio side is worth sanity-checking with InvenioRDM maintainers.

*Last updated: April 2026.*

## Top-line

**Technically feasible, strategically expensive — and probably not the right shape of bet.** The cheap migrations are the things Invenio already does well; the expensive ones are most of what makes BHL distinctive, and those would all need rebuilding on top of Invenio's primitives. A like-for-like swap is unlikely to pay for itself.

## Reasons

### 1. Model mismatch is the headline

Invenio's atomic unit is a **record** — a citable object with attached files. BHL's atomic unit is a **page**: items contain pages contain segments (articles/parts), users browse and search at page granularity, with per-page taxonomic names, per-page OCR, deep-linkable page URLs, and a reader (IA BookReader) that's central to the experience.

The reason Zenodo's UI doesn't fit BHL isn't a skin problem — it's that Zenodo's UI honestly reflects Invenio's record model. Forcing BHL's hierarchy into records-with-files means either fat custom record metadata, custom relation graphs, or a bespoke data layer on top of Invenio's. At which point the framework has been kept but the meaningful parts have been rewritten.

### 2. The expensive bits aren't the database migration

Looking at the [integration-seams view](../diagrams/integration-seams.md), the cheap migrations are the things Invenio already does well:

- OAI-PMH server (`invenio-oaiserver`)
- DOI minting (Invenio supports DataCite natively; CrossRef less directly)
- REST API surface
- Search (Invenio uses OpenSearch)
- Basic citation downloads
- **Task scheduling** — see section below

The expensive bits are everything that makes BHL *BHL*:

- The IA-coupled ingest pipeline — the three-stage IA trio (`IA Analysis Harvest → IA Harvest Async orchestrator → IA Harvest workers`), the `BHLImport DB` staging layer, and the daily `Publish To Production` promotion step.
- The per-page taxonomic name pipeline (`BHLPageNameRefresh` + `gnfinder`), including the separate external `bhlindex` Global Names tool (which writes to its own PostgreSQL database that nothing in BHL reads back).
- The page-level reader integration (IA BookReader iframe; the IIIF code in `BHLUSWeb2` is wired but returns 404 in production).
- The OpenURL resolver.
- The legacy `/api2` and `/api3` surfaces that external tools still consume — breaking these is a commitment to every downstream consumer migrating too.
- The bibliographic-hierarchy browsing UI (title → item → page → reader), which is how users actually consume BHL.
- The **PDF generation lifecycle** — two trigger paths (user-requested via web form, and segment PDFs auto-generated when a part/article is created or modified), a shared iTextSharp-based generator producing searchable PDFs from DJVU + OCR, a 45-day retention cycle, and direct SMTP notification to users.
- The **outbound publishing pipeline** to AWS Open Data — BHL publishes images and OCR text (not just metadata) to the [BHL Open Data dataset on AWS](https://registry.opendata.aws/bhl-open-data/), plus METS XML and name-index files back to Internet Archive S3.

None of those come for free. All would need rebuilding on top of Invenio's primitives. That's most of the codebase.

### 3. The structural dependency on Internet Archive doesn't move

BHL doesn't host its own page images — it embeds IA's BookReader. Macaw uploads to IA. Harvesters pull from IA (on a weekly Saturday schedule — up to 7 days' lag for new content). Name File Generator and METS Upload push metadata XML to IA S3. Whatever platform BHL runs on, Internet Archive is still in the loop in three different roles (source, destination, live image host). Invenio doesn't change that, and Invenio isn't designed for that relationship either.

### 4. The Private API seam is narrower than it looks

An earlier version of this assessment implied the BHL Services Private API was a large fan-in with ~17 callers. A code audit of every REST `Client` class shows only **7 components** have genuine data dependencies on the Private API (Flickr Thumb Grab + 6 Process-tier jobs). The remaining 10 callers use it exclusively for operational logging and email notification — they write their actual data directly to the databases via shared .NET libraries (`BHLImportServer`, `BHLProvider`). See the [integration-seams view](../diagrams/integration-seams.md) for the three-cluster breakdown.

This means the "write gateway" pattern is aspirational, not enforced. A platform migration wouldn't be migrating off a clean gateway — it would be migrating off direct library-level DB coupling in many components.

## Task scheduling: a bright spot

BHL runs ~12 scheduled harvester/processor tasks (see `tasks.csv` and the [README task tables](../README.md#tasks)) plus ~15 DB maintenance jobs. Frequencies range from continuous (Search Indexer, PDF Generator as MQ consumers) through daily (most enrichment) to weekly (IA harvest on Saturdays) and once-weekly (Export Processor on Fridays).

**InvenioRDM handles this well.** The Invenio framework uses **Celery** for background tasks and **Celery Beat** for periodic scheduling. BHL's task schedule would map directly:

| BHL pattern | Invenio equivalent |
|-------------|-------------------|
| Weekly Saturday harvest | Celery Beat crontab: `crontab(hour=X, day_of_week='saturday')` |
| Daily batch jobs | Celery Beat crontab: `crontab(hour=X)` |
| Continuous MQ consumers | Persistent Celery workers consuming from dedicated queues |
| On-demand (user-requested PDF) | Celery task triggered by web request |

Invenio also ships with task-ready modules for OAI harvesting (`invenio-oaiharvester`), indexing (`invenio-indexer`), file integrity checks (`invenio-files-rest`), and DOI registration. So the scheduling infrastructure is a genuine positive — one of the few areas where InvenioRDM would give BHL something better than what it has today.

BHL currently uses a two-layer approach: **Windows Task Scheduler** invokes batch console apps on a schedule (most harvesters and processors are run-once-and-exit executables), and **RabbitMQ** handles async message passing between components (e.g. a batch job drops messages into a queue, and a continuous consumer like the Search Indexer picks them up). RabbitMQ is the async backbone but doesn't handle scheduling — it doesn't decide *when* things run. InvenioRDM's Celery + Celery Beat would unify both layers into one framework: Celery Beat replaces Windows Task Scheduler for cron-style triggering, and Celery workers replace the RabbitMQ consumer daemons.

## Outbound publishing: AWS Open Data and Figshare

BHL's outbound publishing is more substantial than just serving content to users:

- **AWS Open Data** — BHL publishes **images and OCR text** (not just metadata) to the [BHL Open Data dataset on AWS](https://registry.opendata.aws/bhl-open-data/). This is content derived from the IA collection, converted and deposited for open reuse.
- **Figshare (Smithsonian instance)** — BHL deposits data dumps (OCR text corpora) here.
- **Internet Archive S3** — METS XML and name-index XML are uploaded back to IA daily by the Name File Generator and METS Upload tasks.

Neither the AWS nor Figshare sync pipeline lives in the `bhl-us` C# codebase — they're external processes. On Invenio, these could be implemented as Celery tasks, but the conversion/packaging logic would still need writing from scratch. Invenio's file-management layer (`invenio-files-rest`) could help manage the content, and Celery Beat would schedule the sync, but the actual S3-upload and Figshare-deposit logic is BHL-specific.

## A more pragmatic alternative

Use the [integration-seams analysis](../diagrams/integration-seams.md) as a roadmap for incremental modernisation of the existing stack. The strongest seams are already swap-friendly:

- **RabbitMQ** — few producers and consumers, simple message types.
- **Elasticsearch via `SearchElastic/`** — a single shared library owns the interface. This is BHL's best-behaved seam.
- **Per-harvester isolation** — each external source touches one harvester.

The weakest seams are the next refactors:

- **Direct DB writers.** The ingest harvesters and `BHLTextImportProcessor` bypass the Private API and write directly to BHL DB via .NET libraries. Routing these through REST would make the Private API a real architectural gate.
- **Shared abstractions for file storage and email.** The `SearchElastic/` pattern (single library owns the external interface) is worth replicating for Static Files access and SMTP.
- **Outbound publishing.** The AWS Open Data and Figshare sync processes live outside `bhl-us`. Bringing them into the codebase (or at least documenting and scheduling them alongside the other tasks) would make the full pipeline visible and testable.

This path moves BHL toward a modular, maintainable shape *without* committing to a platform replacement that doesn't natively serve the page-centric data model.

## Where Invenio could legitimately help (conservative view)

Niche components alongside BHL, not a wholesale replacement:

- A BHL-adjacent **research-data deposits** service for derived datasets (OCR corpora, name indexes, bulk exports to Figshare/AWS). This is exactly Invenio's sweet spot — citable records with attached files and DOIs.
- A **DOI minting** service, if decoupling from CrossRef via a more modern stack becomes worthwhile.
- A separate **Invenio instance alongside BHL**, not in place of it, for dataset publication workflows that don't fit the page-centric reading experience.

---

## Alternative framing: InvenioRDM as headless backend

The assessment above assumes InvenioRDM would replace everything — backend, frontend, task scheduling, the lot. A more interesting question: **what if InvenioRDM replaces only the backend infrastructure** (data management, file storage, task scheduling, REST API), with a custom frontend for the public-facing UI, and IA treated as a separate concern?

### The architecture

```
┌─────────────────────────────────────────────────────┐
│  Custom public frontend (React / Next.js / etc.)    │
│  Queries OpenSearch directly for search + browse     │
│  Renders page reader, names, citations               │
└────────────────────┬────────────────────────────────┘
                     │ REST API
┌────────────────────▼────────────────────────────────┐
│  InvenioRDM (headless backend)                       │
│  ┌──────────────┐  ┌──────────────┐                 │
│  │ Record mgmt  │  │ File storage │                 │
│  │ (PostgreSQL) │  │ (S3-compat)  │                 │
│  └──────────────┘  └──────────────┘                 │
│  ┌──────────────┐  ┌──────────────┐                 │
│  │ OpenSearch   │  │ Celery +     │                 │
│  │ indexing     │  │ Celery Beat  │                 │
│  └──────────────┘  └──────────────┘                 │
│  Built-in: OAI-PMH · DOI · REST API · file checks   │
└─────────────────────────────────────────────────────┘
                     │ Celery tasks
┌────────────────────▼────────────────────────────────┐
│  BHL-specific Celery tasks                           │
│  Harvesters · gnfinder names · PDF generation        │
│  METS/name-file upload · AWS/Figshare export         │
└─────────────────────────────────────────────────────┘
```

### What this sidesteps

- **UI mismatch** — gone. The frontend is custom either way. Zenodo's record-list UI is irrelevant if it's never shown to users. The custom frontend queries OpenSearch (which InvenioRDM manages) and renders whatever BHL needs: bibliographic hierarchy, page reader, name lists, search results.
- **IA as a monolithic dependency** — factored out. InvenioRDM's `invenio-files-rest` manages content (images, OCR, DJVU) in S3-compatible object storage. IA becomes *one possible source* of content (via harvest tasks) rather than the only place content lives. BHL could serve page images from its own storage, or continue embedding IA BookReader, or adopt IIIF — the backend doesn't care.
- **Task scheduling split** — unified. Celery replaces both Windows Task Scheduler (cron-style triggers) and the current RabbitMQ consumer pattern (async processing) with one framework. BHL's ~12 scheduled tasks become Celery Beat entries; the continuous consumers (Search Indexer, PDF Generator) become persistent Celery workers.

### What changes

| BHL component | Current | On InvenioRDM |
|--------------|---------|---------------|
| BHL DB (SQL Server) | Central schema, `BHLProvider` / `BHLCoreDAL` | InvenioRDM's PostgreSQL + custom record types |
| Static Files (NAS) | Shared filesystem with path-based access | `invenio-files-rest` + S3-compatible object storage |
| Elasticsearch | Written by Search Indexer, read via `SearchElastic/` | OpenSearch managed by InvenioRDM; custom frontend queries it |
| RabbitMQ | Async message passing | Celery (uses RabbitMQ or Redis as broker under the hood) |
| Windows Task Scheduler | Triggers batch .exe apps | Celery Beat crontab entries |
| BHL Services Private API | Aspirational write gateway (7 real data callers) | InvenioRDM REST API + custom endpoints |
| BHLImport DB (staging) | Separate SQL Server database | Invenio deposit/draft workflow (records in draft state before publishing) |
| OAI-PMH server | Custom `OAI2` library in `BHLUSWeb2` | `invenio-oaiserver` (built-in) |
| DOI minting | Custom `BHLDOIService` → CrossRef | `invenio-rdm-records` DOI integration (DataCite native; CrossRef possible) |
| Search Index Queue Load | Batch job that detects DB changes → MQ | Celery signals on record create/update → auto-index |

### The page addressability question

This is the crux. InvenioRDM's record model doesn't natively have sub-record entities like "pages". Three approaches:

1. **Pages as files with rich metadata.** Each page = a file object in the record, with custom metadata (sequence number, OCR text hash, name list). InvenioRDM's `invenio-files-rest` supports per-file metadata. Page images, OCR text files, and DJVU files are all file objects attached to the item record. The custom frontend resolves `/page/{id}` by querying the record's file list. **Trade-off:** addressing a single page means fetching the record and filtering. For items with thousands of pages, this needs efficient indexing.

2. **Pages as nested metadata.** The item record's metadata JSON contains a `pages` array with per-page entries (sequence, OCR text reference, names). Page images are linked by file key. **Trade-off:** large metadata documents for big items, but OpenSearch handles nested objects well for search. This is essentially how IIIF manifests work — a single JSON document describing all pages.

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
- **Outbound publishing.** AWS Open Data and Figshare sync as Celery tasks — the packaging/conversion logic is BHL-specific.

### Revised assessment

Under this framing, the assessment shifts from **"probably not the right bet"** to **"a plausible architecture if you're willing to invest in the page model."**

The key contingencies:

1. **Page addressability must be proven.** Build a prototype: one InvenioRDM instance, one BHL item with 500 pages, confirm that pages are addressable, searchable, and performant. This is a bounded experiment (weeks, not months) that would de-risk the whole approach.
2. **IA decoupling is optional.** InvenioRDM can manage its own files AND still embed IA BookReader for page display — the two aren't mutually exclusive. Start with InvenioRDM managing metadata + OCR + names, keep IA for images initially, and migrate image hosting later if/when BHL wants to.
3. **The frontend is a separate project.** Don't let the backend choice constrain the frontend. A custom frontend on OpenSearch works regardless of whether the backend is InvenioRDM, the current .NET stack, or something else.
4. **Migration is incremental, not big-bang.** Start with one harvester (e.g. Biostor, which is the simplest), one record type (items), and one enrichment task (name extraction). Prove the pipeline end-to-end before committing to a full migration.

## Caveats

- Invenio's most recent extension capabilities may be more flexible than this note assumes — worth checking with InvenioRDM maintainers before ruling out a more ambitious integration.
- CERN's CDS (CERN Document Server) runs on Invenio in a library-shaped configuration; if anyone has built a page-level reading platform on Invenio, CDS is where to look. I suspect the answer is "no", but a yes would strengthen the case.
- The data-flow diagrams in this repo are documentation, not production audits. The BHL source code may have since evolved, and direct conversation with BHL engineers would surface constraints this note doesn't.
- The task-scheduling assessment is based on general knowledge of Celery/Celery Beat, not a direct test with BHL's task definitions. The mapping looks clean on paper but operational details (error handling, retry logic, monitoring, Windows-to-Linux migration for task hosts) would need validating.
- Scale: BHL has millions of pages. If pages become InvenioRDM records (approach 3), the record count is large. OpenSearch handles this but InvenioRDM's record-management layer needs benchmarking at that scale.
