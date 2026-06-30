## Incident Overview

A complete production outage occurred immediately following a standard application code deployment, resulting in continuous HTTP 503 Service Unavailable errors for all end-users.

During the initial triage, the standard GitOps and orchestration metrics failed to surface the underlying issue:

* **ArgoCD** marked the application as completely healthy and synchronized because the manifests matched the repository state.
* **Kubernetes Pods** for both the web application and the database cache were running with zero runtime or initialization crashes.
* **Ingress Routes** were verified as healthy, successfully passing external traffic into the cluster.

The core failure was discovered deeper in the application logs, which showed a total loss of data-layer communication. The database cache logs confirmed that all incoming connection requests from the application were being systematically rejected due to authentication failures. This synchronized perfectly with an automated password rotation routine that had updated the master credentials in the upstream cloud Secret Manager shortly before the deployment.

---

## Root Cause Analysis

The root cause of the outage was **credential desynchronization (state drift)** between the upstream cloud Secret Manager and the internal Kubernetes cluster configuration, caused by an excessive synchronization delay.

The cluster utilizes the External Secrets Operator to bridge cloud-managed security keys into native Kubernetes Secret objects. The synchronization manifest was configured with a long polling delay (`refreshInterval: "1h"`). When the automated routine rotated the master password in the cloud, the operator remained asleep and missed the change, leaving the cluster's internal secret holding the stale password string.

When the application deployment executed shortly after, the newly initialized pods pulled the stale password string from the local secret and attempted to connect to the database. Because the database engine had already updated to match the new cloud configuration requirements, it rejected the application's connection requests. This triggered the cascade failure to HTTP 503, while leaving the external orchestrators unaware because the pods were technically active.

---

## The Solution

The incident was resolved using a two-phased approach: shortening the synchronization loop to pull the correct cloud credentials into the cluster, and recycling the application components to apply the new configuration.

### Phase 1: Remediating the Synchronization Lag

The synchronization manifest (`external-secret.yaml`) was updated to change the `refreshInterval` from `1h` down to a rapid polling cadence (`10s` for testing validation, typically `5m` in production). Once applied, the operator immediately initialized a new pull request to the cloud API, discovered the credential mutation, and updated the internal Kubernetes secret with the correct string value.

### Phase 2: Runtime Environment Recovery

Because Kubernetes environment variables are injected into container runtimes only on startup, the running application pods were still caching the old, stale password in memory. A rolling restart was forced across the deployment, terminating the out-of-sync pods and replacing them with new instances. Upon initialization, the new pods read the freshly updated secret value, established a successful authentication handshake with the database layer, and fully restored public traffic.

---

## Configuration Manifests

### 1. Cloud Secret Bridge (`secret-store.yaml`)

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secretsmanager-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth: {}

```

### 2. Synchronization Sync Loop (`external-secret.yaml`)

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: redis-external-secret
  namespace: default
spec:
  refreshInterval: "10s"
  secretStoreRef:
    name: aws-secretsmanager-store
    kind: ClusterSecretStore
  target:
    name: redis-credentials
    creationPolicy: Owner
  data:
  - remoteRef:
      key: prod/redis/password
      property: password
    secretKey: redis-password

```

### 3. Database Layer Architecture (`redis-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command: ["redis-server", "--requirepass", "NewSecurePassword2026!", "--protected-mode", "no"]
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: default
spec:
  selector:
    app: redis
  ports:
  - protocol: TCP
    port: 6379
    targetPort: 6379

```

### 4. Application Runtime Stack (`app-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: core-application
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: core-app
  template:
    metadata:
      labels:
        app: core-app
    spec:
      containers:
      - name: app
        image: python:3.9-slim
        command: ["/bin/sh", "-c"]
        args:
        - |
          pip install redis flask
          cat << 'EOF' > app.py
          from flask import Flask
          import redis
          import os

          app = Flask(__name__)
          REDIS_PASSWORD = os.environ.get("REDIS_PASSWORD")

          @app.route('/')
          def index():
              try:
                  r = redis.Redis(host='redis-service', port=6379, password=REDIS_PASSWORD, socket_timeout=2)
                  r.ping()
                  return "Application Healthy! Data Layer Online.\n", 200
              except redis.exceptions.AuthenticationError:
                  return "HTTP 503 - Internal Service Error: Redis Authentication Failed\n", 503
              except Exception as e:
                  return f"HTTP 503 - Internal Service Error: {str(e)}\n", 503

          if __name__ == '__main__':
              app.run(host='0.0.0.0', port=8080)
          EOF
          python app.py
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: redis-password
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: core-app-service
  namespace: default
spec:
  type: NodePort
  selector:
    app: core-app
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080

```

---

## Execution & Verification Commands

### Step 1: Verify API Resources

```bash
kubectl api-versions | grep external-secrets

```

### Step 2: Configure Node Group IAM Permissions

```bash
NODE_ROLE=$(aws eks describe-nodegroup --cluster-name production-cluster --nodegroup-name standard-workers --region us-east-1 --query "nodegroup.nodeRole" --output text | awk -F'role/' '{print $2}')

aws iam put-role-policy \
    --role-name "$NODE_ROLE" \
    --policy-name EKS-SecretsManager-Read \
    --policy-document file://secrets-policy.json

```

### Step 3: Apply the Configuration Fix

```bash
kubectl apply -f external-secret.yaml

```

### Step 4: Verify Secret Synchronization Status

```bash
kubectl get externalsecret redis-external-secret
kubectl get secret redis-credentials -o jsonpath='{.data.redis-password}' | base64 --decode

```

### Step 5: Restart the Workload to Clear Memory Cache

```bash
kubectl rollout restart deployment core-application

```

### Step 6: Test Data Layer Connectivity

```bash
kubectl exec -it deployment/core-application -- python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:8080/').read().decode())"

```
