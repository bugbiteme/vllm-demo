# Member Eligibility Verification

*Fictitious operational policy for RAG testing purposes only. Describes how Meridian Health Partners verifies member eligibility for claims processing.*

## Eligibility Data Sources

Member eligibility is determined by the enrollment record on file as of the date of service, sourced from:

- Employer group enrollment feeds (for group commercial plans), updated nightly.
- Marketplace enrollment files (for individual/family plans), updated per CMS exchange transaction schedules, typically within 3 business days of a qualifying event.
- Direct member-submitted applications (for direct-pay individual plans), effective per the coverage effective date on the signed application.

## Verifying Eligibility Before Rendering Services

Providers should verify eligibility through one of the following, in order of preference:

1. **Real-time eligibility (270/271 transaction)** via the Provider Portal or clearinghouse — recommended for same-day accuracy.
2. **Provider Portal member lookup** — shows current plan, effective date, deductible/out-of-pocket accumulator status, and copay amounts.
3. **Member ID card verification** — card shows plan type and group number but does not guarantee active coverage; always confirm with real-time lookup for high-cost or scheduled services.

Claims submitted for a member who is not eligible on the date of service will deny with code CO-109 (see `claim-denial-reason-codes.md`).

## Effective and Termination Dates

- **New enrollment:** Coverage is effective on the date specified in the enrollment record — typically the 1st of the month following enrollment for group plans, or the qualifying event date for special enrollment periods.
- **Termination:** Coverage terminates at 11:59 PM on the last day of the final covered month, except in cases of retroactive termination due to non-payment of premium (see below).

## Retroactive Terminations

If a member's coverage is retroactively terminated due to non-payment of premium (after the applicable grace period), claims paid during the retroactive termination window may be subject to recovery from the provider. Meridian provides a minimum 30-day grace period for individual marketplace plans before retroactive termination is applied, per standard grace period rules; premium-subsidized members receive a 90-day grace period, with claims in the second and third months of the grace period pended until premium status is resolved.

## Newborn and New Dependent Coverage

Newborns are automatically covered under the mother's policy for the first 31 days of life. The subscriber must formally add the newborn to the policy within 31 days to maintain continuous coverage beyond that window; claims for days 32+ will deny if the newborn has not been added to the enrollment record.

## Disputed Eligibility

If a provider believes an eligibility denial (CO-109) is in error — for example, the member asserts they were enrolled as of the date of service — the provider or member may request an eligibility reconsideration by submitting proof of enrollment (confirmation letter, premium payment receipt, or employer HR confirmation) to Member Services at 555-0100. Reconsiderations are typically resolved within 10 business days.
