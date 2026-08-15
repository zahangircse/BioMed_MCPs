# Free & Open Biomedical APIs and MCP Servers
## A Complete Reference Report for Genomics, Epigenomics, Proteomics, and Medical Imaging

**Prepared for:** API vs MCP Workshop — Session 4 (Automation & Agentic AI)
**Scope:** Publicly accessible, no-cost (or free-tier) APIs and ready-made MCP servers covering RNA-seq, DNA methylation, copy number variation (CNV), proteomics, radiology imaging, and digital pathology.

---

## 1. Executive Summary

Biomedical research today is spread across dozens of specialized repositories, each with its own API, authentication model, and data format. This report catalogs the major **free, programmatically accessible sources** for five categories of biomedical data — RNA-seq, DNA methylation, CNV, proteomics, and medical imaging (radiology + pathology) — and separately catalogs **existing MCP servers** that already wrap many of these sources for direct use by an LLM agent.

The short version: for cancer genomics/multi-omics, **NCI's Genomic Data Commons (GDC)** and **cBioPortal** cover almost everything (RNA-seq, CNV, methylation, and even protein data) through one free API each. For proteomics specifically, **PRIDE**/**ProteomeXchange** are the open standard. For imaging, **The Cancer Imaging Archive (TCIA)** covers both radiology and histopathology. On the MCP side, a growing ecosystem (**BioMCP**, **MCPmed**, **gget-mcp**, **dicom-mcp**, **fhir-mcp-server**) already exposes most of this as ready-to-use tools, so in many cases no custom server needs to be written at all.

---

## 2. Methodology

Sources were identified via direct documentation review of each provider (GDC, cBioPortal, PRIDE, Ensembl, TCIA/NBIA, UniProt) plus a survey of current MCP server registries and curated GitHub lists (`awesome-genomic-skills`, `awesome-mcp`, `best-of-mcp-servers`) as of **August 2026**. Every endpoint listed below was confirmed against the provider's own API documentation. Because this is a fast-moving space, MCP server details (stars, maintenance status) should be re-checked before relying on any single project for production use.

---

## 3. Genomics & RNA-seq APIs

### 3.1 NCI Genomic Data Commons (GDC) API
- **Base URL:** `https://api.gdc.cancer.gov`
- **What it provides:** Harmonized RNA-seq gene/miRNA expression quantification, splice-junction data, and raw counts from TCGA, TARGET, and other NCI programs.
- **Auth:** None required for open-access data; a downloadable token is needed only for controlled-access files (e.g., germline variants), which require dbGaP authorization.
- **Query style:** REST, supports both GET and POST (POST recommended for large filter payloads); results in JSON, TSV, or CSV.
- **Example — get RNA-seq files for specific TCGA cases:**
  ```bash
  curl --request POST --header "Content-Type: application/json" \
    --data @payload.json 'https://api.gdc.cancer.gov/files' > response.tsv
  ```
  where `payload.json` filters on `cases.submitter_id` and `files.data_type = "Gene Expression Quantification"`.
- **Simple status check:**
  ```bash
  curl https://api.gdc.cancer.gov/status
  ```

### 3.2 Ensembl REST API
- **Base URL:** `https://rest.ensembl.org`
- **What it provides:** Gene/transcript annotation, genomic sequences, comparative genomics, variant effect prediction — across human and many other species.
- **Auth:** None.
- **Notable MCP wrapper:** `Augmented-Nature/Ensembl-MCP-Server` exposes this entire API as MCP tools (gene lookup, sequence retrieval, batch operations).

### 3.3 NCBI GEO (via E-utilities)
- **Base URL:** `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/`
- **What it provides:** Tens of thousands of public RNA-seq and microarray expression series, searchable by disease, tissue, or platform.
- **Auth:** None required; a free NCBI API key raises the rate limit from 3 to 10 requests/second.
- **Notable MCP wrapper:** referenced directly in the **MCPmed** reference implementation, which exposes GEO as one of its example MCP-enabled services.

### 3.4 EBI Expression Atlas / GTEx APIs
- Curated differential-expression and healthy-tissue baseline-expression data respectively; both free, no auth, REST-based.

---

## 4. DNA Methylation (DNAm) APIs

| Source | Coverage | Access |
|---|---|---|
| **GDC API** (`api.gdc.cancer.gov`) | TCGA methylation data (Illumina HumanMethylation450/27 arrays), harmonized via the SeSAMe pipeline | Free, open-access tier |
| **NCBI GEO** | Large volume of independently deposited methylation array series | Free, no auth |
| **ENCODE REST API** | Methylation alongside broader chromatin/epigenomic assay data | Free, no auth |

There is currently no dedicated, standalone "methylation-only" public API as prominent as PRIDE is for proteomics — DNAm data is typically accessed as one data type within a larger repository (GDC, GEO, ENCODE) rather than through a specialized methylation-specific service.

---

## 5. Copy Number Variation (CNV) APIs

### 5.1 GDC API
Same base URL as above (`api.gdc.cancer.gov`); CNV segment-level calls are available via the `ssms`/`cnv` family of endpoints, derived from SNP arrays and whole-genome sequencing.

### 5.2 cBioPortal API
- **Base URL:** `https://www.cbioportal.org/api`
- **Swagger/OpenAPI spec:** `https://www.cbioportal.org/api/v3/api-docs`
- **What it provides:** Pre-processed, analysis-ready CNV calls (e.g., "amplified," "deep deletion," or log2/linear copy-number values) already joined to the same samples' mutation, RNA-seq, methylation, and protein data — this cross-linking is what makes cBioPortal especially convenient for multi-omic demos.
- **Auth:** None for the public instance.
- **Example (Python, via the `bravado` Swagger client):**
  ```python
  from bravado.client import SwaggerClient
  cbioportal = SwaggerClient.from_url(
      'https://www.cbioportal.org/api/v3/api-docs',
      config={"validate_requests": False, "validate_responses": False, "validate_swagger_spec": False}
  )
  cna = cbioportal.Discrete_Copy_Number_Alterations.getDiscreteCopyNumbersInMolecularProfileUsingGET(
      molecularProfileId="acc_tcga_gistic", sampleListId="acc_tcga_all"
  ).result()
  ```

---

## 6. Proteomics APIs

### 6.1 PRIDE Archive REST API
- **Base URL:** `https://www.ebi.ac.uk/pride/ws/archive/v2`
- **What it provides:** Mass-spectrometry-based protein/peptide identifications and quantification, plus project/assay metadata and links to raw files.
- **Auth:** **None — fully open, no login required for any read access.**
- **Example:**
  ```bash
  curl -H "Accept: application/json" https://www.ebi.ac.uk/pride/ws/archive/projects
  ```
- **File access:** raw files are served over FTP/HTTPS at `https://ftp.pride.ebi.ac.uk/pride/data/archive/<year>/<month>/<accession>/`, or streamed via the PRIDE Archive Downloader service.

### 6.2 ProteomeXchange API
- **Base URL:** `https://proteomecentral.proteomexchange.org/cgi/GetDataset`
- **What it provides:** A single, unified metadata layer across PRIDE, MassIVE, PeptideAtlas, jPOST, and iProX — useful when you don't know in advance which underlying repository holds a given dataset.
- **Auth:** None.

### 6.3 UniProt REST API
- **Base URL:** `https://rest.uniprot.org`
- **What it provides:** Canonical protein sequences, functional annotation, domains, and cross-references to structural/pathway databases.
- **Auth:** None.

### 6.4 cBioPortal (protein layer)
As noted in Section 5.2, cBioPortal also serves RPPA and mass-spectrometry-based protein/phosphoprotein levels for the same cancer cohorts as its genomic data — a practical way to touch proteomics without leaving the same API used for CNV/RNA-seq.

---

## 7. Medical Imaging — Radiology

### 7.1 The Cancer Imaging Archive (TCIA) — NBIA REST API
- **Base URL (current, v4):** `https://services.cancerimagingarchive.net/nbia-api/services/v2` (basic search) and `/nbia-api/services` (advanced/authenticated operations)
- **What it provides:** Search and download DICOM imaging collections (CT, MRI, PET, mammography, etc.) tied to cancer research cohorts, many of which overlap with TCGA cases (enabling combined imaging + omics analysis).
- **Auth:** **No API key required for public collections** as of the current API version; a bearer token is only needed for "limited access" collections that carry an additional data use agreement.
- **Example:**
  ```bash
  curl "https://services.cancerimagingarchive.net/nbia-api/services/v2/getCollectionValues"
  curl "https://services.cancerimagingarchive.net/services/v4/TCIA/query/getPatientStudy?Collection=TCGA-GBM&PatientID=GBM-0123&format=csv"
  ```
- **Python helper package:** `tcia_utils` (community-maintained, published under an NCI-affiliated GitHub org) wraps these calls.

### 7.2 Imaging Data Commons (IDC)
- **What it provides:** A Google Cloud/NCI-hosted superset of TCIA plus additional public imaging collections, queryable via the `idc-index` Python package or directly via Google BigQuery.
- **Auth:** None for public collections.

### 7.3 NLM Open-i API
- **What it provides:** Search over biomedical images (including radiology and pathology figures) extracted from published literature and case reports.
- **Auth:** None.

### 7.4 Orthanc (self-hosted reference PACS)
- Not a public API, but an open-source DICOM server with a full REST interface — the standard local testbed for DICOM-MCP style demos, since it can be spun up in Docker with zero real patient data.

---

## 8. Medical Imaging — Pathology (Digital Histopathology / WSI)

| Source | Coverage | Access |
|---|---|---|
| **TCIA Histopathology Portal** | Whole-slide histopathology images linked to the same TCIA cancer collections (beta search/visualization interface, same NBIA API backend) | Free |
| **Digital Slide Archive** (Kitware) | Open-source WSI management platform with a REST API for slide storage, annotation, and retrieval | Free/open-source, typically self-hosted |
| **GDC** | Diagnostic and tissue slide images directly tied to TCGA molecular data — useful for pairing pathology images with the RNA-seq/CNV/methylation data from the same patient | Free (open-access tier) |

---

## 9. The MCP Server Ecosystem for Biomedical Data

MCP servers matter here because several of the APIs above already have a maintained MCP wrapper — meaning no custom integration code is needed to make them available to an LLM agent.

| MCP Server | What it covers | Repository |
|---|---|---|
| **BioMCP** | PubMed literature, ClinicalTrials.gov, MyVariant.info genetic variants, and cBioPortal-style local cohort/study analysis (survival, co-occurrence, comparisons) | `genomoncology/biomcp` |
| **MCPmed** | Reference MCP implementations for GEO, STRING, UCSC Cell Browser, and PLSDB, plus a cookiecutter template for converting other bioinformatics web services to MCP | published as a call paper in *Briefings in Bioinformatics* |
| **gget-mcp** | Wraps the `gget` Python library, itself a unified query layer over Ensembl, UniProt, NCBI, PDB, and other databases | `longevity-genie/gget-mcp` |
| **Ensembl-MCP-Server** | Full Ensembl REST API — gene lookup, transcripts, sequences, comparative genomics, batch operations | `Augmented-Nature/Ensembl-MCP-Server` |
| **ChatSpatial** | Spatial transcriptomics analysis via natural language, built on Scanpy/Squidpy | `cafferychen777/ChatSpatial` |
| **dicom-mcp** | Query, read, and move data on DICOM servers (PACS/VNA); ships with an Orthanc-based local test setup | `ChristianHinge/dicom-mcp` |
| **fhir-mcp-server** | FHIR (Fast Healthcare Interoperability Resources) API access for clinical/EHR-style data | `wso2/fhir-mcp-server`, `the-momentum/fhir-mcp-server` |
| **omop_mcp** | OMOP common data model, widely used for observational clinical research data | `OHNLP/omop_mcp` |

### Curated lists for ongoing discovery
- `github.com/GoekeLab/awesome-genomic-skills` — actively maintained list of genomics/bioinformatics MCPs and agent skills, including benchmarks (e.g., *BioAgent Bench*, 10 end-to-end pipeline tasks spanning RNA-seq, variant calling, and single-cell workflows)
- `github.com/abordage/awesome-mcp` — general MCP server directory with a dedicated Biology/Medicine/Bioinformatics section
- `github.com/tolkonepiu/best-of-mcp-servers` — a ranked, weekly-updated list

---

## 10. Comparison Matrix

| Data Type | Best single free API | Alternative | Requires auth? |
|---|---|---|---|
| RNA-seq | GDC | GEO, Ensembl | No (open tier) |
| DNA methylation | GDC | GEO, ENCODE | No (open tier) |
| CNV | cBioPortal (pre-processed) | GDC (raw) | No |
| Proteomics | PRIDE | ProteomeXchange, UniProt | No |
| Multi-omic (one API, many types) | cBioPortal | GDC | No |
| Radiology imaging | TCIA (NBIA API) | Imaging Data Commons | No (public collections) |
| Pathology / WSI | TCIA Histopathology Portal | Digital Slide Archive (self-hosted) | No |

---

## 11. Access, Licensing & Rate-Limit Notes

- **"Free" ≠ unrestricted.** Every source above is free to query and download programmatically, but each carries its own terms of use regarding redistribution, publication citation requirements, and (for TCIA) sometimes a per-collection data use agreement for "limited access" sets.
- **GDC tiering matters.** Metadata and derived "open" molecular data (RNA-seq counts, CNV calls, methylation beta values) are open. Raw sequencing reads and germline variant calls are "controlled access" and require dbGaP authorization — this is the one place in this report where a real approval process, not just an API key, is involved.
- **NCBI rate limits** are lifted from 3 to 10 requests/second with a free API key — worth obtaining for any GEO-heavy workflow.
- **TCIA no longer requires an API key** for public collections as of the current API version, simplifying demos considerably compared to older tutorials that still reference API keys.
- **Self-hosted vs. public:** Orthanc and Digital Slide Archive are open-source servers you run yourself, not hosted public APIs — ideal for imaging demos since they let you generate a fully synthetic local dataset with no real patient data anywhere in the pipeline.

---

## 12. Recommendations for the Workshop

1. **For a multi-omic MCP demo:** build on **cBioPortal's API** — one free, no-auth API surface returns RNA-seq, CNV, and protein data for the same cancer cohort, which maps cleanly onto the "multiple tools, one client" pattern already used in the weather/finance examples.
2. **For a proteomics-specific demo:** **PRIDE** is the simplest possible integration — genuinely zero-auth, and a single `GET /projects` call returns real, publication-linked datasets.
3. **For a medical imaging demo:** pair **dicom-mcp** with a local **Orthanc** instance — this avoids any real patient data while still demonstrating a realistic PACS query/retrieve workflow, and is already built as an MCP server (no custom wrapper needed).
4. **For literature + variant context alongside any of the above:** **BioMCP** is worth adding as a second MCP server in an orchestrator-style demo (mirroring the Coordinator/specialist-agent pattern already built), since it independently covers PubMed, ClinicalTrials.gov, and variant pathogenicity lookups.

---

## 13. Sources

- GDC API documentation — docs.gdc.cancer.gov
- cBioPortal API & Swagger docs — cbioportal.org/api, docs.cbioportal.org
- PRIDE Archive API guide — ebi.ac.uk/pride/ws/archive/v2/docs
- TCIA / NBIA REST API guides (v4) — wiki.cancerimagingarchive.net
- Ensembl REST API — rest.ensembl.org
- UniProt REST API — rest.uniprot.org
- `GoekeLab/awesome-genomic-skills` — GitHub
- `abordage/awesome-mcp` — GitHub
- `tolkonepiu/best-of-mcp-servers` — GitHub
- `genomoncology/biomcp`, `longevity-genie/gget-mcp`, `Augmented-Nature/Ensembl-MCP-Server`, `ChristianHinge/dicom-mcp`, `wso2/fhir-mcp-server`, `OHNLP/omop_mcp` — GitHub

*Report current as of August 2026. MCP server maintenance status and API endpoint paths should be re-verified periodically, as this ecosystem is evolving quickly.*
