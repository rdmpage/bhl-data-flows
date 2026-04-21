# BHL data flows

Analysis and diagrams of how data moves through the Biodiversity Heritage Library (BHL) system (`bhl-us` codebase: C# / ASP.NET / SQL Server).

Goals:

1. **Understand the data flow** — where data comes from, how it's transformed, where it ends up.
2. **Support future modularity** — identify integration seams so BHL components can be swapped in and out as the system evolves.

## Diagrams

- [Overview](diagrams/overview.md) — the whole system in one picture.
- Lifecycle sub-diagrams:
  - [Ingest](diagrams/ingest.md) — external sources, harvesters, staging databases.
  - [Process](diagrams/process.md) — search indexing, PDF generation, OCR / text, name tools, exports.
  - [Serve](diagrams/serve.md) — APIs, web sites, and the protocol endpoints (OAI-PMH, OpenURL, IIIF, citations).
- [Integration seams](diagrams/integration-seams.md) — modularity view: who touches each major coupling point (Private API, RabbitMQ, BHL DB, Static Files, Elasticsearch, IA, SMTP) and what swapping it would cost.
- Object lifecycles:
  - [IA item](diagrams/item-lifecycle.md) — how a single IA item travels from discovery to serving, with metadata flowing back to IA.
  - [PDF](diagrams/pdf-lifecycle.md) — how a PDF is requested, generated from DJVU, delivered, and eventually expired.

The original source diagram is preserved as `diagrams/BHL Internal Data Flows.drawio.svg` (and a PNG export) for reference.

## Services

- [gnfinder](services/gnfinder.md) — the Global Names taxonomic name-finding tool used by BHL's Page Name Refresh pipeline.

## Tasks

Automated scheduled tasks extracted from [BHL's Confluence calendar](https://biodiversitylibrary.atlassian.net/wiki/spaces/BHL/calendars/76eed66b-69a3-4361-84f8-180a9902bbb9) (raw data in `tasks.csv`).

### Harvester / processor tasks

| Task | Frequency | Diagram component |
|------|-----------|-------------------|
| IA Analysis Harvest | Sat | [Ingest](diagrams/ingest.md) — discovery stage of IA pipeline |
| IA Harvest | Sat | [Ingest](diagrams/ingest.md) — per-item worker |
| Biostor Harvest | Daily | [Ingest](diagrams/ingest.md) |
| OAI Harvest | Daily | [Ingest](diagrams/ingest.md) |
| Flickr Thumb Grab | Daily | [Ingest](diagrams/ingest.md) |
| Page Name Refresh | Daily | [Process](diagrams/process.md) — taxonomic name extraction |
| DOI Service (Submit) | Daily | [Process](diagrams/process.md) — DOI minting |
| DOI Service (Verify) | Daily | [Process](diagrams/process.md) — DOI verification |
| METS Upload | Daily | [Process](diagrams/process.md) — uploads METS XML to IA S3 |
| Name File Generator | Daily | [Process](diagrams/process.md) — uploads name XML to IA S3 |
| Export Processor | Fri | [Process](diagrams/process.md) |
| IAID Export | Wed | — |

### Database maintenance tasks

| Task | Frequency |
|------|-----------|
| Publish To Production | Daily |
| Remove Inactive NamePages | Daily |
| Archive Audit Data | Daily |
| Update Monthly Stats | Daily |
| Entity Count Refresh | Daily |
| RequestLog Rollup | Daily |
| Reset Pagination Locks | Daily |
| System DB Maintenance | Daily |
| IAAnalysis Rollup | Wed |
| Log Server Perf Metrics | Wed |
| DB Backups (BHL / BHLImport / BHLAuditArchive) | Daily |
| DB Backups (BHLQA) | Tue, Thu, Sun |
| DB Backups (IAAnalysis) | Sat |
| DB Backups (BHLImportQA / BHLAuditArchiveQA / IAAnalysisQA) | Sun |
| CHECKDB Limited (BHL / BHLQA / BHLImport / BHLAuditArchive) | Mon–Wed, Fri–Sat |
| CHECKDB Full (BHL / BHLQA / BHLImport / BHLAuditArchive) | Thu, Sun |
| CHECKDB Limited (BHLImportQA / BHLAuditArchiveQA / IAAnalysisQA) | Mon–Sat |
| CHECKDB Full (BHLImportQA / BHLAuditArchiveQA / IAAnalysisQA) | Sun |
| CHECKDB Limited (IAAnalysis) | Mon–Fri, Sun |
| CHECKDB Full (IAAnalysis) | Sat |
| Index Optimization (Prod) | Sat |
| Index Optimization (QA) | Sun |

### Not in the schedule (continuous services or on-demand)

Search Indexer, PDF Generator, Search Index Queue Load, Flickr Tag Harvest, Wikidata Harvest, OCR Refresh, Text Import Processor.

## Notes

- [Exploratory analysis](notes/analysis.md) — working notes from the initial walk-through of the codebase: infrastructure inventory, subsystem narratives, code pointers, and clarifications (e.g. what "email" edges really mean, and the current status of IIIF).
- [InvenioRDM feasibility](notes/invenio-rdm-feasibility.md) — assessment of whether BHL could move to InvenioRDM as its platform, based on the data flows in this repo.
