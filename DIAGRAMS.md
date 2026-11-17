# CTA Ridership Diagrams

Diagrams of PROJECT_PLAN.md in Mermaid, drawn to the IT 432 rubrics. They render on GitHub and in VS Code's Markdown preview (a Mermaid extension may be needed). Open `cta-diagrams.html` in a browser to see them all rendered.

| Phase | Diagrams |
|---|---|
| 1. Plain Python | 8 Activity, 9 Design class |
| 2. pandas + EDA | 10 DFD context, 11 DFD level 0 |
| 3. SQL | 12 ER, 13 Domain model class |
| 4. S3 | 4 Data lake layout |
| 5. Terraform | 5 Module dependencies |
| 6. Lambda pipeline | 15 Sequence, 14 Design class |
| 7. Model + dashboard | 16 Component, 17 Design class |
| 8. CI/CD | 18 Activity |
| Overall | 3 AWS architecture, 7 Use case, 1 Roadmap, 2 Growth, 6 Open decisions |

# Part 1: Big picture

## 1. Phase roadmap

*Gantt chart (not UML).* Eight phases at about 10 hours a week. Bar lengths use the midpoint of each phase's hour estimate, so dates are approximate. Phases 1–3 alone make a portfolio project.

```mermaid
gantt
    title CTA project at ~10 hrs/week (midpoint estimates)
    dateFormat YYYY-MM-DD
    axisFormat %b
    section Local
    1 Plain Python (10-15h)         :p1, 2026-11-23, 9d
    2 pandas + EDA (15-25h)         :p2, after p1, 14d
    3 SQL / PostgreSQL (10-20h)     :p3, after p2, 11d
    Portfolio-ready milestone       :milestone, m1, after p3, 0d
    section Cloud
    4 S3 + IAM (8-15h)              :p4, after p3, 8d
    5 Terraform (20-35h)            :p5, after p4, 19d
    6 Lambda + Athena (25-40h)      :p6, after p5, 23d
    Cloud pipeline milestone        :milestone, m2, after p6, 0d
    section Ship
    7 Model + Docker + ECS (30-45h) :p7, after p6, 26d
    8 CI/CD with OIDC (10-20h)      :p8, after p7, 11d
    README, diagram, cleanup (8-12h):p9, after p8, 7d
    Resume version                  :milestone, m3, after p9, 0d
```

## 2. How the system grows

*Overview (not UML).* Each phase replaces one piece of the previous one. Functions written early, such as fetch_*, clean_*, and s3_io, are reused later instead of rewritten.

```mermaid
flowchart LR
    subgraph L["Local (Phases 1-3)"]
        direction TB
        C1["CSV + csv module<br/>hand-written stats"] --> C2["pandas<br/>+ Open-Meteo join"]
        C2 --> C3["SQLite then PostgreSQL<br/>4 saved queries"]
    end
    subgraph A["AWS (Phases 4-6)"]
        direction TB
        S["S3 data lake<br/>console first"] --> T["Same bucket<br/>in Terraform"]
        T --> P["Daily Lambda<br/>Glue + Athena"]
    end
    subgraph D["Ship (Phases 7-8)"]
        direction TB
        M["scikit-learn model<br/>beats baseline MAE"] --> UI["Streamlit in Docker<br/>ECS or App Runner"]
        UI --> CI["GitHub Actions<br/>OIDC deploys"]
    end
    C3 -- "same queries run in Athena" --> P
    C2 -- "files move to S3" --> S
    P --> M
```

## 3. Overall · AWS architecture

*Architecture diagram (not UML).* The finished system on one page, and the diagram for your README. Everything inside the AWS box is created by Terraform and deployed from GitHub Actions without stored access keys.

```mermaid
flowchart LR
    CTA["Chicago Data Portal<br/>ridership API"]
    OM["Open-Meteo<br/>weather API"]
    GH["GitHub Actions"]
    U(["You / recruiter"])
    subgraph AWS
        EB["EventBridge<br/>daily schedule"] --> L["Lambda ingest"]
        PS["Parameter Store<br/>config"] -.-> L
        L --> RAW[("S3 raw/<br/>JSON")]
        RAW --> CL["Clean + transform"]
        CL --> CLEAN[("S3 clean/<br/>Parquet")]
        CLEAN --> G["Glue tables"] --> ATH["Athena"]
        ATH --> TR["train.py"] --> MOD[("S3 models/")]
        ATH --> APP["Streamlit<br/>ECS / App Runner"]
        MOD --> APP
        ECR["ECR image"] --> APP
        L -. "logs, failure metric" .-> CW["CloudWatch alarm"]
    end
    CTA --> L
    OM --> L
    GH -- "OIDC role: terraform apply,<br/>push image, deploy" --> ECR
    APP --> U
```

## 4. Phase 4 · Data lake layout

*Folder layout (not UML).* Raw API responses are partitioned by ingest date, clean Parquet by year and month, and models are versioned by training date.

```mermaid
flowchart LR
    B[("project-env-data bucket")] --> R["raw/"]
    B --> C["clean/"]
    B --> M["models/"]
    B --> AR["athena-results/"]
    R --> R1["ridership/ingest_date=YYYY-MM-DD/ridership.json"]
    R --> R2["weather/ingest_date=YYYY-MM-DD/weather.json"]
    C --> C1["ridership/year=YYYY/month=MM/part-0.parquet"]
    C --> C2["weather/year=YYYY/month=MM/part-0.parquet"]
    M --> M1["ridership_model_YYYY-MM-DD.joblib"]
```

## 5. Phase 5 · Terraform module dependencies

*Module diagram (not UML).* Dev and prod call the same modules with different variables. Only storage is built in Phase 5; the other modules arrive with their phases.

```mermaid
flowchart TD
    BS["bootstrap/<br/>state bucket + locking"] -. "remote state" .-> ENV
    ENV["envs/dev and envs/prod"] --> STO["storage<br/>Phase 5"]
    ENV --> ING["ingestion<br/>Phase 6"]
    ENV --> ANA["analytics<br/>Phase 6"]
    ENV --> DSH["dashboard<br/>Phase 7"]
    ENV --> CIM["ci (OIDC role)<br/>Phase 8"]
    STO -- "bucket_name, bucket_arn" --> ING
    STO -- "bucket_name" --> ANA
    STO -- "bucket_arn" --> DSH
    ANA -- "athena_workgroup" --> DSH
    STO --- S1["S3: versioning, encryption,<br/>public access block"]
    ING --- I1["Lambda + IAM role,<br/>EventBridge, CloudWatch alarm"]
    ANA --- A1["Glue database + tables,<br/>Athena workgroup"]
    DSH --- D1["ECR repo,<br/>ECS or App Runner"]
    CIM --- C1["Trust limited to<br/>your repo and branch"]
```

## 6. Open decisions, by when you hit them

*Checklist map (not UML).* Choices the plan leaves to you, placed in the phase where each one comes up.

```mermaid
flowchart LR
    P1["Phase 1"] --> D1["mean of empty list:<br/>return None or raise?"]
    P4["Phase 4"] --> D2["Same day written twice:<br/>overwrite or keep versions?"]
    P5["Phase 5"] --> D3["Hand-made bucket:<br/>terraform import or recreate?"]
    P5 --> D4["Destroy versioned bucket:<br/>force_destroy in dev only?"]
    P6["Phase 6"] --> D5["pandas in Lambda:<br/>layer or container image?"]
    P6 --> D6["Ridership lags a few days:<br/>which target date?"]
    P7["Phase 7"] --> D7["2020 and holidays:<br/>exclude or flag?"]
    P7 --> D8["Dashboard host:<br/>App Runner or ECS Fargate?"]
    P7 --> D9["Add optional<br/>FastAPI endpoint?"]
    P8["Phase 8"] --> D10["Manual approval<br/>for prod deploys?"]
```

# Part 2: UML by phase

## 7. Whole system · Use case diagram

*UML use case diagram.* Actors outside the automation boundary and the use cases inside it. Ingest Daily Data always includes Clean Source Data; Backfill Historical Data extends it when a past date is passed in; Train Ridership Model always includes Evaluate Model Against Baseline. Mermaid has no actor or ellipse shapes, so actors are boxes marked «actor» and use cases are rounded shapes.

```mermaid
flowchart LR
    SCH["«actor»<br/>Scheduler<br/>(time)"]
    AN["«actor»<br/>Analyst"]
    VW["«actor»<br/>Dashboard Viewer"]
    subgraph SYS["CTA Ridership Analysis System"]
        direction TB
        UC1(["Ingest Daily Data"])
        UC2(["Backfill Historical Data"])
        UC3(["Clean Source Data"])
        UC4(["Train Ridership Model"])
        UC5(["Evaluate Model Against Baseline"])
        UC6(["View Ridership Trend"])
        UC7(["View Weather Impact"])
        UC8(["View Model Performance"])
        UC9(["View Tomorrow's Prediction"])
    end
    CDP["«actor»<br/>Chicago Data Portal"]
    OMA["«actor»<br/>Open-Meteo"]
    SCH --- UC1
    AN --- UC2
    AN --- UC4
    VW --- UC6
    VW --- UC7
    VW --- UC8
    VW --- UC9
    UC1 -. "«include»" .-> UC3
    UC2 -. "«extend»" .-> UC1
    UC4 -. "«include»" .-> UC5
    UC1 --- CDP
    UC1 --- OMA
```

## 8. Phase 1 · Activity diagram: clean and summarize the CSV

*UML activity diagram.* Swimlanes for the Analyst and the script. Every decision node has a matching merge node, the three summaries run between a fork and a join, and ridership.csv and the cleaning report are object nodes.

```mermaid
flowchart TB
    subgraph LA["Analyst"]
        direction TB
        I((" ")) --> A1("Run main.py")
        A9("Review report and stats")
        F(((" ")))
    end
    subgraph LS["main.py"]
        direction TB
        O1["ridership.csv"]
        A2("Open ridership file")
        M1{" "}
        D1{" "}
        A3("Read next row")
        D2{" "}
        A4("Count invalid row")
        D3{" "}
        A5("Count duplicate date")
        A6("Keep row")
        M3{" "}
        M2{" "}
        FK[" "]
        S1("Average ridership by weekday")
        S2("Average ridership by month")
        S3("Find top and bottom 5 days")
        JN[" "]
        A7("Print cleaning report and tables")
        O2["cleaning report"]
    end
    A1 --> O1 --> A2 --> M1 --> D1
    D1 -- "[more rows]" --> A3 --> D2
    D2 -- "[invalid]" --> A4 --> M2
    D2 -- "[valid]" --> D3
    D3 -- "[duplicate date]" --> A5 --> M3
    D3 -- "[new date]" --> A6 --> M3
    M3 --> M2
    M2 --> M1
    D1 -- "[no more rows]" --> FK
    FK --> S1 --> JN
    FK --> S2 --> JN
    FK --> S3 --> JN
    JN --> A7 --> O2 --> A9 --> F

    classDef initial fill:#222,stroke:#222,color:#222
    classDef final fill:#222,stroke:#888,color:#222
    classDef bar fill:#222,stroke:#222,color:#222
    classDef node0 fill:transparent,stroke:#888
    classDef obj fill:transparent,stroke:#888,stroke-dasharray:0
    class I initial
    class F final
    class FK,JN bar
```

## 9. Phase 1 · Design class diagram: main.py

*UML design class diagram.* main.py drawn as a «module» class: every function with typed parameters and a return type. The dashed arrow shows the standard-library check your mean and median must match.

```mermaid
classDiagram
    class phase1_main {
        <<module>>
        +load_rows(path: str) list
        +parse_row(raw: dict) dict
        +clean_rows(raw_rows: list) tuple
        +mean(values: list) float
        +median(values: list) float
        +group_by(rows: list, key_func: function) dict
        +stats_by_weekday(rows: list) dict
        +stats_by_month(rows: list) dict
        +top_n_days(rows: list, n: int, highest: bool) list
        +print_table(title: str, mapping: dict) None
        +main() None
    }
    class statistics {
        <<library>>
        +mean(data: list) float
        +median(data: list) float
    }
    phase1_main ..> statistics : results must match
    note for phase1_main "parse_row returns None for an invalid row.\nclean_rows returns (clean rows, report dict)." 
```

## 10. Phase 2 · DFD context diagram

*Data flow diagram (Gane and Sarson).* The whole Phase 2 system as process 0, with its sources and sinks. No data stores appear at the context level.

```mermaid
flowchart LR
    AN["Analyst"]
    CDP["Chicago Data Portal"]
    OMA["Open-Meteo"]
    HC["Holiday Calendar"]
    P0("0<br/>CTA Ridership<br/>Analysis System")
    AN -- "Date Range" --> P0
    P0 -- "Ridership Query" --> CDP
    CDP -- "Ridership Records" --> P0
    P0 -- "Weather Query" --> OMA
    OMA -- "Weather Observations" --> P0
    HC -- "Holiday List" --> P0
    P0 -- "Cleaning Report" --> AN
    P0 -- "EDA Charts and Findings" --> AN
    classDef ext stroke-width:3px
    class AN,CDP,OMA,HC ext
```

## 11. Phase 2 · DFD level 0 diagram

*Data flow diagram (Gane and Sarson).* Every context flow appears here. Processes are numbered verb phrases, data stores are numbered noun phrases (D1–D3), and data only enters or leaves a store through a process. Mermaid has no open-ended rectangle, so stores use the double-sided box.

```mermaid
flowchart LR
    AN["Analyst"]
    CDP["Chicago Data Portal"]
    OMA["Open-Meteo"]
    HC["Holiday Calendar"]
    P1("1<br/>Fetch Ridership")
    P2("2<br/>Fetch Weather")
    P3("3<br/>Clean Ridership")
    P4("4<br/>Clean Weather")
    P5("5<br/>Merge Daily Data")
    P6("6<br/>Produce EDA Charts")
    D1[["D1 | Raw Ridership File"]]
    D2[["D2 | Raw Weather File"]]
    D3[["D3 | Merged Daily Dataset"]]
    AN -- "Date Range" --> P1
    AN -- "Date Range" --> P2
    P1 -- "Ridership Query" --> CDP
    CDP -- "Ridership Records" --> P1
    P1 -- "Raw Ridership" --> D1
    P2 -- "Weather Query" --> OMA
    OMA -- "Weather Observations" --> P2
    P2 -- "Raw Weather" --> D2
    D1 -- "Raw Ridership" --> P3
    D2 -- "Raw Weather" --> P4
    P3 -- "Clean Ridership" --> P5
    P4 -- "Clean Weather" --> P5
    HC -- "Holiday List" --> P5
    P3 -- "Cleaning Report" --> AN
    P5 -- "Merged Daily Record" --> D3
    D3 -- "Merged Daily Record" --> P6
    P6 -- "EDA Charts and Findings" --> AN
    classDef ext stroke-width:3px
    class AN,CDP,OMA,HC ext
```

**Data dictionary**

- **Date Range** = Start_Date + End_Date
- **Ridership Query** = Dataset_ID + Start_Date + End_Date
- **Ridership Records** = 1{Service_Date + Day_Type + Bus_Rides + Rail_Rides + Total_Rides}N
- **Raw Ridership** = Ingest_Date + 1{Service_Date + Day_Type + Bus_Rides + Rail_Rides + Total_Rides}N
- **Weather Query** = Latitude + Longitude + Start_Date + End_Date + Daily_Variables
- **Weather Observations** = 1{Weather_Date + Temp_Max_C + Temp_Min_C + Precipitation_mm + Snowfall_cm}N
- **Raw Weather** = Ingest_Date + 1{Weather_Date + Temp_Max_C + Temp_Min_C + Precipitation_mm + Snowfall_cm}N
- **Clean Ridership** = 1{Service_Date + Day_Type + Bus_Rides + Rail_Rides + Total_Rides}N
- **Clean Weather** = 1{Weather_Date + Temp_Max_C + Temp_Min_C + Precipitation_mm + Snowfall_cm}N
- **Holiday List** = 1{Holiday_Date + Holiday_Name}N
- **Cleaning Report** = Rows_Loaded + Rows_Invalid + Rows_Duplicate + Rows_Kept
- **Merged Daily Record** = Service_Date + Total_Rides + Bus_Rides + Rail_Rides + Temp_Max_C + Temp_Min_C + Precipitation_mm + Snowfall_cm + Weekday + Month + Is_Weekend + Is_Holiday
- **EDA Charts and Findings** = 1{Chart_Title + Chart_Image + Finding_Sentence}N

## 12. Phase 3 · ER diagram

*Entity-relationship diagram.* The three SQL tables, keyed by date so loads can be upserts. Every service day has at most one weather row and at most one holiday.

```mermaid
erDiagram
    DAILY_RIDERSHIP {
        date service_date PK
        string day_type
        int bus_rides
        int rail_rides
        int total_rides
    }
    DAILY_WEATHER {
        date weather_date PK
        float temp_max_c
        float temp_min_c
        float precipitation_mm
        float snowfall_cm
    }
    HOLIDAYS {
        date holiday_date PK
        string holiday_name
    }
    DAILY_RIDERSHIP ||--o| DAILY_WEATHER : "same date"
    DAILY_RIDERSHIP ||--o| HOLIDAYS : "same date"
```

## 13. Phase 3 · Domain model class diagram

*UML domain model class diagram.* The problem-domain classes with {key} identifiers, attributes, named associations, and multiplicities. RidershipCount is abstract with a {disjoint, complete} generalization, IngestRun is a whole made of RawFiles, and Prediction is the association class on Model predicts ServiceDay. Mermaid cannot draw the dashed line to an association, so Prediction is linked to both ends with a note.

```mermaid
classDiagram
    direction LR
    class ServiceDay {
        serviceDate #123;key#125;
        dayType
        isWeekend
    }
    class RidershipCount {
        <<abstract>>
        countID #123;key#125;
        rides
        ingestDate
    }
    class BusRidership
    class RailRidership
    class WeatherObservation {
        observationID #123;key#125;
        tempMaxC
        tempMinC
        precipitationMm
        snowfallCm
    }
    class Location {
        locationID #123;key#125;
        name
        latitude
        longitude
    }
    class Holiday {
        holidayDate #123;key#125;
        holidayName
    }
    class IngestRun {
        runID #123;key#125;
        targetDate
        startTime
        status
        rowsWritten
    }
    class RawFile {
        fileKey #123;key#125;
        source
        format
        uploadTime
    }
    class Model {
        modelID #123;key#125;
        modelType
        trainedDate
        cutoffDate
        mae
    }
    class Prediction {
        predictedRides
        predictionDate
        absoluteError
    }
    RidershipCount <|-- BusRidership : disjoint, complete
    RidershipCount <|-- RailRidership
    ServiceDay "1" -- "1..*" RidershipCount : records ▶
    ServiceDay "1" -- "0..1" WeatherObservation : has ▶
    Location "1" -- "0..*" WeatherObservation : reports ▶
    Holiday "0..1" -- "0..*" ServiceDay : falls on ▶
    IngestRun "1" *-- "1..*" RawFile : contains ▶
    IngestRun "0..*" -- "1" ServiceDay : loads ▶
    Model "0..*" -- "0..*" ServiceDay : predicts ▶
    Model "1" -- "0..*" Prediction
    Prediction "0..*" -- "1" ServiceDay
    note for Prediction "Association class of Model predicts ServiceDay"
    note for RawFile "A raw file exists only within its ingest run" 
```

## 14. Phases 2–6 · Design class diagram: pipeline modules

*UML design class diagram.* The data modules and the Lambda handler with full method signatures. Arrows are navigation visibility: the handler calls ingest, cleaning, and s3_io instead of copying their code.

```mermaid
classDiagram
    class ingest_handler {
        <<control>>
        +handler(event: dict, context: object) dict
        +get_config() dict
        +ingest_source(name: str, fetch_func: function, date: date) int
    }
    class ingest {
        <<boundary>>
        +fetch_ridership(start_date: date, end_date: date) DataFrame
        +fetch_weather(start_date: date, end_date: date, lat: float, lon: float) DataFrame
    }
    class cleaning {
        <<module>>
        +clean_ridership(df: DataFrame) DataFrame
        +clean_weather(df: DataFrame) DataFrame
        +cleaning_report(before: DataFrame, after: DataFrame) dict
    }
    class features {
        <<module>>
        +merge_ridership_weather(rides: DataFrame, weather: DataFrame) DataFrame
        +add_calendar_features(df: DataFrame) DataFrame
    }
    class s3_io {
        <<data access>>
        +upload_raw(bucket: str, source: str, date: date, data: dict) str
        +write_parquet(bucket: str, prefix: str, df: DataFrame) str
        +read_parquet(bucket: str, prefix: str) DataFrame
        +list_keys(bucket: str, prefix: str) list
    }
    class db {
        <<data access>>
        +get_connection(db_url: str) Connection
        +create_tables(conn: Connection) None
        +upsert_ridership(conn: Connection, df: DataFrame) int
        +upsert_weather(conn: Connection, df: DataFrame) int
        +run_query(conn: Connection, sql_path: str) DataFrame
    }
    ingest_handler --> ingest
    ingest_handler --> cleaning
    ingest_handler --> s3_io
    features --> cleaning
    db --> cleaning
```

## 15. Phase 6 · Multilayer sequence diagram: daily ingest

*UML sequence diagram.* Object lifelines (name:Class), messages written as returnValue := message(parameters), a dashed return after each synchronous call, alt frames whose guard conditions Mermaid shows in brackets, «create» for each new raw file, and assumptions in a note.

```mermaid
sequenceDiagram
    actor SCH as :Scheduler
    participant H as :IngestHandler
    participant PS as :ParameterStore
    participant CDP as :ChicagoDataPortal
    participant OM as :OpenMeteo
    participant S3 as :S3DataAccess
    participant CW as :CloudWatch
    Note over H,CDP: Assumptions: targetDate defaults to yesterday. Athena reads clean/ later through Glue tables, outside this run.
    SCH->>H: summary := handler(event, context)
    H->>PS: config := get_config()
    PS-->>H: config
    H->>CDP: rides := fetch_ridership(targetDate, targetDate)
    CDP-->>H: rides
    alt rides returned
        create participant RR as ridershipFile:RawFile
        H->>RR: «create» RawFile(ridership, targetDate, rides)
        H->>S3: rideKey := upload_raw(bucket, ridership, targetDate, rides)
        S3-->>H: rideKey
    else error or no data yet
        H-)CW: putMetric(IngestFailure, 1)
    end
    H->>OM: weather := fetch_weather(targetDate, targetDate, lat, lon)
    OM-->>H: weather
    alt weather returned
        create participant RW as weatherFile:RawFile
        H->>RW: «create» RawFile(weather, targetDate, weather)
        H->>S3: weatherKey := upload_raw(bucket, weather, targetDate, weather)
        S3-->>H: weatherKey
    else error
        H-)CW: putMetric(IngestFailure, 1)
    end
    H->>S3: cleanKey := write_parquet(bucket, clean/, cleanDf)
    S3-->>H: cleanKey
    H-)CW: logSummary(targetDate, rowsWritten)
    H-->>SCH: summary
```

## 16. Phase 7 · Component diagram

*Component diagram.* How the training job, the model file, the container, and the dashboard connect. The course rubrics don't cover component diagrams, so this uses «component» and «artifact» labels to stand in for UML component symbols.

```mermaid
flowchart LR
    ATH["«component»<br/>Athena query layer"]
    TR["«component»<br/>train.py job"]
    EV["«artifact»<br/>evaluation report"]
    MOD["«artifact»<br/>model .joblib in S3"]
    PR["«component»<br/>predict.py"]
    APP["«component»<br/>Streamlit app.py"]
    IMG["«artifact»<br/>Docker image in ECR"]
    RUN["«node»<br/>ECS Fargate or App Runner"]
    U["«actor»<br/>Dashboard Viewer"]
    ATH -- "clean data" --> TR
    TR --> EV
    TR -- "saves" --> MOD
    MOD -- "loads newest" --> PR
    ATH -- "trend and weather data" --> APP
    PR -- "prediction" --> APP
    APP -. "packaged as" .-> IMG
    IMG -. "deployed on" .-> RUN
    RUN --> U
```

## 17. Phase 7 · Design class diagram: model and dashboard

*UML design class diagram.* Training, prediction, and dashboard code with full signatures. Every model is compared with the same-as-7-days-ago baseline before it is saved.

```mermaid
classDiagram
    class dashboard_app {
        <<boundary>>
        +show_ridership_trend(start: date, end: date) None
        +show_weather_impact() None
        +show_model_performance() None
        +show_tomorrow_prediction() None
    }
    class predict {
        <<control>>
        +load_latest_model(bucket: str) Model
        +predict_for_date(model: Model, date: date, weather: dict) int
    }
    class train {
        <<control>>
        +baseline_predict(df: DataFrame) Series
        +train_model(X_train: DataFrame, y_train: Series, kind: str) Model
        +evaluate(y_true: Series, y_pred: Series) dict
        +feature_importance(model: Model, names: list) list
        +save_model(model: Model, bucket: str, date: date) str
    }
    class features {
        <<module>>
        +build_features(df: DataFrame) tuple
        +time_split(df: DataFrame, cutoff_date: date) tuple
    }
    class s3_io {
        <<data access>>
        +read_parquet(bucket: str, prefix: str) DataFrame
        +list_keys(bucket: str, prefix: str) list
    }
    dashboard_app --> predict
    dashboard_app --> s3_io
    predict --> s3_io
    train --> features
    train --> s3_io
    note for dashboard_app "Shows a clear message when no model exists in S3 yet" 
```

## 18. Phase 8 · Activity diagram: CI/CD

*UML activity diagram.* Swimlanes for the Developer, GitHub Actions, and AWS. A pull request runs the checks between a fork and a join; failed checks or a rejected review loop back through merge nodes; only a merge to main reaches AWS.

```mermaid
flowchart TB
    subgraph DEV["Developer"]
        direction TB
        I((" ")) --> A1("Open pull request")
        M2{" "}
        A2("Fix code and push commit")
        A6("Review plan summary")
        D2{" "}
        A7("Merge to main")
        A12("Read failure notice")
    end
    subgraph GA["GitHub Actions"]
        direction TB
        M1{" "}
        FK1[" "]
        B1("Run pytest")
        B2("Check Terraform format and validate")
        JN1[" "]
        B3("Run terraform plan for dev")
        B4("Post plan summary")
        D1{" "}
        B5("Assume AWS role through OIDC")
        B6("Run terraform apply for dev")
        B7("Build and push Docker image")
        FK2[" "]
        B8("Deploy Lambda")
        B9("Deploy dashboard")
        JN2[" "]
        B10("Run smoke tests")
        D3{" "}
        B11("Notify developer")
        M3{" "}
        F(((" ")))
    end
    subgraph AWS["AWS (dev)"]
        direction TB
        C1("Update infrastructure")
        O1["Docker image in ECR"]
    end
    A1 --> M1 --> FK1
    FK1 --> B1 --> JN1
    FK1 --> B2 --> JN1
    JN1 --> B3 --> B4 --> D1
    D1 -- "[checks failed]" --> M2
    D1 -- "[checks passed]" --> A6 --> D2
    D2 -- "[changes requested]" --> M2
    M2 --> A2 --> M1
    D2 -- "[approved]" --> A7 --> B5 --> B6 --> C1 --> B7 --> O1 --> FK2
    FK2 --> B8 --> JN2
    FK2 --> B9 --> JN2
    JN2 --> B10 --> D3
    D3 -- "[tests passed]" --> M3
    D3 -- "[tests failed]" --> B11 --> A12 --> M3
    M3 --> F

    classDef initial fill:#222,stroke:#222,color:#222
    classDef final fill:#222,stroke:#888,color:#222
    classDef bar fill:#222,stroke:#222,color:#222
    classDef node0 fill:transparent,stroke:#888
    classDef obj fill:transparent,stroke:#888,stroke-dasharray:0
    class I initial
    class F final
    class FK1,JN1,FK2,JN2 bar
```
