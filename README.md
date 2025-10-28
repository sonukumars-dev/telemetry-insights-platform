# Telemetry Insights Platform
Real-time vehicle data analytics with Spring Boot, Kafka, and AI-driven anomaly detection.
(Architecture + sprint plan in progress)

#COMPONENTS
| Service                         | Purpose                                                          | Tech                    |
|---------------------------------|------------------------------------------------------------------|-------------------------|
| **telemetry-producer**          | Simulate vehicle data and push to Kafka topic `vehicle-data`     | Python script           |
| **telemetry-ingestion-service** | Consume telemetry events, validate, store, trigger anomaly check | Spring Boot 3 (Java 17) |
| **telemetry-ai-service**        | Serve anomaly detection model over REST                          | FastAPI (Python)        |
| **PostgreSQL**                  | Persistent storage of telemetry + anomalies                      | RDS/local container     |
| **Redis**                       | Cache last 1 000 records / recent anomalies                      | optional                |
| **Next.js dashboard**           | Minimal view for metrics                                         | optional                |
| **CI/CD & Infra**               | GitHub Actions + Terraform + AWS ECS                             | Yet to decide           |

#DATA FLOW
Producer generates JSON payload every few seconds and publishes to Kafka.
Ingestion Service consumes messages, validates schema, stores record in Postgres.
It asynchronously calls AI Service (POST /predict) with telemetry data.
AI Service returns anomaly score ∈ [0, 1].
Ingestion Service tags record as “normal” or “anomaly” and persists result.
GET APIs expose latest telemetry or anomalies for the dashboard.

#DATA SCHEMA (PostgreSQL)
Table: telemetry_records
| Column        | Type                    | Description            |
|---------------|-------------------------|------------------------|
| id            | UUID PK                 | unique record id       |
| vehicle_id    | VARCHAR(50)             | vehicle identifier     |
| timestamp     | TIMESTAMP WITH TZ       | event time             |
| speed         | FLOAT                   | km/h                   |
| engine_temp   | FLOAT                   | °C                     |
| battery_level | FLOAT                   | %                      |
| gps_lat       | FLOAT                   |                        |
| gps_long      | FLOAT                   |                        |
| anomaly_score | FLOAT                   | 0-1                    |
| is_anomaly    | BOOLEAN                 | derived from threshold |
| created_at    | TIMESTAMP DEFAULT now() | insert time            |

*Add an index on (vehicle_id, timestamp)

#API CONTRACTS
Spring Boot – telemetry-ingestion-service
| Method   | Endpoint                                         | Description                           |
| -------- | ------------------------------------------------ | ------------------------------------- |
| `POST`   | `/api/telemetry`                                 | (debug/manual) add a telemetry record |
| `GET`    | `/api/telemetry/latest?vehicleId={id}&limit={n}` | fetch recent telemetry                |
| `GET`    | `/api/telemetry/anomalies?limit={n}`             | list latest anomalies                 |
| Internal | calls AI Service `/predict`                      |                                       |

Request (internal or from producer)
{
"vehicleId": "VHC123",
"speed": 68.2,
"engineTemp": 94.5,
"batteryLevel": 87.0,
"gpsLat": 12.9723,
"gpsLong": 77.5946,
"timestamp": "2025-10-29T09:31:00Z"
}

Response
{
"vehicleId": "VHC123",
"isAnomaly": false,
"anomalyScore": 0.07
}

#FastAPI – telemetry-ai-service
| Method | Endpoint   | Description                                   |
| ------ | ---------- | --------------------------------------------- |
| `POST` | `/predict` | Accepts telemetry JSON, returns anomaly score |

Request
{"speed":68.2,"engineTemp":94.5,"batteryLevel":87.0}

Response
{"anomalyScore":0.07,"isAnomaly":false}

#REPO STRUCTURE
telemetry-insights-platform/
│
├── telemetry-ingestion-service/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/sonu/telemetry/
│   │   │   │   ├── controller/
│   │   │   │   ├── service/
│   │   │   │   ├── repository/
│   │   │   │   ├── model/
│   │   │   │   └── config/
│   │   │   └── resources/
│   │   │       ├── application.yml
│   │   │       └── logback-spring.xml
│   │   └── test/java/com/sonu/telemetry/
│   │       └── TelemetryIngestionServiceTests.java
│   ├── build.gradle
│   ├── settings.gradle
│   ├── gradlew
│   ├── gradlew.bat
│   ├── gradle/
│   │   └── wrapper/
│   │       ├── gradle-wrapper.jar
│   │       └── gradle-wrapper.properties
│   └── Dockerfile
│
├── telemetry-ai-service/
│   ├── app/
│   │   ├── main.py
│   │   ├── model.pkl
│   │   ├── train_model.py
│   │   └── requirements.txt
│   └── Dockerfile
│
├── infra/
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── ci-cd/
│       ├── github-actions.yml
│       └── docker-build.yml
│
├── docker-compose.yml
├── README.md
└── .gitignore