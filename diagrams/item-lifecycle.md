# Lifecycle of an Internet Archive item through BHL

Follow a single digitised book as it moves through BHL — from discovery on Internet Archive through harvest, enrichment, and serving, with metadata flowing back to IA to close the loop.

Each node is a **state the item is in**; each edge is labelled with the **service that transforms it**.

```mermaid
flowchart TD
    A([Item exists on<br/>Internet Archive])

    A -->|IA Analysis Harvest<br/>polls IA OAI API| B[Metadata catalogued<br/>in IAAnalysis DB]

    B -->|IA Harvest Async<br/>transfers to download queue| C[Queued for harvest<br/>in BHLImport DB]

    C -->|IA Harvest worker<br/>fetches from IA| D[Files downloaded<br/>DJVU · OCR · scandata · MARC<br/>→ Static Files]

    D -->|IA Harvest worker<br/>stages page records| E[Pages staged<br/>in BHLImport DB]

    E -->|BHLImportServer<br/>publishes to production| F([In production<br/>BHL DB + Static Files])

    F -->|Search Index Queue Load<br/>+ Search Indexer via MQ| G[Indexed for search<br/>in Elasticsearch]

    F -->|Search Index Queue Load<br/>+ PDF Generator via MQ| H[Pre-gen PDF created<br/>in Static Files]

    F -->|Page Name Refresh<br/>runs gnfinder on OCR| I[Taxonomic names<br/>extracted per page<br/>→ BHL DB]

    F -->|DOI Processor<br/>submits to CrossRef| J[DOI minted<br/>→ BHL DB]

    G & H & I & J --> K([Item fully enriched])

    K --> L[Served to users<br/>biodiversitylibrary.org/item/id]

    L --> M[Metadata from BHL DB<br/>Page images from IA BookReader<br/>OCR text from Static Files<br/>Names from BHL DB<br/>Search from Elasticsearch]

    L -->|OAI-PMH · Public API · OpenURL| N[Available to<br/>external consumers]

    L -->|via IA collection| O[Mirrored to<br/>AWS Open Data]

    F -->|METS Upload| P[METS XML uploaded<br/>to IA S3]

    F -->|Name File Generator| Q[Name index XML uploaded<br/>to IA S3]

    P & Q -.->|metadata returns<br/>to Internet Archive| A

    classDef milestone fill:#ffeeba,stroke:#b8860b,color:#000,stroke-width:2px;
    classDef enrichment fill:#c7e9c0,stroke:#2e7d32,color:#000;
    classDef serving fill:#cfe2f3,stroke:#3d85c6,color:#000;
    class A,F,K milestone;
    class G,H,I,J enrichment;
    class L,M,N,O serving;
```

## The three milestones

The item passes through three key states (highlighted in yellow above):

1. **Exists on Internet Archive** — the starting point. The item is a digitised book hosted on IA, with DJVU page images, scandata, MARC metadata, and OCR text. BHL doesn't know about it yet.
2. **In production** — the item and its pages are now in BHL DB, with files on the shared file store. It's visible on the site but not yet fully enriched (no search entry, no names, no DOI, no PDF).
3. **Fully enriched** — search-indexed, PDF generated, per-page taxonomic names extracted, DOI minted (if applicable), and metadata uploaded back to IA. This is the steady state.

## The loop back to IA

Two services push metadata *back* to Internet Archive after the item is in production:

- **METS Upload** generates a METS XML file and uploads it to IA S3 (daily batch).
- **Name File Generator** generates a name-index XML file and uploads it to IA S3.

This closes the loop: the item started on IA, was harvested into BHL, was enriched, and then BHL's enrichments (structured metadata, taxonomic names) flow back to IA to sit alongside the original scans.

## Timing

Not everything happens at once. The stages have different latencies:

| Stage | Trigger | Typical latency after harvest |
|-------|---------|-------------------------------|
| Search indexing | MQ message (near-real-time consumer) | Seconds to minutes |
| PDF generation | MQ message (batch consumer) | Minutes to hours |
| Name extraction | Scheduled batch (Page Name Refresh) | Hours to days |
| DOI minting | Scheduled batch (DOI Processor) | Hours to days |
| METS / name-file upload to IA | Daily / scheduled batch | Up to 24 hours |

A freshly harvested item may appear on the site without names, a DOI, or a pre-generated PDF for a while. Search availability is the fastest downstream step.

## What this reveals

- **IA is the alpha and the omega.** The item starts on IA, BHL enriches it, and enrichments flow back to IA. The user's browser also talks directly to IA for page images. BHL is never fully independent of IA for any item.
- **The harvest is the critical path.** If the IA Harvest worker fails, nothing downstream happens.
- **Enrichment is loosely coupled but laggy.** Each enrichment step runs on its own schedule, not event-driven. The trade-off is operational simplicity (no complex orchestration) at the cost of eventual consistency.
- **The item lives in many places at once.** After enrichment, data about this item exists in: BHL DB, Static Files, Elasticsearch, Internet Archive (METS + name files + original scans), and eventually AWS Open Data. There is no single source of truth — each store holds a different facet.
