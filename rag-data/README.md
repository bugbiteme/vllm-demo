# RAG Test Corpus — Meridian Health Partners (Fictitious)

This directory contains **synthetic, fictional documents** created to test a Retrieval-Augmented Generation (RAG) pipeline against a healthcare-provider-and-insurer use case (claims validation, member benefits, clinical policy lookup, provider operations).

**Everything here is made up.** "Meridian Health Partners" is not a real company. All member names, IDs, phone numbers, addresses, provider names, NPIs, and case scenarios are fabricated for testing purposes only. Do not treat any figure, policy detail, or clinical criterion here as real-world guidance — this exists purely to give a RAG pipeline realistic-looking documents to chunk, embed, and retrieve against.

## Documents

| File | Simulates | Useful for testing... |
|---|---|---|
| `plan-benefits-summary.md` | A Summary of Benefits and Coverage (SBC) | Member benefit Q&A |
| `claims-submission-policy.md` | Claims filing rules and timelines | Claims-validation queries |
| `prior-authorization-requirements.md` | Services requiring pre-approval | Claims-validation queries |
| `medical-necessity-criteria-imaging.md` | Clinical policy for advanced imaging | Clinical/claims policy lookup |
| `claim-denial-reason-codes.md` | Denial code reference | Claims-validation, denial explanation |
| `member-appeals-grievance-process.md` | Appeals process | Member support queries |
| `provider-network-directory.md` | Sample in-network provider listing | Provider lookup queries |
| `prescription-drug-formulary.md` | Drug tier/coverage rules | Pharmacy benefit queries |
| `care-management-programs.md` | Chronic disease management programs | Care-coordination queries |
| `member-eligibility-verification.md` | Eligibility verification rules | Claims-processing queries |

Good test questions to try once these are ingested: *"Is an MRI of the lumbar spine covered without prior authorization?"*, *"What's the timely filing limit for out-of-network claims?"*, *"Why would a claim be denied with reason code CO-50?"*, *"What tier is metformin on?"*
