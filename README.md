# 🌍 Earthquake Real-Time Analytics Pipeline

**End-to-end data engineering pipeline built on Microsoft Fabric** — ingests live earthquake data from the USGS API, processes it through a medallion architecture (Bronze → Silver → Gold), enriches it with geolocation data, and delivers interactive analytics through Power BI.

![Microsoft Fabric](https://img.shields.io/badge/Microsoft_Fabric-Lakehouse-blue)
![PySpark](https://img.shields.io/badge/PySpark-3.x-orange)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-Storage-green)
![Power BI](https://img.shields.io/badge/Power_BI-Dashboard-yellow)

---

## Architecture

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   USGS API   │────▶│   Bronze Layer   │────▶│   Silver Layer   │────▶│    Gold Layer     │
│  (GeoJSON)   │     │  Raw JSON files  │     │  Structured &    │     │  Enriched with    │
│  Last 30 days│     │  in Lakehouse    │     │  typed Delta     │     │  country code &   │
└──────────────┘     │  Files/          │     │  table           │     │  significance     │
                     └──────────────────┘     └──────────────────┘     │  classification   │
                                                                       └────────┬─────────┘
                                                                                │
                     ┌──────────────────────────────────────────────────────────┘
                     ▼
              ┌──────────────────┐     ┌──────────────────┐
              │  Semantic Model  │────▶│   Power BI       │
              │  (Auto-generated)│     │   Dashboard      │
              └──────────────────┘     └──────────────────┘

              ╔══════════════════════════════════════════════╗
              ║  Fabric Data Pipeline — Orchestrates all     ║
              ║  notebook executions in sequence             ║
              ╚══════════════════════════════════════════════╝
```

---

## Tech Stack

| Component | Technology |
|---|---|
| **Cloud Platform** | Microsoft Fabric (F2 capacity on Azure) |
| **Data Source** | USGS Earthquake Hazards Program — real-time GeoJSON REST API |
| **Storage** | Fabric Lakehouse (`earthquakes_lakehouse`) with Delta Lake tables |
| **Processing** | PySpark (Synapse PySpark kernel) |
| **Enrichment** | `reverse_geocoder` Python library (offline reverse geocoding) |
| **Orchestration** | Fabric Data Pipeline |
| **Visualization** | Power BI (Worldwide Earthquake Events report) |
| **Version Control** | GitHub (Fabric Git integration) |

---

## Data Flow — Layer by Layer

### Bronze Layer — Raw Ingestion
- Calls the USGS API for the past 30 days of earthquake events in GeoJSON format
- Extracts the `features` array from the API response
- Saves raw JSON to Lakehouse Files: `Files/{date}_earthquake_data.json`
- No transformations at this stage — preserving the source data as-is

### Silver Layer — Cleaning & Structuring
- Reads the raw JSON using `spark.read.option("multiline", "true").json()`
- Flattens nested GeoJSON structure:
  - `geometry.coordinates[0]` → `longitude`
  - `geometry.coordinates[1]` → `latitude`
  - `geometry.coordinates[2]` → `elevation`
  - `properties.title` → `title`
  - `properties.place` → `place_description`
  - `properties.sig` → `sig` (significance score)
  - `properties.mag` → `mag` (magnitude)
  - `properties.magType` → `magType`
  - `properties.time` → `time` (converted from epoch ms to timestamp)
  - `properties.updated` → `updated` (converted from epoch ms to timestamp)
- Writes to Delta table: `earthquake_events_silver`

### Gold Layer — Enrichment & Classification
- Reads from `earthquake_events_silver`
- **Reverse Geocoding**: Adds `country_code` column using a PySpark UDF wrapping `reverse_geocoder` — maps lat/lon to ISO country codes
- **Significance Classification**: Categorizes earthquakes into risk tiers:
  - `sig < 100` → **Low**
  - `100 ≤ sig < 500` → **Moderate**
  - `sig ≥ 500` → **High**
- Writes to Delta table: `earthquake_events_gold`

### Power BI — Interactive Dashboard
- Connected to the Gold layer through a Fabric Semantic Model
- Report: **Worldwide Earthquake Events**
- Visualizes earthquake distribution, magnitude trends, and regional risk analysis

---

## Project Structure

```
Earthquake-Analysis-Project/
│
├── Checking Data Using RESTAPI Notebook.Notebook/
│   └── notebook-content.py          # Exploratory notebook — API testing
│
├── Earthquake Events API Data to Bronze Layer Processing.Notebook/
│   └── notebook-content.py          # Bronze: USGS API → raw JSON files
│
├── Earthquake Events API Data to Silver Layer Processing.Notebook/
│   └── notebook-content.py          # Silver: JSON → structured Delta table
│
├── Structured Table data to Gold Processing.Notebook/
│   └── notebook-content.py          # Gold: enrichment + classification
│
├── Earthquake Data Pipeline.DataPipeline/
│   └── ...                          # Fabric pipeline orchestration config
│
├── Worldwide Earthquake Events.Report/
│   └── ...                          # Power BI report definition
│
├── earthquake_lakehouse_semantic_model.SemanticModel/
│   └── ...                          # Semantic model for Power BI
│
├── earthquakes_lakehouse.Lakehouse/
│   └── ...                          # Lakehouse metadata
│
└── README.md
```

> **Note**: Each `.Notebook` folder contains a `notebook-content.py` file — this is how Microsoft Fabric stores notebooks when synced to Git.

---

## Key Engineering Decisions

| Decision | Rationale |
|---|---|
| **Medallion architecture** | Separates raw ingestion, cleaning, and business logic into distinct, testable layers |
| **Delta Lake tables** | ACID transactions, schema enforcement, and time travel for auditability |
| **`saveAsTable()` over `.save()`** | Registers tables in the Fabric metastore, making them queryable via SQL and accessible to Power BI |
| **UDF for reverse geocoding** | `reverse_geocoder` uses an offline dataset — no external API calls needed at runtime, keeping the pipeline self-contained |
| **Significance classification in Gold** | Business logic belongs in the Gold layer, not Silver — Silver is purely structural transformation |
| **Fabric Data Pipeline for orchestration** | Chains notebook executions in order (Bronze → Silver → Gold), enabling scheduled automated runs |

---

## How to Run

### Prerequisites
- Microsoft Fabric workspace (F2 capacity or higher, or Fabric trial)
- Git integration configured in the Fabric workspace

### Setup
1. **Clone the repo** into your Fabric workspace via Git integration:
   - Workspace Settings → Git integration → Connect to this repository
2. **Attach the Lakehouse**: Each notebook expects `earthquakes_lakehouse` as the default Lakehouse. Create it if it doesn't exist.
3. **Install dependencies**: The Gold notebook installs `reverse_geocoder` via `%pip install reverse_geocoder` — this runs automatically in the first cell.
4. **Run the pipeline**: Execute the `Earthquake Data Pipeline` to run all notebooks in sequence, or run each notebook manually in order: Bronze → Silver → Gold.

---

## Sample Data

Each earthquake event in the Gold layer contains:

| Column | Type | Example |
|---|---|---|
| `id` | string | `us7000n1bx` |
| `longitude` | double | `-155.2764` |
| `latitude` | double | `19.4103` |
| `elevation` | double | `31.89` |
| `title` | string | `M 2.3 - 5 km SW of Volcano, Hawaii` |
| `place_description` | string | `5 km SW of Volcano, Hawaii` |
| `sig` | long | `83` |
| `mag` | double | `2.3` |
| `magType` | string | `ml` |
| `time` | timestamp | `2026-02-28 14:23:07` |
| `updated` | timestamp | `2026-02-28 15:01:42` |
| `country_code` | string | `US` |
| `sig_class` | string | `Low` |

---

## Author

**Rajkumar Manala**
- [GitHub](https://github.com/RajkumarManala1)
- [LinkedIn](https://linkedin.com/in/rajkumarmanala) *(update with your actual LinkedIn URL)*

---

## License

This project is for portfolio and educational purposes.
