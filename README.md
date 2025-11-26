# Thmanyah
-- Architecture
PostgreSQL (CDC - Debezium)
          │
          ▼
        Kafka  ----------- Backfill Loader
          │
          ▼
      Flink Job (Enrichment + Transform)
       ┌───────┬────────┬─────────┐
       ▼       ▼        ▼         ▼
BigQuery   Redis   External-System  (Monitoring / DLQ)

--Enrichment + Transformation
JOIN on engagement_events.content_id = content.id
Retrieve: content_type, length_seconds

engagement_seconds = duration_ms / 1000.0
engagement_pct = ROUND(engagement_seconds / length_seconds, 2)
IF length_seconds IS NULL OR duration_ms IS NULL → NULL

-- Unified Enriched Event
{
  "event_id": "...",
  "content_id": "...",
  "user_id": "...",
  "event_type": "play",
  "event_ts": "2025-01-01T12:30:00Z",
  "duration_ms": 12340,
  "device": "ios",
  "content_type": "podcast",
  "length_seconds": 1200,
  "engagement_seconds": 12.34,
  "engagement_pct": 0.01
}


-- To enhance the performance
dataset.engagement_events
PARTITION BY DATE(event_ts)
CLUSTER BY content_type, event_type


source: kafka("engagement-events.raw")
  → map(parse)
  → async_lookup(content)
  → map(transform)
  → side_output(DLQ if errors)
  → tee(
        sink_bigquery(),
        sink_redis(),
        sink_external()
    )

SELECT * FROM engagement_events
WHERE event_ts BETWEEN X AND Y
ORDER BY event_ts


-- Docker Compose
services:
  postgres:
  zookeeper:
  kafka:
  debezium:
  flink-jobmanager:
  flink-taskmanager:
  redis:
  bigquery-emulator (اختياري)
  external-system-mock:
  data-generator:

  -- Structure
  my-engagement-streaming-system/
│
├── docker-compose.yml          
├── flink-job/                  
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/           
│   │   │   └── resources/
│   ├── pom.xml                 
│   ├── requirements.txt        
├── backfill/                   
│   ├── backfill.py
│   └── requirements.txt        
├── data-generator/            
│   └── data_generator.py
├── README.md                   
└── README-backfill.md   
