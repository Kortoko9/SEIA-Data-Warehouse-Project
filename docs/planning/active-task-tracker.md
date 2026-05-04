# Active Task Tracker

| ID | Workstream | Task | Priority | Owner | Dependency | Status | Target Date | Done When | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| T03 | Security | Confirm Fabric production path after trial | High | Kris / IT | None | Waiting | TBD | Capacity / supported production path is documented | Permanent Fabric capacity / production path still needs IT decision. |
| T18 | Gold | Build first reconciliation mart | High | Kris | T12, T13, T14, T16, T17 | In Progress | TBD | Sales-side and tender-side figures available for validation | NB_BUILD_RECON_MART built and running. Void filter applied. Pending: full Bronze backfill (Mar 13–today) for both brz_toast_payments and brz_toast_orders_bulk, Silver re-run, mart re-run, and variance validation. |
| T19 | PeopleVine | Confirm PeopleVine endpoint/export list | High | Kris | None | Not Started | TBD | Final list of PeopleVine sources documented | Needed before PeopleVine bronze ingestion. |
| T20 | PeopleVine | Build PeopleVine bronze ingestion | High | Kris | T19 | Not Started | TBD | Member/account/invoice/transaction raw loads working |  |
| T21 | PeopleVine | Build PeopleVine silver normalization | High | Kris | T20 | Not Started | TBD | Core PV silver tables created |  |
| T22 | PeopleVine | Build first member bridge prototype | High | Kris | T16, T21 | Not Started | TBD | First payment-to-member attribution logic works |  |
| T23 | QA / Recon | Validate orders and payments against Toast reports | High | Kris | T18 | In Progress | TBD | Variances understood and documented | April 22 validation in progress. Sales-side confirmed at $71,279.50. Payment-side pending full backfill completion. |
| T24 | BI | Build first semantic model | Medium | Kris | T18, T22 | Not Started | TBD | Model supports first finance and member views |  |
| T25 | BI | Build first Power BI finance dashboard | Medium | Kris | T24 | Not Started | TBD | Initial finance dashboard published |  |