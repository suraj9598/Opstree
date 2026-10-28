# ClickHouse Error Capture and Error Budgeting

## 1. Overview

This document describes two related parts of the observability POC:

1.  **Automatic error capture:** Use a ClickHouse materialized view to
    select error-related records from `observability.otel_logs` and
    store them in `observability.otel_errors`.
2.  **Error Budgeting:** Define how to measure the reliability of the
    LogQL-to-ClickHouse query path using SLI, SLO, eligible requests,
    eligible failures, and a rolling evaluation window.

The materialized view is the error-recording layer. It helps
investigation and provides useful evidence, but it does **not**
calculate an SLI, SLO, or Error Budget by itself. Error Budgeting needs
a defined request population, a reliable success/failure signal, and a
consistent time window.

## 2. Architecture and request flow

A user runs a LogQL query from Grafana. The request goes through the
NGINX proxy to Cerberus, which translates LogQL into ClickHouse SQL.
ClickHouse executes the SQL and the response returns through the query
path to Grafana.

``` text
User / Grafana
      |
      | LogQL request
      v
NGINX proxy (namespace: log-router)
      |
      | forwards request
      v
Cerberus query translator
      |
      | LogQL -> ClickHouse SQL
      v
ClickHouse
      |
      | query result or query error
      v
Cerberus -> NGINX -> Grafana
```

For Error Budgeting, the service we want to measure is the **query
experience through this path**, not just whether NGINX is running. A
request may reach NGINX successfully but fail during translation,
ClickHouse connection, SQL execution, or response handling.

## 3. Current telemetry flow for logs

The POC has separate log ingestion paths:

-   **OpenTelemetry Collector** sends OpenTelemetry logs to
    `observability.otel_logs`.
-   **Fluent Bit** sends Kubernetes/container logs to
    `observability.kubernetes_logs`.
-   The materialized view described below reads from
    **`observability.otel_logs` only**. It does not currently read from
    `observability.kubernetes_logs`.

``` text
OpenTelemetry Collector
          |
          v
observability.otel_logs
          |
          | new rows inserted
          v
observability.otel_errors_mv
          |
          | rule-based filtering
          v
observability.otel_errors
```

## 4. Error-capture objects

  --------------------------------------------------------------------------------
  Object                           Type                    Purpose
  -------------------------------- ----------------------- -----------------------
  `observability.otel_logs`        Source table            Stores incoming
                                                           OpenTelemetry logs

  `observability.otel_errors_mv`   Materialized view       Checks newly inserted
                                                           source rows against
                                                           error-detection rules

  `observability.otel_errors`      Destination table       Stores rows selected by
                                                           the view as
                                                           error-related records
  --------------------------------------------------------------------------------

The destination table is separate from the source table so that
error-related records can be queried without repeatedly scanning the
full source table for the same patterns.

## 5. Materialized-view SQL and rules

The materialized view uses these rules:

``` sql
CREATE MATERIALIZED VIEW observability.otel_errors_mv
TO observability.otel_errors AS
SELECT *
FROM observability.otel_logs
WHERE
    positionCaseInsensitive(Body, '<Error>') > 0
 OR positionCaseInsensitive(Body, '[ERROR]') > 0
 OR (
    ResourceAttributes['k8s.namespace.name'] = 'log-router'
    AND ResourceAttributes['k8s.container.name'] = 'nginx'
    AND toUInt16OrZero(extract(Body, 'HTTP/[0-9.]+" ([0-9]{3})')) >= 400
    AND toUInt16OrZero(extract(Body, 'HTTP/[0-9.]+" ([0-9]{3})')) < 600
 );
```

### Rule A: Body contains `<Error>`

``` sql
positionCaseInsensitive(Body, '<Error>') > 0
```

Selects log bodies containing `<Error>`, without depending on letter
case. It is intended to catch logs that use this error marker.

### Rule B: Body contains `[ERROR]`

``` sql
positionCaseInsensitive(Body, '[ERROR]') > 0
```

Selects log bodies containing `[ERROR]`, without depending on letter
case. This is useful when application logs put the severity marker
directly in the message body.

### Rule C: HTTP error responses from the NGINX log-router

``` sql
ResourceAttributes['k8s.namespace.name'] = 'log-router'
AND ResourceAttributes['k8s.container.name'] = 'nginx'
AND toUInt16OrZero(extract(Body, 'HTTP/[0-9.]+" ([0-9]{3})')) >= 400
AND toUInt16OrZero(extract(Body, 'HTTP/[0-9.]+" ([0-9]{3})')) < 600
```

This rule is scoped to the `nginx` container in the `log-router`
namespace. It extracts the HTTP status code from the access-log body and
selects statuses from **400 through 599**:

-   `4xx`: client/request errors, such as an invalid or malformed query.
-   `5xx`: server-side errors, which may indicate a proxy, translator,
    or backend failure.

The view captures both classes for investigation. This does **not** mean
every 4xx should count as an availability failure in the SLO. Error
capture and SLO classification are separate concerns.

### Why the rules use `OR`

A row is selected if it matches at least one rule. This lets the view
capture application logs marked with `<Error>` or `[ERROR]`, as well as
HTTP error responses from the specified NGINX container.

## 6. What are we trying to achieve with error capture?

The goal is to identify likely error records automatically and store
them in a dedicated table instead of manually searching the full
`otel_logs` table for every investigation.

This supports:

1.  **Centralized error lookup:** Query likely errors from
    `observability.otel_errors`.
2.  **Faster troubleshooting:** Filter error records by timestamp, body,
    namespace, container, and other available columns.
3.  **HTTP error visibility:** Retain NGINX 4xx and 5xx responses for
    the LogQL-to-SQL route.
4.  **Error Budget investigation:** Use captured error records as
    supporting evidence while validating eligible failures.
5.  **Future dashboards and alerts:** Build error summaries and alerts
    after the classification rules and telemetry coverage are validated.

## 7. Benefits of the materialized view

### 7.1 Automatic capture

After the view is created, matching rows are written to the destination
table as new rows are inserted into `observability.otel_logs`. No
separate scheduled query is needed to filter each new insert.

### 7.2 Separate error-focused dataset

The `otel_errors` table makes it easier to inspect error-related records
without repeating the same text and status filters in every
investigation query.

### 7.3 Less repeated filtering for investigations

When an investigation needs only captured errors, querying the
destination table can avoid scanning and filtering the entire source
table each time. Actual performance depends on table design, data
volume, and query patterns.

### 7.4 Coverage for different error signals

The rules capture both textual error markers and HTTP error responses.
Relevant OpenTelemetry `SeverityText` and `SeverityNumber` fields were
observed to be blank/zero in the data inspected, so this initial
classifier uses body patterns and HTTP status extraction instead.

### 7.5 Evidence for reliability analysis

Captured records can help investigate and validate failure signals. A
trustworthy SLO still needs a complete denominator (all eligible
requests) and a clear policy for which failures count.

## 8. Error Budgeting: concepts and components

Error Budgeting is a reliability practice that defines how much failure
a service can experience while still meeting its SLO. It is calculated
over a chosen evaluation window, such as a rolling 30-day period.

### 8.1 SLI --- Service Level Indicator

An SLI is the measured value describing actual service performance.

For request-based availability:

``` text
Availability SLI (%) =
    Successful eligible requests
    / Total eligible requests
    * 100
```

For latency, an example SLI is:

``` text
Latency SLI (%) =
    Eligible queries completed within the latency limit
    / Total eligible queries
    * 100
```

Availability and latency are different indicators. A query can succeed
but still be too slow.

### 8.2 SLO --- Service Level Objective

An SLO is the target value for an SLI over a specified window.

The current proposal for this POC is:

-   **Availability SLO target:** 99.9%
-   **Evaluation window:** rolling 30 days
-   **Eligible population:** valid production LogQL requests to the
    supported query API routes, subject to the eligibility policy below

This target is **provisional** and should be agreed before being treated
as a production commitment.

Example: if measured availability is 99.5% and the target is 99.9%, the
SLO is not met.

### 8.3 Allowed error rate

The allowed error rate is the portion of eligible requests that can fail
while the service still meets the availability SLO.

``` text
Allowed error rate = 1 - SLO target (as a fraction)
```

For a 99.9% target:

``` text
SLO target = 0.999
Allowed error rate = 1 - 0.999 = 0.001 = 0.1%
```

### 8.4 Eligible requests

Eligible requests are requests included in the SLO denominator. This
population must be defined explicitly and applied consistently to both
successes and failures.

For this POC, the provisional policy is:

**Include:** - Valid production requests to the supported routes: -
`/loki/api/v1/labels` - `/loki/api/v1/query_range` - Requests for which
the query path is expected to provide the service.

**Exclude, when reliably identifiable:** - Invalid LogQL requests that
fail validation with `400 bad_request`, because these are client-input
errors rather than service availability failures. - Health checks and
other non-user traffic. - Intentional test requests such as
`/__test-500`, `/error-budget-test`, and `/error-budget-test-2`.

Do not automatically exclude every 4xx response without review. If a
valid request fails because of an internal issue but is reported as a
4xx, that behavior must be investigated and the SLO policy should
account for it.

### 8.5 Eligible failures

For the availability SLI, a failure is an eligible request that does not
receive the expected successful result due to a service-side problem.

Possible failure sources in this architecture include:

-   NGINX proxy/upstream failures.
-   Cerberus translation or internal processing failures for a valid
    query.
-   ClickHouse connection errors.
-   ClickHouse SQL execution errors for a valid translated query.
-   Timeouts or internal server errors.
-   A request that receives a nominal HTTP 200 but whose response
    indicates a failed query, if such a response pattern exists and is
    detected.

A simple NGINX HTTP status count is useful, but it cannot always prove
that ClickHouse successfully executed the translated SQL. For end-to-end
availability, use Cerberus query-result telemetry and/or response
details as well as NGINX access logs, and avoid double-counting a single
request recorded in multiple places.

### 8.6 Allowed failures

For a request-based SLO:

``` text
Allowed failures = Eligible requests * Allowed error rate
```

Example with 10,000 eligible requests and a 99.9% SLO:

``` text
Allowed failures = 10,000 * 0.001 = 10
```

The service can have up to 10 eligible failed requests in this example
and still meet the request-based 99.9% objective. This is an example,
not a measurement of the current environment.

### 8.7 Error Budget consumed

``` text
Budget consumed (%) =
    Actual eligible failures
    / Allowed failures
    * 100
```

If 10 failures are allowed and 3 eligible failures occur:

``` text
Budget consumed = 3 / 10 * 100 = 30%
```

### 8.8 Error Budget remaining

``` text
Budget remaining (%) = 100 - Budget consumed (%)
```

In the same example:

``` text
Budget remaining = 100 - 30 = 70%
```

If actual failures exceed allowed failures, the remaining budget can
become negative. This indicates that the objective has been breached; it
should not be clamped to zero if the goal is to show the extent of the
breach.

These formulas are meaningful only when actual failures and allowed
failures use the same SLO window and the same eligibility rules. If
there are zero eligible requests, availability and budget percentages
should be reported as undefined/not available, not as 100%.

## 9. How Error Budgeting applies to this architecture

### 9.1 Request journey and what each component tells us

  -----------------------------------------------------------------------
  Component               What it does            Useful reliability
                                                  evidence
  ----------------------- ----------------------- -----------------------
  Grafana                 Sends LogQL and         User-facing query
                          displays the result     outcome; may need query
                                                  response/error
                                                  information

  NGINX proxy             Receives and forwards   Access-log status,
                          requests                route, and possibly
                                                  request duration

  Cerberus                Translates LogQL to     Query count/result
                          ClickHouse SQL and      metrics, route, result,
                          handles query           status class, error
                          processing              reason

  ClickHouse              Executes translated SQL Database query
                                                  success/failure and
                                                  execution latency,
                                                  where available

  Response path           Returns the result to   Confirms whether the
                          Grafana                 caller received the
                                                  expected result
  -----------------------------------------------------------------------

The intended SLI is for the **end-to-end query service**. NGINX logs
alone are an accessible starting point, but Cerberus result metrics and
ClickHouse execution evidence are important for validating that a
nominally successful HTTP response represents a successful query.

### 9.2 Availability calculation plan

The calculation will use a single logical request population:

1.  Select real access-log request records for the supported API paths.
2.  Exclude non-request NGINX lifecycle messages, health checks, and
    intentional test paths.
3.  Classify invalid client LogQL errors according to the agreed policy
    (provisionally exclude `400 bad_request` from availability
    failures).
4.  Count successful eligible requests.
5.  Count eligible service-side failures, including relevant 5xx and
    valid-query failures reported by Cerberus or ClickHouse.
6.  Calculate availability using successful eligible requests divided by
    all eligible requests.
7.  Calculate the Error Budget for the same rolling 30-day window.

Conceptually:

``` text
Total eligible requests = Eligible successes + Eligible failures

Availability SLI (%) =
    Eligible successes / Total eligible requests * 100

Allowed failures =
    Total eligible requests * 0.001

Budget consumed (%) =
    Eligible failures / Allowed failures * 100

Budget remaining (%) =
    100 - Budget consumed (%)
```

The exact production query must use the final eligibility rules and a
reliable failure signal. Counts from NGINX logs and Cerberus metrics
must not simply be added together because one request can appear in both
telemetry sources.

### 9.3 Latency SLI/SLO (next stage)

After validating availability measurement, add a separate latency
objective. For example, a proposed target might be that 99% of eligible
queries complete within 2 seconds. This is only an example and is not
yet an agreed target.

Use a trustworthy request-duration measurement that covers the intended
part of the request journey. Do not assume a database-only duration is
equivalent to the end-to-end time experienced by Grafana.

## 10. Current observed data and interpretation

The latest verified query filtered actual NGINX access-log records in
`observability.otel_logs` for the two LogQL-compatible routes:

- `/loki/api/v1/labels`
- `/loki/api/v1/query_range`

The query uses a strict HTTP request-line pattern and accepts only HTTP
status codes from 200 through 599. It excludes NGINX lifecycle/warning
messages and records without a valid HTTP status.

### 10.1 Verified daily request counts

| Date (2026) | Valid requests | HTTP 2xx/3xx | HTTP 4xx | HTTP 5xx |
|---|---:|---:|---:|---:|
| Oct 24 | 18 | 18 | 0 | 0 |
| Oct 25 | 68 | 68 | 0 | 0 |
| Oct 26 | 17 | 17 | 0 | 0 |
| Oct 27 | 17 | 13 | 4 | 0 |
| **Total** | **120** | **116** | **4** | **0** |

Only four dates currently have matching request records in the queried
rolling 30-day window. This is not a complete 30-day observation period;
missing dates must not be interpreted as zero traffic or zero failures.

### 10.2 Provisional SLI and Error Budget calculation

The current provisional policy counts HTTP 2xx/3xx responses as
successful and HTTP 5xx responses as availability failures. HTTP 4xx
responses are treated as client/request errors and excluded from this
availability SLI for now. This policy still needs formal agreement.

| Field | Latest observed value |
|---|---:|
| Provisional SLO target | 99.9% |
| Eligible requests (success + 5xx) | 116 |
| Successful requests | 116 |
| Eligible failures (5xx) | 0 |
| Observed availability | 100% |
| Observed error rate | 0% |
| Theoretical allowed failures at 0.1% | 0.116 |
| Error budget consumed | 0% |
| Error budget remaining | 100% |

Calculation:

```text
Eligible requests = 116 successful + 0 server errors = 116
Observed availability = 116 / 116 * 100 = 100%
Theoretical allowed failures = 116 * (1 - 0.999) = 0.116
Budget consumed = 0 / 0.116 * 100 = 0%
Budget remaining = 100% - 0% = 100%
```

The `0.116` value is a theoretical fractional allowance, not a literal
fraction of a request. Any report of whole allowed failures needs an
explicit rounding policy. The result is a provisional baseline for the
captured sample, not proof that the service met its SLO over the full
30 days. These NGINX status codes measure HTTP outcomes at the proxy;
they do not independently prove successful ClickHouse execution for
every request.

### 10.3 Cerberus metrics caveat

`cerberus_queries_total` is exposed as a monotonic cumulative counter
(`AggregationTemporality = 2`, `IsMonotonic = true`), but the inspected
ClickHouse samples were repeatedly stored with `Value = 1` across many
samples. Do not use those stored values as a reliable request total until
the metric ingestion/aggregation behavior is validated. NGINX access
logs are the current provisional request-count source.

A 30-day evaluation must continue to check timestamp coverage and log
completeness before it is treated as a full rolling-window SLO result.

## 11. Validation performed for error capture

The error-capture path was tested by sending a request through the NGINX
log-router and checking both tables:

-   The request was recorded in `observability.otel_logs`.
-   The matching HTTP response record was also present in
    `observability.otel_errors`.

A request to `/__test-500` returned **HTTP 404 Not Found** because that
test endpoint was not defined. The 404 was captured by the rules. A
separate historical record with status 500 was also observed in earlier
queries, but the test request itself should not be described as a
successful 500 test.

Earlier inspection also found body markers such as `[ERROR]` and
`<Error>` in source logs. Counts can change as telemetry arrives, so
historical counts are validation evidence, not fixed expected values.

## 12. Important behavior and limitations

### 12.1 Materialized views process new inserts

A ClickHouse incremental materialized view normally processes rows as
they are inserted into the source table. Creating the view does not
automatically scan all historical rows already in
`observability.otel_logs`.

If historical backfill is required, plan it separately and avoid
duplicate records.

### 12.2 The rules are heuristic

The rules search for literal strings in `Body` and parse an HTTP status
from a specific log format. They can miss other error formats, and a
body may contain an error marker without representing a real request
failure.

### 12.3 Error capture and SLO classification are different

`observability.otel_errors` intentionally captures both 4xx and 5xx
records. The SLO policy may exclude invalid client requests while
counting eligible server-side failures. Do not treat every row in
`otel_errors` as an SLO failure automatically.

### 12.4 NGINX lifecycle messages are not requests

Startup, shutdown, and worker lifecycle messages may not contain an HTTP
request line or status code. Filter them out of request-level SLI
calculations.

### 12.5 Separate source and destination scope

`observability.otel_errors` contains only rows from
`observability.otel_logs` that match the current rules. It does not
automatically include records stored only in
`observability.kubernetes_logs`.

### 12.6 Error Budget implementation status

A ClickHouse query has now been run successfully to calculate the
provisional 99.9% SLO, eligible request count, observed availability,
theoretical allowed failures, budget consumption, and budget remaining.
The latest observed result is 116 eligible requests, zero HTTP 5xx
failures, 100% observed availability, 0% budget consumed, and 100%
budget remaining for the captured sample.

The Error Budget is **not yet fully automated**. Remaining work:

-   Confirm the production SLI/SLO and whether particular 4xx responses
    should be excluded.
-   Improve and verify request-log coverage across the rolling 30-day
    window; the current data contains matching requests on only four
    dates (Oct 24–27, 2026).
-   Resolve why `cerberus_queries_total` samples stored in ClickHouse
    repeatedly show a value of 1 before using it as a request counter.
-   Validate end-to-end success/failure classification, including
    Cerberus translation and ClickHouse execution outcomes.
-   Add Grafana panels and alerts after the SLI is validated.

## 13. Useful ClickHouse validation queries

### 13.1 Check the newest captured error records

``` sql
SELECT *
FROM observability.otel_errors
ORDER BY Timestamp DESC
LIMIT 20;
```

### 13.2 Count captured error rows in the last 24 hours

``` sql
SELECT count() AS captured_error_rows
FROM observability.otel_errors
WHERE Timestamp >= now() - INTERVAL 24 HOUR;
```

### 13.3 Inspect NGINX HTTP status codes in the error table

``` sql
SELECT
    toUInt16OrZero(
        extract(Body, 'HTTP/[0-9.]+" ([0-9]{3})')
    ) AS http_status,
    count() AS records
FROM observability.otel_errors
WHERE Timestamp >= now() - INTERVAL 24 HOUR
  AND ResourceAttributes['k8s.namespace.name'] = 'log-router'
  AND ResourceAttributes['k8s.container.name'] = 'nginx'
  AND match(Body, '"[A-Z]+ [^ ]+ HTTP/[0-9.]+')
GROUP BY http_status
ORDER BY http_status;
```

These are investigation queries. SLI queries must additionally restrict
the intended request paths and apply the agreed eligibility policy.

## 14. Summary and next steps

The `observability.otel_errors_mv` materialized view automatically
filters newly inserted `observability.otel_logs` rows using error-text
markers and NGINX HTTP status codes. Matching records are written to
`observability.otel_errors`, creating a dedicated place to investigate
errors.

For Error Budgeting, the proposed first objective is **99.9%
request-based availability over a rolling 30-day window**. The SLI is
actual availability; the SLO is the target; the Error Budget is the
permitted failure allowance under that target.

The next implementation steps are:

1.  Improve and validate request-log coverage over the rolling 30-day
    window.
2.  Finalize the SLI/SLO and eligibility rules, including treatment of
    HTTP 4xx and intentional test requests.
3.  Validate end-to-end success/failure classification against Cerberus
    results and ClickHouse outcomes.
4.  Make the rolling 30-day calculation robust, including a documented
    rounding policy for fractional theoretical allowances.
5.  Add the Error Budget panel and alerts to the existing Grafana setup.
6.  Add latency SLI/SLO after request availability is validated.

The 99.9% target and 30-day window remain provisional. The current
observed 100% availability and 100% remaining budget describe only 116
eligible requests from captured logs on Oct 24–27, 2026; they do not
prove that the full 30-day SLO is met.


## 15. SLI, SLO, and Error Budget SQL queries

This POC calculates a provisional request-based availability SLI for the
NGINX router handling these Loki-compatible API paths:

- `/loki/api/v1/labels`
- `/loki/api/v1/query_range`

**Data sources:** `observability.otel_logs` supplies successful HTTP responses;
`observability.otel_errors` supplies matching HTTP 5xx failures. The
`otel_errors` table contains error records only, so it cannot supply the
successful-request denominator by itself.

**Current policy (provisional):** HTTP 2xx–3xx count as successful; HTTP 5xx
count as availability failures; HTTP 4xx are excluded from this availability
SLI as client/query errors. Test paths are excluded. Confirm this policy before
using the result for production SLO reporting.

### 15.1 Inspect HTTP errors captured in `otel_errors`

```sql
SELECT
    extract(Body, '"[A-Z]+ ([^ ?]+)') AS request_path,
    toUInt16OrZero(
        extract(Body, 'HTTP/[0-9.]+" ([0-9]{3})')
    ) AS http_status,
    count() AS error_count,
    min(Timestamp) AS first_seen,
    max(Timestamp) AS last_seen
FROM observability.otel_errors
WHERE
    ResourceAttributes['k8s.namespace.name'] = 'log-router'
    AND ResourceAttributes['k8s.container.name'] = 'nginx'
    AND match(Body, '"[A-Z]+ [^ ]+ HTTP/[0-9.]+')
    AND toUInt16OrZero(
        extract(Body, 'HTTP/[0-9.]+" ([0-9]{3})')
    ) BETWEEN 400 AND 599
    AND Timestamp >= now() - INTERVAL 30 DAY
GROUP BY request_path, http_status
ORDER BY http_status DESC, error_count DESC;
```

Observed output included:

| Path | HTTP status | Count |
|---|---:|---:|
| `/__test-500` | 500 | 1 |
| `/__test-500` | 404 | 3 |
| `/error-budget-test-2` | 404 | 1 |
| `/loki/api/v1/query_range` | 400 | 4 |

These records demonstrate why every row in `otel_errors` must not automatically
be counted as an availability failure: test-path responses and HTTP 4xx are
handled separately under the provisional policy.

### 15.2 Find eligible production HTTP 5xx failures

```sql
SELECT
    extract(Body, '"[A-Z]+ ([^ ?]+)') AS request_path,
    toUInt16OrZero(
        extract(Body, 'HTTP/[0-9.]+" ([0-9]{3})')
    ) AS http_status,
    count() AS failure_count,
    min(Timestamp) AS first_seen,
    max(Timestamp) AS last_seen
FROM observability.otel_errors
WHERE
    ResourceAttributes['k8s.namespace.name'] = 'log-router'
    AND ResourceAttributes['k8s.container.name'] = 'nginx'
    AND match(Body, '"[A-Z]+ [^ ]+ HTTP/[0-9.]+')
    AND toUInt16OrZero(
        extract(Body, 'HTTP/[0-9.]+" ([0-9]{3})')
    ) BETWEEN 500 AND 599
    AND extract(Body, '"[A-Z]+ ([^ ?]+)') NOT IN (
        '/__test-500',
        '/error-budget-test',
        '/error-budget-test-2'
    )
    AND Timestamp >= now() - INTERVAL 30 DAY
GROUP BY request_path, http_status
ORDER BY failure_count DESC;
```

**Observed result:** `0 rows`. No matching production 5xx record was found in the
available matching `otel_errors` data for the queried window. This does not
prove complete 30-day log coverage.

### 15.3 Calculate SLI, SLO, and remaining Error Budget

This is the combined query validated on 2026-10-28. It gets successful
responses from `otel_logs` and eligible production 5xx failures from
`otel_errors`.

```sql
WITH
successful AS
(
    SELECT count() AS success_count
    FROM observability.otel_logs
    WHERE
        ResourceAttributes['k8s.namespace.name'] = 'log-router'
        AND ResourceAttributes['k8s.container.name'] = 'nginx'
        AND match(Body, '"[A-Z]+ [^ ]+ HTTP/[0-9.]+')
        AND toUInt16OrZero(
            extract(Body, 'HTTP/[0-9.]+" ([0-9]{3})')
        ) BETWEEN 200 AND 399
        AND extract(Body, '"[A-Z]+ ([^ ?]+)') NOT IN (
            '/__test-500',
            '/error-budget-test',
            '/error-budget-test-2'
        )
        AND Timestamp >= now() - INTERVAL 30 DAY
),
failures AS
(
    SELECT count() AS failure_count
    FROM observability.otel_errors
    WHERE
        ResourceAttributes['k8s.namespace.name'] = 'log-router'
        AND ResourceAttributes['k8s.container.name'] = 'nginx'
        AND match(Body, '"[A-Z]+ [^ ]+ HTTP/[0-9.]+')
        AND toUInt16OrZero(
            extract(Body, 'HTTP/[0-9.]+" ([0-9]{3})')
        ) BETWEEN 500 AND 599
        AND extract(Body, '"[A-Z]+ ([^ ?]+)') NOT IN (
            '/__test-500',
            '/error-budget-test',
            '/error-budget-test-2'
        )
        AND Timestamp >= now() - INTERVAL 30 DAY
)
SELECT
    99.9 AS target_slo_percent,
    s.success_count,
    f.failure_count,
    s.success_count + f.failure_count AS eligible_requests,
    round(
        100.0 * s.success_count
        / nullIf(s.success_count + f.failure_count, 0),
        4
    ) AS availability_sli_percent,
    round(
        100.0 * f.failure_count
        / nullIf(s.success_count + f.failure_count, 0),
        4
    ) AS error_rate_percent,
    round(
        (s.success_count + f.failure_count) * (1 - 0.999),
        4
    ) AS allowed_failures_theoretical,
    round(
        100.0 * f.failure_count
        / nullIf(
            (s.success_count + f.failure_count) * (1 - 0.999),
            0
        ),
        2
    ) AS budget_consumed_percent,
    round(
        100.0 - 100.0 * f.failure_count
        / nullIf(
            (s.success_count + f.failure_count) * (1 - 0.999),
            0
        ),
        2
    ) AS budget_remaining_percent
FROM successful AS s
CROSS JOIN failures AS f;
```

**Observed output:**

| Metric | Value |
|---|---:|
| Target SLO | 99.9% |
| Successful requests | 157 |
| Eligible 5xx failures | 0 |
| Eligible requests | 157 |
| Availability SLI | 100% |
| Error rate | 0% |
| Theoretical allowed failures | 0.157 |
| Error budget consumed | 0% |
| Error budget remaining | 100% |

### 15.4 Definitions and limitations

- **SLI:** successful eligible HTTP responses divided by eligible responses,
  multiplied by 100.
- **SLO:** the currently proposed 99.9% availability target over a rolling
  30-day window.
- **Theoretical allowed failures:** eligible requests multiplied by `0.001`
  for a 99.9% SLO.
- **Budget consumed:** observed eligible failures divided by theoretical
  allowed failures, multiplied by 100.
- **Budget remaining:** 100% minus budget consumed.

The theoretical allowance (`0.157` in this small sample) is fractional; it is
not a literal fraction of a request. Define a rounding convention before
operational reporting or alerting.

The current result is based on the matching logs available to the query. Prior
inspection found requests on only a few dates in the 30-day window, so missing
dates must be treated as unknown coverage, not as proof of zero traffic or zero
failures. HTTP 200 at the router also does not by itself prove that a LogQL
query returned correct results from ClickHouse. Validate request coverage,
query correctness, and the provisional HTTP 4xx policy before treating this as
a production SLO.

## 16 Conclusion

The ClickHouse Error Capture and Error Budgeting POC demonstrates automatic error logging using a materialized view and a dedicated `otel_errors` table. By combining error records with successful request logs, we calculated the provisional 99.9% SLO, availability SLI, and remaining Error Budget. The current sample shows 100% availability, 0% budget consumed, and 100% budget remaining, with no eligible production HTTP 5xx failures observed. Full 30-day log coverage and end-to-end query validation are still required before confirming SLO compliance.
