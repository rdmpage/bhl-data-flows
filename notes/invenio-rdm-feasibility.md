# Moving BHL to InvenioRDM — feasibility assessment

An assessment of whether BHL could move from its current stack (C# / ASP.NET / SQL Server, centred on `bhl-us`) to InvenioRDM as its underlying platform. Based on the data-flow diagrams in this repo and the code walkthrough behind them, not on any direct prototyping with Invenio. Treat as an opinion grounded in what we know about BHL; the Invenio side is drawn from general knowledge and is worth sanity-checking with InvenioRDM maintainers.

## Top-line

**Technically feasible, strategically expensive — and probably not the right shape of bet.** The cheap migrations are the things Invenio already does well; the expensive ones are most of what makes BHL distinctive, and those would all need rebuilding on top of Invenio's primitives. A like-for-like swap is unlikely to pay for itself.

## Three reasons

### 1. Model mismatch is the headline

Invenio's atomic unit is a **record** — a citable object with attached files. BHL's atomic unit is a **page**: items contain pages contain segments, users browse and search at page granularity, with per-page taxonomic names, per-page OCR, deep-linkable page URLs, and a reader (IA BookReader) that's central to the experience.

The reason Zenodo's UI doesn't fit BHL isn't a skin problem — it's that Zenodo's UI honestly reflects Invenio's record model. Forcing BHL's hierarchy into records-with-files means either fat custom record metadata, custom relation graphs, or a bespoke data layer on top of Invenio's. At which point the framework has been kept but the meaningful parts have been rewritten.

### 2. The expensive bits aren't the database migration

Looking at the [integration-seams view](../diagrams/integration-seams.md), the cheap migrations are the things Invenio already does well:

- OAI-PMH server
- DOI minting (Invenio supports DataCite natively; CrossRef less directly)
- REST API surface
- Search (Invenio uses OpenSearch)
- Basic citation downloads

The expensive bits are everything that makes BHL *BHL*:

- The IA-coupled ingest pipeline — the three-stage IA trio (`IA Analysis Harvest → IA Harvest Async orchestrator → IA Harvest workers`) and the whole `BHLImport DB` staging layer.
- The per-page taxonomic name pipeline (`BHLPageNameRefresh` + `gnfinder`), plus the integration with the separate `bhlindex` Global Names tool.
- The page-level reader integration (IA BookReader iframe, with the latent IIIF option).
- The OpenURL resolver.
- The legacy `/api2` and `/api3` surfaces that external tools still consume — breaking these is a commitment to every downstream consumer migrating too.
- The bibliographic-hierarchy browsing UI (title → item → page → reader), which is how users actually consume BHL.

None of those come for free. All would need rebuilding on top of Invenio's primitives. That's most of the codebase.

### 3. The structural dependency on Internet Archive doesn't move

BHL doesn't host its own page images — it embeds IA's BookReader. Macaw uploads to IA. Harvesters pull from IA. Name File Generator and METS Upload push XML to IA S3. Whatever platform BHL runs on, Internet Archive is still in the loop in three different roles. Invenio doesn't change that, and Invenio isn't designed for that relationship either.

## A more pragmatic alternative

Use the [integration-seams analysis](../diagrams/integration-seams.md) as a roadmap for incremental modernisation of the existing stack. The strongest seams are already swap-friendly:

- **RabbitMQ** — few producers and consumers, simple message types.
- **Elasticsearch via `SearchElastic/`** — a single shared library owns the interface.
- **Per-harvester isolation** — each external source touches one harvester.

The weakest seams are the next refactors:

- **BHL DB direct writers.** The "Private API as write gateway" story is aspirational. `BHLPageNameRefresh`, `BHLTextImportProcessor`, `BHLOcrRefresh`, `BHLPDFGenerator`, and the Admin Web Site all write directly. Routing these through the Private API would make the gateway real, and only then does it become a credible seam for further changes.
- **Shared abstractions for file storage and email.** The `SearchElastic/` pattern (single library owns the external interface) is worth replicating.

This path moves BHL toward a modular, maintainable shape *without* committing to a platform replacement that doesn't natively serve the page-centric data model.

## Where Invenio could legitimately help

Niche components, not the whole system:

- A BHL-adjacent **research-data deposits** service for derived datasets (OCR corpora, name indexes, bulk exports). This is exactly Invenio's sweet spot.
- A **DOI minting** service, if decoupling from CrossRef via a more modern stack becomes worthwhile.
- A separate **Invenio instance alongside BHL**, not in place of it, for dataset publication workflows that don't fit the page-centric reading experience.

## Caveats

- Invenio's most recent extension capabilities may be more flexible than this note assumes — worth checking with InvenioRDM maintainers before ruling out a more ambitious integration.
- CERN's CDS (CERN Document Server) runs on Invenio in a library-shaped configuration; if anyone has built a page-level reading platform on Invenio, CDS is where to look. I suspect the answer is "no", but a yes would shift the assessment.
- The data-flow diagrams in this repo are documentation, not production audits. The BHL source code may have since evolved, and direct conversation with BHL engineers would surface constraints this note doesn't.
