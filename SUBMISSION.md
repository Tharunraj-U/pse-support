# Invoicely Lite — Product Support Engineer Assignment

## Submission details

- **Name:** Tharun Raj U
- **Date submitted:** 14/11/2026
- **Repository link:** https://github.com/sanjay-complyance/pse-support-lab
- **Total time spent (real time):** Not calculable accurately; the start time was not captured. Work-log end timestamp recorded as 2026-11-14 12:32:04 +05:30.

> All acknowledgement and customer reply times below use the exercise clock: Monday, 16 November 2026, Asia/Dubai (UTC+4). Actual working time is recorded separately.

## Items fully investigated, and why I chose them

1. **Incident 1 — Oasis Builders:** Large month-end upload, retry uncertainty, and potential duplicate invoices; investigation found a measured synchronous validation delay and a plausible web timeout/retry path, while original attempts remain unavailable.
2. **Incident 2 — API restart / Marina Bay:** Critical restart alert plus reports of multi-customer 502s and invoices waiting in Validated warranted P1 triage and an immediate L3 escalation draft; production event-time evidence was not available in the local lab.
3. **Incident 3 — Al Noor Fuel:** The attached workbook and calculation code reproduce BR-CO-15 for three invoices, with an active VAT audit requiring a precise explanation.
4. **Incident 4 — Bluepeak / Sahara Medical:** A 40-invoice batch with blank ASP references and partial receipt exposed a likely unhandled rate-limit response and required careful no-duplicate reconciliation.
5. **Incident 5 — Ghaf Tree Café:** A single-customer date-format rejection was investigated to provide a safe, simple correction without guessing locale-specific dates.

All five incidents are documented. Investigation depth and confidence are stated separately where the matching production import or event-time records are absent.

---

# Incident 1 — Oasis Builders upload stuck

## Acknowledgement

- **Sent at (exercise clock):** 09:07 GST
- **Reply:**

Hello Rashid,

Thank you for reporting this. I understand the urgency and the risk of creating duplicate invoices after repeated upload attempts.

Please do not retry the upload or resend the file for now. We are checking the import history and invoice submission records to establish exactly which invoices were processed and sent.

I will share the verified status and safe next steps as soon as the investigation is complete.

Regards,

Invoicely Lite Support

## Priority and justification

- **Priority:** P2 — provisional, pending reconciliation of the customer's original attempts.
- **Why:** The controlled test demonstrates a major delay (122.647 seconds for 600 invoice lines) and the web proxy times out after 60 seconds. This is consistent with the reported stuck screen and error, and the month-end impact raises urgency. P1 is not supported: there is no evidence that invoices could not reach the ASP generally, and a separate controlled replay was accepted. This remains a single-customer report; lower to P3 if the original account records show no wider processing degradation.
- **SLA:** For P2, meaningful first response within 1 hour (by 10:05 GST), resolution/workaround target 8 business hours (by 17:05 GST), updates every 2–4 hours. The 09:07 GST acknowledgement was within the first-response target; next customer update is due by 11:07 GST. These are exercise-clock times for Monday, 16 November 2026.

## Incident summary and investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| Mon 09:05 | Customer report received | Three upload attempts reported; third showed an error; some documents then showed “already exists.” Customer cannot identify which invoices were sent. |
| Mon 09:07 | Acknowledged Rashid and advised against another full-file upload | Within the P2 first-response target. |
| 14 Nov, 10:09:33–10:11:36 GST (lab clock) | Reviewed the already-completed controlled replay and its logs | Separate import `4565c114-a40d-4b85-8381-b31d44afeaa0`; 600 rows, 8 invoices; import completed in 122.647 seconds. This predates the reported exercise-day attempts and is not evidence of their outcome. |
| Investigation | Queried current Oasis database records read-only | Exactly two imports are present: the October baseline and the controlled November replay. No records for the customer's reported three attempts are available in this lab database. |
| Investigation | Checked invoice/event rows and ASP logs for the controlled replay | All eight generated-suffix invoice records are Accepted, have distinct ASP message IDs, and have empty errors. ASP logged eight `document accepted` events. |
| Investigation | Inspected import, validation, ASP directory, proxy, UI, and duplicate-check code | Found a code-backed explanation for the test duration and a possible timeout/retry/duplicate sequence; this does not establish what occurred in the original attempts. |

Read-only evidence and SQL are preserved in [evidence/oasis-investigation.md](evidence/oasis-investigation.md). No database writes, resets, retries, or customer emails were performed.

## Technical findings and provisional diagnosis

- **Confirmed for the controlled lab replay:** Import `4565c114-a40d-4b85-8381-b31d44afeaa0` parsed 600 rows into 8 invoices, completed with 8 validated and 0 rejected, and then all 8 were accepted by the mock ASP. They have non-empty ASP references and empty error arrays. The invoice numbers include the replay-generated `-HAFW` suffix. These results belong only to this replay.
- **Why the replay took about 122 seconds:** The importer validates invoices synchronously before returning from `POST /v1/imports`. Validation checks a buyer participant on every line with a non-empty Peppol ID. This 600-line workbook therefore made 600 sequential participant lookups. The mock participant endpoint sleeps for 200 ms per lookup: 600 × 0.2 seconds = 120 seconds, plus about 2.647 seconds of other processing. The database event times, roughly 15 seconds per 75-line invoice, support this calculation. ASP submission starts only after import completion; the replay's eight ASP responses took 0–25 ms, so ASP submission did not explain the 122-second import duration. See `services/api/internal/importer/importer.go`, `services/api/internal/validate/validate.go`, `services/api/internal/directory/directory.go`, and `services/asp-mock/main.go`.
- **Why the screen/error may occur:** The upload screen shows a local “Processing” spinner while its POST is pending. The web proxy aborts that request after 60 seconds and returns a gateway timeout; the upload form's 504 message tells the user to try again. The API importer uses its own 10-minute background context rather than the browser request context, so the API work can continue after the proxy times out. This is a strong explanation for the test's long wait and a plausible explanation for the reported error, but original-attempt logs are unavailable to confirm it. See `web/app/api/[...path]/route.ts`, `web/components/UploadForm.tsx`, `services/api/internal/httpapi/httpapi.go`, and `services/api/internal/importer/importer.go`.
- **How duplicate detection works:** The API checks for the same seller and invoice number in a different import, excluding invoices whose status is Rejected; a match produces AE-BR-09. Re-uploading after an earlier import has stored invoice rows can therefore produce “already exists.” The schema has a non-unique lookup index, not a unique constraint, and the upload flow has no idempotency key. This makes repeated requests capable of creating separate imports; the original attempts are not present here, so this is not confirmed as their exact sequence. See `services/api/internal/store/store.go` and `db/init/001_schema.sql`.
- **Unverified risk:** The ASP mock rate-limits a sender after a burst of 8 documents. The submitter code does not explicitly retry or handle HTTP 429; a 429 could leave an invoice in Submitted. The controlled replay shows no such outcome, and there is no evidence that this affected Rashid's attempts.
- **Provisional diagnosis:** A real product latency/timeout mismatch is confirmed by source and replay evidence. It likely caused the observed Processing/error experience; a timed-out first request continuing in the API followed by separate retries could explain later duplicate-number rejections. Confidence is high in the code-path explanation and low-to-medium that it exactly describes the customer's three attempts. The original imports, their invoice status history, and any original ASP message IDs remain unverified.

## Workaround and recovery plan

- Do not upload the whole workbook again and do not use a retry action until the original imports have been reconciled.
- In the customer's actual account, enumerate each November import and compare each source invoice number with the invoice status, ASP reference, and status history. Confirm ambiguous Submitted records against ASP records before deciding anything was not sent.
- Treat Accepted with an ASP reference as accepted by the ASP, not as independent proof that the buyer received it. Do not resend those invoices.
- Only after reconciliation, address individual invoices that are verified as still Validated or Failed through the supported submission retry path. Do not retry Rejected or Accepted invoices, and do not create a replacement upload just to clear the screen.
- This lab cannot perform that customer-account reconciliation: the original three attempts are absent. No retry or data repair has been attempted.

## Customer update draft (not sent)

**To:** Rashid Al Mansoori

**Subject:** Update on your November invoice upload

Hello Rashid,

I understand that the repeated waits and the “document number already exists” messages have left you unsure which November invoices were sent. Please do not upload the full file again while we reconcile the existing records.

In a separate controlled test, this workbook took about two minutes to process, and the test copy's eight invoices were accepted by the test ASP. That test created a different import and does not tell us what happened during your three original attempts. We are checking the original import history and each invoice's ASP reference before advising what, if anything, remains to be sent.

Please reply with any November import IDs or the approximate times shown for the existing attempts; you do not need to upload the file again. I will send you an invoice-by-invoice update and safe next step by 11:07 GST today.

Regards,

Tharun
Product Support Engineer

## Escalation decision and follow-up

- **Decision:** Escalate as a P2 product defect to L3 Engineering through the normal queue, not as a P1 page. Source and controlled replay confirm a synchronous long-running upload behind a 60-second web-proxy timeout and a misleading retry prompt. L1 retains ownership of Rashid's updates at the P2 cadence. This escalation requests product correction; it does not assert that engineering has confirmed the original customer-attempt sequence.
- **Engineering ask:** Decouple upload acceptance from long-running validation or align the client/proxy timeout and provide durable progress; replace the “try again” timeout advice with reconciliation guidance; add idempotent handling or a database-enforced duplicate guard; explicitly handle/retry ASP 429 responses and avoid leaving ambiguous Submitted states.
- **Remaining risks / follow-up:** Obtain the original import IDs/timestamps from the customer's live account; verify all eight source invoice numbers against import and ASP records; reconcile Submitted rows with ASP receipts; send the promised update by 11:07 GST; update priority if more customers or broader ASP delivery failure is found. Preserve this replay as diagnostic evidence only. No dashboard screenshot or customer-attempt logs were available in the evidence reviewed.

## Prevention ideas

- Avoid one remote participant lookup per invoice line; cache or validate once per distinct buyer participant.
- Make upload processing asynchronous or ensure gateway and UI behavior match the measured processing duration.
- Make uploads idempotent and enforce duplicate safety in storage, including concurrent submissions.
- Handle ASP throttling explicitly and show actionable per-invoice progress and final outcomes.

---

# Incident 2 — API restarted / Marina Bay concern

## Separate acknowledgements (drafts; not sent)

### 09:44 GST — Support on-call alert

Hi Support On-call,

Acknowledged. The alert reports three API restarts in 15 minutes, and Customer Success has reported intermittent upload failures and invoices waiting in Validated. I am treating this as a provisional P1 and preparing an immediate L3 on-call escalation while I correlate the available telemetry and affected imports.

I will send the next incident update by 10:14 GST.

Regards,

Tharun

### 09:52 GST — Priya Nair

Hi Priya,

Thanks for flagging the intermittent 502 errors and invoices that have remained in Validated. We are treating this as a potential P1 because more than one customer may be affected, and we are escalating for engineering investigation now.

Please ask affected customers not to repeat full-file uploads or retry invoices until their existing import IDs and invoice statuses have been reconciled. We cannot yet confirm that the reports share the same cause, or that Marina Bay Hospitality is ready for Wednesday's go-live. I will update you by 10:22 GST, including what we can verify about go-live readiness.

Regards,

Tharun
Product Support Engineer

## Priority and justification

- **Priority:** P1 — provisional, based on the critical restart alert plus reports of intermittent failures across multiple customers and invoices not progressing to submission. The reported scope warrants immediate engineering attention. If investigation shows invoices are still reaching the ASP and only a subset is delayed, downgrade to P2 and explain the change.
- **SLA:** P1 first response within 15–30 minutes, resolution/workaround target 4 hours, updates every 30–60 minutes, 24×7. The 09:44 response to the 09:42 alert and the 09:52 response to the 09:50 support-queue receipt are within target. The current target for resolution/workaround is 13:42 GST; next updates are due by 10:14 to on-call and 10:22 to Priya.

## Investigation summary and timeline

| Time (exercise clock, Monday 16 Nov) | What I checked / did | Finding |
|---|---|---|
| 09:42 | Received Grafana alert | Alert payload says `restarts=3` in 15 minutes, severity critical, environment production. This is supplied alert evidence, not independently reproduced from the lab. |
| 09:44 | Drafted acknowledgement to Support on-call | P1 provisional; immediate L3 escalation prepared. |
| 09:48 / 09:50 | Priya posted / support queue received report | Several customers reportedly saw intermittent portal 502s, mostly during upload; a couple of invoices reportedly stayed Validated since about 09:30. Marina Bay Hospitality go-live is scheduled for Wed 18 Nov. |
| 09:52 | Drafted separate acknowledgement to Priya | Asked affected customers to pause repeat uploads and preserve import IDs; set next update time. |
| Investigation on 14 Nov lab snapshot | Checked Docker state, retained API/web logs, Prometheus, and database read-only | Lab API was healthy with restart count 0; Prometheus showed API up, no restart changes, no recent 5xx, queue depth 0; current database had 43 Accepted invoices and no Validated invoices. These observations predate the exercise incident and cannot disprove the 16 Nov production report. |
| Investigation | Inspected alert rule, API startup/shutdown, proxy, and submission worker code | Found a restart-alert threshold mismatch and a non-durable in-memory submission queue. These are actionable risks, not confirmed causes of the reported production event. |

Read-only queries, sanitized log summary, and source references are in [evidence/incident-2-platform.md](evidence/incident-2-platform.md). The incident date in the inbox is 16 Nov, but the available lab runtime and telemetry are dated 14 Nov. No production logs, restart-time container metadata, affected customer import IDs, or historical 16 Nov metrics were available. No fault injection, retry, database write, or customer communication was performed.

## Root cause and confidence

- **Confirmed from the supplied incident report:** Grafana reported three restarts in 15 minutes. Priya reported multiple customers with intermittent portal 502s and some invoices remaining Validated. These are separate signals; their correlation is not established by the lab evidence.
- **Restart cause:** Unknown. The current API container has `restart_count=0` and `oom_killed=false`, but those are current Nov 14 values, not the historical incident container. The API process exits on startup database failure after its retry budget or on a fatal listener error; Docker is configured with `restart: unless-stopped`. No matching historical fatal/error log was available to identify which, if any, occurred.
- **Possible explanation for Validated invoices:** The submission worker queue is an in-memory channel. A process restart discards queued IDs; invoices already stored as Validated could then remain there until manually retried. A restart during an import could also interrupt processing. This is a credible code-based failure mode, not proof it caused the reported rows.
- **502 distinction:** The web proxy returns 502 when its upstream fetch fails and 504 on timeout. A 502 therefore does not, by itself, prove the API returned 502 or identify a restart as the cause. The retained lab web logs had no matching 502/504 entries.
- **Alert-rule issue:** The supplied alert text says “more than twice,” but `rules.yml` uses a threshold `gt 0`; a single process-start-time change can fire it. The reported value of 3 remains what the email states, but the rule does not enforce its stated threshold.
- **Confidence:** High that the code has a volatile submission queue and that the alert threshold is inconsistent with its text. Low on the production restart cause and on whether the queue mechanism caused Priya's affected invoices, because event-time production data and import IDs are missing.

## Workaround and recovery

- Ask affected customers to pause repeated full-file uploads and preserve each existing import ID and approximate upload time.
- Reconcile invoice-level status history and ASP references before any retry. Do not retry an Accepted invoice; verify an ambiguous Submitted invoice with ASP records first.
- Once the backend is stable, support can use the existing retry path only for confirmed Validated or Failed invoices in their original import. Do not replay a whole file to clear a backlog.
- Do not declare Marina Bay ready for go-live until Engineering confirms the restart cause is understood, the queue/backlog is reconciled, and a documented readiness check passes. This is a hold on the readiness decision, not a claim that go-live has been cancelled.

## L3 escalation draft (not sent)

- **Priority:** P1 provisional — critical alert reports three API restarts in 15 minutes; multiple customers reportedly see 502s and invoices waiting in Validated.
- **Affected:** Customer names, seller IDs, import IDs, invoice numbers, and exact counts are not yet provided. Marina Bay Hospitality go-live is scheduled for 18 Nov.
- **Evidence:** Supplied alert at 09:42 GST reports `restarts=3`; Priya's report received 09:50 GST describes intermittent upload 502s and Validated invoices since about 09:30. Current Nov 14 lab telemetry is healthy and cannot be used to rule out the later reported production incident. See [evidence/incident-2-platform.md](evidence/incident-2-platform.md).
- **Suspected cause:** Unknown. In-memory queue loss on process restart could leave stored invoices Validated; no production event logs or container exit details are available to confirm.
- **Ask:** Page the on-call engineer; inspect production restart/exit/OOM events and database/ASP availability from 09:30 onward; correlate affected import IDs with API and ASP records; recover any Validated invoices not in the live queue without replaying customer files; assess and report a go-live readiness decision for Marina Bay before Priya's afternoon call.
- **Next update due:** 10:14 GST to on-call; 10:22 GST to Priya. L1 retains customer ownership.

## Prevention and follow-up

- Persist submission work in an outbox/queue and re-drive stored Validated invoices safely after restart.
- Align alert threshold with the stated policy (for “more than twice,” use a threshold above 2) and distinguish restart detection from customer-impact alerts.
- Preserve event-time container exit/OOM data and correlate API restarts, web 502/504s, queue depth, and invoice statuses on one timeline.
- Add alerting for invoices remaining Validated beyond the documented threshold and test restart recovery before the Marina Bay go-live decision.

---

# Incident 3 — Al Noor Fuel BR-CO-15

## Acknowledgement

- **Sent at (exercise clock):** 10:18 GST
- **SLA:** P2 first response within 1 hour (due 11:15 GST); 8 business-hour resolution/workaround target (Tue 17 Nov, 08:15 GST, allowing for the 18:00 business-hours close); updates every 2–4 hours. Next update due 12:18 GST.
- **Reply:**

Dear Fatima,

Thank you for sharing the details. We understand the importance of explaining these rejections accurately, particularly while your November VAT position is under audit.

We are checking how the portal calculated the invoice totals in the attached file and comparing the affected invoices with those that pass the same rule.

Please do not change the quantities or prices, or upload the whole workbook again, while we confirm the safe next step. I will update you by 12:18 GST today with the invoice-specific findings and our correction plan.

Kind regards,

Tharun
Product Support Engineer

## Priority and justification

- **Priority:** P2 — several invoices in one customer's batch are reportedly rejected; the attached workbook and calculation code independently reproduce three BR-CO-15 mismatches. The issue affects invoice issuance and an active VAT audit, but there is no evidence of a wider platform outage.
- **SLA:** P2 response within 1 hour, resolution/workaround target 8 business hours, updates every 2–4 hours during business hours Monday–Friday 08:00–18:00 GST. The 10:18 acknowledgement was within target. From the 10:15 receipt, the 8-business-hour target falls at Tue 17 Nov 08:15 GST because the clock pauses at Monday 18:00.

## Investigation summary and log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| Mon 10:15 | Received Fatima's email | Several November invoices reportedly rejected with BR-CO-15; she says dispatch quantities, prices, and line amounts agree and requests invoice-specific explanation for her auditor. |
| Mon 10:18 | Acknowledged Fatima | Asked her not to alter line data or re-upload the entire workbook; set next update for 12:18 GST. |
| Investigation on 14 Nov lab snapshot | Queried Al Noor database records read-only | Only the October baseline import exists. There is no November import in this lab database, so no production invoice statuses or ASP responses can be verified here. |
| Investigation | Parsed the attached November workbook without uploading it | 13 invoice-line rows grouped into six invoice numbers, `ANF-2611-0041` through `ANF-2611-0046`. The three-decimal fuel prices and two-decimal line amounts match the source workbook. |
| Investigation | Reproduced the portal and ASP calculation from source code | Invoices `0042`, `0044`, and `0046` produce BR-CO-15 mismatches of AED 0.01, AED 0.02, and AED 0.01 respectively. The other three pass this rule in the same calculation. |

Detailed results and the read-only database query are in [evidence/incident-3-al-noor.md](evidence/incident-3-al-noor.md). The workbook calculation is evidence for the deterministic rule mismatch, not proof of the actual November import's stored statuses or ASP message IDs.

## Root cause and confidence

- **What happened:** The ASP's BR-CO-15 rule compares invoice gross (IBT-112) with net (IBT-109) plus VAT (IBT-110). The attached workbook's three fuel invoices produce gross totals one or two fils below the sum of their rounded net and VAT totals when processed by this code.
- **Where:** `services/api/internal/pint/pint.go` calculates VAT from rounded line net amounts but calculates gross from unrounded quantity × unit price. It rounds those raw gross values only after summing the invoice. `services/asp-mock/main.go` then compares the rounded gross total to rounded net plus VAT in cents.
- **Why:** Different rounding bases create a cent-level discrepancy. For example, invoice `ANF-2611-0042` computes AED 15,022.42 gross, while AED 14,307.08 net + AED 715.35 VAT = AED 15,022.43. Similar mismatches occur for `0044` and `0046`.
- **Confidence:** High that this calculation defect explains BR-CO-15 for those three invoices in the attached workbook. Production database records are absent, so the exact statuses, which other invoices were accepted, and the ASP response IDs remain unverified.

## Workaround and recovery plan

- Do not change quantities or prices from the dispatch records; the workbook's line amounts are consistent with the supplied values.
- Engineering must correct the portal's gross-total calculation before re-submission can reliably pass BR-CO-15. The workbook has no invoice-total fields for the customer to adjust directly.
- Reconcile the original November import in the customer account. After the calculation fix is confirmed, prepare a file containing only invoices verified as rejected: `ANF-2611-0042`, `ANF-2611-0044`, and `ANF-2611-0046`. Do not re-upload the full workbook or resend invoices already Accepted.
- Rejected invoices do not consume their invoice numbers under the documented lifecycle, but verify the actual status before reusing a number. No upload or retry was performed in this investigation.

## Customer resolution update draft (not sent)

Dear Fatima,

We checked the attached workbook against the portal's total calculation. The mismatch is in how the portal calculates the invoice total including VAT, not in the dispatch quantities or prices you supplied.

For `ANF-2611-0042`, the portal calculation gives AED 15,022.42 including VAT, but the net amount plus VAT is AED 15,022.43. For `ANF-2611-0044`, the figures are AED 11,817.57 versus AED 11,817.59. For `ANF-2611-0046`, they are AED 5,610.99 versus AED 5,611.00. These one- or two-fils differences explain why those invoices fail BR-CO-15 in the attached workbook. The other three invoices pass this particular calculation check.

Please do not change the source quantities or prices, and do not upload the full file again. Our lab database does not contain your November import, so I cannot confirm the live status or ASP reference for each invoice from that environment. We are escalating the portal calculation defect and will reconcile the original import before advising you to submit only confirmed rejected invoices. I will update you by 12:18 GST today; I will not promise a fix time before Engineering confirms it.

Kind regards,

Tharun
Product Support Engineer

## L3 escalation decision and follow-up

- **Decision:** Escalate as a P2 product defect to L3 Engineering through the normal queue. A code-based total calculation reproducibly creates the exact rule mismatch reported. This is not a P1 because the evidence indicates a subset of invoices for one customer, not a platform-wide inability to submit.
- **Engineering ask:** Change invoice gross calculation so IBT-112 is derived from the same rounded IBT-109 and IBT-110 values that are sent; add regression tests for three-decimal quantity prices and line rounding; verify the corrected payload against BR-CO-15. Confirm when the fix is available for a safe retry plan.
- **Customer follow-up:** Keep Fatima updated every 2–4 hours; reconcile actual import and ASP records; share invoice-level status and a safe three-invoice-only correction path after the fix is verified. Provide the written calculation explanation for her audit.
- **Remaining risk:** Without the November production import and ASP response records, the exact recorded rejection set and acceptance statuses cannot be confirmed from this lab.

## Prevention ideas

- Use one consistent rounded-total basis for invoice net, VAT, and gross values.
- Add unit and ASP contract tests for rounding at line and invoice level, especially fuel prices with three decimals.
- Include calculated-versus-expected figures in validation diagnostics while preserving the original values for audit.
- Add invoice-level reconciliation details so finance teams can distinguish their source data from portal-calculated totals.

---

# Incident 4 — Bluepeak / Sahara Medical invoices not received

## Acknowledgement

- **Sent at (exercise clock):** 11:33 GST
- **SLA:** P2 first response within 1 hour (due 12:30 GST); 8 business-hour resolution/workaround target (Tue 17 Nov, 09:30 GST, allowing for the 18:00 business-hours close); updates every 2–4 hours. Next update due 13:33 GST.
- **Reply:**

Hi Daniel,

Thank you for the details. An HTTP 201 confirms the import request was accepted by the portal; it does not confirm that every invoice received an ASP acknowledgement or reached the buyer.

Please hold off on resending the batch. I am checking each invoice's status and ASP reference against the available submission records so we can identify what was received and what still needs action.

I will send you a verified reconciliation or a clear evidence gap, with safe next steps, by 13:33 GST today.

Regards,

Tharun
Product Support Engineer

## Priority and justification

- **Priority:** P2 — a 40-invoice batch for one seller is reportedly only partially received, with payment-run impact this week. This is a material subset delivery issue, but no multi-customer outage is evidenced. Raise to P1 if investigation shows that invoices cannot reach the ASP for one or more customers broadly.
- **SLA:** P2 first response within 1 hour, resolution/workaround target 8 business hours, updates every 2–4 hours during business hours Monday–Friday 08:00–18:00 GST. The 11:33 response was within the 12:30 first-response deadline. From receipt at 11:30 Monday, 6.5 business hours elapse before close, with the remaining 1.5 hours due Tuesday 09:30 GST.

## Investigation summary and timeline

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| Mon 11:30 | Received Daniel's email | Connector reports HTTP 201 for a 40-invoice import; portal shows Submitted; hospital reportedly received only a few; ASP references are blank for many. Daniel is holding the resend. |
| Mon 11:33 | Acknowledged Daniel | Explained that HTTP 201 is not invoice-level ASP acceptance; asked him not to resend the batch and set next update for 13:33 GST. |
| Investigation on 14 Nov lab snapshot | Queried Sahara database records read-only | Only the Oct 29 baseline import exists: 12 invoices, all Accepted with ASP message IDs. No matching November import exists in this lab database. |
| Investigation | Queried current mock ASP document list and retained throttle logs | The current in-memory mock returned no Sahara documents and retained logs had no rate-limit entries. This snapshot predates the 16 Nov exercise incident and cannot establish the reported batch outcome. |
| Investigation | Inspected importer, worker, ASP rate limiter, retry path, and sample generator | The sample is 40 invoices from one sender; the ASP allows a burst of 8 then 2 per second. The submitter has four workers and does not handle HTTP 429, leaving a code path where an invoice stays Submitted without an ASP reference. |

See [evidence/incident-4-bluepeak-sahara.md](evidence/incident-4-bluepeak-sahara.md) for read-only results and code references. No upload, replay, retry, database write, or customer email was performed.

## Root cause and confidence

- **Confirmed from source:** `POST /v1/imports` returns after parsing/portal validation and enqueueing; ASP submissions run in background workers. The worker changes an invoice to Submitted before the ASP request. It stores an ASP reference only after HTTP 200. It handles 200, 422, and selected 5xx statuses, but has no HTTP 429 case or `Retry-After` retry. For a 429 response, the invoice remains Submitted with an empty reference; the ASP mock does not store a document for a throttled request.
- **Likely explanation:** The sample contains 40 invoices sent by the same seller, while the mock ASP allows a burst of 8 documents per sender and then 2 per second. Four workers can submit faster than that sustained rate. Excess requests can receive 429, which matches Submitted status, blank references, and partial receipt. The mechanism is strongly supported by code, but the exact response codes and invoice IDs for Daniel's original batch are unverified because its import and logs are absent.
- **Other unverified possibility:** The ASP could have accepted an invoice but the database update that stores its message ID could have failed. Therefore a blank reference alone is not proof that the ASP did not receive it; reconcile with ASP documents before any resend.
- **Important retry limitation:** Both invoice retry and batch retry select only Validated or Failed records. A Submitted invoice is rejected by the retry endpoint, so the existing UI retry path cannot safely recover this likely 429 state.
- **Delivery wording:** Accepted with an ASP message ID confirms the ASP acknowledged receipt; the lab does not expose a buyer-side delivery receipt. Do not equate ASP acceptance with confirmed arrival at Emirates Palm Hospital.
- **Confidence:** High that the code can produce Submitted/blank-reference records on 429. Medium that this explains most of this reported batch, given the 40 same-sender invoices and partial receipt. The exact count, per-invoice ASP outcomes, and buyer delivery remain unknown.

## Workaround and recovery plan

- Do not resend the connector batch. Obtain its import ID and original request time, then reconcile all 40 invoice numbers against portal status history and ASP receipt records.
- Do not resend invoices with an ASP message ID or a matching ASP document. For Submitted rows with no reference, check ASP records by sender and invoice number; blank reference alone is ambiguous.
- The current retry endpoint accepts only Validated or Failed records. Do not manually edit status or invoke a retry for Submitted rows. If an invoice is confirmed absent from the ASP, ask Engineering for an approved, idempotent recovery path before sending it again.
- If ASP acceptance is confirmed but the hospital still cannot find an invoice, confirm the receiver and escalate the ASP-to-buyer delivery question; Invoicely's `Accepted` status alone is not a buyer receipt.

## Customer update draft (not sent)

Hi Daniel,

We have identified a plausible reason for the partial results: the ASP limits how quickly one seller can send documents, and the current submission worker does not correctly recover from an ASP throttle response. That can leave an invoice marked Submitted even though no ASP reference was recorded. We do not yet have the original import record or request logs here, so I cannot confirm that this is what happened to your batch or identify which invoices were affected.

Please continue to hold the resend. Could you send the import ID from the 201 response and the connector request timestamp? We will match each invoice against the portal and ASP records before advising any retry. We will not ask you to resend the full batch. I will update you by 13:33 GST today; ASP acceptance and hospital receipt will be reported separately.

Regards,

Tharun
Product Support Engineer

## L3 escalation decision and follow-up

- **Decision:** Escalate as a P2 product defect to L3 Engineering through the normal queue. This is a code-handling gap with a plausible delivery impact; no actual 429 records from the reported batch are available to prove it.
- **Engineering ask:** Implement bounded retries honoring `Retry-After` for 429; persist submission work; keep throttled invoices in a clearly retryable state; make retries idempotent; and provide a safe reconciliation/re-drive for Submitted rows without ASP references. Add a test for a 40-invoice same-sender batch and verify the imported/ASP outcomes per invoice.
- **Evidence to attach:** Original import ID, request time, invoice-number list, portal event history, ASP 200/429 logs and `Retry-After` values, plus any ASP document list for the sender. Current local lab evidence is not a substitute for these production records.
- **Next customer update:** 13:33 GST. L1 retains ownership and must update even if engineering has no resolution yet.

## Prevention ideas

- Respect the ASP's per-sender throughput limit with a paced queue and retries that honor `Retry-After`.
- Persist queue items and expose throttled/retryable state instead of leaving them Submitted without a reference.
- Make ASP submission idempotent and reconcile ambiguous outcomes before resending.
- Distinguish import accepted, ASP accepted, and buyer delivery in API responses and portal wording.

---

# Incident 5 — Ghaf Tree Café date error

## Acknowledgement

- **Sent at (exercise clock):** 12:13 GST
- **SLA:** P3 first response within 4 hours (due 16:10 GST); resolution target 3 days; daily updates. Next update due Tue 17 Nov at 12:13 GST.
- **Reply:**

Hello Hessa,

Thank you for contacting us. You are right that 13/11/2026 is a valid date when written as day/month/year. The portal currently requires YYYY-MM-DD text or a real Excel date cell, so it does not guess the meaning of slash-formatted text.

I will check the attached file and send you a simple correction for both date columns. Please do not change any invoice amounts or other details, and do not upload again until we have checked the existing import status.

I will update you by 12:13 GST tomorrow.

Regards,

Tharun
Product Support Engineer

## Priority and justification

- **Priority:** P3 — one customer's file uses a date format not accepted by the documented template. The rule message identifies the required format, and there is no evidence of a platform-wide issue.
- **SLA:** P3 first response within 4 hours, resolution target 3 days, daily updates during business hours Monday–Friday 08:00–18:00 GST. The 12:13 response was within the 16:10 deadline.

## Investigation summary and timeline

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| Mon 12:10 | Received Hessa's email | All invoices reportedly rejected with an issue-date error showing `13/11/2026`; Hessa says the date is correct in day/month/year notation. |
| Mon 12:13 | Acknowledged Hessa | Recognized the date is valid in her format, explained the supported format at a high level, and set a next update time. |
| Investigation on 14 Nov lab snapshot | Queried Ghaf database records read-only | No Ghaf import or invoice rows exist in the current database, so the actual exercise-day import status cannot be checked here. |
| Investigation | Inspected the attached workbook without uploading it | Eight rows contain four invoice groups (`GTC-118`–`GTC-121`). Issue and due dates are slash-formatted text: 13/11, 14/11, or 15/11/2026 and 13/12, 14/12, or 15/12/2026. |
| Investigation | Inspected `sheet.ParseDate` and the validation rule | Parser accepts valid `YYYY-MM-DD` text or Excel serial dates; slash-formatted text is rejected with AE-BR-04. Each invoice group therefore fails portal validation and is not submitted to the ASP if the attached file was uploaded unchanged. |

See [evidence/incident-5-ghaf-cafe.md](evidence/incident-5-ghaf-cafe.md) for the read-only workbook and database findings.

## Root cause and confidence

- **What happened:** The sample stores dates as text in `DD/MM/YYYY` format. This format is valid for Hessa's locale but is not one of the portal's accepted representations.
- **Where:** `services/api/internal/sheet/sheet.go` `ParseDate` accepts only a valid ISO `YYYY-MM-DD` string or a numeric Excel date serial. `services/api/internal/validate/validate.go` raises AE-BR-04 when either issue or due date cannot be parsed. The template field guide documents these supported formats.
- **Why:** The parser intentionally does not guess whether a slash-formatted string is day/month or month/day, which avoids silently assigning the wrong tax-invoice date. This is expected validation behavior, not evidence that Hessa's calendar date is invalid.
- **Confidence:** High that the attached sample will fail AE-BR-04 for all four invoices. The live import status is unverified because no November Ghaf record is present in the lab database.

## Workaround and customer resolution draft (not sent)

Hello Hessa,

We checked the attached workbook. Your dates are valid as day/month/year, but the file contains them as text such as `13/11/2026`. The portal accepts `YYYY-MM-DD` text or a real Excel date cell; it does not interpret slash-formatted text because that can mean different dates in different regions.

On a copy of the workbook, update both the invoice issue date and payment due date columns. For example, change `13/11/2026` to `2026-11-13`, `14/11/2026` to `2026-11-14`, and `15/11/2026` to `2026-11-15`. Change the due dates similarly: `13/12/2026` to `2026-12-13`, `14/12/2026` to `2026-12-14`, and `15/12/2026` to `2026-12-15`.

These sample values are text, so changing only the Excel cell display format will not convert them. You can replace the text values directly or use a helper formula to convert each date, then paste the results as values. Please first check that the portal shows the original four invoices as Rejected; portal-rejected invoices were not sent to the ASP. If any status is different, send me the import ID before uploading so we can avoid duplicates. Once confirmed, upload the corrected invoices once. I will update you by 12:13 GST tomorrow.

Regards,

Tharun
Product Support Engineer

## Escalation decision and follow-up

- **Decision:** No L3 escalation. The sample behavior matches the documented accepted date formats, and the error message identifies the required format and received value. Give the customer a clear correction and remain available for the upload reconciliation.
- **Product suggestion:** Consider adding a short UAE day/month example beside the date error or at upload time. Do not silently reinterpret slash-formatted dates.
- **Remaining risk:** Current lab data has no November Ghaf import, so the actual import ID and invoice statuses still need confirmation before the customer re-uploads.
- **Follow-up:** Send the promised update by Tue 17 Nov 12:13 GST, even if the remaining step is only confirming the original import status.

---

# Overall

## Prioritisation

The requested investigation order was followed: Oasis first, then the remaining incidents one at a time after approval. Final evidence-backed triage is:

1. **Incident 2 — P1 provisional:** Critical restart alert plus reports of multiple customers with intermittent 502s and stalled invoices; page L3 on-call and re-prioritise if production evidence narrows the impact.
2. **Incident 1 — P2 provisional:** Major upload delay and possible timeout/retry confusion with month-end impact; do not re-upload before original import reconciliation.
3. **Incident 3 — P2:** Three invoices in the attached workbook reproduce a BR-CO-15 calculation mismatch, with audit impact; escalate the calculation defect through the normal L3 queue.
4. **Incident 4 — P2 provisional:** Partial receipt reported for a 40-invoice batch before a payment run; likely 429 handling gap, but exact outcomes require the original import and ASP records.
5. **Incident 5 — P3:** One customer's dates use a documented unsupported text format; provide conversion steps and verify the original status before upload.

Acknowledgements were drafted within the selected first-response targets: Oasis 09:07; API alert 09:44 and Priya 09:52; Al Noor 10:18; Sahara 11:33; Ghaf 12:13 GST. Incident 2's internal alert and Priya's report have separate acknowledgement drafts.

## What I would do differently

- Capture each original import ID, request timestamp, and invoice list at first contact, before any retry is considered.
- Separate import acceptance, ASP acknowledgement, and buyer-side delivery in every investigation; HTTP 201 and Submitted are not proof of ASP acceptance.
- Query durable event history and ASP records before advising recovery, and preserve the original logs and metric time range.
- Record the difference between the exercise incident date and the local lab snapshot immediately; do not use a healthy or empty pre-incident lab state to disprove a later production report.
- Give a precise next-update time in every acknowledgement and keep customer ownership after escalation.

## What I learned / what I would need to learn

- The upload endpoint parses and validates synchronously, but ASP submission is asynchronous; an import-level 201 does not mean every invoice was accepted.
- A web proxy timeout can end the browser wait while a background API operation continues; idempotency and reconciliation are essential before retrying.
- ASP rate limits, `Retry-After`, queue durability, and per-invoice ASP message IDs determine whether a Submitted state is safe to resend.
- Invoice totals can disagree by cents when rounded line net/VAT values are combined with a gross total calculated from unrounded amounts.
- Date-only values should be normalized without guessing locale; the file's representation matters as well as the calendar date.
- I used read-only PostgreSQL queries, Prometheus HTTP queries, retained Docker logs, source inspection, and in-memory XLSX XML parsing. I would still need production event-time logs, restart/OOM metadata, affected import IDs, and ASP receipts to confirm the unverified timelines.

## Time log (real time)

| Date | From–to | What I worked on |
|---|---|---|
| 14 Nov 2026 | Start not captured – 12:32:04 +05:30 | Final incident investigation, evidence documentation, and submission validation; end timestamp read from the local system clock. |

The start time was not captured, so elapsed hours/minutes cannot be calculated accurately and have not been estimated. Do not use the exercise clock in this section.

## AI usage disclosure

I used GitHub Copilot to help navigate the repository, identify relevant code paths, run read-only database and Prometheus queries, inspect sample workbook contents, calculate expected outcomes, and draft the investigation notes and customer communications.

I checked technical conclusions against the supplied source code, actual workbook values, current local database records, retained logs, and live lab metrics. Hypotheses are labelled as such where original production records were unavailable. No customer email was sent and no database reset or mutation was performed. I reviewed the previously completed controlled Oasis replay as existing evidence and did not run any additional upload replay.

## Final deliverables checklist

- [x] Acknowledgement drafts and priorities for all five inbox items.
- [x] Full investigations for all five incidents, with confidence and evidence gaps stated.
- [x] Separate acknowledgement drafts for the Incident 2 alert and Priya's customer-impact report.
- [x] Customer update/resolution drafts and actionable workarounds; no real emails sent.
- [x] Engineering escalation drafts for product defects and a documented no-escalation decision for the date-format case.
- [x] Sanitized evidence notes: [Oasis](evidence/oasis-investigation.md), [API restart](evidence/incident-2-platform.md), [Al Noor](evidence/incident-3-al-noor.md), [Sahara](evidence/incident-4-bluepeak-sahara.md), and [Ghaf](evidence/incident-5-ghaf-cafe.md).
- [x] AI assistance disclosed.
- [x] Repository URL recorded from the configured Git origin; no commit or push was performed.
- [ ] Candidate to enter actual real-time hours before final submission; this value was not available to the assistant and has not been invented.

**Git status at completion:** `SUBMISSION.md` modified; `evidence/` added; `.vscode/` was already untracked before this work and was left unchanged. No source code was modified and no commit was created.
