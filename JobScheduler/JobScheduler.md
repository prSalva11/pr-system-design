# Job Scheduler Design

## Problem Statement
Design a **Job Scheduler** that can handle scheduling and execution of tasks at scale.

---

## Functional Requirements
1. The system should be able to **schedule millions of tasks per day**.
2. Support **flexible scheduling**:
   - Tasks that run **immediately**.
   - Tasks that run at a **specific time**.
3. Provide **task prioritization** to ensure critical jobs are executed first.
4. Handle **different types of tasks**, such as:
   - Email notifications
   - File processing
   - Error retries
   - Other background jobs
5. Support **cancellation and rescheduling** of tasks.
6. Provide **retry mechanism** for transient failures.

---

## Non-Functional Requirements
1. **High Availability** – The system must not have a single point of failure.
2. **Scalability** – Must scale with growth in both **task volume and execution load**.
3. **Consistency** – Ensure **idempotency** so tasks are not processed multiple times.
4. **Replayability** – Ability to **replay jobs** for auditing or recovery.
5. **Security** – 
   - Encrypt **PII data in transit and at rest**.
   - Services should not be directly exposed to the public.
   - Implement **authentication & authorization**.
6. **Auditing** – Track all job submissions, state changes, and executions.
7. **Dead Letter Queue (DLQ)** – Failed jobs should be captured for later reprocessing or analysis.
8. **Observability** – Provide **metrics, logs, and alerts** to monitor execution health.
9. **Low Latency** – Immediate tasks should be executed with minimal delay.

---

## High-Level Architecture

### Major Components

#### 1. Task Manager (Pods)
- Receives **incoming events** from the Load Balancer.
- Enhances the event model with metadata (priority, status, etc.).
- Persists the enriched task into **Cassandra**.

#### 2. Scheduler
- **Two scheduler jobs run every minute:**

  **a. Scheduler-1**
  - Pulls jobIds from Cassandra that are expected to run within the next **30 minutes**.
  - Retrieves them in a **paginated fashion**, and based on priority, publishes to different **Redis queues**.
  - Uses a **distributed lock (e.g., ShedLock)** to ensure only one scheduler publishes a given set.
  
  **b. Scheduler-2**
  - Pulls jobs from Redis that are scheduled within the **next 1 minute**.
  - Fetches job details from Cassandra.
  - Publishes jobs to **different Kafka topics** based on priority (where worker nodes consume).
  - Jobs failing DB fetch will be pushed to a **delayed retry topic** with status `DB_READ_FAILURE`.

#### 3. Worker Nodes
- Subscribed to appropriate Kafka topics (based on priority).
- Integrated with the right **processing services** (Email Service, File Service, etc.).
- For each task:
  - Builds the required payload according to service contract.
  - Publishes the payload via Kafka or API hook.
  - Performs **idempotency checks** using Cassandra (ensures only jobs in `WAITING` status are processed).
- Failed jobs:
  - Retry up to 3 times within the service.
  - On repeated failure, pushed to **delayed retry topic** with status `PUBLISH_FAILURE`.
  - If all retries fail, jobs are moved to the **DLQ** with the last status update for auditing.

---

## System Flow (End-to-End Lifecycle)

1. **Task Submission**
   - Client → Load Balancer → Task Manager.
   - Task Manager enriches job, assigns priority/status, persists into Cassandra.

2. **Pre-Scheduling (30-min Window)**
   - Scheduler-1 runs every minute.
   - Pulls jobs due in the next 30 minutes from Cassandra (paginated).
   - Pushes them into Redis queues based on priority.

3. **Ready-to-Run Scheduling (1-min Window)**
   - Scheduler-2 runs every minute.
   - Pulls jobs due in the next 1 minute from Redis.
   - Fetches job details from Cassandra.
   - Publishes them to appropriate Kafka topics based on priority.

4. **Execution**
   - Worker nodes consume from Kafka.
   - Validate idempotency using Cassandra (`WAITING` status check).
   - Forward payload to appropriate service (email, file, etc.).

5. **Failure Handling**
   - If DB fetch fails → job goes to delayed retry topic (`DB_READ_FAILURE`).
   - If service publish fails → retry 3 times.
   - If all retries fail → job sent to delayed retry topic (`PUBLISH_FAILURE`).
   - If still unprocessed after retries → job moves to DLQ with audit logs.

---

## Supporting Components

1. **Redis** – Used for **fast access** to near-ready jobIds (quick scheduling decisions).
2. **Kafka** – Used for:
   - Preventing duplicate events.
   - Scaling processing through **partitioned topics**.
3. **Cassandra** – Used for:
   - **Fast writes** (high task ingestion rate).
   - Flexibility in modeling various task types.
   - **Job table schema**:
     ```text
     job (
       jobId,
       jsonPayload,
       priority,
       creationTimeEpoch,
       scheduledTimeEpoch,
       updateTimeEpoch,
       status
     )
     ```

---

## Future Enhancements

1. **Immediate Job Optimization** – Directly publish “immediate” jobs from Task Manager to Redis/Kafka (bypassing schedulers) for sub-second latency.  
2. **Reliable Redis Handoff** – Use Cassandra markers (`ENQUEUED`) to avoid missed jobs if Scheduler-1 crashes mid-process.  
3. **DLQ Granularity** – Maintain separate DLQs for different failure types (`DB_READ_FAILURE_DLQ`, `PUBLISH_FAILURE_DLQ`).  
4. **Retry Backoff** – Replace fixed retry (3 attempts) with **exponential backoff + jitter**.  
5. **Auditing** – Add a **JobHistory table** to capture all state transitions (not just last status).  
6. **Observability** – Monitor metrics like:  
   - Scheduler lag (scheduled vs actual execution time).  
   - Redis/Kafka queue depth per priority.  
   - DLQ growth rate.  
7. **Multi-tenancy** – Add tenantId field in job schema to support multiple clients securely.  

---
