# Incident 3 — Al Noor Fuel BR-CO-15 evidence

## Scope and limitations

The attached `samples/al-noor-fuel-nov-2026.xlsx` workbook was inspected read-only. It was not uploaded or replayed. A read-only database query found only Al Noor's October import and four October invoices, all Accepted. There is no November Al Noor import in the current lab database, so its live import ID, stored statuses, and ASP response IDs are not available here. The inbox report is dated 16 Nov 2026; the local database snapshot is dated 14 Nov 2026.

## Workbook calculation

The checked-in sample contains 13 invoice lines across `ANF-2611-0041` through `ANF-2611-0046`. Reproducing the calculation in `services/api/internal/pint/pint.go` and the cents comparison in `services/asp-mock/main.go` gives:

| Invoice | Lines | Net (IBT-109) | VAT (IBT-110) | Gross from portal (IBT-112) | Net + VAT | Difference | BR-CO-15 calculation |
|---|---:|---:|---:|---:|---:|---:|---|
| `ANF-2611-0041` | 1 | AED 13,749.75 | AED 687.49 | AED 14,437.24 | AED 14,437.24 | AED 0.00 | Pass |
| `ANF-2611-0042` | 3 | AED 14,307.08 | AED 715.35 | AED 15,022.42 | AED 15,022.43 | AED -0.01 | Fail |
| `ANF-2611-0043` | 2 | AED 18,684.00 | AED 934.20 | AED 19,618.20 | AED 19,618.20 | AED 0.00 | Pass |
| `ANF-2611-0044` | 4 | AED 11,254.84 | AED 562.75 | AED 11,817.57 | AED 11,817.59 | AED -0.02 | Fail |
| `ANF-2611-0045` | 1 | AED 11,283.30 | AED 564.17 | AED 11,847.47 | AED 11,847.47 | AED 0.00 | Pass |
| `ANF-2611-0046` | 2 | AED 5,343.80 | AED 267.20 | AED 5,610.99 | AED 5,611.00 | AED -0.01 | Fail |

The file uses three-decimal net prices for fuel products and two-decimal line net amounts. In the PINT mapper, each line VAT is `round2(lineNet * rate / 100)`, but the invoice gross accumulator adds `quantity * unitPrice * (1 + rate / 100)` from the unrounded price before rounding the final sum. The ASP mock compares gross in cents with net plus VAT in cents. For the three failing invoices, the two paths differ by one or two fils. The other three pass BR-CO-15 only; this static calculation does not prove their production acceptance or buyer delivery.

## Source references

- `tools/internal/dataset/dataset.go`: defines the six Al Noor invoice line sets and the three-decimal fuel prices.
- `tools/internal/workbook/workbook.go`: writes quantity, unit price, and two-decimal line net values to the workbook.
- `services/api/internal/pint/pint.go`: maps line VAT and computes invoice totals with different rounding bases.
- `services/asp-mock/main.go`: enforces `TaxInclusiveAmount == TaxExclusiveAmount + TaxAmount` using cents.
- `docs/template-field-guide.md`: explains that invoice totals are calculated by the portal and that ASP BR-CO-15 compares total with VAT against total without VAT plus VAT.

## Read-only database query

```sql
SELECT i.id, i.filename, i.status, i.row_count, i.invoice_count,
       i.duration_ms, i.created_at, i.completed_at
FROM imports i
JOIN sellers s ON s.id = i.seller_id
WHERE s.name ILIKE '%Al Noor%'
ORDER BY i.created_at;
```

Result: only `al-noor-fuel-oct-2026.xlsx`, import `0c6f3a52-1f0e-4f43-9d7a-6a1f1b2f0a03`, Completed, 8 rows, 4 invoices, 3,600 ms. Its four invoices are Accepted with ASP message IDs and empty errors. There is no matching November row in this database.

No customer email, database mutation, ASP submission, retry, or invoice correction was performed.