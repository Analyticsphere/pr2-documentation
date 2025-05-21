# pr2-documentation
Documentation and Issue-tracking for the PR2 Data Pipeline.

- **Issue Tracking:** All [Issues](https://github.com/Analyticsphere/pr2-documentation/issues) will be created in this repo and tracked in the [pr2 GitHub Project](https://github.com/orgs/Analyticsphere/projects/15).
- **Transformations:** Transformation code will be developed/maintained here: [Analyticsphere/pr2-transformation](https://github.com/Analyticsphere/pr2-transformation)
- **Orchestration:** Airflow DAGs will be developed/maintained here: [Analyticsphere/pr2-orchestration](https://github.com/Analyticsphere/pr2-orchestration)

Core logic of the transformations will be implemented in Python, but the Python code will render SQL which will be executed in BigQuery. 

## High-level dataflow diagram

<img width="800" alt="pr2_dataflow_diagram" src="https://github.com/user-attachments/assets/3ddaabca-1b1c-467b-8d3d-9c6c181d0d91" />

# PR2 Transformation Architecture

## System Overview

The PR2 transformation architecture is a serverless ETL pipeline which is built on Google Cloud Platform that transforms Connect data from its raw form to a clean, standardized format for research purposes.

## Architecture Diagram
> *Note:* This currently just includes the cleaning transformations, not-deidentification or aggregations. Each additional transformation should be a new endpoint in the API and should be called from a new "task" in the Airflow DAG.

```mermaid
flowchart LR
    composer["Cloud Composer<br>(Apache Airflow)"] --> |triggers| cloudrun["Cloud Run<br>(PR2 Transformation Service)"]
    cloudrun --> |executes SQL| bq["BigQuery"]
    cloudrun --> |archives SQL| gcs["Cloud Storage"]
    
    subgraph "Data Flow"
        source["FlatConnect<br>(Source Data)"] --> |transformation| staging["Staging Tables"]
        staging --> |transformation| clean["CleanConnect<br>(Clean Data)"]
    end
    
    bq --- source
    bq --- staging
    bq --- clean
    
    classDef primary fill:#4285F4,stroke:#4285F4,color:white
    classDef secondary fill:#34A853,stroke:#34A853,color:white
    classDef datasets fill:#F1F3F4,stroke:#DADCE0,color:black
    
    class composer,cloudrun primary
    class bq,gcs secondary
    class source,staging,clean datasets
```
## First Principles

The following first principles are used to guide development decisions:

- **Unidirectional data flow**: Raw → clean | PII → de-identified | granular → aggregated
- **Modularity**: Small, discrete functions optimized for readability over efficiency
- **Separation of concerns**: Group transformations by type, not by table (e.g., column-level, row/value-level, table-level)
- **Parallelization**: Process multiple tables simultaneously
- **Configuration**: Parameterize constants and avoid hardcoding changeable elements
- **Extensibility**: Design to accomodate arbitrary future requirements, avoid one-off solutions
- **SQL-first**: Generate SQL in python and execute in BigQuery via client library
- **Auditiability and data provanence**: Archive SQL in GCS prior to execution for debugging and analyzing data provanance
- **Serverless**: REST API endpoints for processing. Pass parameters via JSON from orchestrator
- **Centralized orchestration**: Airflow for dependency management, parallelization, and scheduling
  
## Core Components

### Cloud Composer (Apache Airflow)
- Orchestrates the entire ETL workflow
- Schedules and triggers transformation steps in sequence
- Manages dependencies between transformation tasks

### Cloud Run (PR2 Transformation Service)
- Hosts the transformation API endpoints
- Processes transformation requests from Airflow
- Generates SQL for BigQuery execution

### BigQuery
- Functions as both source and destination for data
- Executes the transformation SQL queries
- Houses three key datasets:
  - **FlatConnect**: Original source data
  - **Staging Tables**: Intermediate processed data
  - **CleanConnect**: Final cleaned data

### Cloud Storage
- Archives generated SQL queries for audit purposes
- Provides transparency for transformation logic

## Data Flow

The transformation process follows a simple flow:

1. **Orchestration**: Airflow DAG triggers API calls to the Cloud Run service
2. **Transformation**: The service generates and executes SQL in BigQuery:
   - Source data (FlatConnect) → Column cleaning → Row cleaning → Table merging
   - Intermediate results are stored in staging tables
   - Final results are stored in CleanConnect
3. **Auditing**: Generated SQL is archived in Cloud Storage

## API Endpoints

The PR2 Transformation Service exposes the following RESTful endpoints:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/heartbeat` | GET | Verifies service health |
| `/clean_columns` | POST | Standardizes column names |
| `/clean_rows` | POST | Transforms row values |
| `/merge_table_versions` | POST | Combines multiple table versions |

## Transformation Steps

### 1. Clean Columns
- Standardizes naming conventions for loop variables
- Converts column names to lowercase (except Connect_ID)
- Standardizes version tags by placing them at the end of column names
- Splits or merges columns as needed

### 2. Clean Rows
- Converts binary "0"/"1" values to standardized concept IDs
- Handles array unwrapping for singleton values

### 3. Merge Table Versions
- Performs full outer joins on tables with multiple versions
- Uses COALESCE for combining common columns
- Preserves columns unique to either source table

This architecture provides a clean separation between orchestration (Airflow), processing logic (Cloud Run), and data storage (BigQuery), enabling a maintainable and scalable approach to ETL processing.
## Dataflow diagram for cleaning transformations

```mermaid
flowchart LR
    %% Define Sources
    subgraph SourceTables["Source Tables"]
        mod1_v1["MODULE1_V1"]
        mod1_v2["MODULE1_V2"]
        mod2_v1["MODULE2_V1"]
        mod2_v2["MODULE2_V2"]
        mod3["MODULE3"]
        mod4["MODULE4"]
        bio["BIOSURVEY"]
        clinical_bio["CLINICALBIOSURVEY"]
        covid["COVID19SURVEY"]
        mouthwash["MOUTHWASH"]
        biospecimen["BIOSPECIMEN"]
        participants["PARTICIPANTS"]
        exp2024["EXPERIENCE2024"]
        menstrual["MENSTRUALSURVEY"]
    end
    
    %% Define Staging
    subgraph StagingTables["Staging Tables"]
        %% Cleaned Columns
        mod1_v1_cc["MODULE1_V1_CLEANED_COLUMNS"]
        mod1_v2_cc["MODULE1_V2_CLEANED_COLUMNS"]
        mod2_v1_cc["MODULE2_V1_CLEANED_COLUMNS"]
        mod2_v2_cc["MODULE2_V2_CLEANED_COLUMNS"]
        mod3_cc["MODULE3_CLEANED_COLUMNS"]
        mod4_cc["MODULE4_CLEANED_COLUMNS"]
        bio_cc["BIOSURVEY_CLEANED_COLUMNS"]
        clinical_bio_cc["CLINICALBIOSURVEY_CLEANED_COLUMNS"]
        covid_cc["COVID19SURVEY_CLEANED_COLUMNS"]
        mouthwash_cc["MOUTHWASH_CLEANED_COLUMNS"]
        biospecimen_cc["BIOSPECIMEN_CLEANED_COLUMNS"]
        participants_cc["PARTICIPANTS_CLEANED_COLUMNS"]
        exp2024_cc["EXPERIENCE2024_CLEANED_COLUMNS"]
        
        %% Cleaned Rows
        mod1_v1_cr["MODULE1_V1_CLEANED_ROWS"]
        mod1_v2_cr["MODULE1_V2_CLEANED_ROWS"]
        mod2_v1_cr["MODULE2_V1_CLEANED_ROWS"]
        mod2_v2_cr["MODULE2_V2_CLEANED_ROWS"]
    end
    
    %% Define Clean Tables
    subgraph CleanTables["Clean Tables"]
        mod1_clean["MODULE1"]
        mod2_clean["MODULE2"]
        mod3_clean["MODULE3"]
        mod4_clean["MODULE4"]
        bio_clean["BIOSURVEY"]
        clinical_bio_clean["CLINICALBIOSURVEY"]
        covid_clean["COVID19SURVEY"]
        mouthwash_clean["MOUTHWASH"]
        biospecimen_clean["BIOSPECIMEN"]
        participants_clean["PARTICIPANTS"]
        exp2024_clean["EXPERIENCE2024"]
        menstrual_clean["MENSTRUALSURVEY"]
    end
    
    %% Phase 1: Clean Columns
    mod1_v1 -- "clean_columns" --> mod1_v1_cc
    mod1_v2 -- "clean_columns" --> mod1_v2_cc
    mod2_v1 -- "clean_columns" --> mod2_v1_cc
    mod2_v2 -- "clean_columns" --> mod2_v2_cc
    mod3 -- "clean_columns" --> mod3_cc
    mod4 -- "clean_columns" --> mod4_cc
    bio -- "clean_columns" --> bio_cc
    clinical_bio -- "clean_columns" --> clinical_bio_cc
    covid -- "clean_columns" --> covid_cc
    mouthwash -- "clean_columns" --> mouthwash_cc
    biospecimen -- "clean_columns" --> biospecimen_cc
    participants -- "clean_columns" --> participants_cc
    exp2024 -- "clean_columns" --> exp2024_cc
    menstrual -- "clean_columns" --> menstrual_clean
    
    %% Phase 2: Clean Rows for Tables that need version merging
    mod1_v1_cc -- "clean_rows" --> mod1_v1_cr
    mod1_v2_cc -- "clean_rows" --> mod1_v2_cr
    mod2_v1_cc -- "clean_rows" --> mod2_v1_cr
    mod2_v2_cc -- "clean_rows" --> mod2_v2_cr
    
    %% Phase 2: Clean Rows for Tables that go directly to Clean Tables
    mod3_cc -- "clean_rows" --> mod3_clean
    mod4_cc -- "clean_rows" --> mod4_clean
    bio_cc -- "clean_rows" --> bio_clean
    clinical_bio_cc -- "clean_rows" --> clinical_bio_clean
    covid_cc -- "clean_rows" --> covid_clean
    mouthwash_cc -- "clean_rows" --> mouthwash_clean
    biospecimen_cc -- "clean_rows" --> biospecimen_clean
    participants_cc -- "clean_rows" --> participants_clean
    exp2024_cc -- "clean_rows" --> exp2024_clean
    
    %% Phase 3: Merge Table Versions
    mod1_v1_cr & mod1_v2_cr -- "merge_table_versions" --> mod1_clean
    mod2_v1_cr & mod2_v2_cr -- "merge_table_versions" --> mod2_clean
    
    %% Styling
    classDef sourceStyle fill:#FFF9C4,stroke:#E6BA20,stroke-width:1px
    classDef stagingStyle fill:#FFDFBA,stroke:#FF8C00,stroke-width:1px
    classDef cleanStyle fill:#C8E6C9,stroke:#2E7D32,stroke-width:1px
    
    class mod1_v1,mod1_v2,mod2_v1,mod2_v2,mod3,mod4,bio,clinical_bio,covid,mouthwash,biospecimen,participants,exp2024,menstrual sourceStyle
    class mod1_v1_cc,mod1_v2_cc,mod2_v1_cc,mod2_v2_cc,mod3_cc,mod4_cc,bio_cc,clinical_bio_cc,covid_cc,mouthwash_cc,biospecimen_cc,participants_cc,exp2024_cc,mod1_v1_cr,mod1_v2_cr,mod2_v1_cr,mod2_v2_cr stagingStyle
    class mod1_clean,mod2_clean,mod3_clean,mod4_clean,bio_clean,clinical_bio_clean,covid_clean,mouthwash_clean,biospecimen_clean,participants_clean,exp2024_clean,menstrual_clean cleanStyle
```

## Transformation Steps
- clean_columns:
    - fix naming conventions for loop variables
    - convert collumn names to lower case
    - standardize use of version tag, e.g., '_v2', by putting it at the end of the column names
    - split columns that require splitting
    - merge columns that need merging 
- clean_rows:
    - ensure that binary responses have concept ids for yes/no rather than 0/1
    - ensure that there are now values that should be singletons that are wrapped in brackets like an array
- merge_table_versions
    - join tables that have multiple versions
    - take care to use coalesce appropriately to combine mutual columns
    - take care to include columns unique to either source table in the target table

# Thoughts on the future state of PR2:

## Could we do better a response-centric relational data model? Yes!
> This diagram does not represent a "fully-cooked" data model for the Connect data. It is a conceptual sketch that should be developed further through discussion with stakeholders. The nomenclature can be adjusted to use the terminology from our existing data dictionary as needed ("primary source", "secondary source", etc.). I suggest this type of normalized relational data model as a middle layer between our operational (raw) data and the data that are "curated" for the end-user. 

![survey_data_model](https://github.com/user-attachments/assets/17c0ce7d-e842-42ef-bcb8-fc338ced990c)

## [DRAFT] Diagram of proposed data structure for *Module 4: Where you live and work* data mart.
> This is also just a conceptual sketch, not a fully developed data model. 

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
