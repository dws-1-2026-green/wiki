# Deployment Diagram

```mermaid
flowchart TB
    subgraph external["External"]
        client["Webhook Producers"]
        webhook_url["Webhook Consumers"]
    end

    subgraph ingress["Ingress (nginx)"]
        direction LR
        ing_ctrl["ingress-nginx\nreplicas: 1–3"]
    end

    subgraph ns["Namespace: webhooks"]
        direction TB

        subgraph apps["Applications"]
            direction TB
            er["event-receiver\nreplicas: 1–5\n⇈ HPA (CPU 70%)"]
            sa["subscriptions-api\nreplicas: 1–5\n⇈ HPA (CPU 70%)"]
            sw["subscriptions-worker\nreplicas: 1–5\n⇈ KEDA (lag ≥ 100)"]
            ds["delivery-service\nreplicas: 1–10\n⇈ KEDA (lag ≥ 50)"]
            dd["delivery-dashboard\nreplicas: 1"]
        end

        subgraph messaging["Messaging"]
            direction TB
            kafka["Kafka (Strimzi)\n1 broker (KRaft)\ntopics: 12 partitions each"]
            kui["kafka-ui\nreplicas: 1"]
        end

        subgraph storage["Storage"]
            direction TB
            cassandra["Cassandra (k8ssandra)\ndc1: 1–3 nodes"]
            pg["PostgreSQL\nreplicas: 1\n5Gi PVC"]
            pgb["PgBouncer\nreplicas: 1\npool: 20 / max clients: 1000"]
        end

        subgraph observability["Observability"]
            direction TB
            prom["Prometheus\nreplicas: 1\n15d retention"]
            grafana["Grafana\nreplicas: 1"]
            loki["Loki\nreplicas: 1"]
            alloy["Alloy\nDaemonSet (per node)"]
        end
    end

    client --> ing_ctrl
    ing_ctrl -->|"HTTP :8080"| er
    ing_ctrl -->|"HTTP :8082"| sa
    ing_ctrl -->|"HTTP :8080"| kui
    ing_ctrl -->|"HTTP :9096"| dd

    er -->|"produces"| kafka
    kafka -->|"topic: routing.requests"| sw
    sw -->|"consumes from Cassandra"| cassandra
    sw -->|"produces"| kafka
    kafka -->|"topic: deliveries.to_send"| ds
    ds -->|"writes deliveries"| pgb
    pgb --> pg
    ds -->|"HTTP POST"| webhook_url

    prom -.->|"scrape"| er
    prom -.->|"scrape"| sa
    prom -.->|"scrape"| sw
    prom -.->|"scrape"| ds
    prom -.->|"scrape"| kafka
    alloy -.->|"logs →"| loki
    grafana -.->|"query"| prom
    grafana -.->|"query"| loki
```

## Scaling Summary

| Service | Type | Min | Max | Trigger | Behavior |
|---------|------|-----|-----|---------|----------|
| event-receiver | HPA | 1 | 5 | CPU 70% | scale up +2/30s, scale down -1/60s |
| subscriptions-api | HPA | 1 | 5 | CPU 70% | scale up +2/30s, scale down -1/60s |
| subscriptions-worker | KEDA | 1 | 5 | Kafka lag ≥ 100 on `routing.requests` | cooldown 120s |
| delivery-service | KEDA | 1 | 10 | Kafka lag ≥ 50 on `deliveries.to_send` | cooldown 120s |

## Data Flow

```
External Events → event-receiver → Kafka(routing.requests) → subscriptions-worker → Kafka(deliveries.to_send) → delivery-service → External URLs
                                         ↑ reads subscriptions from Cassandra                ↑ writes delivery status to PostgreSQL via PgBouncer
```
