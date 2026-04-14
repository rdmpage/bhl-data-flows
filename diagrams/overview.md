# BHL data flow — overview

High-level view of how data moves through BHL, organised top-down: external sources feed dedicated harvesters, which push into the core hubs; core data is then processed, indexed, stored, and served via the API / web tier (or pushed to external destinations).

Nodes are colour-coded by role: external sources (yellow), harvesters (green), external destinations (lavender). External sources are drawn independently — they are unrelated systems with distinct harvest pipelines. Audience is split into **public users** (who reach the Public Web Site and Public APIs) and **BHL members** (partner-institution staff who access the Admin Web Site and the Macaw authoring tool).

```mermaid
flowchart TD
    %% --- External sources (each independent) ---
    IA[Internet Archive]
    Flickr[Flickr]
    WD[Wikidata]
    Biostor[Biostor]

    %% --- Harvesters, one per source/mode ---
    IAAnal[IA Analysis Harvest]
    IAH[IA Harvest]
    FlickrTag[Flickr Tag Harvest]
    FlickrThumb[Flickr Thumb Grab]
    WDH[Wikidata Harvest]
    BiostorH[Biostor Harvest]

    subgraph Core["Core hubs"]
        PrivAPI[BHL Services<br/>Private API]
        BHLDB[(BHL DB)]
        ImportDB[(BHLImport DB)]
        MQ([RabbitMQ])
        SMTP([SMTP])
    end

    subgraph Processing["Batch processors"]
        Processors[Search Indexer · PDF generators<br/>OCR / Text · Name tools · bhlindex<br/>DOI · METS · Export · AWS Sync]
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
    end

    %% --- Standalone authoring tool (not connected to BHL internals) ---
    Scans[Image scans]
    Macaw[Macaw<br/>authoring tool]

    BHLMembers((BHL members))

    subgraph Outbound["External destinations"]
        AWS[AWS Open Data<br/>mirrored from IA]
        Figshare[Figshare<br/>Smithsonian instance]
    end

    PublicUsers((Public users))

    %% --- Source -> harvester ---
    IA --> IAAnal
    IA --> IAH
    Flickr --> FlickrTag
    Flickr --> FlickrThumb
    WD --> WDH
    Biostor --> BiostorH

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

    %% --- Core internals ---
    ImportDB --> PrivAPI
    PrivAPI --> BHLDB
    BHLDB --> MQ
    MQ --> Processing
    BHLDB --> Processing
    Files --> Processing
    Processing --> PrivAPI
    Processing --> ES
    Processing --> Files
    Processing --> BhlIndexDB

    %% --- Serving ---
    BHLDB --> SiteSvc
    BHLDB --> PubAPI
    Files --> SiteSvc
    ES --> PubWeb
    SiteSvc --> PubWeb
    SiteSvc --> AdminWeb
    PubAPI --> PubWeb

    %% --- Audience (public users receive data; BHL members contribute) ---
    PubWeb --> PublicUsers
    PubAPI --> PublicUsers
    BHLMembers --> AdminWeb
    BHLMembers --> Macaw

    %% --- Macaw: separate authoring tool. No BHL integration; content
    %% reaches BHL only via IA Harvest after Macaw uploads to IA. ---
    Scans --> Macaw
    Macaw -- "image files<br/>+ metadata" --> IA

    %% --- Outbound / side-effects ---
    %% AWS Open Data is mirrored from IA, not pushed from BHL's Static Files
    IA --> AWS
    Files --> Figshare
    PrivAPI --> SMTP
    SiteSvc --> SMTP
    Processing -. "direct, errors only" .-> SMTP

    %% --- Styling ---
    classDef source fill:#fff2b3,stroke:#b8860b,color:#000;
    classDef harvester fill:#c7e9c0,stroke:#2e7d32,color:#000;
    classDef destination fill:#d9d2e9,stroke:#674ea7,color:#000;
    class IA,Flickr,WD,Biostor source;
    class IAAnal,IAH,FlickrTag,FlickrThumb,WDH,BiostorH harvester;
    class AWS,Figshare destination;
```

## Architectural hubs

Three components sit at the centre of the system and are worth naming explicitly:

- **BHL Services Private API** — the write-back gateway. Almost every background job commits data to the production database through this API rather than touching the DB directly.
- **RabbitMQ** — the async backbone decoupling search indexing and PDF generation from the processes that trigger them.
- **BHL DB** — the central production database that everything else orbits.

**BHLImport DB** acts as a staging layer: harvesters write raw / partial records here before the Private API promotes them into BHL DB. **bhlindex** (the Global Names tool) is grouped with the other batch processors on the diagram, but writes to its own PostgreSQL `bhlindex DB` — and **nothing in BHL reads that database back**. Treat it as a parallel name-index data product built from BHL content, not as a name source for the BHL site. Per-page names shown on biodiversitylibrary.org come from **BHL DB** instead, populated by `Page Name Refresh` using a different Global Names library (`gnfinder`) on OCR text.

## Macaw (standalone authoring tool)

Macaw (`/Users/rpage/Sites/macaw-book-metadata-tool`; see `diagrams/macaw.png`) is a PHP/CodeIgniter tool that partner institutions use to prepare page-level metadata for scanned books. **It does not interact with any BHL component.** A member feeds image scans and title metadata into Macaw, fills in item- and page-level metadata, and the export plugin uploads the resulting package (`_scandata.xml`, `_marc.xml`, `_jp2.zip`, `_bhlcreators.xml`) to Internet Archive via S3 using each partner's own IA credentials. Content reaches BHL only indirectly, via the standard IA Harvest pipeline — so Internet Archive appears on the diagram as both a source (for BHL) and a destination (for Macaw).

(The earlier source diagram's "Macaw Server" / "Macaw OAI Harvest" nodes misrepresented the flow: Macaw has no OAI-PMH endpoint — it consumes OAI feeds for import but does not expose one — and has no direct link to BHL's Public APIs, database, or services.)

## Harvest paths at a glance

| Source | Harvester(s) | Destination(s) |
|--------|-------------|---------------|
| Internet Archive | IA Analysis Harvest → IA Harvest | BHLImport DB, Static Files, RabbitMQ, BHL Services API |
| Flickr | Flickr Tag Harvest | BHLImport DB, BHL Services API |
| Flickr | Flickr Thumb Grab | BHL Services API |
| Wikidata | Wikidata Harvest | BHL Services API |
| Biostor | Biostor Harvest | BHLImport DB, BHL Services API |

Macaw-authored content reaches BHL indirectly via IA (see above).

## What's deliberately hidden here

- Individual batch processors (and `IAAnalysis DB`, which is specific to the IA ingest pipeline). Each is shown in the relevant lifecycle sub-diagram.
- Email-sender markers. ~20 components POST to an internal `/v1/Email` endpoint; only the two API → SMTP edges (plus one direct edge from the Search Indexer) are shown here. Individual senders are marked in the sub-diagrams.
- **IIIF.** Code exists in `BHLUSWeb2/Controllers/IIIFController.cs` and `IIIFUtility/`, but IIIF is not in production use. Page-image display currently delegates to the Internet Archive BookReader.
