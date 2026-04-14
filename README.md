# BHL Internal Data Flows — Analysis

> Source: `BHL Internal Data Flows.drawio.svg` (revised March 23, 2026)  
> Codebase: `bhl-us` (C# / ASP.NET / SQL Server)

---

## Overview

The BHL system is built around two central hubs:

- **BHL Services Private API** — the write-back gateway through which almost every background job commits data to the production database
- **Message Queue (RabbitMQ)** — the async backbone decoupling search indexing and PDF generation from the processes that trigger them

The diagram covers approximately 30 components across four physical tiers: Web Server, App Server, Database Server, and File/Search Server.

---

## Infrastructure & Storage

| Component | Technology | Role |
|-----------|-----------|------|
| BHL DB | SQL Server | Central production database |
| BHLImport DB | SQL Server | Import staging database |
| IAAnalysis DB | SQL Server | Internet Archive analysis database |
| bhlindex DB | PostgreSQL | Name/taxon indexing database |
| Search Engine | Elasticsearch | Full-text and faceted search |
| Static Files | File share | OCR text, scandata, images, PDFs |
| Message Queue | RabbitMQ | Async messaging between services |
| SMTP Service | — | Outbound email |

---

## Subsystems & Data Flows

### Internet Archive Import

The most deeply coupled subsystem — touches five storage systems.

```
IAAnalysis DB ──────────────────────────────► IA Harvest
IA Analysis Harvest ──► IAAnalysis DB            │
                                                  ├──► BHLImport DB        (IA item metadata)
                                                  ├──► BHL DB
                                                  ├──► Static Files        (Scandata/DJVU/MARC/OCR)
                                                  ├──► Message Queue       (index + PDF messages)
                                                  └──► BHL Services API
```

Key code: `IAHarvest/`, `IAHarvestAsync/`, `IAAnalysisHarvest/`, `BHLImportServer/`  
Database: `Database-BHLImport/` (~120 stored procedures)

---

### Search Indexing

```
BHL DB ──► Search Queue Load ──► Message Queue ──► Search Indexer ──► Elasticsearch
OCR Refresh ──────────────────► Message Queue
Text Import Processor ────────► Message Queue
```

Key code: `BHLSearchIndexer/`, `SearchElastic/`

---

### OCR & Text

```
Static Files ──► SiteServices Private API     (served to web)
Text Import Processor ──► OCR Files
OCR Refresh ──────────► OCR Files
OCR Files ─┬──► OCR Export to Figshare
           └──► Name Refresh
```

Key code: `BHLSearchIndexer/DataAccess.cs` (OCR retrieval), `IAHarvest/` (DJVU extraction)

---

### PDF Generation

```
Message Queue ──────────────────► Pre-Gen PDF Generator ──► Pre-gen PDF Files
Public APIs ────────────────────► Pre-Gen PDF Generator
DJVU Files ─────────────────────► Custom PDF Generator ──► Custom PDF Files
Custom PDF Generator ───────────► BHL Services API        (data + email)
```

---

### Name / Taxon Processing

```
BHL DB ──► Name File Generator ──► BHL Services API
OCR Files ──────────────────────► Name Refresh ──► BHL Services API
bhlindex (GN tool) ─────────────► bhlindex DB
```

Key code: `BHLSearchIndexer/` (name indexing), `IIIFUtility/NameManifest.cs` (name display)

---

### External Harvests

All follow the same pattern: fetch from external source → write via BHL Services API.

| Harvester | Source | Destinations |
|-----------|--------|-------------|
| Flickr Tag Harvest | Flickr API | BHLImport DB, BHL Services API |
| Flickr Thumb Grab | Flickr API | BHL Services API (data + email) |
| Wikidata Harvest | Wikidata | BHL DB, BHL Services API |
| Biostor Harvest | Biostor | BHLImport DB, BHL DB, BHL Services API |
| Macaw OAI Harvest | Macaw Server | Macaw Web Site |

---

### APIs & Web Tier

```
BHL DB ──► SiteServices Private API ─┬──► Public Web Site  (files + email)
                                     ├──► Admin Web Site   (files)
                                     └──► Public APIs ─────► Public Web Site
                                                       └──── Macaw Web Site
BHL DB ──► Public APIs ──► Public Web Site
BHL Services API ──► BHL DB
```

---

### Other Processors

| Processor | Inputs | Outputs |
|-----------|--------|---------|
| Export Processor | BHL Services API | Public Web Site (export files) |
| METS Upload | — | BHL Services API (data + email) |
| DOI Processor | — | BHL Services API (data + email) |
| AWS Sync | Static Files, Message Queue | AWS |
| IIIF | BHL DB, Static Files (scandata) | IIIF manifests (Presentation API 2.x) |

Key code: `BHLUSWeb2/Controllers/IIIFController.cs`, `IIIFUtility/`

---

## Mermaid Overview Diagram

```mermaid
flowchart LR
    subgraph Databases
        BHLDB[(BHL DB)]
        ImportDB[(BHLImport DB)]
        IAAnalDB[(IAAnalysis DB)]
        bhlindexDB[(bhlindex DB)]
        ES[(Elasticsearch)]
    end

    subgraph Files
        StaticFiles[Static Files]
        OCRFiles[OCR Files]
        PreGenPDF[Pre-gen PDF Files]
        CustomPDF[Custom PDF Files]
    end

    MQ([RabbitMQ])
    SMTP([SMTP Service])

    subgraph IA Import
        IAAnalHarvest[IA Analysis Harvest]
        IAHarvest[IA Harvest]
    end

    subgraph Harvests
        FlickrTag[Flickr Tag Harvest]
        FlickrThumb[Flickr Thumb Grab]
        Wikidata[Wikidata Harvest]
        Biostor[Biostor Harvest]
        MacawOAI[Macaw OAI Harvest]
    end

    subgraph Processing
        SearchIndexer[Search Indexer]
        SearchQueueLoad[Search Queue Load]
        TextImport[Text Import Processor]
        OCRRefresh[OCR Refresh]
        NameRefresh[Name Refresh]
        NameFileGen[Name File Generator]
        bhlindex[bhlindex GN tool]
        PreGenPDFGen[Pre-Gen PDF Generator]
        CustomPDFGen[Custom PDF Generator]
        ExportProc[Export Processor]
        DOIProc[DOI Processor]
        METSUpload[METS Upload]
        OCRExport[OCR Export to Figshare]
        AWSSync[AWS Sync]
    end

    subgraph APIs
        BHLSvcAPI[BHL Services Private API]
        SiteSvcAPI[SiteServices Private API]
        PublicAPI[Public APIs]
    end

    subgraph Web
        PublicWeb[Public Web Site]
        AdminWeb[Admin Web Site]
        MacawWeb[Macaw Web Site]
    end

    IAAnalHarvest --> IAAnalDB
    IAAnalDB --> IAHarvest
    IAHarvest --> ImportDB
    IAHarvest --> BHLDB
    IAHarvest --> StaticFiles
    IAHarvest --> MQ
    IAHarvest --> BHLSvcAPI

    BHLDB --> SearchQueueLoad
    SearchQueueLoad --> MQ
    TextImport --> MQ
    OCRRefresh --> MQ
    MQ --> SearchIndexer
    SearchIndexer --> ES

    TextImport --> OCRFiles
    OCRRefresh --> OCRFiles
    OCRFiles --> OCRExport
    OCRFiles --> NameRefresh
    NameRefresh --> BHLSvcAPI
    bhlindex --> bhlindexDB

    MQ --> PreGenPDFGen
    PreGenPDFGen --> PreGenPDF
    CustomPDFGen --> CustomPDF
    CustomPDFGen --> BHLSvcAPI

    FlickrTag --> ImportDB
    FlickrTag --> BHLSvcAPI
    FlickrThumb --> BHLSvcAPI
    Wikidata --> BHLDB
    Wikidata --> BHLSvcAPI
    Biostor --> ImportDB
    Biostor --> BHLDB
    Biostor --> BHLSvcAPI
    MacawOAI --> MacawWeb

    BHLSvcAPI --> BHLDB
    BHLDB --> SiteSvcAPI
    StaticFiles --> SiteSvcAPI
    SiteSvcAPI --> PublicWeb
    SiteSvcAPI --> AdminWeb
    SiteSvcAPI --> PublicAPI
    BHLDB --> PublicAPI
    PublicAPI --> PublicWeb
    PublicAPI --> MacawWeb

    ExportProc --> BHLSvcAPI
    ExportProc --> PublicWeb
    DOIProc --> BHLSvcAPI
    METSUpload --> BHLSvcAPI
    StaticFiles --> AWSSync
    MQ --> AWSSync