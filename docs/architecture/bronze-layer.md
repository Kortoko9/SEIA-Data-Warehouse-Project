# 📦 Bronze Layer

**Purpose**

*The Bronze Layer is the raw landing zone for source-system data. It stores API payloads and source extracts with minimal transformation so data can be replayed, audited, and normalized downstream into the Silver Layer.*

**Bronze Layer Status Table**

| Source | Table Group | Status | Notes |
| --- | --- | --- | --- |
| Toast | Restaurants | Complete | Metadata landed successfully |
| Toast | Menus Metadata | Complete | Refreshable metadata source |
| Toast | Menus | Complete | Full menu payload landed |
| Toast | Core Config | Complete | Revenue centers, dining options, taxes, discounts, etc. |
| Toast | Orders Bulk | Complete | Raw order payloads ingested |
| Toast | Payments | Complete | Payment-detail payloads ingested |
| Toast | Labor | Not Started | Future phase |
| Toast | Cash Management | Not Started | Future phase |
| PeopleVine | Raw Feeds | Not Started | Pending endpoint/export confirmation |

**Primary Bronze Tables**

| Table Name | Purpose |
| --- | --- |
| brz_toast_restaurants | Restaurant metadata |
| brz_toast_menus_metadata | Menu refresh metadata |
| brz_toast_menus | Full menu payloads |
| brz_toast_orders_bulk | Raw order payloads |
| brz_toast_payments | Raw payment payloads |
| brz_toast_config_revenue_centers | Revenue center config |
| brz_toast_config_sales_categories | Sales categories |
| brz_toast_config_dining_options | Dining options |
| brz_toast_config_discounts | Discount setup |
| brz_toast_config_service_charges | Service charge setup |
| brz_toast_config_tax_rates | Tax configuration |

**Bronze Standards**

| Standard | Requirement |
| --- | --- |
| Raw Preservation | Store source payloads |
| Auditability | Track ingestion timestamps |
| Replay Capability | Re-run historical windows if needed |
| Minimal Transformation | Raw-first philosophy |
| Source Metadata | Include restaurant_guid / entity IDs |

**Immediate Bronze Next Steps**

| Priority | Task |
| --- | --- |
| High | Add PeopleVine bronze ingestion |
| Medium | Add labor endpoints |
| Medium | Add cash management endpoints |
| Medium | Add automated pipeline scheduling |
| Low | Add webhook ingestion (future phase) |

[**Toast API Endpoint & Scope Map**
](%F0%9F%93%A6%20Bronze%20Layer/Toast%20API%20Endpoint%20&%20Scope%20Map%2034cb9a82a84c80d3af0fecd7f8923af9.md)