# External Data Pipeline (EDP) - Direct Deployment

## Overview

This is a streamlined deployment of the EDP stack directly to an OpenShift cluster, without requiring ACM (Advanced Cluster Management).

The EDP consists of:

### Infrastructure Services
- **PostgreSQL**: Two instances for aggregator and notification databases
- **Kafka**: Message broker for data pipeline communication (Strimzi operator)
- **Redis**: Cache for aggregator results
- **MinIO**: S3-compatible storage for uploaded archives
- **Mock OAuth2 Server**: For development/testing authentication
- **RHOBS Mock**: For development/testing observability API

### Processing Services
- **ingress**: HTTP endpoint for receiving archive uploads (port 3000)
- **ccx-data-pipeline**: Processes incoming cluster telemetry data
- **dvo-extractor**: Extracts DVO (Deployment Validation Operator) insights
- **db-writer**: Writes OCP recommendations to PostgreSQL
- **dvo-writer**: Writes DVO recommendations to PostgreSQL
- **cache-writer**: Writes results to Redis cache
- **aggregator**: REST API for querying stored recommendations (port 8082)
- **smart-proxy**: Unified API that aggregates multiple backend services (port 8080)
- **content-service**: Provides recommendation content and metadata (port 8081)
- **ccx-upgrades-data-eng**: Upgrade risk prediction data engineering service
- **ccx-upgrades-inference**: ML inference service for upgrade risks
- **notification-writer**: Writes notification events to database

Total: **19 pods**

## Prerequisites

- An OpenShift cluster (4.18+)
- Cluster admin access
- `oc` CLI installed and logged in

## Quick Start

### Step 1: Install Strimzi Operator

```bash
# Create kafka namespace
oc create namespace kafka

# Install Strimzi operator
oc create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka

# Verify operator is running
oc get pods -n kafka
# Wait for: strimzi-cluster-operator-xxxxx   1/1   Running
```

### Step 2: Deploy Kafka Cluster

```bash
# Apply Kafka cluster and topics
oc apply -f deploy/03-kafka-strimzi.yaml

# Wait for Kafka to be ready (takes 2-3 minutes)
oc wait kafka/edp-kafka --for=condition=Ready --timeout=300s -n kafka

# Verify Kafka is running
oc get kafka -n kafka
oc get kafkatopic -n kafka
```

### Step 3: Deploy EDP Stack

```bash
# Create namespace
oc apply -f deploy/00-namespace.yaml

# Create secrets
oc apply -f deploy/01-secrets.yaml

# Deploy infrastructure (PostgreSQL, Redis, MinIO, mocks)
oc apply -f deploy/02-infrastructure.yaml

# Wait for infrastructure to be ready
oc wait --for=condition=ready pod -l app=postgresql -n edp-processing --timeout=300s
oc wait --for=condition=ready pod -l app=minio -n edp-processing --timeout=300s

# Deploy application services
oc apply -f deploy/04-application-services.yaml

# Wait for all pods to be ready
oc wait --for=condition=ready pod --all -n edp-processing --timeout=600s
```

### Step 4: Verify Deployment

```bash
# Check all pods are running
oc get pods -n edp-processing

# Check Kafka
oc get pods -n kafka

# You should see all 19 pods in Running state
```

Expected output:
```
NAME                                      READY   STATUS    RESTARTS
aggregator-xxx                            1/1     Running   0
cache-writer-xxx                          1/1     Running   0
ccx-data-pipeline-xxx                     1/1     Running   0
ccx-upgrades-data-eng-xxx                 1/1     Running   0
ccx-upgrades-inference-xxx                1/1     Running   0
content-service-xxx                       1/1     Running   0
db-writer-xxx                             1/1     Running   0
dvo-extractor-xxx                         1/1     Running   0
dvo-writer-xxx                            1/1     Running   0
ingress-xxx                               1/1     Running   0
minio-0                                   1/1     Running   0
minio-create-buckets-xxx                  0/1     Completed 0
mock-oauth2-server-xxx                    1/1     Running   0
notification-db-0                         1/1     Running   0
notification-writer-xxx                   1/1     Running   0
postgresql-0                              1/1     Running   0
redis-xxx                                 1/1     Running   0
rhobs-mock-xxx                            1/1     Running   0
smart-proxy-xxx                           1/1     Running   0
```

## Testing the Pipeline

### Expose Services

To test the data pipeline, expose the ingress and aggregator services:

```bash
# Expose ingress for archive uploads
oc create route edge ingress \
  --service=ingress \
  --port=3000 \
  -n edp-processing

# Expose aggregator for querying results
oc create route edge aggregator \
  --service=aggregator \
  --port=8082 \
  -n edp-processing

# Expose smart-proxy (optional, provides enriched API)
oc create route edge smart-proxy \
  --service=smart-proxy \
  --port=8080 \
  -n edp-processing
```

### Upload Test Archive

```bash
# Setup Python environment (first time only)
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Install molodec for generating test archives
export PIP_INDEX_URL=https://repository.engineering.redhat.com/nexus/repository/insights-qe/simple
pip install -U molodec

# Upload test archive
INGRESS_URL=$(oc get route ingress -n edp-processing -o jsonpath='{.spec.host}')
python test_upload.py upload https://$INGRESS_URL
```

Expected output:
```
Uploading to: https://ingress-edp-processing.apps...
Status Code: 202
✅ Archive uploaded successfully!
```

### Watch Processing Flow

```bash
# Watch ingress receive the upload
oc logs -n edp-processing deployment/ingress --tail=20 -f

# Watch ccx-data-pipeline process rules
oc logs -n edp-processing deployment/ccx-data-pipeline --tail=50 -f

# Watch db-writer write to PostgreSQL
oc logs -n edp-processing deployment/db-writer --tail=20 -f
```

### Query Results

```bash
# Get the aggregator URL
AGGREGATOR_URL=$(oc get route aggregator -n edp-processing -o jsonpath='{.spec.host}')

# Query cluster reports (replace with your test cluster ID)
CLUSTER_ID="181862b9-c53b-4ea9-ae22-ac4415e2cf21"
curl -sk "https://$AGGREGATOR_URL/api/v1/organizations/1/clusters/$CLUSTER_ID/reports" | jq
```

## Configuration

### Default Credentials (Development Only)

**PostgreSQL:**
- Username: `user`
- Password: `password`
- Aggregator DB: `aggregator`
- Notification DB: `notifications`

**Redis:**
- Password: `password`

**Kafka:**
- No authentication (PLAINTEXT)

**MinIO:**
- Access Key: `minio`
- Secret Key: `minio123`

**IMPORTANT:** For production, update secrets in `deploy/01-secrets.yaml` before deploying.

### Service Endpoints

**Infrastructure:**
- `postgresql.edp-processing.svc.cluster.local:5432` - Aggregator PostgreSQL
- `notification-db.edp-processing.svc.cluster.local:5432` - Notification PostgreSQL
- `edp-kafka-kafka-bootstrap.kafka.svc.cluster.local:9092` - Kafka broker
- `redis.edp-processing.svc.cluster.local:6379` - Redis cache
- `minio.edp-processing.svc.cluster.local:9000` - MinIO S3 API
- `minio.edp-processing.svc.cluster.local:9001` - MinIO Console

**Application Services:**
- `ingress:3000` - Archive upload endpoint
- `aggregator:8082` - Aggregator REST API
- `smart-proxy:8080` - Smart Proxy unified API
- `content-service:8081` - Content Service API

## Resource Requirements

The full stack requires approximately:

**CPU:**
- Minimum: ~3.5 cores (requests)
- Maximum: ~11 cores (limits)

**Memory:**
- Minimum: ~7.5 GB (requests)
- Maximum: ~15 GB (limits)

**Storage:**
- PostgreSQL aggregator: 10Gi
- PostgreSQL notifications: 5Gi
- MinIO: 20Gi

**Recommended minimum cluster size:**
- 3 worker nodes
- 4 vCPUs per node
- 16 GB RAM per node

For development/testing on smaller clusters, you can reduce resource requests in the deployment files.

## Troubleshooting

### Check Pod Status
```bash
oc get pods -n edp-processing
oc describe pod <pod-name> -n edp-processing
```

### Check Logs
```bash
oc logs -n edp-processing deployment/<service-name> -f
```

### Verify Kafka Topics
```bash
oc get kafkatopic -n kafka

# Exec into Kafka pod to inspect topics
oc exec -it -n kafka edp-kafka-dual-role-0 -- /bin/bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

### Check Database
```bash
# Check PostgreSQL
oc exec -n edp-processing postgresql-0 -- \
  psql -U user -d aggregator -c "SELECT * FROM rule_hit LIMIT 5;"
```

### MinIO Bucket Issues
If the minio-create-buckets Job fails:
```bash
# Delete and recreate the job
oc delete job minio-create-buckets -n edp-processing
oc apply -f deploy/02-infrastructure.yaml
```

## Cleanup

To remove the entire EDP stack:

```bash
# Delete application services
oc delete -f deploy/04-application-services.yaml

# Delete infrastructure
oc delete -f deploy/02-infrastructure.yaml

# Delete secrets
oc delete -f deploy/01-secrets.yaml

# Delete Kafka
oc delete -f deploy/03-kafka-strimzi.yaml

# Delete namespace
oc delete -f deploy/00-namespace.yaml

# Delete Kafka namespace
oc delete namespace kafka
```

## Architecture

The data flow is:

```
Upload → Ingress → MinIO → Kafka → ccx-data-pipeline →
Kafka (results) → db-writer → PostgreSQL → Aggregator REST API
                                          ↓
                                       Smart Proxy
```

## Differences from ACM Addon Approach

This deployment is simpler than the ACM addon approach:

| ACM Addon | Direct Deployment |
|-----------|-------------------|
| Requires 2 clusters (hub + managed) | Requires 1 cluster |
| Requires ACM installation | No ACM needed |
| Requires placement/import setup | Direct oc apply |
| Manifests wrapped in AddOnTemplates | Plain Kubernetes YAML |
| More complex troubleshooting | Simpler to debug |

## Next Steps

- Update credentials in `deploy/01-secrets.yaml` for production use
- Adjust resource limits based on your cluster capacity
- Set up proper monitoring and alerting
- Configure persistent backup for PostgreSQL and MinIO
