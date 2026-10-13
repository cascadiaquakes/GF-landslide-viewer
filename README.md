# Cascadia Ground Failure Viewer

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.18895836.svg)](https://doi.org/10.5281/zenodo.18895836)
[![License: BSD-3-Clause](https://img.shields.io/badge/License-BSD_3--Clause-blue.svg)](LICENSE)

A cloud-native geospatial application for exploring ground failure inventories across the Cascadia region. Features interactive visualization, intelligent filtering, vector-tile-based rendering, and a fully serverless export pipeline capable of efficiently delivering large filtered datasets.

**Live viewer:** https://ground-failure.cascadiaquakes.org

---

## Documentation

The [Cascadia landslide book](https://cascadiaquakes.github.io/cascadia-landslide-book/) documents the ground failure data, and [cascadia-landslide-data](https://github.com/cascadiaquakes/cascadia-landslide-data) holds the preprocessing.

---

## Features

### Interactive 2D Visualization
- Vector tiles served via Martin (ECS Fargate)
- MapLibre-based rendering with smooth zoom transitions
- PGA contours overlay (USGS M9 scenario)
- Detail-rich popups and overlays
- Cascadia-bounded default view

### Advanced Filters

Data have been aligned for all datasets using the following fields:
- **Categorical:** Material, Movement, Confidence  (computed from original datasets)
- **Numerical:** PGA, PGV, rainfall ranges  (Using USGS M9 scenario and 30-Year (1990-2019) Annual Average of DAYMET Precipitation)
- Real-time updates reflected in tile requests  
- Filter summary panel showing active constraints

### Data Export (Serverless)
- Fully asynchronous AWS-based pipeline
- API Gateway → Lambda creates job + posts to SQS
- Worker Lambda queries PostGIS and writes results to S3
- DynamoDB stores job progress + errors
- Frontend polls job endpoint until download is ready
- Supports GeoJSON export (optionally zipped)

---

## Data

Landslide inventories from:

- California Geological Survey (CGS)
- Oregon Department of Geology and Mineral Industries (DOGAMI)
- Washington Geological Survey (WGS)
- British Columbia Geological Survey (Canada)

The inventories are aligned to common fields (material, movement, confidence) and joined with PGA/PGV from the USGS M9 scenario and 30-year (1990-2019) annual average DAYMET precipitation. The preprocessing is documented in [cascadia-landslide-data](https://github.com/cascadiaquakes/cascadia-landslide-data).

---

## Technology Stack

| Component | Technology | Version |
|----------|------------|---------|
| Frontend Build | Vite | Latest |
| Mapping | MapLibre GL JS | Latest |
| Tile Server | Martin | 0.14.x |
| Backend API | AWS Lambda (Python) | |
| Queue | Amazon SQS | |
| Job Tracking | DynamoDB | |
| Database | PostgreSQL + PostGIS | 16 / 3.4 |
| Deployment | AWS CDK | Latest |
| Hosting | S3 + CloudFront | |

---

## Architecture Overview

![Architecture](landslide_DB_AWS.png)

---

## Running Locally

### Prerequisites
- Node.js 18+
- Docker Desktop (for local Martin/PostGIS)
- AWS CLI + CDK (for deployment)
- Python 3.10+ (Lambda dependencies)

### Installation

**1. Clone repository**
```bash
git clone https://github.com/cascadiaquakes/GF-landslide-viewer.git
cd GF-landslide-viewer
```

**2. Start local Martin + PostGIS**
```bash
cd martin-server
docker-compose up
```

**3. Initialize frontend**
```bash
cd frontend
npm install
npm run dev
```

**4. Open application**
- Frontend: http://localhost:5173  
- Tiles: http://localhost:3000  

### Development Commands

**Frontend:**
```bash
cd frontend
npm run dev
```

**Worker Lambda test:**
```bash
python download_api/worker_main.py
```

---

## Project Structure

```
GF-landslide-viewer/
├── README.md
│   # Documentation for the entire project
│
├── aws_cdk/
│   ├── app.py
│   │   # CDK entrypoint — bootstraps the AWS application and instantiates the stack
│   ├── cdk.json
│   │   # CDK configuration file (context, toolkit settings)
│   └── landslide_stack.py
│       # Main CDK stack defining all AWS resources:
│       # - VPC, RDS (PostGIS), ECS Fargate (Martin server)
│       # - Lambda functions, API Gateway
│       # - SQS queue, DynamoDB job table
│       # - CloudFront + S3 hosting for frontend
│
├── download_api/
│   ├── lambda_main.py
│   │   # Lambda handler for /download endpoint:
│   │   # Validates request, creates a job entry, pushes job to SQS
│   ├── worker_main.py
│   │   # Worker Lambda triggered by SQS:
│   │   # Executes PostGIS query, writes export file to S3, updates DynamoDB
│   ├── main.py
│   │   # Shared helpers used by both API + worker Lambdas:
│   │   # (filter validation, SQL builder, error handling)
│   └── requirements.txt
│       # Python dependencies bundled into Lambda layers or zipped packages
│
├── frontend/
│   ├── index.html
│   │   # HTML entrypoint for the Vite application
│   ├── package.json
│   │   # Frontend dependencies and scripts
│   ├── vite.config.js
│   │   # Vite build + dev server configuration
│   │
│   ├── public/
│   │   └── vite.svg
│   │       # Public assets served as-is (no bundling)
│   │
│   └── src/
│       ├── main.js
│       │   # Frontend entrypoint: initializes UI, map, filter panels
│       ├── style.css
│       │   # Global styling for the viewer
│       │
│       ├── download/
│       │   ├── download_api.js
│       │   │   # Frontend wrapper for POST /download and GET /download/{jobId}
│       │   └── downloadPanel.js
│       │       # UI modal for download requests + polling workflow
│       │
│       ├── filter-panel/
│       │   ├── FiltersPanel.js
│       │   │   # Creates and manages the full filter panel UI
│       │   ├── filters.js
│       │   │   # Event listeners + logic for applying filters
│       │   ├── filterState.js
│       │   │   # Central state manager for all filter values
│       │   ├── filters-panel.css
│       │   │   # Styling for the filter UI
│       │   └── landslide-filters-config.js
│       │       # Definitions of categorical + numeric filters (labels, ranges, options)
│       │
│       ├── maplibre/
│       │   ├── baselayer.js
│       │   │   # Background basemap configuration (imagery, tiles)
│       │   ├── config.js
│       │   │   # Map constants (style IDs, source IDs, bounds, colors)
│       │   ├── layers.js
│       │   │   # Render layers for landslides + PGA overlays
│       │   ├── overlay.js
│       │   │   # Additional visual overlays (labels, boxes, etc.)
│       │   ├── viewer.js
│       │   │   # Main MapLibre map initialization + interaction handlers
│       │   └── zoom.js
│       │       # Zoom helpers for syncing UI + map behavior
│       │
│       ├── summary/
│       │   └── summary.js
│       │       # Sidebar summary panel showing active filters and counts
│       │
│       └── resources/
│           ├── Crescent_Logo.png
│           ├── USNSF_Logo.png
│           ├── favicon.ico
│           └── pga_contours.json
│               # Static assets + contour GeoJSON for overlays
│
├── martin-server/
│   ├── config.yaml
│   │   # Martin configuration: database connection, tile sources, vector layers
│   └── docker-compose.yml
│       # Local development stack (PostGIS + Martin tile server)
│
└── sql/
    └── setup_db.sql
        # PostGIS schema setup sample
```

---

## API Endpoints

### **POST `/download`**
Initiate a new export job.

**Payload:**
```json
{
  "filters": {
    "materials": ["Earth", "Debris"],
    "movements": ["Flow"],
    "confidences": ["High"],
    "pga_min": 0.05,
    "pga_max": 0.5
  },
  "compress": true
}
```

**Response:**
```json
{
  "jobId": "1234-5678",
  "status": "pending"
}
```

### **GET `/download/{jobId}`**
Polls job status (DynamoDB).  
Returns download URL when ready.

---

## Database Schema

### `landslides` Table (example)

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Unique landslide identifier |
| geom | Geometry | Landslide point/polygon |
| material | TEXT | Material classification |
| movement | TEXT | Movement type |
| confidence | TEXT | Inventory confidence |
| pga | FLOAT | Peak Ground Acceleration |
| pgv | FLOAT | Peak Ground Velocity |
| rainfall | FLOAT | Annual rainfall estimate |

(Accompanying tile-optimized tables/views are defined in `sql/setup_db.sql`.)

We also store in separate tables the original data (e.g. WDNR, DOGAMI...) and use it for downloads.

---

## Deployment

| Workflow | Trigger | What it does |
|---|---|---|
| `ci.yml` | PRs to `master` or `dev` | Validates the frontend build and infrastructure |
| `deploy-frontend-dev.yml` | Push to `dev` | Builds and deploys the dev frontend |
| `deploy-frontend.yml` | Manual | Builds and deploys the production frontend |
| `deploy-infra.yml` | Manual | Deploys the CDK stack (API Gateway, Lambdas, SQS, DynamoDB, ECS, RDS) |

The frontend is served from S3 + CloudFront; everything else is managed by the CDK app in `aws_cdk/`. See [CICD_SETUP.md](CICD_SETUP.md) for the full pipeline.

---

## Troubleshooting

### Frontend not loading
- Hard refresh: `Ctrl + Shift + R`
- Check console errors
- Verify tile server reachable

### Tiles missing
- Ensure Martin is running locally
- Check PostGIS connection string in `config.yaml`

### Downloads failing
- Check Lambda logs (API + worker)
- Verify SQS queue not stalled
- Check DynamoDB job entry for error messages

---

## Performance

- Vector tiles minimize data transfer  
- Simplified geometry tables optimized for rendering  
- Serverless download pipeline prevents API timeouts  
- CloudFront caching improves global latency

---

## How to Cite

> Bachelot, L., Leshchinsky, B., Acharya, P., Olsen, M., Share, P.-E., & Marfo, W. (2026). *The Cascadia Region Earthquake Science Center (CRESCENT) Ground Failure Viewer* (v1.0.0). Zenodo. https://doi.org/10.5281/zenodo.18895836

Please also credit the state and provincial geological surveys whose landslide inventories are shown (listed under [Data](#data)).

---

## License

This project is licensed under the BSD 3-Clause License. See [LICENSE](LICENSE).

---

## Contacts

- **Loïc Bachelot**, University of Oregon: [lbachelo@uoregon.edu](mailto:lbachelo@uoregon.edu)
- **William Marfo**, University of California, Davis: [wmarfo@ucdavis.edu](mailto:wmarfo@ucdavis.edu)
- **Amanda M. Thomas**, University of California, Davis: [amthom@ucdavis.edu](mailto:amthom@ucdavis.edu)

For bugs or feature requests, please [open an issue](https://github.com/cascadiaquakes/GF-landslide-viewer/issues).

---

## Acknowledgments

<p align="center">
  <img src="https://raw.githubusercontent.com/cascadiaquakes/.github/main/assets/crescent-logo.png" alt="CRESCENT" height="70">
  &nbsp;&nbsp;&nbsp;&nbsp;
  <img src="https://raw.githubusercontent.com/cascadiaquakes/.github/main/assets/nsf-logo.png" alt="U.S. National Science Foundation" height="70">
</p>

This project is supported by the National Science Foundation through the Cascadia Region Earthquake Science Center (CRESCENT), Cooperative Agreement EAR-2225286.

Developed by the CRESCENT development team based on the work of Prasaya Acharya. Landslide data courtesy of the geological surveys listed under [Data](#data).
