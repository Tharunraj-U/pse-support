# Oasis Builders — controlled replay evidence

This evidence describes the already-completed lab replay only. It does not establish what happened during the customer's reported three attempts. No customer email, database mutation, reset, or additional replay was performed during this investigation.

## Read-only database checks

Commands were run against the existing PostgreSQL container with `SELECT` statements only.

```sql
SELECT i.id, i.filename, i.status, i.row_count, i.invoice_count,
       i.duration_ms, i.created_at, i.completed_at
FROM imports i
JOIN sellers s ON s.id = i.seller_id
WHERE s.name ILIKE '%Oasis%'
ORDER BY i.created_at;
```

Result: two imports are present:

| Import ID | File | Status | Rows | Invoices | Duration | Created (UTC) |
|---|---|---:|---:|---:|---:|---|
| `0c6f3a52-1f0e-4f43-9d7a-6a1f1b2f0a02` | `oasis-builders-oct-2026.xlsx` | Completed | 16 | 8 | 5,400 ms | 2026-10-31 07:40:00 |
| `4565c114-a40d-4b85-8381-b31d44afeaa0` | `oasis-builders-nov-2026.xlsx` | Completed | 600 | 8 | 122,647 ms | 2026-11-14 06:09:33 |

No additional November imports for Oasis are present in this lab database. In particular, it contains no records identifiable as the three customer attempts.

The invoice check was:

```sql
SELECT v.import_id, v.invoice_number, v.status, v.asp_message_id, v.errors
FROM invoices v
JOIN sellers s ON s.id = v.seller_id
WHERE s.name ILIKE '%Oasis%'
ORDER BY v.created_at, v.invoice_number;
```

For controlled replay import `4565c114-a40d-4b85-8381-b31d44afeaa0`, result: all 8 rows have status `Accepted`, a non-empty `asp_message_id`, and `errors = []`. The replay-created invoice numbers carry a `-HAFW` suffix, so these must not be presented as the customer's original invoice identifiers.

## Sanitized log excerpts

Container logs are UTC. Sender/receiver participant IDs and internal invoice UUIDs are omitted because they are not needed to establish the outcome.

```text
2026-11-14T06:09:33.475Z api  import received  import=4565c114-a40d-4b85-8381-b31d44afeaa0  file=oasis-builders-nov-2026.xlsx
2026-11-14T06:09:33.864Z api  import parsed    import=4565c114-a40d-4b85-8381-b31d44afeaa0  rows=600 invoices=8
2026-11-14T06:11:36.088Z api  import completed import=4565c114-a40d-4b85-8381-b31d44afeaa0  rows=600 invoices=8 validated=8 rejected=0 duration_ms=122647
2026-11-14T06:11:36.176Z asp-mock  document accepted  OBS-2611-001-HAFW  message=0c9dad55-804a-459e-b0f9-c7352ed22e35
2026-11-14T06:11:36.176Z asp-mock  document accepted  OBS-2611-002-HAFW  message=40b2e3c8-233d-4736-9885-8de70a05a473
2026-11-14T06:11:36.176Z asp-mock  document accepted  OBS-2611-003-HAFW  message=6ec71cff-3144-40e1-a958-ce99c5169a90
2026-11-14T06:11:36.176Z asp-mock  document accepted  OBS-2611-004-HAFW  message=5384c383-5227-40ca-98da-8ab33696488f
2026-11-14T06:11:36.195Z asp-mock  document accepted  OBS-2611-005-HAFW  message=40b5432f-91a4-4579-ba5e-ea926a6d130d
2026-11-14T06:11:36.196Z asp-mock  document accepted  OBS-2611-008-HAFW  message=26bd74e8-eff7-41d1-ab32-1307ba9b165d
2026-11-14T06:11:36.197Z asp-mock  document accepted  OBS-2611-006-HAFW  message=cbefb410-bf93-438e-a23a-b65b5415d043
2026-11-14T06:11:36.198Z asp-mock  document accepted  OBS-2611-007-HAFW  message=7ed55cd8-1cf5-4bcc-8d12-bf74a1d24704
```

API logs also recorded HTTP 200 ASP responses for all 8 submissions; recorded response durations were 0–25 ms. Subsequent GETs of the replay import returned HTTP 200. No ASP 429 appears in the replay evidence reviewed.

## Timing explanation from code

- `services/api/internal/importer/importer.go`: `Import` uses a 10-minute internal context and validates invoices in a sequential loop before it returns to the HTTP handler. It enqueues successful invoices only after marking the import completed.
- `services/api/internal/validate/validate.go`: validation loops over every invoice row, and `validateRow` calls participant lookup for each non-empty buyer Peppol ID.
- `services/api/internal/directory/directory.go`: each participant lookup makes an HTTP request to the ASP mock.
- `services/asp-mock/main.go`: the participant endpoint sleeps for 200 ms per request.
- The workbook has 600 invoice lines, each with a buyer participant ID: 600 × 200 ms = 120 seconds of lookup delay. The observed 122.647 seconds is consistent with this plus parsing, database writes, and other overhead. Event history advances by about 15 seconds per 75-line invoice.
- `web/app/api/[...path]/route.ts`: the browser-facing API proxy timeout is 60 seconds. `web/components/UploadForm.tsx` displays a local Processing spinner while its upload POST waits and advises “Please try again” on 504. Since the importer uses an independent background context, the API can continue processing after the proxy stops waiting. This is a plausible explanation for the reported experience, not proof about the three original attempts.

## Duplicate handling

`services/api/internal/store/store.go` checks for the same seller and invoice number in another import, excluding status `Rejected`; a match is reported as AE-BR-09. `db/init/001_schema.sql` defines a lookup index on seller and invoice number but no unique constraint. The upload path has no idempotency key. These facts explain how a later upload may be reported as a duplicate, but the missing original import records prevent confirming that this happened to Rashid's three attempts.

## What remains unverified

- Import IDs and invoice status history for each customer attempt.
- Whether any of those attempts received an ASP message ID or were accepted/rejected by the real account's ASP.
- Whether the third attempt's error was a 504, a different gateway error, or another failure.
- Whether buyer-side delivery occurred for any accepted original invoice.