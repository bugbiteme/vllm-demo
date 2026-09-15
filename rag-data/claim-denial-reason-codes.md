# Claim Denial Reason Code Reference

*Fictitious reference document for RAG testing purposes only. Uses standard ANSI X12 Claim Adjustment Reason Code (CARC) conventions for realism.*

## Common Denial Codes

| Code | Description | Typical Resolution |
|---|---|---|
| CO-29 | Timely filing limit expired | Submit appeal with proof of timely submission or qualifying exception (see `claims-submission-policy.md`) |
| CO-50 | Non-covered service — not deemed a medical necessity | Submit clinical documentation supporting necessity via appeal, or verify against `medical-necessity-criteria-imaging.md` if applicable |
| CO-96 | Non-covered charge — service excluded under member's plan | Verify plan exclusions in `plan-benefits-summary.md`; not typically appealable unless plan document was applied incorrectly |
| CO-97 | Benefit included in payment for another procedure already adjudicated | Verify bundling/unbundling rules; submit corrected claim if procedures were billed in error |
| CO-197 | Precertification/authorization absent | Submit retroactive authorization request if criteria met, or file appeal with clinical urgency documentation |
| CO-18 | Duplicate claim/service | No action needed if original claim was paid; submit corrected claim if services were genuinely distinct |
| CO-16 | Claim lacks information needed for adjudication | Resubmit corrected claim with missing information |
| CO-109 | Claim not covered by this payer/contractor — member not covered by Meridian on date of service | Verify member eligibility via `member-eligibility-verification.md`; redirect claim to correct payer if applicable |
| CO-45 | Charge exceeds contracted/allowed amount | Provider write-off required per network contract; not billable to member beyond applicable cost-share |
| PR-1 | Patient responsibility — deductible not yet met | Informational; bill member per plan cost-sharing terms |
| PR-2 | Patient responsibility — coinsurance | Informational; bill member per plan cost-sharing terms |
| PR-3 | Patient responsibility — copayment | Informational; bill member per plan cost-sharing terms |
| CO-11 | Diagnosis code inconsistent with procedure billed | Verify ICD-10/CPT code pairing and resubmit corrected claim |
| CO-B7 | Provider not certified/eligible for this procedure on the service date | Verify provider credentialing status was active on date of service |

## Denial Category Definitions

- **CO (Contractual Obligation):** The provider cannot bill the member for this amount; it is written off per network contract terms.
- **PR (Patient Responsibility):** The member is responsible for this amount per their plan's cost-sharing structure. This is not a "denial" in the traditional sense — it reflects normal cost-sharing.
- **OA (Other Adjustment):** Adjustment not attributable to contractual obligation or patient responsibility, such as an interest payment adjustment.

## Escalation Path for Disputed Denials

1. **Reconsideration request** — informal review, no formal appeal filed, typically resolved within 15 business days. Appropriate for suspected processing errors (e.g., wrong code pairing, COB misapplication).
2. **Formal appeal** — see `member-appeals-grievance-process.md` for the complete process, timelines, and required documentation.
3. **External review** — available after formal appeal is exhausted, for adverse determinations involving medical necessity or experimental/investigational status.

## Most Frequently Overturned Denials on Appeal

Based on Meridian's internal quality review, the denial codes most frequently overturned on appeal are CO-50 (medical necessity, when additional clinical documentation is provided) and CO-197 (missing authorization, when retroactive authorization criteria are met). Providers are encouraged to submit complete clinical documentation with the original claim to reduce avoidable appeals.
