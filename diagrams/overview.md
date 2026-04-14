# BHL data flow — overview

High-level view of how data moves through BHL, organised top-down: external sources feed dedicated harvesters, which push into the core hubs; core data is then processed, indexed, stored, and served via the API / web tier (or pushed to external destinations).

External sources are drawn independently — they are unrelated systems with distinct harvest pipelines — and colour-coded along with their harvesters to make the ingest paths legible at a glance.

```mermaid
flowchart TD
    %% --- External sources (each independent) ---
    IA[Internet Archive]
    Flickr[Flickr]
    WD[Wikidata]
    Biostor[Biostor]
    MacawSrv[Macaw Server]

    %% --- Harvesters, one per source/mode ---
    IAAnal[IA Analysis Harvest]
    IAH[IA Harvest]
    FlickrTag[Flickr Tag Harvest]
    FlickrThumb[Flickr Thumb Grab]
    WDH[Wikidata Harvest]
    BiostorH[Biostor Harvest]
    MacawOAI[Macaw OAI Harvest]

    subgraph Core["Core hubs"]
        PrivAPI[BHL Services<br/>Private API]
        BHLDB[(BHL DB)]
        ImportDB[(BHLImport DB)]
        MQ([RabbitMQ])
        SMTP([SMTP])
    end

    subgraph Processing["Batch processors"]
        Processors[Search Indexer · PDF generators<br/>OCR / Text · Name tools<br/>DOI · METS · Export · AWS Sync]
        BhlIndexTool[bhlindex<br/>taxonomic name indexer]
    end

    subgraph Storage["Indexes & files"]
        ES[(Elasticsearch)]
        BhlIndexDB[(bhlindex DB)]
        Files[Static Files<br/>OCR · PDFs · scandata]
    end

    subgraph Serving["APIs & web tier"]
        SiteSvc[SiteServices API]
        PubAPI[Public APIs]
        PubWeb[Public Web Site]
        AdminWeb[Admin Web Site]
        MacawWeb[Macaw Web Site]
    end

    AWS[AWS]
    Figshare[Figshare]
    Users((Users))

    %% --- Source -> harvester ---
    IA --> IAAnal
    IA --> IAH
    Flickr --> FlickrTag
    Flickr --> FlickrThumb
    WD --> WDH
    Biostor --> BiostorH
    MacawSrv --> MacawOAI

    %% --- Harvester -> destinations ---
    IAAnal --> IAH
    IAH --> ImportDB
    IAH --> PrivAPI
    IAH --> Files
    IAH --> MQ
    FlickrTag --> ImportDB
    FlickrTag --> PrivAPI
    FlickrThumb --> PrivAPI
    WDH --> PrivAPI
    BiostorH --> ImportDB
    BiostorH --> PrivAPI
    MacawOAI --> MacawWeb

    %% --- Core internals ---
    ImportDB --> PrivAPI
    PrivAPI --> BHLDB
    BHLDB --> MQ
    MQ --> Processing
    Processing --> PrivAPI
    Processing --> ES
    Processing --> Files
    BHLDB --> BhlIndexTool
    Files --> BhlIndexTool
    BhlIndexTool --> BhlIndexDB

    %% --- Serving ---
    BHLDB --> SiteSvc
    BHLDB --> PubAPI
    Files --> SiteSvc
    ES --> PubAPI
    BhlIndexDB --> PubAPI
    SiteSvc --> PubWeb
    SiteSvc --> AdminWeb
    PubAPI --> PubWeb
    PubAPI --> MacawWeb
    Users --> PubWeb
    Users --> AdminWeb
    Users --> PubAPI

    %% --- Outbound / side-effects ---
    Files --> AWS
    Processing --> Figshare
    PrivAPI --> SMTP
    SiteSvc --> SMTP
    Processing -. "direct, errors only" .-> SMTP

    %% --- Styling ---
    classDef source fill:#fff2b3,stroke:#b8860b,color:#000;
    classDef harvester fill:#c7e9c0,stroke:#2e7d32,color:#000;
    class IA,Flickr,WD,Biostor,MacawSrv source;
    class IAAnal,IAH,FlickrTag,FlickrThumb,WDH,BiostorH,MacawOAI harvester;
```

## Architectural hubs

Three components sit at the centre of the system and are worth naming explicitly:

- **BHL Services Private API** — the write-back gateway. Almost every background job commits data to the production database through this API rather than touching the DB directly.
- **RabbitMQ** — the async backbone decoupling search indexing and PDF generation from the processes that trigger them.
- **BHL DB** — the central production database that everything else orbits.

**BHLImport DB** acts as a staging layer: harvesters write raw / partial records here before the Private API promotes them into BHL DB. **bhlindex** is a separate subsystem (PostgreSQL, running the Global Names tool) for taxonomic name indexing — it reads from BHL DB and Static Files but keeps its output in its own database rather than feeding back through the Private API.

## Harvest paths at a glance

| Source | Harvester(s) | Destination(s) |
|--------|-------------|---------------|
| Internet Archive | IA Analysis Harvest → IA Harvest | BHLImport DB, Static Files, RabbitMQ, BHL Services API |
| Flickr | Flickr Tag Harvest | BHLImport DB, BHL Services API |
| Flickr | Flickr Thumb Grab | BHL Services API |
| Wikidata | Wikidata Harvest | BHL Services API |
| Biostor | Biostor Harvest | BHLImport DB, BHL Services API |
| Macaw Server | Macaw OAI Harvest | Macaw Web Site (BHL-hosted archive) |

## What's deliberately hidden here

- Individual batch processors (and `IAAnalysis DB`, which is specific to the IA ingest pipeline). Each is shown in the relevant lifecycle sub-diagram.
- Email-sender markers. ~20 components POST to an internal `/v1/Email` endpoint; only the two API → SMTP edges (plus one direct edge from the Search Indexer) are shown here. Individual senders are marked in the sub-diagrams.
- **IIIF.** Code exists in `BHLUSWeb2/Controllers/IIIFController.cs` and `IIIFUtility/`, but IIIF is not in production use. Page-image display currently delegates to the Internet Archive BookReader.
