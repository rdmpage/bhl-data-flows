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

## Notes

- [Exploratory analysis](notes/analysis.md) — working notes from the initial walk-through of the codebase: infrastructure inventory, subsystem narratives, code pointers, and clarifications (e.g. what "email" edges really mean, and the current status of IIIF).
- [InvenioRDM feasibility](notes/invenio-rdm-feasibility.md) — assessment of whether BHL could move to InvenioRDM as its platform, based on the data flows in this repo.
