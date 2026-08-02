# Data Modeling Playbook

Reference repo comparing modeling approaches **3NF**, **Kimball star schema**, and **Data Vault 2.0** built from the **same source dataset**, plus decision guidance on when to use each.

**Stack:** SQL · dbt · ERD diagrams (runs locally on DuckDB no cloud account needed)

Most modeling resources explain one approach in isolation. This repo builds **all three from identical source data** so the differences are concrete.

```
                          ┌───────────────┐
                          │  Same raw data│
                          │  (seeds/)     │
                          └───────┬───────┘
                                  │
                          ┌───────┴───────┐
                          │  Staging layer│  <- shared, identical for all three
                          └───────┬───────┘
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
      ┌───────────────┐  ┌───────────────┐  ┌───────────────────┐
      │  models/nf3/  │  │models/kimball/│  │ models/data_vault/│
      │ 3NF normalized│   Star schema    │  │ Hubs/Links/Sats   │
      └───────────────┘  └───────────────┘  └───────────────────┘
```

---

## Comparison

| | **3NF** | **Kimball Star Schema** | **Data Vault 2.0** |
|---|---|---|---|
| **Optimized for** | Transactional integrity, storage efficiency | Query performance, BI/reporting simplicity | Auditability, historization, source agility |
| **Structure** | Fully normalized, minimal redundancy | Denormalized dimensions + fact tables | Hubs (keys), Links (relationships), Satellites (attributes + history) |
| **Query complexity** | High many joins for reporting | Low 1/2 joins, intuitive for analysts | High more joins than Kimball, but consistent pattern |
| **Handles source schema changes** | Poorly normalization is brittle to change | Moderately dimensions need rebuilding | Well new sources add new satellites, no rework of existing structures |
| **History tracking** | Not built in requires manual SCD logic | SCD Type 1/2 on dimensions (manual pattern) | Built into the model every satellite row is a historized fact |
| **Best team fit** | OLTP systems, small reporting needs | Analytics engineering teams building BI facing marts | Large orgs with many source systems, compliance/audit needs |
| **Learning curve** | Familiar to anyone who's learned relational DBs | Low widely taught, intuitive for consumers | Steep hub/link/satellite pattern takes time to internalize |

---

## Project Structure

```
data-modeling-playbook-dbt/
├── dbt_project/
│   ├── seeds/                  # Shared raw source data (CSV) identical input for all 3 models
│   ├── models/
│   │   ├── staging/            # Shared cleaning layer (1:1 with seeds)
│   │   ├── nf3/                # 3NF: customers, categories, products, orders, order_items
│   │   ├── kimball/            # Star schema: dim_customers, dim_products, dim_date, fct_order_items
│   │   └── data_vault/            # Hubs, Links, Satellites
│   ├── dbt_project.yml
│   └── profiles_example.yml    # DuckDB profile no cloud warehouse required
├── erd_diagrams/               # One ERD per approach, in Mermaid
│   ├── erd_3nf.md
│   ├── erd_kimball.md
│   └── erd_data_vault.md
└── docs/
    ├── decision_guide.md       # Deep-dive: when to use which approach, and why
    └── setup_guide.md
```
---

## ERD Diagrams

Each approach has its own entity relationship diagram, written in Mermaid.

# 3NF Entity Relationship Diagram

```mermaid
erDiagram
    CUSTOMERS ||--o{ ORDERS : places
    ORDERS ||--o{ ORDER_ITEMS : contains
    PRODUCTS ||--o{ ORDER_ITEMS : "appears in"
    CATEGORIES ||--o{ PRODUCTS : classifies

    CUSTOMERS {
        string customer_id PK
        string first_name
        string last_name
        string email
        string country
        date signup_date
    }

    CATEGORIES {
        string category_id PK
        string category_name
    }

    PRODUCTS {
        string product_id PK
        string product_name
        string category_id FK
        decimal unit_price
        boolean is_active
    }

    ORDERS {
        string order_id PK
        string customer_id FK
        date order_date
        string order_status
    }

    ORDER_ITEMS {
        string order_item_id PK
        string order_id FK
        string product_id FK
        int quantity
        decimal unit_price
    }
```

**Key characteristic:** every entity is normalized `category_name` lives only in `CATEGORIES`, order level attributes live only in `ORDERS`. Reporting on "revenue by category" requires joining all five tables.

# Kimball Star Schema Entity Relationship Diagram

```mermaid
erDiagram
    DIM_CUSTOMERS ||--o{ FCT_ORDER_ITEMS : "referenced by"
    DIM_PRODUCTS ||--o{ FCT_ORDER_ITEMS : "referenced by"
    DIM_DATE ||--o{ FCT_ORDER_ITEMS : "referenced by"

    DIM_CUSTOMERS {
        string customer_key PK
        string customer_id
        string customer_name
        string email
        string country
    }

    DIM_PRODUCTS {
        string product_key PK
        string product_id
        string product_name
        string category
        boolean is_active
    }

    DIM_DATE {
        date date_key PK
        int year
        int month
        int day
        int quarter
    }

    FCT_ORDER_ITEMS {
        string order_item_key PK
        string order_item_id
        string customer_key FK
        string product_key FK
        date date_key FK
        string order_status
        int quantity
        decimal unit_price
        decimal line_amount
    }
```

**Key characteristic:** `category` is denormalized directly onto `DIM_PRODUCTS` no join to a separate categories table needed. "Revenue by category" is one join (fact → dim_products), not four.

# Data Vault 2.0 Entity Relationship Diagram

```mermaid
erDiagram
    HUB_CUSTOMER ||--o{ SAT_CUSTOMER : describes
    HUB_PRODUCT ||--o{ SAT_PRODUCT : describes
    HUB_ORDER ||--o{ SAT_ORDER : describes

    HUB_ORDER ||--o{ LINK_ORDER_CUSTOMER : participates
    HUB_CUSTOMER ||--o{ LINK_ORDER_CUSTOMER : participates

    HUB_ORDER ||--o{ LINK_ORDER_PRODUCT : participates
    HUB_PRODUCT ||--o{ LINK_ORDER_PRODUCT : participates

    HUB_CUSTOMER {
        string customer_hk PK
        string customer_bk
        timestamp load_date
        string record_source
    }

    HUB_PRODUCT {
        string product_hk PK
        string product_bk
        timestamp load_date
        string record_source
    }

    HUB_ORDER {
        string order_hk PK
        string order_bk
        timestamp load_date
        string record_source
    }

    SAT_CUSTOMER {
        string customer_hk FK
        string first_name
        string last_name
        string email
        string country
        string hash_diff
        timestamp load_date
    }

    SAT_PRODUCT {
        string product_hk FK
        string product_name
        string category
        decimal unit_price
        string hash_diff
        timestamp load_date
    }

    SAT_ORDER {
        string order_hk FK
        date order_date
        string order_status
        string hash_diff
        timestamp load_date
    }

    LINK_ORDER_CUSTOMER {
        string order_customer_lk PK
        string order_hk FK
        string customer_hk FK
        timestamp load_date
    }

    LINK_ORDER_PRODUCT {
        string order_product_lk PK
        string order_hk FK
        string product_hk FK
        string order_item_id
        timestamp load_date
    }
```
**Key characteristic:** business keys (Hubs) are structurally separated from relationships (Links) and descriptive attributes (Satellites). Every satellite row carries a `load_date` and `hash_diff` new attribute values are always inserted as new rows, never overwritten, so full history is preserved by construction rather than by a manual SCD Type 2 pattern.

---
