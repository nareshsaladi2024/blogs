# OIC Error Hospital: Architecture, Agents, and Database Schema

**How we built a resilient Oracle Integration Cloud (OIC) error management hub**  
*By Naresh Saladi — August 14, 2025*

> This post distills a working implementation described in the internal brief "OIC Error Hospital" and turns it into an end‑to‑end guide you can share with engineering and operations teams.

## What is the OIC Error Hospital?

An operational hub that continuously **collects, stores, reconciles, and resubmits** Oracle Integration Cloud (OIC) errors so teams can trace failures over time and act on them quickly.

### Key capabilities

- Collect all OIC Integration errors **every 15 minutes**
- Persist error details to **ATP** for historical analysis
- Reconcile **resubmitted transactions** (completed/failed) hourly
- Support **bulk resubmission** with optional throttling

## Agent Architecture (High Level)

Two scheduled agents coordinate error lifecycle management:

1. **Error Collection Agent** — polls OIC for both *recoverable and non‑recoverable* integration errors and logs them to the ATP store.
2. **Error Reconcile Agent** — inspects recovery jobs and updates each instance’s lifecycle status (completed / failed), enabling dashboards and automated follow‑ups.

---

## Error Collection Agent

- **Integration Name:** *IT SERVICES OIC ERROR COLLECTION AGENT*  
- **Style:** Scheduled Event Integration  
- **Identifier:** `IT_SERVI_OIC_ERROR_COLLE_AGENT`  
- **Revision:** 1.0  
- **Schedule:** Every **15 minutes**  
- **Objective:** Capture OIC errors (recoverable & non‑recoverable) for all OIC integrations and persist with rich context (tracking vars, endpoints, status, timestamps).

> Typical flow: *Schedule trigger → Query OIC runtime errors → Normalize → Upsert into ATP → Tag environment & endpoint metadata.*

## Error Reconcile Agent

- **Integration Name:** *IT SERVICES OIC ERROR RECONCILE AGENT*  
- **Style:** Scheduled Event Integration  
- **Identifier:** `IT_SERVI_OIC_ERROR_RECON_AGENT`  
- **Revision:** 1.0  
- **Schedule:** Every **1 hour**  
- **Objective:** Track recovery jobs for previously captured errors and **reconcile** their outcome, enabling SLA reporting and targeted retries.

> Typical flow: *Schedule trigger → Fetch recovery job status → Update instance lifecycle → Flag completed vs. failed → Emit metrics.*

---

## Data Model (ATP)

Below is the core table that powers historical analysis and reconciliation joins.

```sql
CREATE TABLE "OIC_ERROR_HOSPITAL" 
(
 "INSTANCE_ID" VARCHAR2(100 BYTE) COLLATE "USING_NLS_COMP" NOT NULL ENABLE,
 "INTEGRATION_NAME" VARCHAR2(100 BYTE) COLLATE "USING_NLS_COMP" NOT NULL ENABLE,
 "INTEGRATION_CODE" VARCHAR2(100 BYTE) COLLATE "USING_NLS_COMP" NOT NULL ENABLE,
 "INTEGRATION_ID" VARCHAR2(100 BYTE) COLLATE "USING_NLS_COMP" NOT NULL ENABLE,
 "INTEGRATION_VERSION" VARCHAR2(100 BYTE) COLLATE "USING_NLS_COMP" NOT NULL ENABLE,
 "TRACKING_VAR_1" VARCHAR2(2000 BYTE) COLLATE "USING_NLS_COMP",
 "TRACKING_VAR_2" VARCHAR2(2000 BYTE) COLLATE "USING_NLS_COMP",
 "TRACKING_VAR_3" VARCHAR2(2000 BYTE) COLLATE "USING_NLS_COMP",
 "ENVIRONMENT" VARCHAR2(100 BYTE) COLLATE "USING_NLS_COMP",
 "ENDPOINT_NAME" VARCHAR2(100 BYTE) COLLATE "USING_NLS_COMP",
 "ENDPOINT_OPERATION" VARCHAR2(100 BYTE) COLLATE "USING_NLS_COMP",
 "ENDPOINT_TYPE" VARCHAR2(100 BYTE) COLLATE "USING_NLS_COMP",
 "STATUS" VARCHAR2(100 BYTE) COLLATE "USING_NLS_COMP",
 "CREATED_DT" TIMESTAMP (3) WITH LOCAL TIME ZONE NOT NULL ENABLE,
 "MODIFIED_DT" TIMESTAMP (3) WITH LOCAL TIME ZONE,
 "ERROR_TIME" TIMESTAMP (3) WITH LOCAL TIME ZONE,
 "RECOVERABLE" VARCHAR2(100 BYTE) COLLATE "USING_NLS_COMP",
 "RETRY_COUNT" NUMBER(2,0),
 "FAULT_LOCATION" VARCHAR2(2000 BYTE) COLLATE "USING_NLS_COMP",
 "ERROR_CODE" VARCHAR2(2000 BYTE) COLLATE "USING_NLS_COMP",
 "ERROR_DETAILS" CLOB COLLATE "USING_NLS_COMP",
 "ERROR_LOCATION" VARCHAR2(2000 BYTE) COLLATE "USING_NLS_COMP",
 "ERROR_MESSAGE" CLOB COLLATE "USING_NLS_COMP",
 "ERROR_ITEMS" CLOB COLLATE "USING_NLS_COMP",
 "CAPTURE_DT" TIMESTAMP (3) WITH LOCAL TIME ZONE NOT NULL ENABLE,
 PRIMARY KEY ("INSTANCE_ID")
);
```

### Practical tips

- **Throttled resubmission:** batch retry in small bursts to protect downstreams; backoff on repeated failures.
- **Observability:** emit metrics for counts, latency to recover, and % auto‑healed; alert on spikes in *non‑recoverable* errors.
- **Traceability:** store `TRACKING_VAR_*` to relate failed instances back to business entities.
- **Environment tagging:** keep `ENVIRONMENT` normalized (e.g., `DEV`, `TEST`, `UAT`, `PROD`).

## Scheduling Strategy

- Collection every **15 minutes** balances near‑real‑time signal with OIC limits.
- Reconciliation hourly keeps state fresh without overloading the OIC recovery APIs.
- For peak windows, shorten intervals temporarily and increase throttling.

## Closing Thoughts

With just two scheduled agents and a simple but rich schema, the OIC Error Hospital turns scattered transient errors into **actionable, auditable** operational data and automated recoveries.
