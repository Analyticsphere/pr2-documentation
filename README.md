# pr2-documentation
Documentation and Issue-tracking for the PR2 Data Pipeline.

- **Issue Tracking:** All [Issues](https://github.com/Analyticsphere/pr2-documentation/issues) will be created in this repo and tracked in the [pr2 GitHub Project](https://github.com/orgs/Analyticsphere/projects/15).
- **Transformations:** Transformation code will be developed/maintained here: [Analyticsphere/transformation](https://github.com/Analyticsphere/pr2-transformation)
- **Orchestration:** Airflow DAGs will be developed/maintained here: [Analyticsphere/pr2-orchestration](https://github.com/Analyticsphere/pr2-orchestration)

Core logic of the transformations will be implemented in Python, but the Python code will render SQL which will be executed in BigQuery. 
## High-level dataflow diagram

<img width="800" alt="pr2_dataflow_diagram" src="https://github.com/user-attachments/assets/3ddaabca-1b1c-467b-8d3d-9c6c181d0d91" />

## Dataflow diagram for cleaning transformations

```mermaid
flowchart LR
 subgraph FlatConnect["FlatConnect"]
        fc_mod1_v1["module1_v1"]
        fc_mod1_v2["module1_v2"]
        fc_mod2_v1["module2_v1"]
        fc_mod2_v2["module2_v2"]
        fc_mod3["module_3_v1"]
        fc_mod4["module_4_v1"]
        fc_bs["bioSurvey_v1"]
        fc_c19["covid19Survey_v1"]
        fc_prom["promis_v1"]
  end
 subgraph CleanConnect["CleanConnect"]
        cc_mod1["module1"]
        cc_mod2["module2"]
        cc_mod3["module3"]
        cc_mod4["module4"]
        cc_bs["bioSurvey"]
        cc_c19["covid19Survey"]
        cc_prom["promis"]
  end
    fc_mod1_v1 -- coalesce loop vars --> fc_coal_mod1_v1["stg_coalesced_module1_v1"]
    fc_mod1_v2 -- coalesce loop vars --> fc_coal_mod1_v2["stg_coalesced_module1_v2"]
    fc_mod2_v1 -- coalesce loop vars --> fc_coal_mod2_v1a["stg_coalesced_module2_v1"]
    fc_mod2_v2 -- coalesce loop vars --> fc_coal_mod2_v1b["stg_coalesced_module2_v1"]
    fc_mod3 -- clean --> cc_mod3
    fc_mod4 -- clean --> cc_mod4
    merge_mod1["stg_merged_module_1"] -- clean --> cc_mod1
    merge_mod2["stg_merged_module_2"] -- clean --> cc_mod2
    fc_coal_mod1_v1 -- merge --> merge_mod1
    fc_coal_mod1_v2 -- merge --> merge_mod1
    fc_coal_mod2_v1a -- merge --> merge_mod2
    fc_coal_mod2_v1b -- merge --> merge_mod2
    fc_bs -- merge covid variables --> stg_cov19["stg_covid19Survey"]
    fc_c19 -- merge covid variables --> stg_cov19
    stg_cov19 -- clean --> cc_c19
    fc_bs -- "clean non-covid variables" --> cc_bs
    fc_prom --> cc_prom
    style FlatConnect fill:#FFF9C4
    style CleanConnect fill:#C8E6C9
```

## [DRAFT] Diagram of proposed data structure changes to *Module 4: Where you live and work*

```mermaid
erDiagram
    PARTICIPANT ||--o{ ADDRESS : ""
    ADDRESS ||--|| JOB : ""
    ADDRESS ||--|| SCHOOL : ""
    ADDRESS }|--|| COMMUTE : ""
    PARTICIPANT ||--o{ JOB : ""
    PARTICIPANT ||--o{ SCHOOL : ""
      PARTICIPANT ||--o{ COMMUTE : ""
    
    PARTICIPANT {
        int connect_id PK
        str name
    }
    
    ADDRESS {
        int connect_id PK, FK
        int address_id PK
        str address_type
        int street_number
        str street_name
        str city
        str state
        int zip
        float latitude
        float longitude
        date start_date
        date end_date
    }
    
    JOB {
        int job_id PK
        int connect_id FK
        int address_id FK
        int occupation
        str employer
    }

    SCHOOL {
        int connect_id
        int school_id
        str school_name
    }

    COMMUTE {
        int connect_id FK, PK
        int departure_address_id FK, PK
        int destination_address_id FK, PK
        str commute_type
        str commute_days_per_week
        str commute_time
    }
```
