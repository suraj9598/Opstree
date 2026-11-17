# ClickHouse Error Log Segregation POC

## 1. Project Overview

### 1.1 Objective

The objective of this POC is to implement and validate a separate error-log segregation mechanism in ClickHouse using Materialized Views.

Currently, all application and Kubernetes logs are ingested into the main ClickHouse table:

```sql
observability.otel_logs
```

This table contains different types of logs, including:

- INFO
- WARN
- ERROR
- FATAL
- Application logs
- Kubernetes logs
- ClickHouse logs
- Other infrastructure logs

Searching for errors in a large table containing all types of logs can become difficult.

To address this, we implemented a separate error table:

```sql
observability.otel_error_logs
```

A ClickHouse Materialized View automatically evaluates newly inserted records in the main table and routes matching error logs into the separate error table.

The main table continues to store all incoming logs. Matching error logs are copied into the error table.

### 1.2 Goals of the POC

The main goals were:

1. Understand the existing ClickHouse log-ingestion architecture.
2. Analyze the existing OTel log schema.
3. Identify different error formats in actual ingested logs.
4. Understand why OTel severity fields alone may not be sufficient.
5. Create a separate error-log table.
6. Implement automatic error segregation using a Materialized View.
7. Support multiple error formats.
8. Identify detection gaps.
9. Generate controlled test logs for validation.
10. Test false positives.
11. Evaluate the initial feasibility of this approach.
12. Identify future performance, storage, and scalability requirements.

### 1.3 POC Scope

This POC focuses on error-log segregation, not complete error detection for every possible application and infrastructure log format.

The implementation demonstrates how selected error patterns can be detected and copied into a separate ClickHouse table.

---

## 2. Architecture

### 2.1 Overall Architecture

```text
              Applications / Kubernetes
                         |
                         |
                         v
               OpenTelemetry Collector
                         |
                         |
                         v
                ClickHouse Exporter
                         |
                         |
                         v
                ClickHouse Server
                   Port 9000
                         |
                         v
             observability.otel_logs
                  (Main Table)
                         |
                         |
                         v
              Materialized View
              mv_otel_error_logs
                         |
                         |
                  Error Detection
                      Rules
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
       JSON           ClickHouse       Klog
    ERROR/FATAL       Error Tags     E-prefix
          |              |              |
          +--------------+--------------+
                         |
                         v
             observability.otel_error_logs
                    (Error Table)
```

### 2.2 Architecture Explanation

**Step 1: Log Generation and Collection**

Applications and Kubernetes workloads generate logs.

The OpenTelemetry Collector collects the logs and processes them through its configured pipeline.

**Step 2: Log Export**

The OpenTelemetry Collector sends the logs to ClickHouse using the ClickHouse exporter.

The ClickHouse native protocol endpoint is exposed on port `9000`.

**Step 3: Main Table**

All incoming logs are stored in:

```sql
observability.otel_logs
```

This is the primary log-storage table.

**Step 4: Materialized View**

The Materialized View is attached to the main table.

Whenever new records are inserted, ClickHouse evaluates the configured error-detection conditions.

**Step 5: Error Detection**

The configured conditions identify selected error formats:

- JSON ERROR and FATAL levels.
- ClickHouse-style `<Error>` tags.
- ClickHouse-style `<Fatal>` tags.
- ClickHouse-style `<Critical>` tags.
- Klog-style `E` prefix.
- Structured timestamp-based ERROR messages.

**Step 6: Error Table**

Records matching at least one condition are inserted into:

```sql
observability.otel_error_logs
```

### 2.3 Important Architecture Decision

We did not remove matching error records from the main table.

The same matching record exists in both tables:

```text
observability.otel_logs
              |
              +---- All logs remain here
              |
              +---- Matching error logs are copied here
                            |
                            v
                  observability.otel_error_logs
```

This design allows us to maintain the complete original log dataset while providing a separate table for error-specific queries.

---

## 3. Environment and Components

### 3.1 Environment Details

| Component | Configuration |
|---|---|
| Kubernetes | Kind |
| Kubernetes context | `kind-clickhouse-logql` |
| ClickHouse namespace | `clickhouse` |
| ClickHouse pod | `clickhouse-clickhouse-0-0-0` |
| ClickHouse version | `26.10.3.7` |
| Database | `observability` |
| Database engine | Atomic |
| Main table | `otel_logs` |
| Error table | `otel_error_logs` |
| Materialized View | `mv_otel_error_logs` |
| Generator namespace | `log-generator-poc` |
| Generator Deployment | `log-generator` |
| Generator ConfigMap | `log-generator-script` |
| Generator ConfigMap key | `generator.py` |
| Generator image | `python:3.12-slim` |

### 3.2 Important Kubernetes Resources

ClickHouse:

```text
Namespace: clickhouse
Pod: clickhouse-clickhouse-0-0-0
Service: clickhouse-clickhouse-headless
```

OpenTelemetry Collector:

```text
Namespace: logging
ConfigMap: otel-collector-opentelemetry-collector-agent
```

Log Generator:

```text
Namespace: log-generator-poc
Deployment: log-generator
ConfigMap: log-generator-script
```

### 3.3 Verify Kubernetes Context

```bash
kubectl config current-context
```

**Purpose:**

Displays the currently selected Kubernetes context.

This is important because we must ensure that all commands are executed against the intended Kind cluster.

Expected context:

```text
kind-clickhouse-logql
```

### 3.4 Verify Kubernetes Namespaces

```bash
kubectl get namespaces
```

**Purpose:**

Displays all namespaces available in the current Kubernetes cluster.

### 3.5 Check ClickHouse Pod

```bash
kubectl get pods -n clickhouse
```

**Purpose:**

Checks whether the ClickHouse pod is running.

### 3.6 Check ClickHouse Version

```bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \
  clickhouse-client --query "SELECT version()"
```

**Purpose:**

Executes a ClickHouse query inside the ClickHouse pod and returns the installed server version.

### 3.7 Open ClickHouse Client

```bash
kubectl exec -it -n clickhouse clickhouse-clickhouse-0-0-0 -- \
  clickhouse-client
```

**Explanation:**

- `kubectl exec`: Executes a command inside a Kubernetes pod.
- `-it`: Opens an interactive terminal.
- `-n clickhouse`: Selects the ClickHouse namespace.
- `clickhouse-clickhouse-0-0-0`: Specifies the ClickHouse pod.
- `clickhouse-client`: Starts the ClickHouse SQL client.

After executing this command, SQL queries can be run directly.

---

## 4. Understanding the Existing Log Ingestion Pipeline

Before implementing error segregation, we inspected the existing log-ingestion pipeline.

The OpenTelemetry Collector was already configured to collect Kubernetes container logs and export them to ClickHouse.

### 4.1 Existing Collector Configuration

The relevant configuration included:

- `file_log` receiver for Kubernetes container logs.
- Container log parser.
- OTLP gRPC receiver on port `4317`.
- OTLP HTTP receiver on port `4318`.
- `memory_limiter` processor.
- `batch` processor.
- ClickHouse exporter.
- Database: `observability`.
- Destination table: `otel_logs`.
- Native ClickHouse endpoint on port `9000`.
- Asynchronous inserts enabled.
- Sending queue and retry configuration.

The collector's processing flow:

```text
Kubernetes Container Logs
           |
           v
      file_log Receiver
           |
           v
      memory_limiter
           |
           v
          batch
           |
           v
     ClickHouse Exporter
           |
           v
 observability.otel_logs
```

### 4.2 Inspect the OpenTelemetry Collector ConfigMap

```bash
kubectl get configmap \
  otel-collector-opentelemetry-collector-agent \
  -n logging -o yaml
```

**Purpose:**

Displays the collector configuration, including:

- Receivers.
- Processors.
- Exporters.
- Service pipelines.
- ClickHouse export settings.

### 4.3 Why Did We Inspect the Collector?

We wanted to understand:

1. Where the logs were coming from.
2. How the logs were being collected.
3. Which table received the logs.
4. Which attributes and fields were available.
5. Whether multiline processing was configured.
6. How the logs were exported to ClickHouse.

The inspected `file_log` configuration used the container parser. We did not find a multiline operator in that configuration.

This became important later because multiline stack traces may require additional processing.

---

## 5. ClickHouse Database and Table Analysis

Before creating or modifying the error-detection mechanism, we inspected the existing database and tables.

### 5.1 Check Database Engine

```sql
SELECT
    name,
    engine
FROM system.databases
WHERE name = 'observability';
```

**Purpose:**

Checks the database engine used by the `observability` database.

Observed result:

```text
observability | Atomic
```

The database uses the Atomic engine.

### 5.2 List Existing Tables

```sql
SHOW TABLES FROM observability;
```

**Purpose:**

Lists all tables and views available in the `observability` database.

### 5.3 Check Table Engines

```sql
SELECT
    name,
    engine
FROM system.tables
WHERE database = 'observability';
```

**Purpose:**

Displays the database objects and their respective engines.

This helps distinguish between normal tables and Materialized Views.

### 5.4 Inspect Main Table Schema

```sql
SHOW CREATE TABLE observability.otel_logs;
```

**Purpose:**

Displays the complete DDL of the main OTel log table.

The main table contains the following important columns:

| Column | Purpose |
|---|---|
| `Timestamp` | Log timestamp |
| `TraceId` | Trace identifier |
| `SpanId` | Span identifier |
| `TraceFlags` | Trace flags |
| `SeverityText` | Text severity |
| `SeverityNumber` | Numeric severity |
| `ServiceName` | Service name |
| `Body` | Actual log message |
| `ResourceSchemaUrl` | Resource schema URL |
| `ResourceAttributes` | Resource attributes |
| `ScopeSchemaUrl` | Scope schema URL |
| `ScopeName` | Instrumentation scope name |
| `ScopeVersion` | Instrumentation scope version |
| `ScopeAttributes` | Scope attributes |
| `LogAttributes` | Log attributes |
| `EventName` | Event name |

The `Body` column was especially important because different applications and infrastructure components use different log formats.

### 5.5 Inspect Error Table Schema

```sql
SHOW CREATE TABLE observability.otel_error_logs;
```

**Purpose:**

Displays the complete DDL of the error table.

The error table uses the MergeTree engine.

Its observed storage design included:

```sql
ENGINE = MergeTree
PARTITION BY toDate(Timestamp)
ORDER BY
(
    toStartOfFiveMinutes(Timestamp),
    ServiceName,
    Timestamp
)
```

The error table contains the OTel log columns and additional materialized columns used for Kubernetes log information.

The exact complete DDL should be retrieved from the cluster rather than reconstructed from partial information.

### 5.6 Check Existing Materialized Views

```sql
SELECT
    name,
    engine,
    create_table_query
FROM system.tables
WHERE database = 'observability'
  AND engine = 'MaterializedView';
```

**Purpose:**

Checks whether Materialized Views exist in the database and displays their definitions.

During the initial investigation, the result contained zero rows.

This indicated that automatic error routing was not active at that stage.

### 5.7 Check Error Table Record Count

```sql
SELECT
    count() AS total_error_logs,
    min(Timestamp) AS first_error_time,
    max(Timestamp) AS latest_error_time
FROM observability.otel_error_logs;
```

**Purpose:**

Checks the total number of records in the error table and the oldest and newest timestamps.

Initial observed result:

```text
total_error_logs = 0
first_error_time = 1970-01-01 00:00:00
latest_error_time = 1970-01-01 00:00:00
```

This confirmed that the error table was empty at that stage.

---

## 6. Initial Log Analysis and Error Detection Investigation

The next step was to understand how errors were represented in the actual ingested logs.

We did not want to rely only on generic keywords such as:

- Error
- Failed
- Exception
- Fatal
- Panic
- Timeout

These words can also appear in normal messages.

### 6.1 Analyze Severity Distribution

```sql
SELECT
    SeverityText,
    SeverityNumber,
    count() AS log_count
FROM observability.otel_logs
WHERE Timestamp >= now() - INTERVAL 24 HOUR
GROUP BY
    SeverityText,
    SeverityNumber
ORDER BY log_count DESC;
```

**Purpose:**

Groups logs by their OTel severity text and numeric severity.

During the investigation, we found a significant number of logs with:

```text
SeverityText = ''
SeverityNumber = 0
```

One observed result showed:

```text
1,461,037 rows
```

with blank severity text and severity number zero.

An earlier observation also showed `129,276` rows.

These values represent observations from different query executions and time windows, not fixed totals.

### Finding

We could not depend only on OTel severity fields because a large number of logs did not have populated severity information.

Therefore, the log body also needed to be evaluated.

### 6.2 Inspect Log Attributes

```sql
SELECT
    LogAttributes,
    count() AS log_count
FROM observability.otel_logs
WHERE Timestamp >= now() - INTERVAL 24 HOUR
GROUP BY LogAttributes
ORDER BY log_count DESC
LIMIT 20;
```

**Purpose:**

Helps understand which log attributes are available with the incoming records.

Attributes such as the following were relevant during investigation:

- `log.file.path`
- `logtag`
- `log.iostream`

### 6.3 Search Common Error Keywords

```sql
SELECT
    Timestamp,
    ServiceName,
    SeverityText,
    SeverityNumber,
    Body
FROM observability.otel_logs
WHERE Timestamp >= now() - INTERVAL 1 HOUR
  AND
  (
      positionCaseInsensitive(Body, 'error') > 0
      OR positionCaseInsensitive(Body, 'exception') > 0
      OR positionCaseInsensitive(Body, 'panic') > 0
      OR positionCaseInsensitive(Body, 'fatal') > 0
      OR positionCaseInsensitive(Body, 'oom') > 0
  )
ORDER BY Timestamp DESC
LIMIT 50;
```

**Purpose:**

Searches for common error-related keywords in the log body.

This helped us discover different types of errors, including:

- Kubernetes Unauthorized errors.
- ClickHouse Keeper connection problems.
- ClickHouse exceptions.
- Application error messages.
- Panic and fatal messages.

### Finding

A keyword-based approach can produce false positives.

Example:

```text
Error budget report generated successfully
```

This message contains the word `Error`, but it represents a successful operation.

Therefore, we explored more structured detection conditions.

### 6.4 Search ClickHouse-Style Error Tags

```sql
SELECT
    Timestamp,
    ServiceName,
    Body
FROM observability.otel_logs
WHERE Timestamp >= now() - INTERVAL 7 DAY
  AND match(Body, '<(Information|Error|Warning|Fatal|Critical)>')
ORDER BY Timestamp DESC
LIMIT 50;
```

**Purpose:**

Searches for ClickHouse-style severity tags in the log body.

Examples:

```text
<Information>
<Error>
<Warning>
<Fatal>
<Critical>
```

This helped us distinguish explicit severity tags from normal messages containing error-related keywords.

### 6.5 Search Klog Error Patterns

```sql
SELECT
    Timestamp,
    ServiceName,
    Body
FROM observability.otel_logs
WHERE Timestamp >= now() - INTERVAL 7 DAY
  AND match(Body, '^E[0-9]{4} ')
ORDER BY Timestamp DESC
LIMIT 50;
```

**Purpose:**

Searches for Klog-style error prefixes.

Example:

```text
E1116 20:16:10.123456 ...
```

The `E` prefix indicates an error-level Klog message.

The four digits represent the date portion of the prefix.

### 6.6 Search Native ClickHouse Severity Tags

```sql
SELECT
    extract(Body, '<(Information|Error|Warning|Fatal|Critical)>') AS native_severity,
    count() AS log_count
FROM observability.otel_logs
WHERE Timestamp >= now() - INTERVAL 7 DAY
  AND Body NOT LIKE '%POC_LOG_GENERATOR%'
  AND match(Body, '<(Information|Error|Warning|Fatal|Critical)>')
GROUP BY native_severity
ORDER BY log_count DESC;
```

**Purpose:**

Extracts the severity tag from matching ClickHouse-style log messages and counts the occurrences of each tag.

The `extract()` function returns the matched severity value.

The generator exclusion ensures that the query focuses on actual ingested logs rather than controlled test messages.

### 6.7 Identified Log Formats

During the investigation, we identified multiple formats that needed separate handling.

| Log format | Example | Detection method |
|---|---|---|
| JSON | `"level":"ERROR"` | JSON extraction |
| JSON | `"level":"FATAL"` | JSON extraction |
| ClickHouse-style | `<Error>` | Body pattern |
| ClickHouse-style | `<Fatal>` | Body pattern |
| ClickHouse-style | `<Critical>` | Body pattern |
| Klog | `E1116...` | Regular expression |
| Structured timestamp | `2026-11-16T20:16:10Z ERROR ...` | Regular expression |

The main learning was that different applications can represent error severity differently.

---

## 7. Error Table and Materialized View Design

### 7.1 Why a Separate Error Table?

The design uses two tables:

```sql
observability.otel_logs
observability.otel_error_logs
```

The main table stores all incoming logs.

The error table stores copies of records that match the configured error-detection conditions.

Benefits:

- Separate error-specific querying.
- Easier error investigation.
- Automatic routing during ingestion.
- Original logs remain available in the main table.
- Detection rules can be updated as new patterns are discovered.

### 7.2 Why Use a Materialized View?

A Materialized View allows ClickHouse to process inserted data and write the query result into a destination table.

Our implementation uses:

```sql
TO observability.otel_error_logs
```

This specifies the destination table.

The processing flow is:

```text
INSERT INTO observability.otel_logs
                 |
                 v
        Materialized View
                 |
                 v
       Evaluate WHERE conditions
                 |
          +------+------+
          |             |
       Match         No match
          |             |
          v             v
   Error table       No routing
```

### 7.3 Important Difference: View vs Materialized View

A normal View stores a query definition and executes it when queried.

A Materialized View with a destination table processes inserted data and writes matching query results to that destination.

In our POC, the Materialized View is attached to the main table and writes matching records to the error table.

### 7.4 Important Historical Data Behaviour

The Materialized View processes new inserted blocks.

It does not automatically backfill historical records into the error table when created.

Therefore, historical error records already present in the main table are not automatically copied.

A separate backfill operation would be required if historical segregation is needed.

---

## 8. Initial Materialized View Implementation

The initial detection conditions covered:

1. JSON ERROR/FATAL.
2. ClickHouse-style Error tags.
3. Klog-style error prefixes.

### 8.1 Initial Materialized View SQL

```sql
CREATE MATERIALIZED VIEW observability.mv_otel_error_logs
TO observability.otel_error_logs
AS
SELECT
    Timestamp,
    TraceId,
    SpanId,
    TraceFlags,
    SeverityText,
    SeverityNumber,
    ServiceName,
    Body,
    ResourceSchemaUrl,
    ResourceAttributes,
    ScopeSchemaUrl,
    ScopeName,
    ScopeVersion,
    ScopeAttributes,
    LogAttributes,
    EventName
FROM observability.otel_logs
WHERE
    upperUTF8(JSONExtractString(Body, 'level')) IN ('ERROR', 'FATAL')
    OR position(Body, '<Error>') > 0
    OR position(Body, '<Fatal>') > 0
    OR position(Body, '<Critical>') > 0
    OR match(Body, '^E[0-9]{4} ');
```

### 8.2 Explanation of the SQL

**1. Create the Materialized View**

```sql
CREATE MATERIALIZED VIEW observability.mv_otel_error_logs
```

Creates a Materialized View named `mv_otel_error_logs` inside the `observability` database.

**2. Specify the destination table**

```sql
TO observability.otel_error_logs
```

Matching records are written into the error table.

**3. Select the required columns**

```sql
SELECT
    Timestamp,
    TraceId,
    SpanId,
    TraceFlags,
    SeverityText,
    SeverityNumber,
    ServiceName,
    Body,
    ResourceSchemaUrl,
    ResourceAttributes,
    ScopeSchemaUrl,
    ScopeName,
    ScopeVersion,
    ScopeAttributes,
    LogAttributes,
    EventName
```

These are the OTel columns selected from the source table.

**4. Specify the source table**

```sql
FROM observability.otel_logs
```

The Materialized View listens to inserts into the main table.

**5. JSON severity detection**

```sql
upperUTF8(JSONExtractString(Body, 'level')) IN ('ERROR', 'FATAL')
```

Explanation:

- `JSONExtractString()` extracts the JSON `level` field.
- `upperUTF8()` converts the extracted value to uppercase.
- `IN ('ERROR', 'FATAL')` checks whether the value is one of the selected error levels.

This supports JSON levels written in different letter cases.

**6. ClickHouse-style error detection**

```sql
position(Body, '<Error>') > 0
OR position(Body, '<Fatal>') > 0
OR position(Body, '<Critical>') > 0
```

`position()` checks whether the specified text exists in the log body.

A result greater than zero indicates that the pattern is present.

**7. Klog detection**

```sql
match(Body, '^E[0-9]{4} ')
```

This regular expression checks whether the message starts with `E`, followed by four digits and a space.

### 8.3 Check the Initial Materialized View

```sql
SHOW CREATE TABLE observability.mv_otel_error_logs;
```

**Purpose:**

Displays the actual Materialized View definition stored in ClickHouse.

### 8.4 Check the Error Table

```sql
SELECT
    count() AS total_error_logs
FROM observability.otel_error_logs;
```

**Purpose:**

Checks the number of records stored in the error table.

---

## 9. Controlled Log Generator POC

### 9.1 Why Did We Create a Log Generator?

Existing Kubernetes and application logs were not sufficient for validating every error-detection scenario.

We needed controlled test data where we knew:

- Which messages should be NORMAL.
- Which messages should be WARNING.
- Which messages should be ERROR.
- Which messages contain error-related keywords but are actually normal.

For this purpose, we created a Python-based log generator and deployed it inside Kubernetes.

### 9.2 Generator Environment

| Component | Value |
|---|---|
| Working directory | `~/log-generator` |
| Python file | `generator.py` |
| Namespace | `log-generator-poc` |
| Deployment | `log-generator` |
| ConfigMap | `log-generator-script` |
| ConfigMap key | `generator.py` |
| Container image | `python:3.12-slim` |
| JSON generation interval | Approximately 1 second |
| Structured ERROR interval | Approximately 30 seconds |

### 9.3 Complete generator.py Code

The following is the actual Python script used in the POC.

```python
import json
import random
import time
from datetime import datetime, timezone

test_cases = [
    # INFO logs
    {
        "level": "INFO",
        "message": "Application started successfully",
        "expected_classification": "NORMAL"
    },
    {
        "level": "INFO",
        "message": "User login successful",
        "expected_classification": "NORMAL"
    },

    # WARNING logs
    {
        "level": "WARN",
        "message": "High memory usage detected",
        "expected_classification": "WARNING"
    },
    {
        "level": "WARN",
        "message": "CPU utilization is above threshold",
        "expected_classification": "WARNING"
    },

    # Genuine ERROR logs
    {
        "level": "ERROR",
        "message": "Database connection failed",
        "expected_classification": "ERROR"
    },
    {
        "level": "ERROR",
        "message": "HTTP 500 Internal Server Error",
        "expected_classification": "ERROR"
    },
    {
        "level": "ERROR",
        "message": "Kubernetes API request failed",
        "expected_classification": "ERROR"
    },

    # FATAL logs
    {
        "level": "FATAL",
        "message": "Critical service failure",
        "expected_classification": "ERROR"
    },

    # EXCEPTION logs
    {
        "level": "ERROR",
        "message": "NullPointerException occurred",
        "expected_classification": "ERROR"
    },

    # TIMEOUT logs
    {
        "level": "ERROR",
        "message": "Database request timeout occurred",
        "expected_classification": "ERROR"
    },

    # FALSE POSITIVE logs
    {
        "level": "INFO",
        "message": "Error budget report generated successfully",
        "expected_classification": "NORMAL"
    },
    {
        "level": "INFO",
        "message": "No exception occurred during processing",
        "expected_classification": "NORMAL"
    },
    {
        "level": "INFO",
        "message": "Failed requests count is zero",
        "expected_classification": "NORMAL"
    },
    {
        "level": "INFO",
        "message": "Timeout configuration updated successfully",
        "expected_classification": "NORMAL"
    }
]

while True:
    # Generate existing JSON test logs
    test_case = random.choice(test_cases)

    log_entry = {
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "service": "poc-log-generator",
        "marker": "POC_LOG_GENERATOR",
        "test_case_id": f"TEST-{random.randint(100000, 999999)}",
        "level": test_case["level"],
        "message": test_case["message"],
        "expected_classification": test_case["expected_classification"]
    }

    print(json.dumps(log_entry), flush=True)

    # Generate one controlled non-JSON log every 30 seconds
    if int(time.time()) % 30 == 0:
        timestamp = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
        print(
            f"{timestamp} ERROR structured-test-service "
            "POC_STRUCTURED_ERROR_TEST simulated structured timestamp failure",
            flush=True
        )

    time.sleep(1)
```

### 9.4 Understanding the Generator Code

**1. Import Python libraries**

```python
import json
import random
import time
from datetime import datetime, timezone
```

- `json`: Converts Python dictionaries into JSON log messages.
- `random`: Selects a random test case.
- `time`: Controls the generator interval and checks the 30-second condition.
- `datetime`: Generates timestamps.
- `timezone`: Ensures timestamps use UTC.

**2. Define test cases**

```python
test_cases = [...]
```

We defined 14 different test cases.

| Category | Number of cases | Expected classification |
|---|---:|---|
| INFO | 2 | NORMAL |
| WARN | 2 | WARNING |
| Genuine ERROR | 3 | ERROR |
| FATAL | 1 | ERROR |
| EXCEPTION | 1 | ERROR |
| TIMEOUT | 1 | ERROR |
| False-positive messages | 4 | NORMAL |
| **Total** | **14** | |

**3. Select a random test case**

```python
test_case = random.choice(test_cases)
```

Selects one random entry from the test-case list.

Each test case has an equal probability of selection.

**4. Create the JSON log entry**

```python
log_entry = {
    "timestamp": datetime.now(timezone.utc).isoformat(),
    "service": "poc-log-generator",
    "marker": "POC_LOG_GENERATOR",
    "test_case_id": f"TEST-{random.randint(100000, 999999)}",
    "level": test_case["level"],
    "message": test_case["message"],
    "expected_classification": test_case["expected_classification"]
}
```

| Field | Purpose |
|---|---|
| `timestamp` | UTC timestamp of the generated log |
| `service` | Identifies the generator service |
| `marker` | Identifies generator JSON logs |
| `test_case_id` | Generates a random test identifier |
| `level` | Actual log level |
| `message` | Actual log message |
| `expected_classification` | Expected classification for testing |

**Important:** `expected_classification` is testing metadata. It is not used by the Materialized View to decide whether a log should be routed.

**5. Print the JSON log**

```python
print(json.dumps(log_entry), flush=True)
```

- `json.dumps()` converts the dictionary into JSON.
- `flush=True` immediately flushes output so the container log stream receives the message.

**6. Generate non-JSON structured ERROR logs**

```python
if int(time.time()) % 30 == 0:
    timestamp = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
    print(
        f"{timestamp} ERROR structured-test-service "
        "POC_STRUCTURED_ERROR_TEST simulated structured timestamp failure",
        flush=True
    )
```

This produces an additional non-JSON ERROR message when the current Unix timestamp is divisible by 30.

Example:

```text
2026-11-17T17:57:30Z ERROR structured-test-service POC_STRUCTURED_ERROR_TEST simulated structured timestamp failure
```

This message was specifically introduced to reproduce a detection gap in the initial Materialized View.

**7. Add a one-second delay**

```python
time.sleep(1)
```

Pauses the loop for one second before the next JSON log is generated.

Therefore, the generator produces approximately one JSON message per second.

The additional structured message can be printed during the same loop iteration.

### 9.5 Test Case Explanation

**INFO logs**

```text
Application started successfully
User login successful
```

Expected classification: NORMAL.

These messages verify that normal informational logs are not unnecessarily routed into the error table.

**WARNING logs**

```text
High memory usage detected
CPU utilization is above threshold
```

Expected classification: WARNING.

These cases help evaluate how warning-level messages behave.

The current Materialized View does not explicitly include JSON WARN as an error condition.

**Genuine ERROR logs**

```text
Database connection failed
HTTP 500 Internal Server Error
Kubernetes API request failed
```

Expected classification: ERROR.

These are the primary error cases.

**FATAL logs**

```text
Critical service failure
```

Expected classification: ERROR.

These validate the JSON FATAL condition.

**EXCEPTION and TIMEOUT logs**

```text
NullPointerException occurred
Database request timeout occurred
```

Both use JSON level ERROR.

They test whether different application failure messages are routed based on their structured severity.

**False-positive logs**

```text
Error budget report generated successfully
No exception occurred during processing
Failed requests count is zero
Timeout configuration updated successfully
```

Expected classification: NORMAL.

These are important because they contain error-related words but represent normal or successful operations.

The purpose is to ensure that the detection mechanism does not classify every message containing words such as `Error`, `Failed`, `Exception`, or `Timeout` as an actual error.

### 9.6 Validate Python Syntax

Move to the generator directory:

```bash
cd ~/log-generator
```

Check the Python syntax:

```bash
python3 -m py_compile generator.py
```

**Purpose:**

Checks for Python syntax errors without running the generator.

Inspect the script:

```bash
cat generator.py
```

**Purpose:**

Displays the complete generator script.

### 9.7 Inspect Kubernetes Generator Resources

Check the Deployment:

```bash
kubectl get deployment log-generator -n log-generator-poc
```

Check the pods:

```bash
kubectl get pods -n log-generator-poc
```

Check the ConfigMap:

```bash
kubectl get configmap log-generator-script \
  -n log-generator-poc -o yaml
```

Check generator logs:

```bash
kubectl logs deployment/log-generator -n log-generator-poc
```

Search for the structured ERROR test:

```bash
kubectl logs deployment/log-generator -n log-generator-poc \
  | grep POC_STRUCTURED_ERROR_TEST
```

### 9.8 Update the ConfigMap

After modifying the local generator script, the ConfigMap needs to be updated.

The actual ConfigMap YAML filename depends on the working directory.

Apply the updated file:

```bash
kubectl apply -f <generator-configmap-file>.yaml
```

Restart the Deployment:

```bash
kubectl rollout restart deployment/log-generator \
  -n log-generator-poc
```

Check rollout status:

```bash
kubectl rollout status deployment/log-generator \
  -n log-generator-poc
```

**Purpose:**

- `kubectl apply` updates the Kubernetes ConfigMap.
- `kubectl rollout restart` restarts the Deployment.
- `kubectl rollout status` verifies that the new Deployment rollout has completed.

The generator ConfigMap backup was maintained locally at:

```text
~/log-generator/log-generator-script-backup.yaml
```

### 9.9 Why Did We Use Markers?

We used two markers:

```text
POC_LOG_GENERATOR
POC_STRUCTURED_ERROR_TEST
```

`POC_LOG_GENERATOR` identifies JSON generator messages.

`POC_STRUCTURED_ERROR_TEST` identifies the controlled non-JSON timestamp-based ERROR message.

These markers allow us to query only the controlled test data without mixing it with unrelated application and Kubernetes logs.

---

## 10. Reproducing the Structured Timestamp Detection Gap

After introducing the structured timestamp test, we checked whether the generated messages were reaching the main table.

### 10.1 Check the Main Table

```sql
SELECT
    Timestamp,
    ServiceName,
    Body
FROM observability.otel_logs
WHERE Timestamp >= now() - INTERVAL 10 MINUTE
  AND position(Body, 'POC_STRUCTURED_ERROR_TEST') > 0
ORDER BY Timestamp DESC;
```

**Purpose:**

Checks whether the controlled structured ERROR messages are being ingested into the main table.

Observed result:

```text
20 matching rows
```

This confirmed that the generator and ingestion pipeline were producing the controlled test messages.

### 10.2 Check the Error Table

```sql
SELECT
    Timestamp,
    ServiceName,
    Body
FROM observability.otel_error_logs
WHERE Timestamp >= now() - INTERVAL 10 MINUTE
  AND position(Body, 'POC_STRUCTURED_ERROR_TEST') > 0
ORDER BY Timestamp DESC;
```

Observed result:

```text
0 rows
```

### Finding

The structured timestamp ERROR messages were present in the main table but were missing from the error table.

This proved that the initial Materialized View did not cover this particular log format.

This was the detection gap we needed to address.

---

## 11. Testing the New Timestamp Regex

Before changing the Materialized View, we tested the regular expression independently.

### 11.1 Regex

```sql
match(
    Body,
    '^[0-9]{4}-[0-9]{2}-[0-9]{2}T.*[[:space:]]ERROR[[:space:]]'
)
```

### 11.2 Regex Explanation

| Regex part | Meaning |
|---|---|
| `^` | Start of the log message |
| `[0-9]{4}` | Four-digit year |
| `-` | Date separator |
| `[0-9]{2}` | Two-digit month |
| `-` | Date separator |
| `[0-9]{2}` | Two-digit day |
| `T` | Date/time separator |
| `.*` | Any characters |
| `[[:space:]]` | Whitespace |
| `ERROR` | Exact uppercase ERROR token |
| `[[:space:]]` | Whitespace after ERROR |

The regex is designed for messages that start with an ISO-like timestamp and contain an uppercase `ERROR` token followed by whitespace.

### 11.3 Validate the Regex Against Controlled Logs

```sql
SELECT
    count() AS total_test_logs,
    countIf(
        match(
            Body,
            '^[0-9]{4}-[0-9]{2}-[0-9]{2}T.*[[:space:]]ERROR[[:space:]]'
        )
    ) AS matched_count
FROM observability.otel_logs
WHERE Timestamp >= now() - INTERVAL 10 MINUTE
  AND position(Body, 'POC_STRUCTURED_ERROR_TEST') > 0;
```

Observed result:

```text
total_test_logs = 20
matched_count = 20
```

### Finding

The regex successfully matched all 20 controlled structured timestamp ERROR messages.

This validated that the new condition could identify the previously missed format.

### 11.4 Check Regex False Positives Against JSON Generator Logs

```sql
SELECT
    count() AS total_test_logs,
    countIf(
        match(
            Body,
            '^[0-9]{4}-[0-9]{2}-[0-9]{2}T.*[[:space:]]ERROR[[:space:]]'
        )
    ) AS error_matches,
    countIf(
        NOT match(
            Body,
            '^[0-9]{4}-[0-9]{2}-[0-9]{2}T.*[[:space:]]ERROR[[:space:]]'
        )
    ) AS non_error_matches
FROM observability.otel_logs
WHERE Timestamp >= now() - INTERVAL 10 MINUTE
  AND position(Body, 'POC_LOG_GENERATOR') > 0;
```

Observed result:

```text
total_test_logs = 600
error_matches = 0
non_error_matches = 600
```

### Finding

The new timestamp regex did not match the 600 tested JSON generator messages.

This was an important check before adding the condition to the Materialized View.

---

## 12. Backup Before Updating the Materialized View

Before replacing the Materialized View, we took a backup of its current definition.

### 12.1 Create and Open Working Directory

```bash
mkdir -p ~/log-generator
cd ~/log-generator
```

**Purpose:**

Creates the working directory if it does not already exist and changes the current directory to it.

### 12.2 Back Up Materialized View Definition

```bash
kubectl exec -n clickhouse clickhouse-clickhouse-0-0-0 -- \
  clickhouse-client --query "SHOW CREATE TABLE observability.mv_otel_error_logs" \
  > mv_otel_error_logs_backup.sql
```

**Explanation:**

- `kubectl exec`: Executes a command inside a pod.
- `-n clickhouse`: Selects the ClickHouse namespace.
- `clickhouse-clickhouse-0-0-0`: Identifies the ClickHouse pod.
- `clickhouse-client`: Runs the ClickHouse client.
- `SHOW CREATE TABLE`: Retrieves the current Materialized View DDL.
- `>`: Redirects the output to a local file.

Backup file:

```text
~/log-generator/mv_otel_error_logs_backup.sql
```

### 12.3 Inspect the Backup

```bash
cat mv_otel_error_logs_backup.sql
```

This displays the saved Materialized View definition.

### 12.4 Generator ConfigMap Backup

The generator ConfigMap backup was also maintained locally:

```text
~/log-generator/log-generator-script-backup.yaml
```

Inspect it:

```bash
cat ~/log-generator/log-generator-script-backup.yaml
```

---

## 13. Updating the Materialized View

We needed to add the structured timestamp ERROR regex without removing the existing detection conditions.

### 13.1 Temporarily Stop the Generator

Before replacing the Materialized View, we temporarily scaled down the controlled generator.

```bash
kubectl scale deployment log-generator \
  -n log-generator-poc --replicas=0
```

**Purpose:**

Changes the desired replica count of the generator Deployment to zero.

This temporarily stops the controlled test generator.

### 13.2 Wait for Generator Pod Deletion

```bash
kubectl wait --for=delete pod \
  -l app=log-generator \
  -n log-generator-poc \
  --timeout=120s
```

**Explanation:**

- `--for=delete`: Waits for the selected pod to be deleted.
- `-l app=log-generator`: Selects pods with the specified label.
- `-n log-generator-poc`: Selects the generator namespace.
- `--timeout=120s`: Waits for a maximum of 120 seconds.

### 13.3 Verify Generator Is Stopped

```bash
kubectl get pods -n log-generator-poc
```

At this stage, the generator namespace showed no generator resources.

**Important:** Only the controlled generator was stopped. Other log sources and the OTel Collector were not intentionally stopped.

### 13.4 Replace the Materialized View

We used `CREATE OR REPLACE MATERIALIZED VIEW` to update the existing definition.

The final SQL was:

```sql
CREATE OR REPLACE MATERIALIZED VIEW observability.mv_otel_error_logs
TO observability.otel_error_logs
AS
SELECT
    Timestamp,
    TraceId,
    SpanId,
    TraceFlags,
    SeverityText,
    SeverityNumber,
    ServiceName,
    Body,
    ResourceSchemaUrl,
    ResourceAttributes,
    ScopeSchemaUrl,
    ScopeName,
    ScopeVersion,
    ScopeAttributes,
    LogAttributes,
    EventName
FROM observability.otel_logs
WHERE
    upperUTF8(JSONExtractString(Body, 'level')) IN ('ERROR', 'FATAL')
    OR position(Body, '<Error>') > 0
    OR position(Body, '<Fatal>') > 0
    OR position(Body, '<Critical>') > 0
    OR match(Body, '^E[0-9]{4} ')
    OR match(Body, '^[0-9]{4}-[0-9]{2}-[0-9]{2}T.*[[:space:]]ERROR[[:space:]]');
```

### 13.5 What Changed?

Only one additional condition was introduced:

```sql
OR match(
    Body,
    '^[0-9]{4}-[0-9]{2}-[0-9]{2}T.*[[:space:]]ERROR[[:space:]]'
)
```

The existing conditions were preserved.

### 13.6 Final Detection Conditions Explained

**Condition 1: JSON ERROR/FATAL**

```sql
upperUTF8(JSONExtractString(Body, 'level')) IN ('ERROR', 'FATAL')
```

Extracts the JSON level and checks whether it is ERROR or FATAL.

**Condition 2: ClickHouse Error**

```sql
position(Body, '<Error>') > 0
```

Checks whether the log body contains `<Error>`.

**Condition 3: ClickHouse Fatal**

```sql
position(Body, '<Fatal>') > 0
```

Checks whether the log body contains `<Fatal>`.

**Condition 4: ClickHouse Critical**

```sql
position(Body, '<Critical>') > 0
```

Checks whether the log body contains `<Critical>`.

**Condition 5: Klog error prefix**

```sql
match(Body, '^E[0-9]{4} ')
```

Checks whether the log starts with the Klog-style error prefix.

**Condition 6: Structured timestamp ERROR**

```sql
match(
    Body,
    '^[0-9]{4}-[0-9]{2}-[0-9]{2}T.*[[:space:]]ERROR[[:space:]]'
)
```

Detects the tested ISO-like timestamp-based ERROR format.

### 13.7 Why Did We Use OR?

All detection conditions are connected using `OR`.

This means a record is routed to the error table if it matches at least one condition.

For example:

- A JSON log with level ERROR matches condition 1.
- A ClickHouse log containing `<Error>` matches condition 2.
- A Klog message starting with `E1116` matches condition 5.
- A timestamp-based ERROR message matches condition 6.

### 13.8 Verify the Updated Materialized View

```sql
SHOW CREATE TABLE observability.mv_otel_error_logs;
```

**Purpose:**

Displays the currently active Materialized View definition.

We verified that:

- The new timestamp regex was present.
- JSON ERROR/FATAL detection was preserved.
- ClickHouse error-tag conditions were preserved.
- Klog detection was preserved.

### 13.9 Restart the Generator

```bash
kubectl scale deployment log-generator \
  -n log-generator-poc --replicas=1
```

**Purpose:**

Restores the generator Deployment to one replica.

Verify the Deployment:

```bash
kubectl get deployment log-generator -n log-generator-poc
```

Verify the pod:

```bash
kubectl get pods -n log-generator-poc
```

Check rollout status:

```bash
kubectl rollout status deployment/log-generator \
  -n log-generator-poc
```

---

## 14. Final Validation After Updating the Materialized View

After updating the Materialized View, we performed validation again.

The objective was to verify:

1. Structured timestamp ERROR routing.
2. Existing JSON ERROR routing.
3. Existing JSON FATAL routing.
4. False-positive behaviour for normal JSON messages.

### 14.1 Verify Structured Timestamp ERROR Routing

```sql
SELECT
    Timestamp,
    ServiceName,
    Body
FROM observability.otel_error_logs
WHERE Timestamp >= now() - INTERVAL 5 MINUTE
  AND position(Body, 'POC_STRUCTURED_ERROR_TEST') > 0
ORDER BY Timestamp DESC;
```

Observed result:

One controlled structured timestamp ERROR message was found in the error table.

Example:

```text
2026-11-17T17:57:30Z ERROR structured-test-service POC_STRUCTURED_ERROR_TEST simulated structured timestamp failure
```

### Finding

The structured timestamp ERROR message was successfully routed into the error table after updating the Materialized View.

This validated the newly added regex condition.

### 14.2 Verify JSON ERROR and FATAL Routing

```sql
SELECT
    upperUTF8(JSONExtractString(Body, 'level')) AS log_level,
    count() AS error_count
FROM observability.otel_error_logs
WHERE Timestamp >= now() - INTERVAL 5 MINUTE
  AND position(Body, 'POC_LOG_GENERATOR') > 0
GROUP BY log_level
ORDER BY log_level;
```

Observed result:

| Log level | Count |
|---|---:|
| ERROR | 55 |
| FATAL | 13 |

No INFO, WARN, or DEBUG levels appeared in this result.

### Finding

The existing JSON ERROR/FATAL routing continued to work after the Materialized View replacement.

These counts are from the specific five-minute validation window and are not fixed expected totals.

### 14.3 Final False-Positive Validation

We then checked whether non-ERROR/FATAL JSON generator messages were present in the error table.

```sql
SELECT
    count() AS total_normal_logs,
    countIf(
        upperUTF8(JSONExtractString(Body, 'level')) NOT IN ('ERROR', 'FATAL')
    ) AS non_error_logs_in_error_table
FROM observability.otel_error_logs
WHERE Timestamp >= now() - INTERVAL 5 MINUTE
  AND position(Body, 'POC_LOG_GENERATOR') > 0;
```

Final observed result:

```text
total_normal_logs = 65
non_error_logs_in_error_table = 0
```

### Explanation

**`total_normal_logs`**

Despite the alias name, this counts all matching JSON generator records found in the error table during the last five minutes.

**`non_error_logs_in_error_table`**

This counts records whose JSON level is not ERROR or FATAL.

The result was zero.

### Finding

Among the 65 matching JSON generator records in the error table, zero had a JSON level other than ERROR or FATAL.

This provides evidence that the tested normal JSON messages were not incorrectly routed by the current detection rules.

---

## 15. Final POC Results

### 15.1 Validation Summary

| Test case | Result | Observation |
|---|---|---|
| JSON ERROR detection | PASS | ERROR logs routed |
| JSON FATAL detection | PASS | FATAL logs routed |
| ClickHouse `<Error>` condition | Configured | Present in final MV |
| ClickHouse `<Fatal>` condition | Configured | Present in final MV |
| ClickHouse `<Critical>` condition | Configured | Present in final MV |
| Klog `E` prefix condition | Configured | Present in final MV |
| Structured timestamp ERROR | PASS | Previously missed format now routed |
| Timestamp regex validation | PASS | 20 out of 20 matched |
| Regex false-positive test on JSON source logs | PASS | 0 of 600 matched |
| JSON ERROR/FATAL after MV replacement | PASS | ERROR 55, FATAL 13 |
| Final JSON false-positive validation | PASS | 0 non-ERROR/FATAL records among 65 |

**Important:** The ClickHouse-style and Klog conditions were configured in the final Materialized View. Their presence in the definition is not the same as a separately documented end-to-end controlled test result.

### 15.2 What Did We Successfully Achieve?

1. Analyzed the existing ClickHouse log schema.
2. Identified different error formats in actual ingested logs.
3. Found that many logs had blank OTel severity information.
4. Created a separate error-log segregation mechanism.
5. Implemented automatic routing using a Materialized View.
6. Added multiple detection conditions for different log formats.
7. Created a controlled Kubernetes log generator.
8. Generated JSON ERROR, FATAL, WARN, INFO, and false-positive messages.
9. Reproduced a structured timestamp ERROR detection gap.
10. Added a regex condition to address the gap.
11. Validated the new routing condition.
12. Verified that JSON ERROR/FATAL routing remained functional.
13. Performed false-positive validation on controlled JSON logs.

---

## 16. Limitations and Open Areas

The initial implementation and controlled validation are complete.

However, the current implementation should be treated as a validated POC, not a production-ready error-detection solution.

### 16.1 Detection Coverage

The current Materialized View does not cover every possible error format.

The timestamp regex currently detects the tested uppercase `ERROR` token in the supported timestamp-based format.

Further testing is needed for:

- Timestamp-based FATAL messages.
- Timestamp-based CRITICAL messages.
- Lowercase or mixed-case non-JSON severity values.
- WARN messages that represent actual failures.
- Application-specific error formats.
- Messages where errors are embedded inside HTTP 200 responses.
- Schema variations across services.
- Other infrastructure-specific severity formats.

### 16.2 Multiline Stack Traces

Some applications generate multiline errors.

For example:

```text
Database connection failed
    at application.database.connect()
    at application.service.initialize()
    at application.main()
```

The error indicator may appear in the first line while the remaining stack trace contains additional information.

The current POC has not implemented or validated complete multiline error assembly.

Further investigation is needed at the ingestion layer and in the detection logic.

### 16.3 Performance and Scalability

The current tests prove functional behaviour on controlled test data.

They do not establish production-scale performance.

Further testing is required for:

- High-volume log ingestion.
- Materialized View insert overhead.
- Regex evaluation cost.
- CPU utilization.
- Memory usage.
- Insert latency.
- Query latency.
- Concurrent ingestion.
- Large-volume data behaviour.

### 16.4 Storage Overhead and Duplicate Data

Matching error logs are stored in both tables:

```text
observability.otel_logs
observability.otel_error_logs
```

This introduces additional storage consumption.

The actual storage overhead must be measured through a controlled benchmark.

### 16.5 Historical Data

The Materialized View processes new inserted blocks.

Creating or replacing the Materialized View does not automatically backfill historical records.

If historical error data is required, a separate backfill process needs to be designed and tested.

### 16.6 WARN and Misclassified Errors

The current JSON detection rule selects only:

```text
ERROR
FATAL
```

It does not select JSON WARN messages.

However, some applications may report actual failures with WARN severity.

This requires a more advanced detection strategy instead of blindly routing every WARN message into the error table.

### 16.7 Multiline Processing and Upstream Validation

The current collector configuration did not show a multiline operator in the inspected `file_log` pipeline.

A future implementation should investigate whether multiline assembly can be performed upstream before records reach ClickHouse.

---

## 17. Useful Kubernetes Commands Reference

### Check Current Context

```bash
kubectl config current-context
```

### List Namespaces

```bash
kubectl get namespaces
```

### Check ClickHouse Pods

```bash
kubectl get pods -n clickhouse
```

### Check Generator Deployment

```bash
kubectl get deployment log-generator -n log-generator-poc
```

### Check Generator Pods

```bash
kubectl get pods -n log-generator-poc
```

### Check Generator Logs

```bash
kubectl logs deployment/log-generator -n log-generator-poc
```

### Inspect Generator ConfigMap

```bash
kubectl get configmap log-generator-script \
  -n log-generator-poc -o yaml
```

### Inspect OTel Collector ConfigMap

```bash
kubectl get configmap \
  otel-collector-opentelemetry-collector-agent \
  -n logging -o yaml
```

### Restart Generator

```bash
kubectl rollout restart deployment/log-generator \
  -n log-generator-poc
```

### Check Generator Rollout

```bash
kubectl rollout status deployment/log-generator \
  -n log-generator-poc
```

### Stop Generator

```bash
kubectl scale deployment log-generator \
  -n log-generator-poc --replicas=0
```

### Start Generator

```bash
kubectl scale deployment log-generator \
  -n log-generator-poc --replicas=1
```

### Inspect Generator ConfigMap YAML

```bash
kubectl get configmap log-generator-script \
  -n log-generator-poc -o yaml
```

---

## 18. Useful ClickHouse Commands Reference

### Open ClickHouse Client

```bash
kubectl exec -it -n clickhouse clickhouse-clickhouse-0-0-0 -- \
  clickhouse-client
```

### Check Version

```sql
SELECT version();
```

### Check Database Engine

```sql
SELECT
    name,
    engine
FROM system.databases
WHERE name = 'observability';
```

### List Tables

```sql
SHOW TABLES FROM observability;
```

### Check Table Engines

```sql
SELECT
    name,
    engine
FROM system.tables
WHERE database = 'observability';
```

### Inspect Main Table

```sql
SHOW CREATE TABLE observability.otel_logs;
```

### Inspect Error Table

```sql
SHOW CREATE TABLE observability.otel_error_logs;
```

### Inspect Materialized View

```sql
SHOW CREATE TABLE observability.mv_otel_error_logs;
```

### Check Materialized Views

```sql
SELECT
    name,
    engine,
    create_table_query
FROM system.tables
WHERE database = 'observability'
  AND engine = 'MaterializedView';
```

### Check Error Table Count

```sql
SELECT count()
FROM observability.otel_error_logs;
```

### Check Latest Error Records

```sql
SELECT
    Timestamp,
    ServiceName,
    Body
FROM observability.otel_error_logs
ORDER BY Timestamp DESC
LIMIT 20;
```

### Check Recent JSON Error Levels

```sql
SELECT
    upperUTF8(JSONExtractString(Body, 'level')) AS log_level,
    count() AS error_count
FROM observability.otel_error_logs
WHERE Timestamp >= now() - INTERVAL 5 MINUTE
  AND position(Body, 'POC_LOG_GENERATOR') > 0
GROUP BY log_level
ORDER BY log_level;
```

### Check Structured Timestamp Test Records

```sql
SELECT
    Timestamp,
    ServiceName,
    Body
FROM observability.otel_error_logs
WHERE Timestamp >= now() - INTERVAL 5 MINUTE
  AND position(Body, 'POC_STRUCTURED_ERROR_TEST') > 0
ORDER BY Timestamp DESC;
```

### Check False Positives

```sql
SELECT
    count() AS total_normal_logs,
    countIf(
        upperUTF8(JSONExtractString(Body, 'level')) NOT IN ('ERROR', 'FATAL')
    ) AS non_error_logs_in_error_table
FROM observability.otel_error_logs
WHERE Timestamp >= now() - INTERVAL 5 MINUTE
  AND position(Body, 'POC_LOG_GENERATOR') > 0;
```

---

## 19. Phase 2 – Future Work

After completing the initial implementation and controlled validation, the next phase should focus on detection coverage, scalability, and production suitability.

### 19.1 Extended Error Detection

Test additional patterns:

- Timestamp-based FATAL and CRITICAL.
- Lowercase and mixed-case severity.
- Additional Kubernetes/Klog formats.
- Application-specific error formats.
- WARN messages representing actual failures.

### 19.2 Multiline Error Handling

Investigate:

- Upstream multiline assembly.
- Stack trace grouping.
- Handling of partial or incomplete stack traces.
- Preservation of the original message.

### 19.3 Hybrid Detection Strategy

A future design can combine:

1. Structured severity extraction.
2. Explicit log-format patterns.
3. Application-specific conditions.
4. Carefully selected keyword and contextual rules.
5. Validation of unexpected or malformed records.

The objective should be to improve coverage without introducing unnecessary false positives.

### 19.4 Performance Benchmarking

Run controlled benchmarks at different ingestion volumes.

Measure:

- Insert throughput.
- Insert latency.
- CPU utilization.
- Memory utilization.
- Materialized View processing overhead.
- Query execution time.

Compare the ingestion behaviour with and without the Materialized View under equivalent workloads.

### 19.5 Storage Benchmarking

Measure:

- Main table storage.
- Error table storage.
- Total additional storage.
- Compression behaviour.
- Storage overhead as error volume increases.

### 19.6 Historical Backfill Strategy

Design a controlled process to populate the error table from historical main-table records if required.

This must be tested carefully to avoid duplicate insertion when combined with ongoing Materialized View processing.

---

## 20. Final Conclusion

The ClickHouse Error Log Segregation POC demonstrates that Materialized Views can automatically route selected error logs from a main OTel log table into a separate error table.

During the POC, we:

- Analyzed the existing log-ingestion architecture.
- Inspected the ClickHouse database and table schema.
- Investigated different error formats.
- Identified limitations in OTel severity fields.
- Created a separate error table.
- Implemented Materialized View-based routing.
- Developed a controlled Python log generator.
- Tested JSON ERROR, FATAL, WARN, INFO, and false-positive cases.
- Identified a structured timestamp ERROR detection gap.
- Added a regex condition to address the gap.
- Validated the updated Materialized View.
- Confirmed that tested non-ERROR/FATAL JSON messages were not routed into the error table.

### Final Status

**Phase 1 – Error Log Segregation Implementation and Controlled Validation: COMPLETE**

**Phase 2 – Extended Detection Coverage, Multiline Handling, Performance Benchmarking, and Storage Analysis: PENDING**

The current implementation is a functionally validated POC for the tested formats. Further testing is required before considering it production-ready.
