# Prefect Complete Learning Guide
## Master Prefect 3.0 for Data Pipeline Orchestration

---

## 📚 Table of Contents

1. [Introduction to Prefect](#introduction-to-prefect)
2. [Installation & Setup](#installation--setup)
3. [Core Concepts](#core-concepts)
4. [Flows: The Orchestration Layer](#flows-the-orchestration-layer)
5. [Tasks: The Building Blocks](#tasks-the-building-blocks)
6. [States & Lifecycle](#states--lifecycle)
7. [Deployments & Scheduling](#deployments--scheduling)
8. [Work Pools & Workers](#work-pools--workers)
9. [Advanced Patterns](#advanced-patterns)
10. [Best Practices](#best-practices)
11. [Real-World Examples](#real-world-examples)
12. [Troubleshooting](#troubleshooting)
13. [Prefect in CDP-Profiler](#prefect-in-cdp-profiler)

---

## Introduction to Prefect

### What is Prefect?

**Prefect** is a modern workflow orchestration platform built for Python. It helps you build, run, and monitor data pipelines and workflows with:

- ✅ **Simple Python API** - Write workflows like regular Python functions
- ✅ **Automatic Retries** - Built-in retry logic for failed tasks
- ✅ **Observability** - Beautiful UI to monitor workflow runs
- ✅ **Scheduling** - Cron-based or event-driven scheduling
- ✅ **Parallel Execution** - Run tasks concurrently
- ✅ **State Management** - Track every step of your workflow

### Why Prefect?

**Traditional Approach:**
```python
# Without Prefect - fragile, no retries, no monitoring
def process_data():
    data = fetch_data()  # What if this fails?
    processed = transform(data)  # What if this fails?
    save(processed)  # What if this fails?
    # No way to retry, no visibility, no monitoring
```

**With Prefect:**
```python
# With Prefect - robust, retries, full observability
@flow
def process_data():
    data = fetch_data()  # Automatic retries on failure
    processed = transform(data)  # State tracking
    save(processed)  # Full visibility in UI
    # Built-in retries, monitoring, and error handling
```

### Prefect Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Prefect Server                       │
│  • Workflow Definitions                                 │
│  • Execution History                                   │
│  • UI Dashboard (http://localhost:4200)                │
└───────────────────┬─────────────────────────────────────┘
                    │
                    │ Work Queue
                    │
┌───────────────────▼─────────────────────────────────────┐
│                  Prefect Worker                          │
│  • Polls work queue                                     │
│  • Executes tasks                                       │
│  • Reports status back                                  │
└─────────────────────────────────────────────────────────┘
```

### Key Components

1. **Prefect Server** - Centralized orchestration hub
2. **Workers** - Execute tasks pulled from work queues
3. **Work Pools** - Groups of workers that can execute similar work
4. **Flows** - Your workflow definitions (decorated Python functions)
5. **Tasks** - Individual units of work within flows
6. **Deployments** - Configured, scheduled versions of flows

---

## Installation & Setup

### Step 1: Install Prefect

```bash
# Basic installation
pip install prefect

# With Azure integration (for CDP-Profiler)
pip install prefect prefect-azure

# Verify installation
prefect version
```

### Step 2: Choose Your Hosting Option (The "Brain")

In Prefect, you need a central server or cloud instance to manage your UI, schedules, and logs.

#### 1. Prefect Cloud (Recommended for Production)
*   **Need to deploy?** ❌ No. Managed SaaS by Prefect.
*   **Best for:** Most production use cases (Fast, Zero-maintenance).
*   **Setup:** Just sign up at [app.prefect.cloud](https://app.prefect.cloud).

#### 2. Self-Hosted Prefect Server
*   **Need to deploy?** ✅ Yes (Docker/VM).
*   **Best for:** Strict compliance where data metadata cannot leave your network.
*   **Command:** `prefect server start` (local dev) or run a persistent container on Azure/AWS.

#### 3. Ephemeral Mode (Local Only)
*   **Need to deploy?** ❌ No. Runs directly against a local SQLite file.
*   **Best for:** Single-user development or one-off scripts. No remote UI.

### Step 3: Production Hosting Comparison

| Feature | Prefect Cloud | Self-Hosted (Azure/AWS) |
| :--- | :--- | :--- |
| **Maintenance** | Zero (Managed) | High (You manage DB & Uptime) |
| **Infrastructure** | SaaS | Your VPC/VNet (DB + Compute) |
| **Security** | Metadata in Cloud | 100% Private |
| **Setup Time** | 5 mins | 2-3 hours |

---

### Step 4: Self-Hosting Requirements (Azure/AWS)

If you choose to host the Prefect Server yourself instead of using Prefect Cloud, you need to provision the following:

| Component | AWS Equivalent | Azure Equivalent |
| :--- | :--- | :--- |
| **Database** | RDS (PostgreSQL) | Azure Database (PostgreSQL) |
| **Server/UI** | ECS (Fargate) / EKS | Container Instances (ACI) / AKS |
| **Networking** | ALB (Load Balancer) | Application Gateway |
| **Auth** | AWS IAM / Cognito | Managed Identity / Entra ID |

**Setup Flow:**
1.  Deploy a PostgreSQL instance (Prefect needs this for its internal metadata).
2.  Deploy the `prefecthq/prefect:3.0-python3.11` Docker image.
3.  Set `PREFECT_API_DATABASE_CONNECTION_URL` to your Postgres string.
4.  Expose the container via a Load Balancer (HTTPS).

---

### Step 5: Step-by-Step Cloud Setup

1.  **Create Account:** Sign up at [app.prefect.cloud](https://app.prefect.cloud).
2.  **Create Workspace:** Name it (e.g., `cdp-profiler`).
3.  **Generate API Key:** Go to Profile Settings → API Keys → Create Key.
4.  **Login via CLI:**
    ```bash
    prefect auth login --key YOUR_API_KEY
    ```
5.  **Configure API URL:** Look at your browser URL to get Account ID and Workspace ID.
    ```bash
    export PREFECT_API_URL="https://api.prefect.cloud/api/accounts/ACC_ID/workspaces/WS_ID"
    export PREFECT_API_KEY="YOUR_API_KEY"
    ```

---

### Step 5: Start a Worker (Local or Cloud)

Once logged in (to local or cloud), start the worker to listen for jobs:

```bash
# Create work pool first
prefect work-pool create cdp-workers --type process

# Start the worker
prefect worker start --pool cdp-workers
```

---

## Core Concepts

### The Mental Model

Think of Prefect workflows like a recipe:

- **Flow** = The entire recipe (e.g., "Make a cake")
- **Task** = Individual steps (e.g., "Mix ingredients", "Bake for 30 minutes")
- **State** = Current status (e.g., "Mixing...", "Baking...", "Done!")
- **Deployment** = Scheduled version of the recipe (e.g., "Make cake every Sunday at 2 PM")

### Basic Example

```python
from prefect import flow, task

@task
def fetch_data():
    """Fetch data from API"""
    return {"users": 100, "orders": 50}

@task
def process_data(data):
    """Process the data"""
    return data["users"] * 2

@flow
def my_workflow():
    """Main workflow"""
    data = fetch_data()  # Task 1
    result = process_data(data)  # Task 2 (depends on Task 1)
    print(f"Result: {result}")
    return result

# Run the workflow
if __name__ == "__main__":
    my_workflow()
```

**What happens:**
1. `my_workflow()` is called → Flow starts
2. `fetch_data()` executes → Task 1 runs
3. `process_data(data)` executes → Task 2 runs (waits for Task 1)
4. Flow completes → All tasks finished

---

## Flows: The Orchestration Layer

### What is a Flow?

A **flow** is a container for tasks. It defines the order and dependencies of operations.

### Creating Flows

```python
from prefect import flow

@flow(name="my-workflow")
def my_workflow(param1: str, param2: int = 10):
    """
    Flow function - orchestrates tasks
    
    Args:
        param1: Required parameter
        param2: Optional parameter with default
    """
    # Your workflow logic here
    pass
```

### Flow Characteristics

1. **Decorated Functions** - Use `@flow` decorator
2. **Can Call Tasks** - Flows orchestrate tasks
3. **Can Call Other Flows** - Nested flows (subflows)
4. **Parameter Limit** - ⚠️ **512KB max** for flow parameters
5. **Return Values** - Can return results

### Flow Parameters

```python
@flow(name="client-profile")
def profile_client(client_id: str, sample_size: int = 10000):
    """
    Profile a client's data
    
    Args:
        client_id: Client identifier (required)
        sample_size: Sample size for profiling (optional, default 10000)
    """
    print(f"Profiling client {client_id} with sample size {sample_size}")
    # ... workflow logic
```

**Important:** Flow parameters are serialized, so keep them small (< 512KB).

### Nested Flows (Subflows)

```python
@flow(name="ingestion-subflow")
def ingest_data(client_id: str):
    """Subflow for data ingestion"""
    # ... ingestion logic
    return {"status": "ingested"}

@flow(name="main-workflow")
def main_workflow(client_id: str):
    """Main workflow that calls subflow"""
    result = ingest_data(client_id)  # Calls another flow
    # ... rest of workflow
```

**Benefits:**
- Organize complex workflows
- Reuse common patterns
- Better observability (subflows appear separately in UI)

### Flow Runners

Control how tasks execute:

```python
from prefect import flow
from prefect.task_runners import ConcurrentTaskRunner, SequentialTaskRunner

# Sequential (default) - tasks run one after another
@flow
def sequential_flow():
    task1()
    task2()  # Waits for task1
    task3()  # Waits for task2

# Concurrent - tasks run in parallel
@flow(task_runner=ConcurrentTaskRunner())
def concurrent_flow():
    task1()  # Runs immediately
    task2()  # Runs immediately (parallel with task1)
    task3()  # Runs immediately (parallel with task1 and task2)
```

### Real Example from CDP-Profiler

```python
@flow(name="client-profile-pipeline")
def full_profile_workflow(
    client_id: str,
    sample_size: int = 10000,
    skip_ingestion: bool = False
):
    """
    Complete profiling workflow:
    1. Ingest data (if needed)
    2. Profile data
    3. Generate insights
    """
    # Step 1: Check if data exists
    data_exists = check_parquet_files_exist(client_id)
    
    # Step 2: Ingest if needed
    if not skip_ingestion or not data_exists:
        ingest_result = ingest_client_data(client_id)
    
    # Step 3: Profile data
    profile_result = profile_client_data(client_id, sample_size)
    
    # Step 4: Generate insights (parallel)
    insights_result = generate_insights_task(client_id)
    
    return {
        "status": "completed",
        "client_id": client_id
    }
```

---

## Tasks: The Building Blocks

### What is a Task?

A **task** is an atomic unit of work. It's a Python function decorated with `@task`.

### Creating Tasks

```python
from prefect import task

@task
def fetch_data():
    """Simple task"""
    return {"data": "example"}

@task(name="process-data", retries=2)
def process_data(input_data):
    """Task with name and retries"""
    # Process input_data
    return processed_result
```

### Task Parameters

Common task parameters:

```python
@task(
    name="my-task",              # Custom name (default: function name)
    retries=2,                    # Number of retry attempts
    retry_delay_seconds=60,       # Delay between retries
    timeout_seconds=300,          # Task timeout
    log_prints=True,              # Capture print statements
    tags=["ingestion", "heavy"],  # Tags for filtering
    cache_key_fn=lambda: "key",   # Caching (advanced)
    cache_expiration=None         # Cache expiration
)
def my_task(param1: str):
    """Task with all common parameters"""
    print(f"Processing {param1}")
    return {"result": "done"}
```

### Task Retries

```python
@task(retries=3, retry_delay_seconds=30)
def unreliable_api_call():
    """
    Task that might fail
    - Will retry up to 3 times
    - Waits 30 seconds between retries
    """
    response = requests.get("https://api.example.com/data")
    if response.status_code != 200:
        raise Exception("API call failed")
    return response.json()
```

**Retry Behavior:**
- Task fails → Wait `retry_delay_seconds` → Retry
- If all retries exhausted → Task marked as `FAILED`
- Flow can continue or fail based on error handling

### Task Timeouts

```python
@task(timeout_seconds=60)
def slow_operation():
    """
    Task with timeout
    - If execution takes > 60 seconds, task is cancelled
    """
    time.sleep(120)  # This will timeout!
    return "done"
```

### Task Logging

```python
@task(log_prints=True)
def task_with_logging():
    """
    Task that captures print statements
    - All print() calls appear in Prefect UI logs
    """
    print("Starting task...")
    print("Processing data...")
    print("Task complete!")
    return "done"
```

### Task Tags

```python
@task(tags=["ingestion", "heavy", "production"])
def heavy_ingestion_task():
    """
    Task with tags
    - Useful for filtering in UI
    - Can be used for concurrency limits
    """
    # Heavy processing
    pass
```

**Use Cases:**
- Filter tasks in UI: "Show only ingestion tasks"
- Concurrency limits: "Max 2 tasks with 'heavy' tag"
- Organization: Group related tasks

### Task Dependencies

Tasks automatically create dependencies based on call order:

```python
@task
def task_a():
    return "A"

@task
def task_b():
    return "B"

@task
def task_c(a_result, b_result):
    return f"{a_result} + {b_result}"

@flow
def my_flow():
    a = task_a()      # Runs first
    b = task_b()      # Runs in parallel with task_a
    c = task_c(a, b)  # Waits for both a and b
    return c
```

**Dependency Graph:**
```
task_a ──┐
         ├──> task_c
task_b ──┘
```

### Real Example from CDP-Profiler

```python
@task(retries=2, retry_delay_seconds=60, log_prints=True)
def ingest_client_data(client_id: str, chunk_size: int = 100000):
    """
    Export client data from OLTP to ADLS parquet.
    
    Why Prefect:
    - Long-running (5-15 minutes)
    - Needs retry logic (database timeouts)
    - Memory-intensive (chunking required)
    """
    from cdp.ingestion import Ingestion
    
    log.info(f"Starting data ingestion for client {client_id}")
    
    ingestion = Ingestion(client_id, datalake_name, oltp_db)
    partition_files = ingestion.export_profile_data(
        f"cdp/{client_id}/profiles/data.parquet",
        chunk_size=chunk_size
    )
    
    log.info(f"✓ Ingestion completed: {len(partition_files)} partitions")
    return {
        "client_id": client_id,
        "status": "ingested",
        "partitions": len(partition_files)
    }
```

### Task Best Practices

1. **Keep Tasks Focused** - One task = one responsibility
2. **Use Retries** - For unreliable operations (APIs, databases)
3. **Add Timeouts** - Prevent tasks from hanging forever
4. **Log Important Steps** - Use `log_prints=True` or logging
5. **Don't Pass Large Data** - Pass paths/IDs, not dataframes
6. **Use Tags** - Organize and filter tasks

---

## States & Lifecycle

### What are States?

Prefect tracks the **state** of every flow run and task run. States tell you what's happening.

### Common States

| State | Description | Example |
|-------|-------------|---------|
| **SCHEDULED** | Waiting to run | Flow scheduled for 2 AM |
| **PENDING** | Infrastructure being prepared | Worker pulling task |
| **RUNNING** | Currently executing | Task is running code |
| **COMPLETED** | Finished successfully | Task returned result |
| **FAILED** | Crashed with exception | Task raised error |
| **CRASHED** | Infrastructure failure | Worker OOM, network loss |
| **CANCELLED** | Manually cancelled | User cancelled run |
| **PAUSED** | Paused for input | Waiting for user approval |

### State Transitions

```
SCHEDULED → PENDING → RUNNING → COMPLETED
                              ↓
                           FAILED
                              ↓
                          CRASHED
```

### Checking States

```python
from prefect import flow, task, get_client

@task
def my_task():
    return "done"

@flow
def my_flow():
    result = my_task()
    return result

if __name__ == "__main__":
    # Run flow and get state
    state = my_flow()
    print(f"Flow state: {state}")
    
    # Check if completed
    if state.is_completed():
        print("Flow completed successfully!")
```

### State Details

```python
from prefect import flow, task

@task
def failing_task():
    raise ValueError("Something went wrong!")

@flow
def my_flow():
    try:
        result = failing_task()
    except Exception as e:
        print(f"Task failed: {e}")
        # Flow continues or fails based on error handling
```

### State Persistence

Prefect persists states to:
- **SQLite** (default, development)
- **PostgreSQL** (production)
- **Ephemeral** (no persistence, ephemeral mode)

---

## Deployments & Scheduling

### What is a Deployment?

A **deployment** is a configured, scheduled version of a flow. It tells Prefect:
- Which flow to run
- When to run it (schedule)
- Where to run it (work pool)
- What parameters to use

### Creating Deployments

#### Method 1: Using `.deploy()`

```python
from prefect import flow

@flow(name="my-workflow")
def my_workflow(param1: str):
    print(f"Running with {param1}")
    return "done"

# Deploy the flow
my_workflow.deploy(
    name="my-deployment",
    work_pool_name="cdp-workers",
    tags=["production"],
    description="My workflow deployment",
    version="1.0.0",
    cron="0 2 * * *",  # Daily at 2 AM UTC
    parameters={"param1": "default_value"}
)
```

#### Method 2: Using `prefect.yaml`

```yaml
# prefect.yaml
name: cdp-profiler
prefect-version: 3.0.0

deployments:
  - name: nightly-profiles
    entrypoint: src/cdp/flows/client_pipeline.py:nightly_profile_all_clients
    work_pool_name: cdp-workers
    schedule:
      cron: "0 2 * * *"
    parameters:
      client_ids: ["70132", "70210"]
    tags:
      - production
      - scheduled
```

Then deploy:
```bash
prefect deploy
```

### Scheduling Options

#### Cron Schedule

```python
# Daily at 2 AM UTC
cron="0 2 * * *"

# Every hour
cron="0 * * * *"

# Every Monday at 9 AM
cron="0 9 * * 1"

# Every 15 minutes
cron="*/15 * * * *"
```

#### Interval Schedule

```python
from prefect import flow
from datetime import timedelta

@flow
def my_flow():
    pass

my_flow.deploy(
    name="interval-deployment",
    schedule=timedelta(hours=1)  # Run every hour
)
```

#### No Schedule (On-Demand)

```python
my_flow.deploy(
    name="on-demand-deployment",
    cron=None  # No schedule, triggered manually or via API
)
```

### Real Example from CDP-Profiler

```python
# From deployments.py

# On-Demand Profile Deployment
client_pipeline.deploy(
    name="on-demand-profile",
    work_pool_name="cdp-workers",
    tags=["production", "on-demand"],
    description="On-demand client profiling triggered via API",
    version="1.0.0",
    cron=None  # No schedule, triggered on-demand
)

# Nightly Profile Deployment
nightly_profile_all_clients.deploy(
    name="nightly-profiles",
    work_pool_name="cdp-workers",
    tags=["production", "scheduled", "batch"],
    description="Nightly batch profiling for all active clients",
    version="1.0.0",
    cron="0 2 * * *",  # Daily at 2 AM UTC
    parameters={
        "client_ids": ["70132", "70210", "70045"]
    }
)
```

### Triggering Deployments

#### Via UI
1. Go to Prefect UI (http://localhost:4200)
2. Navigate to Deployments
3. Click "Run" on a deployment

#### Via API

```python
from prefect import get_client

async with get_client() as client:
    # Trigger deployment
    flow_run = await client.create_flow_run_from_deployment(
        deployment_id="your-deployment-id",
        parameters={"param1": "value"}
    )
    print(f"Flow run created: {flow_run.id}")
```

#### Via CLI

```bash
# Run a deployment
prefect deployment run "my-workflow/my-deployment"

# Run with parameters
prefect deployment run "my-workflow/my-deployment" --param param1=value
```

---

## Work Pools & Workers

### 💡 The Simple Explanation

**What are Workers in Prefect? (plain English)**

A worker is a long-running process that waits for Prefect to give it work and then runs your flow code.

That’s it. No magic.

**If Prefect were a company:**
*   **Prefect Server** = Manager
*   **Deployment** = Job description
*   **Work pool** = Team
*   **Worker** = Employee
*   **Flow run** = Task assigned to employee

**Why workers exist at all**
Prefect does not execute your code. It only:
1.  Knows what should run
2.  Knows when it should run
3.  Knows where it can run

Workers are the **“where”**.

**Work Pool vs Worker (this is crucial)**

| Component | Description | Answers... |
| :--- | :--- | :--- |
| 🧠 **Work Pool** | A label + configuration. | What kind of execution is allowed? On what infrastructure? With what runtime behavior? |
| 💪 **Worker** | A running process that belongs to a work pool. | "Any jobs for my pool?" (Polls Prefect Server), Executes flow runs, Sends status/logs back. |

> [!IMPORTANT]
> A work pool itself does nothing. No worker = nothing runs.

**How they work together (runtime flow)**
```mermaid
graph TD
    A[Deployment] -->|says 'run in pool: cdp-workers'| B[Work Pool: cdp-workers]
    B -->|waits| C[Worker]
    C -->|picks job| D[Flow Run Executes]
```

**Why pools + workers are separated**
Because Prefect wants:
*   Multiple workers per pool
*   Dynamic scaling
*   Multiple execution styles
*   Clean separation of concerns

This is very different from cron.

---

### What are Work Pools?

A **work pool** is a group of workers that can execute similar work. Think of it as a "job queue" or "worker pool".

### Creating Work Pools

```bash
# Create a work pool
prefect work-pool create cdp-workers --type process

# List work pools
prefect work-pool ls

# Inspect work pool
prefect work-pool inspect cdp-workers
```

### Work Pool Types (Demystified)

1️⃣ **Process** (default, simplest)
`prefect work-pool create cdp-workers --type process`
*   **What happens:** Worker runs on a VM; Flow runs as a new OS process; Uses local Python + environment.
*   **Use when:** Same machine, Simple ETL, Easy setup.
*   💡 *This is usually where people start.*

2️⃣ **Thread**
`--type thread`
*   **What happens:** Runs inside same Python process; Uses threads.
*   **Pros:** Faster startup.
*   **Cons:** No isolation, Python GIL.
*   ⚠️ *Not recommended for production ETL.*

3️⃣ **Docker**
`--type docker`
*   **What happens:** Worker launches a Docker container per run; Uses Docker image; Clean isolation.
*   **Use when:** Different dependencies, Cleaner builds, CI/CD friendly.
*   🔥 *Very popular in production.*

4️⃣ **Kubernetes**
`--type kubernetes`
*   **What happens:** Each flow run = Kubernetes Pod; Auto-scaling; Resource limits.
*   **Use when:** High scale, Enterprise, Multi-tenant CDP.

5️⃣ **ECS / Azure / Cloud containers**
*   Same idea as Docker/K8s, but cloud-native.

### What are Workers?

A **worker** is a lightweight service that:
1. Polls a work pool for tasks
2. Executes tasks when available
3. Reports status back to Prefect server

**Starting a worker (what this REALLY does)**
`prefect worker start --pool cdp-workers`

This:
*   Starts a daemon
*   Connects to Prefect Server
*   Waits forever
*   Executes jobs when assigned

You usually run this as a system service, Docker container, or Kubernetes deployment.

**Multiple workers = parallelism**
`prefect worker start --pool cdp-workers &`
`prefect worker start --pool cdp-workers &`

Now you have 2 workers, meaning 2 flow runs can execute simultaneously. This is how you scale Prefect.

**Where workers run (important)**
Workers run:
*   ❌ NOT inside Prefect Server
*   ❌ NOT inside FastAPI
*   ✅ On machines with: Access to data, Credentials, Compute.

> [!TIP]
> Think: “Where do I want my jobs to run?” That’s where workers go.

**How FastAPI fits in (quick recap)**
*   **FastAPI:** Triggers deployments, Never runs flows.
*   **Workers:** Do all the heavy work.

**One diagram that locks it in**
```mermaid
graph TD
    A[Prefect Cloud/Server] -->|assigns work| B[Work Pool: cdp-workers]
    B -->|polled by| C[Worker 1]
    B -->|polled by| D[Worker 2]
    C --> E[Flow Run A]
    D --> F[Flow Run B]
```

**One-line definitions**
*   **Work Pool:** A category of execution environment.
*   **Worker:** A running service that executes flow runs.
*   **Deployment:** Instructions that say which pool to use.

**Why this design is powerful:**
*   **Scale workers independently**
*   **Multiple pools for different workloads**
*   **Clean isolation**
*   **No tight coupling to FastAPI**
*   **Production-grade control**

---

### Starting Workers

```bash
# Start a worker for a specific pool
prefect worker start --pool cdp-workers

# Start with specific work queue
prefect worker start --pool cdp-workers --work-queue default

# Start multiple workers (for parallel execution)
prefect worker start --pool cdp-workers &
prefect worker start --pool cdp-workers &
```

### Worker Configuration

Workers can be configured via environment variables:

```bash
# Set work pool
export PREFECT_WORK_POOL_NAME="cdp-workers"

# Set work queue
export PREFECT_WORK_QUEUE_NAME="default"

# Set concurrency (max parallel tasks)
export PREFECT_WORKER_CONCURRENCY=4
```

### Work Queues

**Work queues** filter which tasks a worker executes.

#### 💡 The Simple Explanation

**Short definition (plain English)**
A work queue is a filter inside a work pool that decides which jobs a worker can pick up.

**Analogy:**
*   Work pool = Team
*   Worker = Employee
*   **Work queue = Task category**

**Why work queues exist**
Imagine you have the same infrastructure but different priorities or types of jobs. You don’t want every worker to pick up every job. Work queues let you say: *“These workers only do THIS kind of work.”*

**Hierarchy (important)**
`Deployment` → `Work Pool` → `Work Queue` → `Worker` → `Flow Run`

**Concrete Example:**
1.  **Create work pool:** `prefect work-pool create cdp-workers --type process`
2.  **Create work queues:** 
    *   `prefect work-queue create high-priority --pool cdp-workers`
    *   `prefect work-queue create low-priority --pool cdp-workers`
3.  **Deployment chooses a queue:**
    ```python
    Deployment.build_from_flow(
        flow=nightly_etl,
        name="nightly-etl",
        work_pool_name="cdp-workers",
        work_queue_name="low-priority"
    )
    ```
4.  **Worker listens to a queue:**
    `prefect worker start --pool cdp-workers --work-queue high-priority`
    *(That worker will ignore low-priority jobs)*

**Why queues matter (real-world reasons):**
1.  **Priority handling:** Urgent jobs → high-priority.
2.  **Resource control:** Heavy jobs → fewer workers; Light jobs → more workers.
3.  **Isolation:** Risky jobs separated; Experimental flows isolated.
4.  **SLA guarantees:** Customer-facing jobs never blocked by batch ETL.

**What queues are NOT:**
*   ❌ Not a message broker (not Kafka/RabbitMQ)
*   ❌ Not task-level queues
*   ❌ Not required for beginners

**When you actually need work queues:**
*   **You DON’T need them if:** Single worker, single type of job. Just use the `default` queue.
*   **You DO need them if:** Multiple job types, priority differences, shared infra, SLA concerns.

**Mapping to things you already know:**

| Prefect | Equivalent |
| :--- | :--- |
| Work Pool | Node group / Infra type |
| Work Queue | Priority queue |
| Worker | Executor |
| Deployment | Job config |

> [!TIP]
> **One-sentence summary:** Work pools decide **where** jobs can run; work queues decide **which** jobs a worker should pick.

**Typical beginner mistake:**
Creating too many pools or queues. Start simple: 1 pool, `default` queue, 1–2 workers. Add queues only when needed.

---

**Use Cases:**
- Priority: High-priority tasks vs. low-priority tasks
- Environment: Production vs. development workers
- Resource: CPU-intensive vs. I/O-intensive tasks

### Real-World Setup

**CDP-Profiler Production Setup:**

```python
# 1. Create work pool
prefect work-pool create cdp-workers --type process

# 2. Create work queue
prefect work-queue create default --pool cdp-workers

# 3. Deploy flows to work pool
client_pipeline.deploy(
    name="on-demand-profile",
    work_pool_name="cdp-workers"  # Assigns to work pool
)

# 4. Start workers (in production, runs as service)
prefect worker start --pool cdp-workers
```

---

## Advanced Patterns

### Parallel Execution

#### Using `.submit()` for Parallel Tasks

```python
from prefect import flow, task
from prefect.task_runners import ConcurrentTaskRunner

@task
def task_a():
    time.sleep(5)
    return "A"

@task
def task_b():
    time.sleep(5)
    return "B"

@flow(task_runner=ConcurrentTaskRunner())
def parallel_flow():
    # Submit tasks (non-blocking)
    future_a = task_a.submit()
    future_b = task_b.submit()
    
    # Get results (blocks until complete)
    result_a = future_a.result()
    result_b = future_b.result()
    
    return f"{result_a} + {result_b}"
```

**Execution Time:** ~5 seconds (parallel) vs. ~10 seconds (sequential)

#### Using `.map()` for Batch Processing

```python
@task
def process_item(item: str):
    """Process a single item"""
    return f"Processed {item}"

@flow
def batch_flow():
    items = ["item1", "item2", "item3", "item4"]
    
    # Process all items in parallel
    results = process_item.map(items)
    
    return results
```

**Use Case:** Process multiple clients, files, or batches in parallel.

### Real Example: Parallel Insights Generation

```python
@flow(task_runner=ConcurrentTaskRunner())
def full_profile_workflow(client_id: str):
    # Sequential steps
    ingest_result = ingest_client_data(client_id)
    profile_result = profile_client_data(client_id)
    
    # Parallel steps (run simultaneously)
    insights_future = generate_insights_task.submit(client_id)
    semantic_future = generate_semantic_index_task.submit(client_id)
    
    # Wait for both to complete
    insights_result = insights_future.result()
    semantic_result = semantic_future.result()
    
    return {
        "insights": insights_result,
        "semantic": semantic_result
    }
```

### Error Handling

#### Task-Level Error Handling

```python
@task(retries=2)
def unreliable_task():
    """Task with built-in retries"""
    if random.random() < 0.5:
        raise Exception("Random failure")
    return "success"

@flow
def resilient_flow():
    """Flow that handles task failures"""
    try:
        result = unreliable_task()
        return result
    except Exception as e:
        print(f"Task failed after retries: {e}")
        # Flow can continue or fail
        return None
```

#### Flow-Level Error Handling

```python
@flow
def workflow_with_error_handling():
    """Flow with comprehensive error handling"""
    try:
        result = risky_task()
        return result
    except ValueError as e:
        print(f"Value error: {e}")
        # Handle specific error type
        return None
    except Exception as e:
        print(f"Unexpected error: {e}")
        # Handle all other errors
        raise  # Re-raise to fail flow
```

### Conditional Execution

```python
@flow
def conditional_workflow(client_id: str, skip_ingestion: bool = False):
    """Flow with conditional steps"""
    
    # Check if data exists
    data_exists = check_parquet_files_exist(client_id)
    
    # Conditional ingestion
    if not skip_ingestion and not data_exists:
        ingest_result = ingest_client_data(client_id)
    else:
        print("Skipping ingestion")
    
    # Always profile
    profile_result = profile_client_data(client_id)
    
    return profile_result
```

### Loops in Flows

```python
@flow
def batch_process_clients(client_ids: list[str]):
    """Process multiple clients sequentially"""
    results = []
    
    for client_id in client_ids:
        result = profile_client(client_id)
        results.append(result)
    
    return results

# Or process in parallel
@flow(task_runner=ConcurrentTaskRunner())
def batch_process_clients_parallel(client_ids: list[str]):
    """Process multiple clients in parallel"""
    # Use map for parallel execution
    results = profile_client.map(client_ids)
    return results
```

### Passing Data Between Tasks

#### ⚠️ Important: Parameter Size Limit

**❌ BAD - Passing Large Data:**

```python
@task
def load_data():
    return pd.read_csv("large_file.csv")  # 100MB dataframe

@task
def process_data(df):
    return df.groupby("category").sum()  # Receives 100MB

@flow
def bad_flow():
    df = load_data()  # 100MB serialized
    result = process_data(df)  # ERROR: Exceeds 512KB limit!
```

**✅ GOOD - Passing Paths:**

```python
@task
def load_data():
    df = pd.read_csv("large_file.csv")
    # Save to disk/ADLS
    path = "cdp/client123/data.parquet"
    df.to_parquet(path)
    return path  # Pass path, not data

@task
def process_data(path):
    df = pd.read_parquet(path)  # Load from disk
    result = df.groupby("category").sum()
    result.to_parquet("cdp/client123/processed.parquet")
    return "cdp/client123/processed.parquet"  # Return path

@flow
def good_flow():
    path = load_data()  # Returns path string
    result_path = process_data(path)  # Works!
```

**CDP-Profiler Pattern:**

```python
@task
def ingest_client_data(client_id: str):
    """Ingest and save to ADLS"""
    # ... ingestion logic ...
    path = f"cdp/{client_id}/profiles/data.parquet"
    save_to_adls(data, path)
    return path  # Return path, not data

@task
def profile_client_data(client_id: str, data_path: str):
    """Profile data from ADLS"""
    data = load_from_adls(data_path)  # Load when needed
    metrics = calculate_metrics(data)
    metrics_path = f"cdp/{client_id}/metrics/metrics.parquet"
    save_to_adls(metrics, metrics_path)
    return metrics_path  # Return path

@flow
def workflow(client_id: str):
    data_path = ingest_client_data(client_id)
    metrics_path = profile_client_data(client_id, data_path)
    return metrics_path
```

### Caching

```python
from prefect import task
from datetime import timedelta

@task(cache_key_fn=lambda: "static-key", cache_expiration=timedelta(hours=1))
def expensive_computation():
    """Task that caches results"""
    print("Computing...")  # Only prints on first run
    return {"result": "expensive"}
```

**Use Cases:**
- Expensive API calls
- Data transformations that don't change
- Computations with same inputs

---

## Best Practices

### 1. Keep Tasks Focused

**❌ BAD:**
```python
@task
def do_everything():
    fetch_data()
    transform_data()
    save_data()
    send_notification()
    generate_report()
```

**✅ GOOD:**
```python
@task
def fetch_data():
    return data

@task
def transform_data(data):
    return transformed

@task
def save_data(data):
    save(data)

@flow
def workflow():
    data = fetch_data()
    transformed = transform_data(data)
    save_data(transformed)
```

### 2. Use Retries for Unreliable Operations

```python
# APIs, databases, network calls
@task(retries=3, retry_delay_seconds=30)
def api_call():
    response = requests.get("https://api.example.com")
    return response.json()

# File operations (less reliable)
@task(retries=2, retry_delay_seconds=10)
def read_file(path):
    return pd.read_csv(path)
```

### 3. Add Timeouts

```python
@task(timeout_seconds=300)  # 5 minutes max
def long_running_task():
    # Task will be cancelled if it takes > 5 minutes
    process_large_dataset()
```

### 4. Use Tags for Organization

```python
@task(tags=["ingestion", "heavy"])
def heavy_ingestion():
    pass

@task(tags=["profiling", "cpu-intensive"])
def cpu_profiling():
    pass

# Filter in UI: "Show only ingestion tasks"
```

### 5. Don't Pass Large Data

**Always pass paths/IDs, not dataframes:**

```python
# ✅ GOOD
@task
def save_data(data):
    path = "cdp/client/data.parquet"
    data.to_parquet(path)
    return path  # Return path

@task
def process_data(path):
    data = pd.read_parquet(path)  # Load when needed
    return process(data)
```

### 6. Use Logging

```python
@task(log_prints=True)
def task_with_logging():
    print("Starting task...")  # Appears in UI
    logger.info("Processing data")  # Also use logging module
    print("Task complete!")
```

### 7. Handle Errors Gracefully

```python
@flow
def resilient_workflow():
    try:
        result = risky_task()
        return result
    except SpecificError as e:
        # Handle specific error
        fallback_task()
    except Exception as e:
        # Handle all other errors
        log_error(e)
        raise  # Or return default value
```

### 8. Use Descriptive Names

```python
# ✅ GOOD
@flow(name="client-profile-pipeline")
def profile_client():
    pass

@task(name="ingest-client-data")
def ingest():
    pass

# ❌ BAD
@flow(name="flow1")
def f():
    pass
```

### 9. Document Your Workflows

```python
@flow(name="client-profile-pipeline")
def full_profile_workflow(
    client_id: str,
    sample_size: int = 10000
):
    """
    Complete profiling workflow for a client.
    
    Steps:
    1. Ingest data from OLTP to ADLS
    2. Profile data and calculate metrics
    3. Generate AI insights
    
    Args:
        client_id: Client identifier
        sample_size: Sample size for profiling
        
    Returns:
        Dict with profiling results
    """
    # ... implementation
```

### 10. Test Locally First

```python
# Test flow locally before deploying
if __name__ == "__main__":
    result = my_workflow(param1="test")
    print(f"Result: {result}")
```

---

## Real-World Examples

### Example 1: Data Ingestion Pipeline

```python
from prefect import flow, task
from prefect.task_runners import ConcurrentTaskRunner

@task(retries=2, retry_delay_seconds=60, log_prints=True)
def connect_to_database(client_id: str):
    """Connect to client's OLTP database"""
    print(f"Connecting to database for client {client_id}")
    # ... connection logic
    return connection

@task(retries=2, retry_delay_seconds=60)
def export_table(connection, table_name: str, output_path: str):
    """Export table to parquet"""
    print(f"Exporting {table_name} to {output_path}")
    # ... export logic
    return output_path

@task
def upload_to_adls(local_path: str, adls_path: str):
    """Upload parquet to Azure Data Lake"""
    print(f"Uploading {local_path} to {adls_path}")
    # ... upload logic
    return adls_path

@flow(name="data-ingestion-pipeline")
def ingestion_pipeline(client_id: str):
    """
    Complete data ingestion pipeline
    
    1. Connect to database
    2. Export tables
    3. Upload to ADLS
    """
    # Step 1: Connect
    connection = connect_to_database(client_id)
    
    # Step 2: Export tables (parallel)
    tables = ["profiles", "purchases", "events"]
    paths = []
    
    for table in tables:
        local_path = export_table(connection, table, f"temp/{table}.parquet")
        adls_path = upload_to_adls(local_path, f"cdp/{client_id}/{table}.parquet")
        paths.append(adls_path)
    
    print(f"✓ Ingestion complete: {len(paths)} tables uploaded")
    return paths
```

### Example 2: Profiling with Parallel Processing

```python
@task
def load_data(path: str):
    """Load data from ADLS"""
    return pd.read_parquet(path)

@task
def calculate_basic_stats(df):
    """Calculate basic statistics"""
    return df.describe()

@task
def calculate_correlations(df):
    """Calculate correlations"""
    return df.corr()

@task
def detect_patterns(df):
    """Detect data patterns"""
    patterns = {}
    for col in df.columns:
        if df[col].dtype == 'object':
            patterns[col] = detect_email_phone_patterns(df[col])
    return patterns

@flow(task_runner=ConcurrentTaskRunner())
def profile_data(client_id: str):
    """
    Profile data with parallel metric calculation
    """
    # Load data
    data_path = f"cdp/{client_id}/profiles/data.parquet"
    df = load_data(data_path)
    
    # Calculate metrics in parallel
    stats_future = calculate_basic_stats.submit(df)
    correlations_future = calculate_correlations.submit(df)
    patterns_future = detect_patterns.submit(df)
    
    # Wait for all to complete
    stats = stats_future.result()
    correlations = correlations_future.result()
    patterns = patterns_future.result()
    
    # Save results
    save_metrics(client_id, {
        "stats": stats,
        "correlations": correlations,
        "patterns": patterns
    })
    
    return "completed"
```

### Example 3: Batch Processing Multiple Clients

```python
@flow
def profile_single_client(client_id: str):
    """Profile a single client"""
    # ... profiling logic
    return {"client_id": client_id, "status": "completed"}

@flow(task_runner=ConcurrentTaskRunner())
def batch_profile_clients(client_ids: list[str]):
    """
    Profile multiple clients in parallel
    """
    # Process all clients in parallel
    results = profile_single_client.map(client_ids)
    
    # Wait for all to complete
    completed = [r for r in results if r["status"] == "completed"]
    failed = [r for r in results if r["status"] == "failed"]
    
    print(f"✓ Completed: {len(completed)}")
    print(f"✗ Failed: {len(failed)}")
    
    return {
        "completed": len(completed),
        "failed": len(failed),
        "results": results
    }
```

---

## Troubleshooting

### Common Issues

#### 1. "Database is locked" (SQLite)

**Problem:** SQLite database locked when running Prefect server.

**Solution:**
```bash
# Restart Prefect server
prefect server stop
prefect server start

# Or use PostgreSQL for production
export PREFECT_API_DATABASE_CONNECTION_URL="postgresql://user:pass@localhost/prefect"
```

#### 2. Flow Parameters Too Large

**Problem:** `ValueError: Flow parameters exceed 512KB limit`

**Solution:**
```python
# ❌ BAD - Passing large dataframe
@flow
def bad_flow(df):
    process(df)  # ERROR: df is too large

# ✅ GOOD - Pass path instead
@flow
def good_flow(data_path: str):
    df = pd.read_parquet(data_path)  # Load when needed
    process(df)
```

#### 3. Tasks Not Running

**Problem:** Tasks stuck in PENDING state.

**Solution:**
```bash
# Check if worker is running
prefect worker ls

# Start worker if not running
prefect worker start --pool cdp-workers

# Check work pool
prefect work-pool inspect cdp-workers
```

#### 4. Retries Not Working

**Problem:** Task fails but doesn't retry.

**Solution:**
```python
# Make sure retries are set
@task(retries=3, retry_delay_seconds=30)
def my_task():
    # Task logic
    pass

# Check if exception is being caught
@task(retries=3)
def my_task():
    try:
        risky_operation()
    except Exception as e:
        # Don't catch exceptions if you want retries
        raise  # Re-raise to trigger retry
```

#### 5. Timeout Not Working

**Problem:** Task exceeds timeout but doesn't stop.

**Solution:**
```python
# Timeout only works if task is cancellable
@task(timeout_seconds=60)
def my_task():
    # Long-running operation
    # Make sure it can be interrupted
    for i in range(1000000):
        process(i)  # This might not be cancellable
```

#### 6. Print Statements Not Appearing

**Problem:** `print()` statements don't show in logs.

**Solution:**
```python
# Enable log_prints
@task(log_prints=True)
def my_task():
    print("This will appear in logs!")
    
# Or use logging module
import logging
logger = logging.getLogger(__name__)

@task
def my_task():
    logger.info("This will appear in logs!")
```

### Debugging Tips

1. **Check Prefect UI** - http://localhost:4200
   - View flow runs
   - Check task states
   - Read logs

2. **Enable Verbose Logging**
```python
import logging
logging.basicConfig(level=logging.DEBUG)
```

3. **Test Tasks Individually**
```python
# Test task without flow
@task
def my_task(param):
    return param * 2

# Test directly
result = my_task("test")
print(result)
```

4. **Check Worker Logs**
```bash
# Worker logs show task execution
prefect worker start --pool cdp-workers --verbose
```

---

## Prefect in CDP-Profiler

### Architecture Overview (Production)

In production, we separate the **Orchestration Layer** (the brain) from the **Compute Layer** (the hands).

```mermaid
graph TD
    subgraph "Orchestration Layer (Managed)"
        A["Prefect Cloud / Self-Hosted Server"]
    end

    subgraph "Compute Layer (Your Cloud/Instance)"
        B["FastAPI (cdp-api)"]
        C["Prefect Worker (cdp-worker)"]
    end

    subgraph "Data Layer (Azure)"
        D["Azure Data Lake Storage (ADLS)"]
        E["Azure SQL / Synapse"]
    end

    B -->|triggers| A
    A -->|assigns work| C
    C -->|reads/writes| D
    C -->|reads| E
    B -->|serves metrics| D
```

### Production Components

| Component | Responsibility | Scaling Strategy |
| :--- | :--- | :--- |
| **Prefect Cloud** | UI, Scheduling, Logs, States | Zero (SaaS) |
| **API (`cdp-api`)** | Receives UI requests, triggers flows | Scale horizontally (multiple instances) |
| **Worker (`cdp-worker`)** | Executes heavy data jobs (Ingestion, AI) | Scale vertically (High RAM) or multiple workers |

#### Why separate API and Worker?
1.  **Isolation:** If a massive data profiling job crashes the **Worker**, your **API** stays alive to serve other users.
2.  **Efficiency:** API server stays "thin" (low cost), while Worker server can be "beefy" (high compute) and potentially scaled down when idle.

---

### Key Workflows

#### 1. Full Profile Workflow

**File:** `src/cdp/flows/client_pipeline.py`

```python
@flow(name="client-profile-pipeline")
def full_profile_workflow(
    client_id: str,
    sample_size: int = 10000,
    skip_ingestion: bool = False
):
    """
    Complete profiling workflow:
    1. Ingest data (if needed)
    2. Profile data
    3. Generate insights (parallel)
    4. Generate semantic index (parallel)
    """
    # Check if data exists
    data_exists = check_parquet_files_exist(client_id)
    
    # Ingest if needed
    if not skip_ingestion or not data_exists:
        ingest_result = ingest_client_data(client_id)
    
    # Profile data
    profile_result = profile_client_data(client_id, sample_size)
    
    # Generate insights in parallel
    insights_future = generate_insights_task.submit(client_id)
    semantic_future = generate_semantic_index_task.submit(client_id)
    
    insights_result = insights_future.result()
    semantic_result = semantic_future.result()
    
    return {"status": "completed"}
```

#### 2. CSV Upload Workflow

**File:** `src/cdp/flows/csv_upload_pipeline.py`

```python
@flow(name="csv-upload-pipeline")
def csv_upload_workflow(file_path: str, client_id: str):
    """
    CSV upload and processing:
    1. Validate CSV
    2. Convert to parquet
    3. Upload to ADLS
    4. Trigger profiling
    """
    # Validate
    validation_result = validate_csv(file_path)
    
    # Ingest
    adls_path = ingest_csv_to_adls(file_path, client_id)
    
    # Profile
    profile_result = profile_client_data(client_id)
    
    return {"status": "completed", "adls_path": adls_path}
```

#### 3. Nightly Batch Workflow

**File:** `src/cdp/flows/client_pipeline.py`

```python
@flow(name="nightly-profile-all-clients")
def nightly_profile_all_clients(client_ids: list[str]):
    """
    Nightly batch profiling for all clients
    Scheduled: Daily at 2 AM UTC
    """
    results = []
    
    for client_id in client_ids:
        result = full_profile_workflow(
            client_id=client_id,
            sample_size=10000,
            skip_ingestion=False
        )
        results.append(result)
    
    return results
```

### Deployment Configuration

**File:** `src/cdp/flows/deployments.py`

```python
def deploy_all():
    """Deploy all workflows"""
    
    # On-demand profile
    client_pipeline.deploy(
        name="on-demand-profile",
        work_pool_name="cdp-workers",
        tags=["production", "on-demand"],
        cron=None  # Triggered via API
    )
    
    # Nightly batch
    nightly_profile_all_clients.deploy(
        name="nightly-profiles",
        work_pool_name="cdp-workers",
        tags=["production", "scheduled"],
        cron="0 2 * * *",  # Daily at 2 AM UTC
        parameters={"client_ids": ["70132", "70210"]}
    )
```

### Triggering Workflows from FastAPI

```python
from prefect import get_client

@app.post("/api/profiler/run")
async def trigger_profiling(client_id: str):
    """Trigger profiling workflow via API"""
    
    async with get_client() as client:
        # Find deployment
        deployments = await client.read_deployments()
        deployment = next(d for d in deployments if d.name == "on-demand-profile")
        
        # Create flow run
        flow_run = await client.create_flow_run_from_deployment(
            deployment_id=deployment.id,
            parameters={"client_id": client_id}
        )
        
        return {
            "flow_run_id": str(flow_run.id),
            "status": "triggered"
        }
```

### Key Patterns Used

1. **Path-Based Data Passing** - Always pass ADLS paths, not dataframes
2. **Retry Logic** - All ingestion tasks have retries (database timeouts)
3. **Parallel Execution** - Insights and semantic index generated in parallel
4. **Conditional Execution** - Skip ingestion if data already exists
5. **Error Handling** - Comprehensive error handling at task and flow level

---

## Next Steps

### Learning Path

1. **Start Simple** - Create a basic flow with 2-3 tasks
2. **Add Retries** - Add retry logic to unreliable tasks
3. **Parallel Execution** - Use `.submit()` and `.map()` for parallel tasks
4. **Deploy** - Create a deployment and schedule it
5. **Monitor** - Use Prefect UI to monitor workflow runs
6. **Advanced** - Explore caching, subflows, and error handling

### Resources

- **Prefect Docs:** https://docs.prefect.io/
- **Prefect UI:** http://localhost:4200 (when server running)
- **Prefect Community:** https://discourse.prefect.io/
- **CDP-Profiler Docs:** See `docs/profiler-docs/` directory

### Practice Exercises

1. **Exercise 1:** Create a flow that fetches data from an API and saves it to a file
2. **Exercise 2:** Add retry logic to handle API failures
3. **Exercise 3:** Create a flow that processes multiple files in parallel
4. **Exercise 4:** Deploy a flow with a cron schedule
5. **Exercise 5:** Create a subflow for reusable logic

---

## Summary

### Key Takeaways

1. **Flows** orchestrate tasks - Use `@flow` decorator
2. **Tasks** are atomic units - Use `@task` decorator
3. **Deployments** schedule flows - Use `.deploy()` or `prefect.yaml`
4. **Workers** execute tasks - Start workers to process work
5. **States** track execution - Monitor in Prefect UI
6. **Retries** handle failures - Set `retries` parameter
7. **Parallel** execution - Use `.submit()` and `.map()`
8. **Don't pass large data** - Pass paths/IDs instead

### Prefect vs. Alternatives

| Feature | Prefect | Airflow | Luigi |
|---------|---------|---------|-------|
| Python-native | ✅ | ⚠️ | ✅ |
| Learning curve | Easy | Steep | Medium |
| UI | Excellent | Good | Basic |
| Retries | Built-in | Built-in | Manual |
| Scheduling | Flexible | Cron-based | Limited |

**Prefect is ideal for:**
- Python-first teams
- Modern data pipelines
- Fast iteration
- Beautiful observability

---

**Happy Orchestrating! 🚀**

For questions or issues, check:
- Prefect UI: http://localhost:4200
- Prefect Docs: https://docs.prefect.io/
- CDP-Profiler Issues: See project repository

