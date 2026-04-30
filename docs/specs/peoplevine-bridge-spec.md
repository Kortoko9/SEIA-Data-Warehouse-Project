# PeopleVine Bridge Spec

## Status

Design Spec / Not Yet Implemented

Implementation details will be updated after PeopleVine Bronze/Silver queries or notebooks are built.

## Purpose

Defines the planned bridge between Toast payments/checks and PeopleVine members/accounts.

This bridge is required for member spend reporting, house account attribution, unresolved member-match queues, and manual override workflows.

## Why This Exists

Toast captures POS orders, checks, selections, payments, house account references, and tab names.

PeopleVine is the member/account system of record.

Toast alone may not always identify the correct member or PeopleVine account with enough confidence. This bridge defines how we intend to connect the two systems in a controlled and auditable way.

## Planned Output

Primary planned bridge table:

`bridge_toast_payment_to_member`

Status:

`Planned / Not Built`

Expected grain:

`One row per Toast payment-to-member attribution result`

Planned primary key:

`restaurant_guid + payment_guid`

## Source Inputs

### Existing Toast Inputs

| Source Table | Field | Purpose |
| --- | --- | --- |
| stg_toast_payment | payment_guid | Toast payment identifier |
| stg_toast_payment | check_guid | Link payment to check |
| stg_toast_payment | order_guid | Link payment to order |
| stg_toast_payment | paid_business_date | Payment reporting date |
| stg_toast_payment | amount | Payment amount |
| stg_toast_payment | house_account_reference | Best Toast-side member/account reference |
| stg_toast_check | tab_name | Fallback name matching field |
| stg_toast_check | guest_reference | Possible customer/member reference |

### Planned PeopleVine Inputs

| Planned Table | Purpose | Status |
| --- | --- | --- |
| brz_peoplevine_member | Raw PeopleVine member payload/export | Not Built |
| brz_peoplevine_account | Raw PeopleVine account payload/export | Not Built |
| brz_peoplevine_invoice | Raw PeopleVine invoice payload/export | Not Built |
| brz_peoplevine_transaction | Raw PeopleVine transaction/payment payload/export | Not Built |
| stg_peoplevine_member | Normalized member table | Not Built |
| stg_peoplevine_account | Normalized account table | Not Built |
| stg_peoplevine_invoice | Normalized invoice table | Not Built |
| stg_peoplevine_transaction | Normalized transaction table | Not Built |

## Planned Bridge Columns

| Column | Description |
| --- | --- |
| restaurant_guid | Toast restaurant/location identifier |
| payment_guid | Toast payment identifier |
| check_guid | Toast check identifier |
| order_guid | Toast order identifier |
| paid_business_date | Payment business date |
| payment_amount | Toast payment amount |
| house_account_reference | Toast house account/account reference |
| toast_tab_name | Toast check/tab name |
| guest_reference | Toast guest/customer reference, if available |
| matched_member_id | Resolved PeopleVine member ID |
| matched_account_id | Resolved PeopleVine account ID |
| match_method | Matching tier/method used |
| match_confidence | Confidence score for the match |
| match_status | MATCHED / UNRESOLVED / MANUAL_OVERRIDE |
| unresolved_reason | Why no match was found, if unresolved |
| manual_override_id | Link to manual override table, if used |
| first_seen_at_utc | First time attribution row was created |
| last_seen_at_utc | Last time attribution row was observed |
| created_at_utc | Row creation timestamp |
| updated_at_utc | Row update timestamp |
| run_id | Pipeline run identifier |

## Matching Priority

### Tier 1 — Deterministic Account ID Match

Preferred match:

`stg_toast_payment.house_account_reference → stg_peoplevine_account.account_id`

Use when the Toast house account reference directly maps to a PeopleVine account ID.

Expected confidence:

`1.000000`

Match status:

`MATCHED`

### Tier 2 — Deterministic External Reference Match

Fallback match:

`house_account_reference → PeopleVine external account reference`

Use if PeopleVine provides an external ID or integration key that maps to Toast.

Expected confidence:

`0.950000–1.000000`

### Tier 3 — Account Name Match

Fallback match:

`house_account_reference → PeopleVine account name`

Use normalized account names only.

Rules:

- trim spaces
- lowercase
- remove punctuation where safe
- avoid matching if multiple accounts share same normalized name

Expected confidence:

`0.750000–0.900000`

### Tier 4 — Tab Name / Member Name Match

Fallback match:

`stg_toast_check.tab_name → PeopleVine member/account name`

Use only as a weaker candidate match.

Rules:

- do not auto-accept ambiguous matches
- avoid if multiple candidates exist
- send low-confidence results to unresolved queue

Expected confidence:

`0.500000–0.750000`

### Tier 5 — Manual Override

If automated matching fails or is ambiguous, use:

`ctl_manual_override_member_match`

Manual overrides should be auditable and include:

- override reason
- resolved member ID
- resolved account ID
- effective date range
- created by
- active flag

## Unresolved Queue Rule

If no acceptable match is found, create or update a record in:

`ctl_unresolved_member_match`

Required unresolved fields:

| Field | Purpose |
| --- | --- |
| payment_guid | Payment needing attribution |
| check_guid | Check tied to payment |
| order_guid | Order tied to payment |
| business_date | Toast sales business date |
| paid_business_date | Toast payment business date |
| house_account_reference | Best Toast reference available |
| toast_tab_name | Fallback name field |
| payment_amount | Payment amount |
| candidate_member_id | Candidate member, if any |
| candidate_account_id | Candidate account, if any |
| match_method_attempted | Matching method attempted |
| match_confidence | Confidence score |
| unresolved_reason | Why unresolved |
| status | OPEN / RESOLVED / IGNORED |

## Manual Override Rule

Manual override records must not be treated as informal edits.

They must be stored in:

`ctl_manual_override_member_match`

Manual override logic should apply before weak fuzzy matching when an active override exists.

Override precedence:

1. Active manual override
2. Deterministic PeopleVine account match
3. External reference match
4. Account name match
5. Tab/member name match
6. Unresolved queue

## Match Status Values

| Status | Meaning |
| --- | --- |
| MATCHED | Confident automated match |
| UNRESOLVED | No acceptable match |
| MANUAL_OVERRIDE | Match resolved by approved override |
| REVIEW | Candidate match exists but needs human review |
| IGNORED | Record intentionally excluded |

## Match Method Values

| Match Method | Description |
| --- | --- |
| MANUAL_OVERRIDE | Matched using ctl_manual_override_member_match |
| HOUSE_ACCOUNT_ID | house_account_reference matched to PeopleVine account ID |
| EXTERNAL_REFERENCE | house_account_reference matched to external reference |
| ACCOUNT_NAME_EXACT | normalized account name exact match |
| ACCOUNT_NAME_FUZZY | normalized account name fuzzy match |
| TAB_NAME_MEMBER_EXACT | tab name matched to member name |
| TAB_NAME_MEMBER_FUZZY | tab name fuzzy matched to member name |
| UNMATCHED | No match found |

## Confidence Guidance

| Confidence Range | Interpretation | Action |
| --- | --- | --- |
| 1.000000 | Deterministic | Auto-match |
| 0.900000–0.999999 | Very strong | Auto-match if unique |
| 0.750000–0.899999 | Moderate | Review if material/ambiguous |
| 0.500000–0.749999 | Weak | Send to unresolved/review |
| Below 0.500000 | Not reliable | Do not match |

## Success Criteria

The PeopleVine bridge is successful when:

| Requirement | Complete? |
| --- | --- |
| PeopleVine source list confirmed | ☐ |
| PeopleVine Bronze ingestion built | ☐ |
| PeopleVine Silver member/account tables built | ☐ |
| Toast payment-to-member bridge table built | ☐ |
| Manual override process tested | ☐ |
| Unresolved queue populated for failed matches | ☐ |
| Match confidence rules implemented | ☐ |
| Member spend mart can use the bridge | ☐ |
| Query / Notebook Inventory updated | ☐ |
| Data Dictionary updated | ☐ |

## Not Yet Implemented

The following are intentionally not implemented yet:

- PeopleVine Bronze queries
- PeopleVine Silver normalization
- bridge_toast_payment_to_member
- member spend Gold mart
- automated confidence scoring

## Development Rule

Do not treat this page as implemented code.

When PeopleVine ingestion begins, implementation must follow this order:

1. Confirm PeopleVine endpoint/export list
2. Build PeopleVine Bronze ingestion
3. Build PeopleVine Silver member/account/invoice tables
4. Build first deterministic bridge prototype
5. Add unresolved match writes
6. Add manual override precedence
7. Register all code in Query / Notebook Inventory
8. Update this page with actual field names and tested logic

## Open Questions

| Question | Status |
| --- | --- |
| What exact PeopleVine export/API endpoints are available? | Open |
| What is the most reliable PeopleVine account key? | Open |
| Does Toast house_account_reference directly contain a PeopleVine account identifier? | Open |
| Are corporate accounts structured differently from individual member accounts? | Open |
| Should manual overrides be maintained by Finance only? | Open |
| What confidence threshold should be required for auto-match? | Open |

## Next Step

Confirm the PeopleVine source list and available account/member fields before implementation.