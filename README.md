# HELEP — Emergency Response Platform

Microservices-based emergency dispatch platform built for the 24-hour capstone exercise.

## Architecture

| Service | Port | Responsibility |
|---|---|---|
| user-service | 8001 | Registration, login, JWT issuance |
| sos-service | 8002 | SOS intake, incident lifecycle |
| dispatch-service | 8003 | Responder matching, saga orchestration |
| notification-service | 8004 | SMS/push delivery |
| analytics-service | 8005 | Metrics aggregation |

Events flow through **Apache Kafka** (Strimzi). Services are deployed to **Kubernetes** via a **Helm umbrella chart**.

## Quick start (local dev)

```bash
# 1. Smoke test with Docker Compose (no K8s needed)
docker compose -f docker-compose.dev.yml up --build

# 2. Sign up
curl -X POST localhost:8001/signup \
  -H 'content-type: application/json' \
  -d '{"phone":"+237600000001","password":"hunter22","role":"citizen"}'

# 3. Send an SOS (use token from step 2)
TOKEN=eyJ...
curl -X POST localhost:8002/sos \
  -H "authorization: Bearer $TOKEN" \
  -H 'content-type: application/json' \
  -d '{"lat":4.0500,"lon":9.7700,"mode":"online"}'

# 4. Watch notification logs
docker compose -f docker-compose.dev.yml logs -f notification-service
```

## Deploy to Kubernetes

```bash
# Install Strimzi operator
kubectl apply -f https://strimzi.io/install/latest?namespace=helep

# Deploy Kafka cluster + topics
kubectl apply -f k8s/kafka/kafka-cluster.yaml
kubectl wait kafka/helep-kafka --for=condition=Ready --timeout=300s -n helep

# Apply base config (namespace, secrets, configmap)
kubectl apply -f k8s/base/secrets-and-config.yaml

# Apply network policies
kubectl apply -f k8s/network/network-policies.yaml

# Apply ingress
kubectl apply -f k8s/ingress/ingress.yaml

# Deploy all services via Helm
helm dependency update helm/helep
helm upgrade --install helep helm/helep \
  --namespace helep --create-namespace \
  --set global.jwtSecret="$(openssl rand -hex 32)"
```

## Useful commands

```bash
# Check all pod status
kubectl get pods -n helep

# View logs for a service
kubectl logs -n helep -l app=sos-service -f

# Check circuit breaker stats (dispatch-service)
kubectl exec -n helep deploy/helep-dispatch-service -- \
  python -c "from app.circuit_breaker import CircuitBreakerRegistry; import json; print(json.dumps(CircuitBreakerRegistry.all_stats(), indent=2))"

# Helm rollback
helm rollback helep -n helep

# Port-forward Grafana
kubectl port-forward svc/helep-kube-prometheus-stack-grafana 3000:80 -n helep
# Open http://localhost:3000 — user: admin / password: helep-grafana
```

## Project structure

```
helep/
├── services/
│   ├── user-service/         # FastAPI, port 8001
│   ├── sos-service/          # FastAPI, port 8002
│   ├── dispatch-service/     # FastAPI, port 8003 — circuit breaker here
│   ├── notification-service/ # FastAPI, port 8004
│   └── analytics-service/   # FastAPI, port 8005
├── helm/
│   └── helep/                # Umbrella Helm chart
│       ├── Chart.yaml
│       ├── values.yaml
│       └── charts/           # Sub-charts (one per service)
├── k8s/
│   ├── base/                 # Namespace, Secrets, ConfigMap
│   ├── kafka/                # Strimzi Kafka CR + KafkaTopics
│   ├── ingress/              # NGINX Ingress
│   ├── monitoring/           # ServiceMonitor, Grafana dashboard
│   └── network/              # NetworkPolicies
├── .github/workflows/
│   └── ci-cd.yaml            # Test → Build → Deploy pipeline
├── design-process-template.md  # Part G — ADRs, data flow, security
└── patterns-template.md        # Part H — Saga, Strategy, Circuit Breaker
```

## Patterns implemented

- **Saga (choreography)** — `sos → dispatch → notification` via Kafka events with compensating transactions
- **Strategy** — pluggable responder matching (`MATCHER=nearest|least_loaded|specialisation`)
- **Circuit Breaker** — fault isolation on all inter-service HTTP calls (`dispatch-service/app/circuit_breaker.py`)
- **Repository / Unit of Work** — DB abstraction per service
