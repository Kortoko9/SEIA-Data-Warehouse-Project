# 🧱 Architecture

**Architecture Overview**
*Toast Standard API and PeopleVine are the primary source systems. Microsoft Fabric Lakehouse is the landing and transformation environment. Power BI will consume curated Gold reporting tables*.

**Data Flow**
*Toast API → Bronze raw tables → Silver normalized tables → Gold reporting marts → Power BI dashboards*

**Primary Source Systems**

| Component | Role | Current Status | Notes |
| --- | --- | --- | --- |
| Toast Standard API | Primary transaction source | Active | Orders, payments, menus, config data source |
| PeopleVine | Member/account attribution source | Pending Integration | Required for member spend + house accounts |
| Microsoft Fabric Lakehouse | Raw + transformed data platform | Active | Bronze and Silver layers in progress |
| Gold Reporting Layer | Curated reporting marts | Not Started | Finance-grade reporting layer |
| Power BI | Dashboards / semantic model | Pending | Final reporting interface |
| Control Tables | Audit / watermarks / logging | Active | Core ctl_* tables created |

**Primary Outputs**

| Output | Purpose |
| --- | --- |
| Daily Sales | Revenue tracking |
| Payments / Tenders | Cash & card reconciliation |
| Member Spend | Club/member analytics |
| Discounts / Comps | Operational controls |
| Finance Reconciliation | Tie-out to Toast |
| Executive Dashboard | High-level KPIs |

[Scope / Phase 1 Definition](%F0%9F%A7%B1%20Architecture/Scope%20Phase%201%20Definition%2034cb9a82a84c80df9df3eb86e7c9036a.md)

[Fabric Object Registry](%F0%9F%A7%B1%20Architecture/Fabric%20Object%20Registry%2034cb9a82a84c8013b9fde5b6f3b8d1ca.md)

[Incremental Refresh Rules](%F0%9F%A7%B1%20Architecture/Incremental%20Refresh%20Rules%2034cb9a82a84c8039ac6ef8bc43986464.md)

[Control Tables](%F0%9F%A7%B1%20Architecture/Control%20Tables%2034fb9a82a84c81d19c1ec1602a05420f.md)

[Reconciliation Rules](%F0%9F%A7%B1%20Architecture/Reconciliation%20Rules%2034fb9a82a84c81b6bfd5f6ce1b412c55.md)

[PeopleVine Bridge Spec](%F0%9F%A7%B1%20Architecture/PeopleVine%20Bridge%20Spec%2034fb9a82a84c810eba7af6705bbdbf0d.md)

[KPI Catalog / Dashboard Requirements](%F0%9F%A7%B1%20Architecture/KPI%20Catalog%20Dashboard%20Requirements%2034fb9a82a84c81a7863df176407b0b5e.md)

[Validation / QA Checklist](%F0%9F%A7%B1%20Architecture/Validation%20QA%20Checklist%2034fb9a82a84c81ec8f7adc21ed914a59.md)

[Phase 1 Definition of Done](%F0%9F%A7%B1%20Architecture/Phase%201%20Definition%20of%20Done%2034fb9a82a84c8129af77d6e03715c47e.md)

[Known Issues / Lessons Learned](%F0%9F%A7%B1%20Architecture/Known%20Issues%20Lessons%20Learned%2034fb9a82a84c81cf8454ca265844cfe9.md)