# Incident 2 — API restart and upload impact

This evidence combines the supplied exercise reports with read-only observations from the local lab. The exercise email is dated Monday 16 Nov 2026; the available local container, database, and Prometheus snapshot is dated 14 Nov 2026. The local snapshot is not the reported production environment and cannot establish what happened on 16 Nov.

## Supplied incident reports

- Grafana alert received 09:42 GST: `InvoicelyApiRestarted`, severity `critical`, environment `production`, value `restarts=3`; annotation says the API restarted three times in the last 15 minutes.
- Priya's message was posted at 09:48 and entered the support queue at 09:50 GST. She reported intermittent portal 502s for several customers, mostly during uploads, and a couple of invoices reportedly in Validated since about 09:30. Marina Bay Hospitality is scheduled to go live on 18 Nov.
- No affected seller names/IDs, import IDs, invoice numbers, or customer-side error timestamps were included in the report.

## Read-only local lab checks

No reset, retry, replay, or database write was performed.

| Check | Result in the local lab snapshot |
|---|---|
| Docker API container | Healthy; `restart_count=0`; `oom_killed=false`; container started 2026-11-14 04:57:53 UTC |
| API logs | Retained output showed startup/listening only; no matching panic, fatal exit, or API error lines in the filtered 24-hour output |
| Web logs | No matching proxy 502/504 records in the filtered retained output |
| Prometheus `up{job="api"}` | `1` |
| `changes(process_start_time_seconds{job="api"}[15m])` | `0` |
| `sum(increase(invoicely_http_request_duration_seconds_count{code=~"5.."}[5m])) or vector(0)` | `0` |
| `invoicely_submission_queue_depth` | `0` |
| Current PostgreSQL invoice statuses | 43 Accepted; 0 Validated and 0 Submitted |
| Current PostgreSQL imports | 6 Completed; no Processing or Failed imports |

These are only current local lab observations. They do not prove that the production alert or customer reports are false.

## Alert rule check

`observability/grafana/provisioning/alerting/rules.yml` evaluates:

```promql
changes(process_start_time_seconds{job="api"}[15m])
```

However, its threshold condition is `gt 0`, while the alert email's description says the container restarted more than twice. Thus any observed process-start-time change can fire the configured rule. The supplied email's value of 3 is recorded as reported, but the local snapshot cannot independently reproduce it.

## Code-based failure modes

- `docker-compose.yml` configures the API with `restart: unless-stopped`, a 128 MB memory limit, and a healthcheck. The restart policy responds to process exit; Docker healthcheck failure alone marks a container unhealthy and does not itself restart it.
- `services/api/main.go` exits on initial database configuration failure, database ping failure after 30 attempts, or a fatal `ListenAndServe` error. No event-time logs or exit metadata are available to distinguish these causes or establish an OOM event.
- `services/api/internal/submit/submit.go` stores pending invoice IDs in a buffered Go channel in process memory. The queue is not durable across process restarts. An invoice already stored as Validated but whose queued work is lost could remain Validated. This is a plausible mechanism, not a verified cause for the reported invoices.
- `web/app/api/[...path]/route.ts` returns 502 when an upstream fetch fails and 504 when its 60-second timeout fires. The customer's 502 report alone does not distinguish these conditions or prove the API itself returned 502.

## Missing evidence and next checks

- Production API container exit code, restart timestamps, OOM events, and logs spanning 09:30–09:50 GST on 16 Nov.
- Production API `up`, restart, 5xx, request-duration, and queue-depth metrics for that same window, plus database and ASP health evidence.
- Each affected import ID and invoice status/event history, matched to ASP receipts/message IDs.
- The number of affected customers/invoices and whether any invoices continued reaching the ASP during the alert window.
- A documented Marina Bay readiness result after restart recovery and queue reconciliation are verified.