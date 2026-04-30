# 📈 Power BI Roadmap

**Purpose**

*This roadmap outlines the phased rollout of Power BI reporting for SEIA Club. The goal is to move from manual spreadsheet reporting to automated, trusted, executive-grade dashboards powered by the Fabric warehouse.*

**Roadmap Overview**

| Phase | Status | Objective | Target Outcome |
| --- | --- | --- | --- |
| Phase 1 | Upcoming | Build first finance-grade reporting set | Reconciliation + core sales visibility |
| Phase 2 | Planned | Add member analytics and operational dashboards | Member spend + management reporting |
| Phase 3 | Planned | Executive dashboard suite | Leadership KPI command center |
| Phase 4 | Future | Advanced analytics / forecasting | Predictive insights + alerts |

**Phase 1 – Foundation Dashboards**

| Dashboard | Purpose | Priority | Dependency |
| --- | --- | --- | --- |
| Finance Reconciliation | Tie sales, payments, and variances | High | First Gold mart |
| Daily Sales Dashboard | Revenue by day / meal period / outlet | High | fact_sales_check |
| Tender Dashboard | Card, cash, house account mix | High | fact_payments |
| Discounts & Comps | Controls and leakage monitoring | High | fact_discounts |

Phase 2 – Club Operations Dashboards

| Dashboard | Purpose | Priority | Dependency |
| --- | --- | --- | --- |
| Member Spend Dashboard | Spend by member/account | High | PeopleVine bridge |
| Menu Mix Dashboard | Top sellers / category trends | Medium | fact_sales_selection |
| Service Charges Dashboard | Track gratuity/service charge flows | Medium | fact_service_charges |
| Reservations / Events (Future) | Club activity analytics | Low | Tripleseat integration |

**Phase 3 – Executive Suite**

| Dashboard | Purpose | Priority |
| --- | --- | --- |
| Executive KPI Dashboard | Revenue, covers, trends, membership metrics | High |
| Monthly Finance Pack | Automated board-ready summaries | High |
| Operational Scorecard | Department KPIs | Medium |

**Phase 4 – Advanced Analytics**

| Dashboard / Tool | Purpose |
| --- | --- |
| Forecasting Models | Revenue and spend forecasting |
| Variance Alerts | Notify unusual drops/spikes |
| Cohort Analysis | Member behavior trends |
| AI Insights Layer | Natural language summaries |

**Visual Standards**

| Standard | Requirement |
| --- | --- |
| Executive Friendly | Clean, simple, fast to read |
| Trusted Numbers | Reconciled to source systems |
| Drillthrough Enabled | Ability to inspect detail |
| Mobile Friendly | Leadership phone access where possible |
| Role Based | Finance vs Ops vs Executive views |

**Immediate Next Actions**

| Priority | Task |
| --- | --- |
| High | Build first semantic model in Power BI. |
| High | Launch Finance Reconciliation dashboard first. |
| High | Publish Daily Sales + Tender dashboards. |
| Medium | Build Member Spend dashboard after PV bridge. |
| Medium | Design executive KPI dashboard. |