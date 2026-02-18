# CDP-Profiler: Required Deployments (Backend Only)

## Summary

**Total Deployments Required: 3 Core Components**

The CDP-Profiler backend requires **3 main components** to be deployed, each serving a specific purpose in the architecture.

---

## Deployment Components

### 1. **Profiler API** (FastAPI Server)
**Purpose:** Serves REST API endpoints for profile metrics, insights, and workflow management

**Responsibilities:**
- Handle HTTP requests from clients/UI
- Serve profile metrics via `/profile/{client_id}` endpoint
- Generate and serve AI insights via `/profile/{client_id}/insights`
- Trigger Prefect workflows via `/workflows/profile/{client_id}`
- Check workflow status via `/workflows/status/{workflow_id}`
- Handle CSV uploads via `/upload/csv/{client_id}`
- Query metrics from DuckDB or Synapse backends

**Deployment by Environment:**
| Environment | Resource Type | Resource Name | Resource Group |
|------------|--------------|---------------|----------------|
| **Dev** | Azure Web App | `profiler-api-dev` | `exp_ai_rg` |
| **Staging** | Azure Container App | `cdp-staging-api` | `rg-cdp-stg` |
| **Prod** | Azure Container App | `cdp-prod-api` | `rg-cdp-prod` |

**Docker Image:** `Dockerfile.api`
**Port:** 8000
**Health Check:** `/health` endpoint

---

### 2. **Profiler Worker** (Prefect Worker)
**Purpose:** Executes Prefect workflows (data ingestion, profiling, insights generation)

**Responsibilities:**
- Poll Prefect server for queued jobs
- Execute workflow tasks:
  - Data ingestion (SQL Server → ADLS)
  - Data profiling (generate metrics)
  - Custom objects profiling
  - AI insights generation
  - Semantic index generation
- Report task status back to Prefect
- Handle retries and error recovery

**Deployment by Environment:**
| Environment | Resource Type | Resource Name | Resource Group |
|------------|--------------|---------------|----------------|
| **Dev** | Azure Container Instance | `cdp-worker-dev` | `exp_ai_rg` |
| **Staging** | Azure Container App | `cdp-staging-workers` | `rg-cdp-stg` |
| **Prod** | Azure Container App | `cdp-prod-workers` | `rg-cdp-prod` |

**Docker Image:** `Dockerfile.worker`
**Prefect Work Pool:** `cdp-workers`
**Scaling:**
- Dev: Single instance (ACI)
- Staging/Prod: Auto-scale 1-5 instances (Container App)

---

### 3. **Prefect Server** (Orchestration Server)
**Purpose:** Manages workflow orchestration, scheduling, and state tracking

**Responsibilities:**
- Manage work pools and worker connections
- Schedule workflows (e.g., nightly batch at 2 AM UTC)
- Queue on-demand workflow triggers
- Track workflow execution state
- Store workflow history and logs
- Provide Prefect UI dashboard for monitoring

**Deployment by Environment:**
| Environment | Resource Type | Resource Name | Resource Group |
|------------|--------------|---------------|----------------|
| **Dev** | Azure Web App | `prefect-dev` | `exp_ai_rg` |
| **Staging** | Azure Web App | `prefect-dev` | `exp_ai_rg` (shared with dev) |
| **Prod** | Azure Container App | `prefect-prod` | `rg-cdp-prod` |

**Prefect API URL:**
- Dev/Staging: `https://prefect-dev.azurewebsites.net/api`
- Prod: `https://prefect-prod.bluetree-15861917.eastus.azurecontainerapps.io/api`

**UI Access:**
- Dev/Staging: `https://prefect-dev.azurewebsites.net`
- Prod: `https://prefect-prod.bluetree-15861917.eastus.azurecontainerapps.io`

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    User/Client Requests                     │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              1. Profiler API (FastAPI Server)                │
│  - Handles HTTP requests                                     │
│  - Serves metrics and insights                               │
│  - Triggers workflows                                        │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ Triggers Workflow
                        ▼
┌─────────────────────────────────────────────────────────────┐
│           3. Prefect Server (Orchestration)                  │
│  - Manages work pools                                        │
│  - Queues jobs                                               │
│  - Tracks execution state                                     │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ Queues Work
                        ▼
┌─────────────────────────────────────────────────────────────┐
│         2. Profiler Worker (Prefect Agent)                   │
│  - Polls for work                                            │
│  - Executes workflow tasks                                   │
│  - Reports status                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Deployment Count Summary

### Per Environment

**Development:**
- 1x Profiler API (Web App)
- 1x Profiler Worker (Container Instance)
- 1x Prefect Server (Web App)
- **Total: 3 deployments**

**Staging:**
- 1x Profiler API (Container App)
- 1x Profiler Worker (Container App)
- 1x Prefect Server (Web App - shared with dev)
- **Total: 3 deployments** (1 shared)

**Production:**
- 1x Profiler API (Container App)
- 1x Profiler Worker (Container App)
- 1x Prefect Server (Container App)
- **Total: 3 deployments**

### Across All Environments

**Total Unique Deployments:**
- **3 core components** × **3 environments** = **9 total deployments**
- However, Prefect Server is shared between Dev and Staging
- **Actual unique deployments: 8**

---

## Additional Infrastructure (Not Deployments)

These are Azure resources that support the deployments but are not "deployments" themselves:

### Storage & Data
- **Azure Data Lake Storage (ADLS)** - Stores parquet files and metrics
- **Azure Synapse Analytics** - Alternative backend for metrics (optional)
- **DuckDB** - Embedded database in containers (no separate deployment)

### Supporting Services
- **Azure Container Registry (ACR)** - Stores Docker images
- **Azure Key Vault** - Stores secrets (API keys, passwords)
- **Log Analytics Workspace** - Centralized logging
- **Application Insights** - Application monitoring

### Networking
- **Container App Environment** - Network isolation (staging/prod)
- **Virtual Network** - Network configuration (if needed)

---

## Deployment Dependencies

### Order of Deployment

1. **Prefect Server** (must be deployed first)
   - Workers need Prefect API URL to connect
   - API needs Prefect API URL to trigger workflows

2. **Profiler Worker** (deploy after Prefect Server)
   - Requires Prefect Server to be running
   - Connects to Prefect work pool

3. **Profiler API** (can be deployed independently)
   - Can run without workers (but can't execute workflows)
   - Requires Prefect Server URL to trigger workflows

### Configuration Dependencies

**All components need:**
- Azure Key Vault access (for secrets)
- ADLS access (read/write permissions)
- Prefect API URL (for workers and API)

**Worker additionally needs:**
- SQL Server access (for data ingestion)
- OpenAI API key (for insights generation)

---

## Docker Images Required

### Image 1: `cdp-api`
**Dockerfile:** `Dockerfile.api`
**Used by:** Profiler API
**Base:** `python:3.10-slim`
**Key Dependencies:**
- FastAPI
- Uvicorn (4 workers)
- DuckDB with Azure extension
- Azure SDK

### Image 2: `cdp-worker`
**Dockerfile:** `Dockerfile.worker`
**Used by:** Profiler Worker
**Base:** `python:3.10-slim`
**Key Dependencies:**
- Prefect
- Prefect Azure integration
- DuckDB with Azure extension
- All profiling libraries
- OpenAI SDK (for insights)

**Note:** Both images share the same codebase (`src/cdp/`) but have different entrypoints.

---

## Scaling Configuration

### Profiler API
- **Dev:** Fixed 1 instance (Web App)
- **Staging/Prod:** Auto-scale 2-10 instances (Container App)
  - Scale based on HTTP requests
  - CPU threshold: 70%

### Profiler Worker
- **Dev:** Fixed 1 instance (Container Instance)
- **Staging/Prod:** Auto-scale 1-5 instances (Container App)
  - Scale based on queue depth
  - CPU threshold: 80%

### Prefect Server
- **Dev/Staging:** Fixed 1 instance (Web App)
- **Prod:** Auto-scale 1-3 instances (Container App)

---

## Summary Table

| Component | Purpose | Dev | Staging | Prod | Total |
|-----------|---------|-----|---------|------|-------|
| **Profiler API** | REST API server | Web App | Container App | Container App | 3 |
| **Profiler Worker** | Workflow executor | Container Instance | Container App | Container App | 3 |
| **Prefect Server** | Orchestration | Web App | Web App (shared) | Container App | 2 |
| **TOTAL** | | **3** | **3** | **3** | **8 unique** |

---

## Quick Reference

**Minimum Required for Basic Operation:**
- ✅ 1x Profiler API
- ✅ 1x Profiler Worker
- ✅ 1x Prefect Server
- **Total: 3 deployments**

**For Full Production Setup:**
- ✅ 3x Profiler API (dev/staging/prod)
- ✅ 3x Profiler Worker (dev/staging/prod)
- ✅ 2x Prefect Server (dev+staging shared, prod separate)
- **Total: 8 unique deployments**

---

## Notes

1. **Prefect Server Sharing:** Dev and Staging share the same Prefect Server (`prefect-dev`). This is intentional to reduce infrastructure costs and simplify management.

2. **Container Apps vs Web Apps:** 
   - Dev uses Web Apps for simplicity
   - Staging/Prod use Container Apps for better scaling and modern architecture

3. **Worker Scaling:** Workers can scale horizontally to process multiple workflows in parallel. The Prefect work pool automatically distributes work.

4. **No Database Deployment:** DuckDB is embedded in containers. Synapse is an external service (not deployed by this system).

5. **Frontend Excluded:** As requested, frontend deployments are not included in this count.

