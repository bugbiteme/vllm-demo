# Claims Submission Policy

*Fictitious policy document for RAG testing purposes only. Applies to Meridian Health Partners commercial plans.*

## Timely Filing Limits

| Claim Type | Filing Deadline |
|---|---|
| In-network provider claims | 90 days from date of service |
| Out-of-network provider claims | 180 days from date of service |
| Member-submitted reimbursement claims | 365 days from date of service |
| Corrected claims / resubmissions | 60 days from original remittance advice date |

Claims received after the applicable deadline will be **denied for timely filing** (see denial code CO-29 in `claim-denial-reason-codes.md`) unless the provider or member demonstrates a qualifying exception (see Exceptions section below).

## Required Claim Information

A claim will be rejected prior to adjudication if any of the following are missing or invalid:

1. Member ID and group number
2. Rendering provider National Provider Identifier (NPI)
3. Valid ICD-10 diagnosis code(s) supporting medical necessity
4. Valid CPT/HCPCS procedure code(s)
5. Date(s) of service
6. Place of service code
7. Billed charge amount
8. For facility claims: type of bill and revenue codes

## Submission Methods

- **Electronic (preferred):** EDI 837 transaction via Meridian's clearinghouse, payer ID 77401.
- **Paper:** CMS-1500 (professional) or UB-04 (institutional) forms mailed to Meridian Claims Processing, PO Box 44210, Meridian, ST 55501.
- **Member reimbursement claims:** Meridian Member Portal or the paper Member Claim Reimbursement Form.

## Coordination of Benefits (COB)

When a member has coverage under more than one plan, Meridian determines primary/secondary payer status using the **birthday rule** for dependents (the parent whose birthday falls earlier in the calendar year is primary) and standard COB rules for employer vs. individual coverage. Claims submitted without COB information when other coverage is on file will be pended for up to 30 days pending COB verification before denial.

## Exceptions to Timely Filing

Meridian will waive timely filing denials when the provider or member demonstrates one of the following, with supporting documentation submitted within 30 days of the original denial:

- Retroactive member eligibility determination.
- System outage on Meridian's clearinghouse, confirmed by Meridian's EDI operations team.
- Natural disaster or declared state of emergency affecting the provider's billing operations.
- Newborn claims submitted within 90 days of the retroactive addition of the newborn to the policy.

## Claim Status and Remittance

Adjudicated claims generate an Electronic Remittance Advice (ERA, ANSI 835) or paper Explanation of Payment (EOP) within 14 business days of receipt for clean claims, and within 30 business days for claims requiring manual review. Providers may check real-time claim status through the Meridian Provider Portal.
