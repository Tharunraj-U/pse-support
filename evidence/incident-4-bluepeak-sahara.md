# Incident 4 — Bluepeak / Sahara batch evidence

## Customer report

Daniel's 16 Nov 2026 email says the connector received HTTP 201 for a 40-invoice import, the portal displayed Submitted, Emirates Palm Hospital received only a few invoices, and many portal ASP references were blank. He is holding the resend pending guidance. The email does not include the import ID, individual invoice IDs, connector timestamp, or raw connector response body.

## Read-only local database and ASP checks

No November Sahara import exists in the local database. The only Sahara import found is the Oct 29 baseline (`0c6f3a52-1f0e-4f43-9d7a-6a1f1b2f0a04`), Completed with 12 invoices; all 12 are Accepted and have non-empty ASP message IDs. No current November invoice status or ASP reference can therefore be correlated to Daniel's report.

The current local mock ASP query `GET /v1/documents?sender=0235:1006453719` returned zero documents, and the retained 24-hour ASP log search returned zero `rate limit exceeded` lines. The mock stores received documents in memory and is separate from the reported production environment. This Nov 14 snapshot does not verify the Nov 16 batch.

## Code-backed response path

- `tools/internal/dataset/dataset.go`: Sahara sample builder creates 40 invoices for the same seller/sender (`SMS-1116-001` through `SMS-1116-040`).
- `services/asp-mock/main.go`: per sender, token bucket allows a burst of 8 documents, then replenishes at 2 documents per second. Excess requests receive HTTP 429 and a `Retry-After`; throttled documents are not added to the ASP document store.
- `services/api/main.go`: starts four submission workers.
- `services/api/internal/importer/importer.go`: after parsing and portal validation, marks the import complete, enqueues validated invoice IDs, then returns the import to the HTTP handler.
- `services/api/internal/submit/submit.go`: sets status Submitted before sending; stores Accepted and the ASP message ID only on HTTP 200. It does not handle 429 or `Retry-After`, so a 429 leaves the database row Submitted with its default empty ASP reference. A failure to persist an HTTP 200 response is another possible ambiguous outcome that must be checked rather than assumed away.
- `services/api/internal/store/store.go` and `services/api/internal/httpapi/httpapi.go`: built-in retry queries/status guards allow only Validated and Failed, not Submitted. The current UI retry cannot recover Submitted rows with missing references.
- `docs/architecture.md`: `GET /v1/documents?sender=<peppolId>` lists documents received by the simulated ASP. The mock has no buyer-side delivery acknowledgement endpoint.

## Interpretation and safe next diagnostic

The 201 response is for the import request; it is not evidence that all 40 background ASP requests were accepted. The combination of a 40-invoice same-sender batch, an 8-request burst limit, four workers, and no 429 recovery makes throttling a strong explanation for partial receipt and Submitted rows with blank references. This is a code-supported mechanism, not a confirmed event for Daniel's missing invoices.

For the original production import, first obtain the import ID and request time. Then use read-only queries to list each invoice's status, ASP message ID, and event history, and query the ASP document list/logs by sender and invoice number. Do not resend a blank-reference Submitted invoice until absence from the ASP has been established and Engineering approves a safe retry path. No replay or status change was performed.