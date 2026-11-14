# Incident 5 — Ghaf Tree Café date-format evidence

## Workbook inspection

The attached `samples/ghaf-cafe-nov-2026.xlsx` workbook was read from its XLSX worksheet XML without modifying or uploading it.

| Invoice | Line rows | Issue date text | Due date text |
|---|---:|---|---|
| `GTC-118` | 2 | `13/11/2026` | `13/12/2026` |
| `GTC-119` | 2 | `13/11/2026` | `13/12/2026` |
| `GTC-120` | 3 | `14/11/2026` | `14/12/2026` |
| `GTC-121` | 1 | `15/11/2026` | `15/12/2026` |

Total: 8 non-empty invoice lines, 4 distinct invoices. The date values are text strings, not Excel date serial values.

## Code and documented behavior

- `services/api/internal/sheet/sheet.go`: `Read` requests raw cell values. `ParseDate` accepts valid `YYYY-MM-DD` strings or positive numeric Excel date serials; it does not parse `DD/MM/YYYY` text.
- `services/api/internal/validate/validate.go`: each row's issue and due date is checked; an unparseable value adds AE-BR-04. An invoice is rejected by the portal if any of its rows fails validation and is not submitted to the ASP.
- `tools/internal/dataset/dataset.go` and `tools/internal/workbook/workbook.go`: the Ghaf sample defines slash-formatted date strings and writes them as text.
- `docs/template-field-guide.md`: explicitly documents ISO `YYYY-MM-DD` or real Excel dates and provides conversion guidance for text dates.

For the unchanged sample, each of the four invoice groups contains slash-formatted issue and due dates and will fail portal date validation. This is consistent with the reported message, although the actual Nov 16 portal record is not available in the local database.

## Read-only database check

```sql
SELECT i.id, i.filename, i.status, i.row_count, i.invoice_count,
       i.created_at, i.completed_at
FROM imports i
JOIN sellers s ON s.id = i.seller_id
WHERE s.name ILIKE '%Ghaf%'
ORDER BY i.created_at;
```

Result: zero rows in the current local lab database for Ghaf. This database snapshot is dated 14 Nov 2026, while the inbox report is dated 16 Nov 2026, so it cannot confirm the reported import ID or live status.

No customer email, upload, retry, or database write was performed.