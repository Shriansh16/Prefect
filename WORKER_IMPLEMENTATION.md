# CDP-Profiler Worker Implementation

## Overview

The **Profiler Worker** is a containerized Prefect agent that executes workflow tasks. It runs continuously, polling the Prefect server for queued jobs and executing them when available.

**Key Relationship:** Prefect server schedules and queues jobs. Workers run independently in containers, connect to the Prefect server, poll for queued jobs, and execute them. Jobs only execute when workers are available and connected.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│         Prefect Server (Orchestrator)                   │
│  - Manages work pool: "cdp-workers"                     │
│  - Queues workflow runs                                 │
│  - Tracks execution state                                │
└────────────────────┬────────────────────────────────────┘
                      │
                      │ HTTP Polling (every few seconds)
                      ▼
┌─────────────────────────────────────────────────────────┐
│         Worker Container (Prefect Agent)                 │
│  ┌──────────────────────────────────────────────────┐  │
│  │  1. Entrypoint Script (worker-entrypoint.sh)     │  │
│  │     - Wait for Prefect server                     │  │
│  │     - Deploy flows                                │  │
│  │     - Start worker                                │  │
│  └──────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  2. Prefect Worker Process                        │  │
│  │     - Polls work pool: "cdp-workers"             │  │
│  │     - Picks up queued flow runs                   │  │
│  │     - Executes tasks                              │  │
│  │     - Reports status                              │  │
│  └──────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  3. Workflow Execution                            │  │
│  │     - Runs Python code from flows/                │  │
│  │     - Executes tasks in sequence/parallel         │  │
│  │     - Handles retries and errors                  │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## Implementation Details

### 1. Docker Container Setup

**File: `Dockerfile.worker`**

```dockerfile
FROM python:3.10-slim

# Install system dependencies (SQL Server ODBC driver, etc.)
RUN apt-get update && apt-get install -y \
    curl gnupg2 ca-certificates unixodbc-dev \
    msodbcsql18

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source code
COPY src/ ./src/
COPY prefect.yaml ./

# Copy entrypoint script
COPY docker/worker-entrypoint.sh /app/worker-entrypoint.sh
RUN chmod +x /app/worker-entrypoint.sh

# Set environment
ENV CONTAINER_ENV=true
ENV PYTHONPATH=/app/src

# Start entrypoint script
ENTRYPOINT ["/app/worker-entrypoint.sh"]
```

**Key Components:**
- **Base Image:** `python:3.10-slim` (lightweight Python runtime)
- **Dependencies:** Prefect, Azure SDK, DuckDB, profiling libraries
- **SQL Server Driver:** Microsoft ODBC Driver 18 (for data ingestion)
- **Entrypoint:** `worker-entrypoint.sh` (startup script)

---

### 2. Entrypoint Script Flow

**File: `docker/worker-entrypoint.sh`**

The entrypoint script runs when the container starts and performs these steps:

#### Step 1: Load Authentication (if needed)
```bash
# Load Prefect auth string from Azure Key Vault
if [ -z "${PREFECT_API_AUTH_STRING:-}" ]; then
    PREFECT_API_AUTH_STRING="$(python - <<'PY'
from cdp.secrets import get_secret
value = get_secret("PREFECT-API-AUTH-STRING")
print(value)
PY
)"
    export PREFECT_API_AUTH_STRING
fi
```

**Purpose:** Retrieves Prefect authentication token from Azure Key Vault if not provided via environment variable.

#### Step 2: Wait for Prefect Server
```bash
# Wait for Prefect server to be available
timeout=60
until curl -s "${PREFECT_API_URL}/health" > /dev/null 2>&1 || [ $timeout -le 0 ]; do
    echo "Waiting for Prefect server at ${PREFECT_API_URL}..."
    sleep 2
    timeout=$((timeout - 2))
done
```

**Purpose:** Ensures Prefect server is ready before attempting to connect. Prevents connection errors on startup.

#### Step 3: Deploy Flows
```bash
# Deploy Prefect flows (idempotent - safe to run multiple times)
cd /app
prefect deploy --all
```

**Purpose:** Registers all workflow definitions with Prefect server. Reads from `prefect.yaml` and creates deployments.

**What Gets Deployed:**
- `on-demand-profile` - Main profiling workflow
- `nightly-profiles` - Scheduled batch workflow
- `regenerate-insights-only` - Insights regeneration workflow
- `csv-upload` - CSV upload workflow

#### Step 4: Start Worker
```bash
# Start the Prefect worker
exec prefect worker start --pool cdp-workers
```

**Purpose:** Starts the Prefect worker process that polls for work and executes tasks.

---

### 3. Prefect Worker Process

**Command: `prefect worker start --pool cdp-workers`**

#### How It Works:

1. **Connects to Prefect Server**
   - Uses `PREFECT_API_URL` environment variable
   - Authenticates using `PREFECT_API_AUTH_STRING` (if needed)
   - Registers itself as available worker in `cdp-workers` pool

2. **Polls for Work**
   - Every few seconds, checks Prefect server for queued flow runs
   - Looks for runs assigned to `cdp-workers` work pool
   - Picks up runs in FIFO order (or priority-based if configured)

3. **Executes Flow Runs**
   - When a flow run is found, worker:
     - Updates status to "Running"
     - Loads flow code from deployment
     - Executes flow function with provided parameters
     - Runs tasks in sequence (or parallel if configured)
     - Reports progress after each task

4. **Reports Status**
   - Sends task completion status to Prefect server
   - Logs output to Prefect UI
   - Updates flow run state (Running → Completed/Failed)

#### Worker Lifecycle:

```
Start → Connect to Prefect → Register in Pool → Poll for Work
                                                      ↓
                                              Work Available?
                                                      ↓
                                              Yes → Execute Flow
                                                      ↓
                                              Report Status → Poll Again
```

---

### 4. Workflow Execution

When the worker picks up a flow run, it executes the workflow defined in `src/cdp/flows/client_pipeline.py`:

#### Example: `client_pipeline` Flow

```python
@flow(
    name="client-profile-pipeline",
    task_runner=ConcurrentTaskRunner(),
    timeout_seconds=7200  # 2 hours max
)
def client_pipeline(
    client_id: str,
    sample_size: int = 10000,
    skip_ingestion: bool = False,
    domain_context: Optional[str] = None
):
    # Step 1: Check if data exists
    parquet_files_exist = check_parquet_files_exist(client_id)
    
    # Step 2: Ingest data (if needed)
    if not parquet_files_exist or not skip_ingestion:
        ingestion_result = ingest_client_data(client_id)
    
    # Step 3: Profile data
    profile_result = profile_client_data(client_id, sample_size)
    
    # Step 4: Generate insights
    insights_result = generate_column_insights_task(client_id, domain_context)
    
    # Step 5: Build semantic index
    semantic_index_result = generate_semantic_index_task(client_id)
    
    return result
```

#### Task Execution:

Each `@task` decorated function runs as a separate unit:

```python
@task(retries=2, retry_delay_seconds=60, log_prints=True)
def ingest_client_data(client_id: str, chunk_size: int = 100000):
    """
    Task: Export client data from SQL Server to ADLS
    - Automatically retries 2 times on failure
    - Logs all output to Prefect UI
    """
    # ... ingestion logic ...
```

**Task Features:**
- **Retries:** Automatic retry on failure (2 retries with 60-second delay)
- **Logging:** All print statements captured in Prefect UI
- **Dependencies:** Tasks wait for prerequisites to complete
- **Parallel Execution:** Multiple tasks can run concurrently (if using `ConcurrentTaskRunner`)

---

### 5. How Work Gets Queued

#### Scenario 1: API Triggers Workflow

```
1. User/API calls: POST /workflows/profile/70132
   ↓
2. API Server calls Prefect API:
   POST https://prefect-dev.azurewebsites.net/api/deployments/{id}/create_flow_run
   ↓
3. Prefect Server creates flow run and queues it in "cdp-workers" pool
   ↓
4. Worker polls Prefect server (every few seconds)
   ↓
5. Worker finds queued run and picks it up
   ↓
6. Worker executes workflow
```

#### Scenario 2: Scheduled Workflow

```
1. Prefect Server checks schedule (cron: "0 2 * * *")
   ↓
2. At 2 AM UTC, Prefect creates flow run automatically
   ↓
3. Flow run queued in "cdp-workers" pool
   ↓
4. Worker picks up and executes
```

---

### 6. Error Handling & Retries

#### Task-Level Retries

```python
@task(retries=2, retry_delay_seconds=60)
def ingest_client_data(client_id: str):
    # If this fails, Prefect automatically:
    # 1. Waits 60 seconds
    # 2. Retries the task
    # 3. Repeats up to 2 times
    # 4. If all retries fail, marks task as "Failed"
```

#### Flow-Level Handling

```python
@flow(timeout_seconds=7200)
def client_pipeline(...):
    try:
        result = ingest_client_data(client_id)
    except Exception as e:
        # Flow can catch task failures and handle gracefully
        log.error(f"Ingestion failed: {e}")
        # Continue with other tasks or fail entire flow
```

#### Worker-Level Resilience

- **Automatic Reconnection:** If worker loses connection to Prefect server, it automatically reconnects
- **Graceful Shutdown:** Worker finishes current task before stopping
- **Health Checks:** Container orchestrator (Azure) monitors worker health

---

### 7. Configuration

#### Environment Variables

**Required:**
```bash
PREFECT_API_URL=https://prefect-dev.azurewebsites.net/api
PYTHONPATH=/app/src
CONTAINER_ENV=true
```

**Optional:**
```bash
PREFECT_API_AUTH_STRING=<token>  # If Prefect server requires auth
DATALAKE_LOCAL_PATH=/app/cdp      # For local development
```

**Secrets (from Azure Key Vault):**
```bash
OPENAI_API_KEY          # For insights generation
DB_PWD_DEV              # SQL Server password
DATALAKE_ADLS_TOKEN     # ADLS access token
```

#### Prefect Configuration

**File: `prefect.yaml`**

```yaml
deployments:
  - name: on-demand-profile
    entrypoint: /app/src/cdp/flows/client_pipeline.py:client_pipeline
    work_pool:
      name: cdp-workers  # Worker polls this pool
    tags:
      - production
      - on-demand
```

**Key Settings:**
- **work_pool:** `cdp-workers` - Worker must join this pool
- **entrypoint:** Path to flow function
- **tags:** For filtering and organization

---

### 8. Monitoring & Observability

#### Prefect UI

**Access:** `https://prefect-dev.azurewebsites.net`

**What You Can See:**
- All flow runs (running, completed, failed)
- Task execution logs
- Execution duration
- Error messages and stack traces
- Worker status and activity

#### Container Logs

**Azure Container Apps:**
```bash
az containerapp logs show \
  --name cdp-worker-dev \
  --resource-group exp_ai_rg \
  --tail 100
```

**Local Docker:**
```bash
docker logs cdp-worker --tail 100 -f
```

**What Logs Show:**
- Worker startup sequence
- Prefect server connection status
- Flow deployment status
- Task execution progress
- Error messages

---

### 9. Scaling

#### Horizontal Scaling

**Multiple Workers:**
- Deploy multiple worker containers
- All join the same work pool: `cdp-workers`
- Prefect automatically distributes work
- Each worker processes different flow runs in parallel

**Example:**
```
Work Pool: cdp-workers
├── Worker 1 (processing flow run A)
├── Worker 2 (processing flow run B)
└── Worker 3 (idle, waiting for work)
```

#### Auto-Scaling (Azure Container Apps)

**Configuration:**
```yaml
scale:
  minReplicas: 1
  maxReplicas: 5
  rules:
    - name: cpu-scaling
      type: cpu
      metadata:
        target: 80
```

**Behavior:**
- Scales up when CPU > 80%
- Scales down when CPU < 20%
- Minimum 1 worker always running
- Maximum 5 workers for cost control

---

### 10. Local Development

#### Using Docker Compose

**File: `docker-compose.yml`**

```yaml
services:
  worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    environment:
      PREFECT_API_URL: http://prefect:4200/api
      DATALAKE_LOCAL_PATH: /app/cdp
    volumes:
      - ./cdp:/app/cdp  # Mount local data directory
    depends_on:
      - prefect
```

**Start:**
```bash
docker-compose up worker
```

#### Manual Start (for debugging)

```bash
# Start Prefect server first
prefect server start

# In another terminal, start worker
cd src/cdp/flows
PYTHONPATH=/app/src \
PREFECT_API_URL=http://localhost:4200/api \
prefect worker start --pool cdp-workers
```

---

## Complete Execution Flow Example

### Scenario: User Triggers Profile via API

```
1. API receives: POST /workflows/profile/70132
   ↓
2. API calls Prefect: create_flow_run(deployment="on-demand-profile", parameters={...})
   ↓
3. Prefect Server:
   - Creates flow run with ID: abc-123
   - Queues in "cdp-workers" pool
   - Status: "Scheduled"
   ↓
4. Worker (polling every 5 seconds):
   - Checks Prefect API: GET /api/work_pools/cdp-workers/runs
   - Finds flow run abc-123
   - Picks it up
   - Status: "Running"
   ↓
5. Worker executes:
   - check_parquet_files_exist("70132") → Task 1 ✓
   - ingest_client_data("70132") → Task 2 ✓
   - profile_client_data("70132", 10000) → Task 3 ✓
   - generate_column_insights_task("70132", ...) → Task 4 ✓
   - generate_semantic_index_task("70132") → Task 5 ✓
   ↓
6. Worker reports completion:
   - Updates Prefect: flow run abc-123 → "Completed"
   - Logs: "✓ Pipeline completed successfully for client 70132"
   ↓
7. API polls status:
   - GET /workflows/status/abc-123
   - Returns: {"status": "completed", "result": {...}}
   ↓
8. User sees completion in UI
```

---

## Key Takeaways

1. **Worker is a Prefect Agent**
   - Runs continuously, polling for work
   - Executes workflows defined in `flows/` directory
   - Reports status back to Prefect server

2. **Entrypoint Script Handles Startup**
   - Waits for Prefect server
   - Deploys flows automatically
   - Starts worker process

3. **Work Pool Model**
   - Multiple workers can join same pool
   - Prefect distributes work automatically
   - Enables horizontal scaling

4. **Task Execution**
   - Each `@task` runs as separate unit
   - Automatic retries on failure
   - Logs captured in Prefect UI

5. **Resilience**
   - Automatic reconnection to Prefect
   - Task-level retries
   - Graceful error handling

---

## Troubleshooting

### Worker Not Connecting

**Check:**
```bash
# Verify Prefect server is running
curl https://prefect-dev.azurewebsites.net/api/health

# Check worker logs
docker logs cdp-worker

# Verify PREFECT_API_URL is set correctly
echo $PREFECT_API_URL
```

### Worker Not Picking Up Work

**Check:**
- Work pool name matches: `cdp-workers`
- Flow deployment exists in Prefect UI
- Flow run is in "Scheduled" state (not "Failed")
- Worker is registered in work pool

### Tasks Failing

**Check:**
- Prefect UI logs for error messages
- Container logs for Python exceptions
- Verify environment variables (secrets, API keys)
- Check Azure permissions (Key Vault, ADLS)

---

## Summary

The worker implementation is a **containerized Prefect agent** that:
- ✅ Starts automatically with entrypoint script
- ✅ Connects to Prefect server
- ✅ Deploys workflows on startup
- ✅ Polls for queued work continuously
- ✅ Executes workflows and tasks
- ✅ Reports status and logs
- ✅ Handles retries and errors automatically
- ✅ Scales horizontally with multiple instances

This architecture provides **reliability, observability, and scalability** for the CDP profiling workflows.

