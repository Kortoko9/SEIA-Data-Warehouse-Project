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
| T26 | Tripleseat | Confirm Tripleseat endpoint/export list | Medium | Kris | None | Not Started | TBD | Final list of Tripleseat sources documented | Private dining and event booking source. Needed before Tripleseat bronze ingestion. |
| T27 | Tripleseat | Build Tripleseat bronze ingestion | Medium | Kris | T26 | Not Started | TBD | Raw event/booking payloads landing in Bronze | Follow same PQM append pattern as Toast Bronze. |
| T28 | Tripleseat | Build Tripleseat silver normalization | Medium | Kris | T27 | Not Started | TBD | stg_tripleseat_event and stg_tripleseat_booking created |  |
| T29 | Tripleseat | Build events daily mart | Medium | Kris | T28 | Not Started | TBD | mart_events_daily available for reporting |  |
| T30 | SevenRooms | Confirm SevenRooms endpoint/export list | Medium | Kris | None | Not Started | TBD | Final list of SevenRooms sources documented | Reservation and guest profile source. Needed before SevenRooms bronze ingestion. |
| T31 | SevenRooms | Build SevenRooms bronze ingestion | Medium | Kris | T30 | Not Started | TBD | Raw reservation/guest payloads landing in Bronze | Follow same PQM append pattern as Toast Bronze. |
| T32 | SevenRooms | Build SevenRooms silver normalization | Medium | Kris | T31 | Not Started | TBD | stg_sevenrooms_reservation and stg_sevenrooms_guest created |  |
| T33 | SevenRooms | Build reservations daily mart | Medium | Kris | T32 | Not Started | TBD | mart_reservations_daily available for reporting |  |