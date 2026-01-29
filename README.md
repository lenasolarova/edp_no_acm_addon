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

## Quick Start

In Cluster Bot: 
```bash
launch 4.20 aws,large
```

Once the cluster is ready:

```bash
# Log in into the cluster (server URL in kubeconfig)
oc login -u kubeadmin -p <password> <URL>

# Get the cluster URL or simply use the one Clusterbot gave
oc whoami --show-console
```

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

# Configure image pull credentials - testing version
oc create secret docker-registry quay-pull-secret \
  --docker-server=quay.io \
  --docker-username=<your-quay-username> \
  --docker-password=<your-quay-password> \
  -n edp-processing

# Link the secret to the default service account
oc secrets link default quay-pull-secret --for=pull -n edp-processing

# Create secrets
oc apply -f deploy/01-secrets.yaml

# Deploy infrastructure (PostgreSQL, Redis, MinIO, mocks)
oc apply -f deploy/02-infrastructure.yaml

# Wait for infrastructure to be ready
oc wait --for=condition=ready pod -l app=postgresql -n edp-processing --timeout=300s
oc wait --for=condition=ready pod -l app=minio -n edp-processing --timeout=300s

# Deploy application services
oc apply -f deploy/04-application-services.yaml

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
> oc get pods -n edp-processing

NAME                                      READY   STATUS      RESTARTS   AGE
aggregator-6494dc7976-5csd7               1/1     Running     0          104s
cache-writer-5f4c47497b-ksp55             1/1     Running     0          104s
ccx-data-pipeline-b465d9bf-m49t4          1/1     Running     0          105s
ccx-upgrades-data-eng-7685565f99-88rnx    1/1     Running     0          102s
ccx-upgrades-inference-6f86d86854-rcg6r   1/1     Running     0          101s
content-service-67cd5f4889-2ccdf          1/1     Running     0          103s
db-writer-77dd578f6d-qcgv5                1/1     Running     0          105s
dvo-extractor-558bbf6ddc-scvdk            1/1     Running     0          105s
dvo-writer-55994d9b5-bclj2                1/1     Running     0          104s
ingress-67f47c644-9qkjb                   1/1     Running     0          100s
minio-0                                   1/1     Running     0          2m23s
minio-create-buckets-cc5xg                0/1     Completed   0          2m22s
mock-oauth2-server-5bd8bd579-pmbrk        1/1     Running     0          2m24s
notification-db-0                         1/1     Running     0          2m25s
notification-writer-5497b7d57d-k4mjx      1/1     Running     0          100s
postgresql-0                              1/1     Running     0          2m26s
redis-5f6b544485-wmzbj                    1/1     Running     0          2m25s
rhobs-mock-85895c697b-ct5b7               1/1     Running     0          2m23s
smart-proxy-58f8ddffb9-xxwkl              1/1     Running     0          102s

> oc get pods -n kafka

NAME                                         READY   STATUS    RESTARTS   AGE
edp-kafka-dual-role-0                        1/1     Running   0          4m53s
edp-kafka-entity-operator-5dc4b5fb54-k5nrl   1/1     Running   0          4m15s
strimzi-cluster-operator-6c84667cb8-2n9f9    1/1     Running   0          5m59s
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

# Expose smart-proxy
oc create route edge smart-proxy \
  --service=smart-proxy \
  --port=8080 \
  -n edp-processing
```

### Option 1: Manual Upload Test Archive

For quick testing, you can manually upload a test archive:

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

### Option 2: Configure Local Insights Pipeline

**Prerequisites:**
- ACM installed on the cluster
- `insights-client` deployment running in the `open-cluster-management` namespace

This option configures the full insights pipeline to use your local EDP stack:
1. **insights-operator** → uploads cluster data to local **ingress:3000**
2. Processing pipeline → processes the data
3. **insights-client** → fetches results from local **aggregator:8082**

#### Step 2.1: Configure Ingress to Accept Unauthenticated Uploads

The ingress service requires authentication by default. Disable it for local development:

```bash
# Disable authentication requirement in ingress
oc set env deployment/ingress -n edp-processing INGRESS_AUTH=false

# Wait for rollout to complete
oc rollout status deployment/ingress -n edp-processing
```

#### Step 2.2: Configure insights-operator to Upload Locally

By default, insights-operator uploads to Red Hat's cloud (`console.redhat.com`). Configure it to use your local ingress instead:

```bash
# Create a support Secret to override the upload endpoint
# Note: Must be a Secret (not ConfigMap) and must include full path
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: support
  namespace: openshift-config
type: Opaque
stringData:
  endpoint: "http://ingress.edp-processing.svc.cluster.local:3000/api/ingress/v1/upload"
EOF

# Restart insights-operator to pick up the new configuration
oc delete pod -n openshift-insights -l app=insights-operator

# Wait for it to come back up
oc wait --for=condition=ready pod -l app=insights-operator -n openshift-insights --timeout=120s
```

**Verify insights-operator configuration:**

```bash
# Check the configuration is loaded
oc logs -n openshift-insights deployment/insights-operator --tail=100 | grep -A 15 "Configuration is"

# Expected output should show:
# uploadEndpoint: http://ingress.edp-processing.svc.cluster.local:3000/api/ingress/v1/upload
```

**Trigger immediate upload (instead of waiting 2 hours):**

```bash
# Force insights operator to gather and upload immediately
oc annotate -n openshift-insights insightsoperator cluster \
  insightsoperator.openshift.io/gather='{}'

# Watch for upload activity
oc logs -n openshift-insights deployment/insights-operator -f | grep -i upload

# Expected output:
# Uploading application/vnd.redhat.openshift.periodic to http://ingress.edp-processing.svc.cluster.local:3000/api/ingress/v1/upload
# Uploaded report successfully in XXXms
```

#### Step 2.3: Configure insights-client to Fetch from Local Aggregator

The insights-client needs to fetch processed reports from the aggregator service (not ingress). It appends the cluster path to CCX_SERVER, so we need to include the base API path:

```bash
# Configure insights-client to use local aggregator
# Note: insights-client appends "/clusters/{id}/reports" to this URL
oc set env deployment/insights-client -n open-cluster-management \
  CCX_SERVER=http://aggregator.edp-processing.svc.cluster.local:8082/api/v1/organizations/1

# Restart the deployment
oc rollout restart deployment/insights-client -n open-cluster-management

# Wait for the rollout to complete
oc rollout status deployment/insights-client -n open-cluster-management
```

**Verify insights-client configuration:**

```bash
# Check the environment variable is set correctly
oc exec -n open-cluster-management deployment/insights-client -- env | grep CCX_SERVER
# Expected output: CCX_SERVER=http://aggregator.edp-processing.svc.cluster.local:8082/api/v1/organizations/1

# Watch the insights-client logs
oc logs -n open-cluster-management deployment/insights-client -f

# Expected output:
# Creating Request for cluster local-cluster (...) using Insights URL http://aggregator.edp-processing.svc.cluster.local:8082/api/v1/organizations/1/clusters/.../reports
```

### Watch the Complete Processing Flow

After configuring both insights-operator and insights-client, watch the data flow through the pipeline:

```bash
# 1. Watch insights-operator upload archive to ingress
oc logs -n openshift-insights deployment/insights-operator -f | grep -i upload

# 2. Watch ingress receive the upload and send to Kafka
oc logs -n edp-processing deployment/ingress --tail=20 -f

# 3. Watch ccx-data-pipeline consume from Kafka and process rules
oc logs -n edp-processing deployment/ccx-data-pipeline --tail=50 -f

# 4. Watch db-writer write results to PostgreSQL
oc logs -n edp-processing deployment/db-writer --tail=20 -f

# 5. Watch insights-client fetch results from aggregator
oc logs -n open-cluster-management deployment/insights-client -f

# 6. Check PolicyReports created by insights-client
oc get policyreports -A
```

### Query Results from Aggregator

```bash
# Get the aggregator URL
AGGREGATOR_URL=$(oc get route aggregator -n edp-processing -o jsonpath='{.spec.host}')

# Query cluster reports (replace with your test cluster ID)
CLUSTER_ID="9f1511c6-6ef4-48ef-8fe9-e6dfea7076f0"
curl -sk "https://$AGGREGATOR_URL/api/v1/organizations/1/clusters/$CLUSTER_ID/reports" | jq
```

### Query Results from Smart Proxy

```bash
# Get the smart-proxy URL
SMART_PROXY_URL=$(oc get route smart-proxy -n edp-processing -o jsonpath='{.spec.host}')

# Identity header (base64 encoded: {"identity": {"type": "User", "account_number": "0000001", "org_id": "000001", "internal": {"org_id": "000001"}}})
IDENTITY_HEADER="eyJpZGVudGl0eSI6IHsidHlwZSI6ICJVc2VyIiwgImFjY291bnRfbnVtYmVyIjogIjAwMDAwMDEiLCAib3JnX2lkIjogIjAwMDAwMSIsICJpbnRlcm5hbCI6IHsib3JnX2lkIjogIjAwMDAwMSJ9fX0="

# Query cluster reports with enriched content
CLUSTER_ID="9f1511c6-6ef4-48ef-8fe9-e6dfea7076f0"
curl -sk -H "x-rh-identity: $IDENTITY_HEADER" \
  "https://$SMART_PROXY_URL/api/v1/clusters/$CLUSTER_ID/report" | \
  jq '.report.data[] | {rule_id, description, total_risk, resolution}'
```

## Configuration

### Image Registry Credentials

Before deploying, ensure you have valid credentials for the image registry (quay.io). These are required to pull the EDP service images. You'll need:
- Quay.io username
- Quay.io password or robot token

The credentials are configured during Step 3 of the deployment process.

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


## Troubleshooting

### Insights Pipeline Issues

**Problem: insights-client getting 404 errors**
```bash
# Check insights-client logs for the URL it's using
oc logs -n open-cluster-management deployment/insights-client --tail=50 | grep "Insights URL"

# Should show: http://aggregator.edp-processing.svc.cluster.local:8082/api/v1/organizations/1/clusters/.../reports
# If wrong path, fix: Re-run Step 2.3 to configure CCX_SERVER correctly
```

**Problem: insights-operator still uploading to console.redhat.com**
```bash
# Check insights-operator configuration
oc logs -n openshift-insights deployment/insights-operator --tail=100 | grep -A 15 "Configuration is"

# Should show: uploadEndpoint: http://ingress.edp-processing.svc.cluster.local:3000/api/ingress/v1/upload
# If it shows console.redhat.com, the support Secret wasn't applied correctly

# Check if the support Secret exists
oc get secret support -n openshift-config

# Fix: Re-run Step 2.2 to create the support Secret and restart insights-operator
```

**Problem: ingress rejecting uploads with "missing x-rh-identity header"**
```bash
# Check if INGRESS_AUTH is set to false
oc exec -n edp-processing deployment/ingress -- env | grep INGRESS_AUTH
# Expected: INGRESS_AUTH=false

# If not set or set to true, fix:
oc set env deployment/ingress -n edp-processing INGRESS_AUTH=false
oc rollout status deployment/ingress -n edp-processing
```

**Problem: No data in aggregator database**
```bash
# Check if ingress received any uploads
oc logs -n edp-processing deployment/ingress --tail=100

# Check if ccx-data-pipeline processed anything
oc logs -n edp-processing deployment/ccx-data-pipeline --tail=100 | grep -i "cluster"

# Check database directly
oc exec -n edp-processing postgresql-0 -- \
  psql -U user -d aggregator -c "SELECT cluster, reported_at FROM report ORDER BY reported_at DESC LIMIT 5;"

# If empty, data isn't flowing through the pipeline
# Trigger a manual upload:
oc annotate -n openshift-insights insightsoperator cluster insightsoperator.openshift.io/gather='{}'
```

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
The bucket creation job now automatically waits for MinIO to be ready (with retries). If you still encounter issues:
```bash
# Check the job logs
oc logs -n edp-processing job/minio-create-buckets

# If needed, delete and recreate the job
oc delete job minio-create-buckets -n edp-processing
oc apply -f deploy/02-infrastructure.yaml

# Verify buckets were created by checking the logs again
oc logs -n edp-processing job/minio-create-buckets
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

