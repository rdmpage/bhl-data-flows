# BHL data flow — overview

High-level view of how data moves through BHL, organised by the data's journey: from external sources, through ingest, into the core hubs, out to processing and storage, and then served via the API / web tier (or pushed to external destinations).

```mermaid
flowchart LR
    subgraph External["External Sources"]
        IA[Internet Archive]
        Flickr[Flickr]
        WD[Wikidata]
        Biostor[Biostor]
        MacawSrv[Macaw Server]
    end

    subgraph Ingest["Harvesters"]
        Harvesters[IA / Flickr / Wikidata /<br/>Biostor / OAI harvesters]
    end

    subgraph Core["Core hubs"]
        PrivAPI[BHL Services<br/>Private API]
        BHLDB[(BHL DB)]
        MQ([RabbitMQ])
    end

    subgraph Processing["Batch processors"]
        Processors[Search Indexer · PDF generators<br/>OCR / Text · Name tools<br/>DOI · METS · Export · AWS Sync]
    end

    subgraph Storage["Indexes & files"]
        ES[(Elasticsearch)]
        Files[Static Files<br/>OCR · PDFs · scandata]
    end

    subgraph Serving["APIs & web tier"]
        SiteSvc[SiteServices API]
        PubAPI[Public APIs]
        Web[Public / Admin /<br/>Macaw web sites]
    end

    subgraph Outbound["External destinations"]
        AWS[AWS]
        Figshare[Figshare]
        SMTP([SMTP])
    end

    Users((Users))

    External --> Ingest
    Ingest --> PrivAPI
    Ingest --> Files
    Ingest --> MQ

    PrivAPI --> BHLDB
    BHLDB --> MQ
    MQ --> Processing
    Processing --> PrivAPI
    Processing --> ES
    Processing --> Files

    BHLDB --> SiteSvc
    BHLDB --> PubAPI
    Files --> SiteSvc
    ES --> PubAPI
    SiteSvc --> Web
    PubAPI --> Web
    Users --> Web
    Users --> PubAPI

    Files --> AWS
    Processing --> Figshare
    PrivAPI --> SMTP
    SiteSvc --> SMTP
    Processing -. "direct, errors only" .-> SMTP
```

## Architectural hubs

Two components sit at the centre of the system and are worth naming explicitly:

- **BHL Services Private API** — the write-back gateway. Almost every background job commits data to the production database through this API rather than touching the DB directly.
- **RabbitMQ** — the async backbone decoupling search indexing and PDF generation from the processes that trigger them.

## What's deliberately hidden here

- Individual harvesters, processors, and auxiliary databases (`BHLImport DB`, `IAAnalysis DB`, `bhlindex DB`). Each is shown in the relevant lifecycle sub-diagram.
- Email-sender markers. ~20 components POST to an internal `/v1/Email` endpoint; only the two API → SMTP edges (plus one direct edge from the Search Indexer) are shown here. Individual senders are marked in the sub-diagrams.
- **IIIF.** Code exists in `BHLUSWeb2/Controllers/IIIFController.cs` and `IIIFUtility/`, but IIIF is not in production use. Page-image display currently delegates to the Internet Archive BookReader.
