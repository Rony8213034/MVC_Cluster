# vinyl-api on Kubernetes (Helm)

This chart deploys the **vinyl-api** Flask service and exposes it via a Kubernetes `Service` of type `LoadBalancer`.
MongoDB is deployed separately using the Bitnami MongoDB Helm chart (replicaset mode).

## Repo layout

```
charts/
└─ vinyl-api/
   ├─ Chart.yaml            # Chart metadata
   ├─ values.yaml           # Default configuration (image, env, Mongo URL, resources, etc.)
   └─ templates/
      ├─ _helpers.tpl       # Helper template funcs (names/labels)
      ├─ configmap.yaml     # App config (env like DB_NAME, COLL_NAME, SEED_DATA)
      ├─ deployment.yaml    # Deployment with liveness/readiness probes
      ├─ secret.yaml        # (Optional) Creates Secret vinyl-api-secrets with MONGO_URL
      └─ service.yaml       # Service (type LoadBalancer)
mongo-values.yml             # Values used to install Bitnami MongoDB (replicaset)
```

## Prereqs

* Helm v3 and kubectl configured for your cluster
* A namespace (defaults to `dev`)
* Bitnami MongoDB chart locally (or install from remote)
* Your app image in a registry the cluster can pull from (ECR in your case)

## 1) Install MongoDB (replicaset)

### Option A – from your local copy (recommended, pinned)

```bash
# create namespace once
kubectl create namespace dev

# install from local chart dir with your values
helm upgrade --install mongo charts/mongodb \
  -n dev -f mongo-values.yml --wait
```

### Option B – from Bitnami registry (if you prefer remote)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install mongo bitnami/mongodb \
  -n dev -f mongo-values.yml --wait
```

This creates headless Services like:

* `mongo-headless.dev.svc.cluster.local`
* `mongo-arbiter-headless.dev.svc.cluster.local`

Your app’s connection string will reference those hostnames and `replicaSet=rs0`.

## 2) Provide MongoDB to the API

You have **two** ways to pass `MONGO_URL` to the app pod:

### A) Let the chart create the Secret for you (simple)

In `charts/vinyl-api/values.yaml`, set:

```yaml
mongo:
  existingSecret: ""  # keep empty
  mongoUrl: "mongodb://appuser:ChangeMeAppPass!@mongo-0.mongo-headless.dev.svc.cluster.local:27017,mongo-1.mongo-headless.dev.svc.cluster.local:27017/appdb?replicaSet=rs0&authSource=appdb"
```

> Note: the `!` in the password is already quoted inside a YAML string—keep it quoted.

The template will create `Secret/vinyl-api-secrets` with a key `MONGO_URL`.

### B) Use a pre-created Secret (advanced)

Create the secret yourself:

```bash
kubectl -n dev create secret generic vinyl-api-secrets \
  --from-literal=MONGO_URL='mongodb://appuser:ChangeMeAppPass!@mongo-0.mongo-headless.dev.svc.cluster.local:27017,mongo-1.mongo-headless.dev.svc.cluster.local:27017/appdb?replicaSet=rs0&authSource=appdb'
```

Then point the chart at it:

```yaml
mongo:
  existingSecret: "vinyl-api-secrets"
  mongoUrl: ""   # ignored when existingSecret is set
```

## 3) Configure the app (values you likely want to change)

Inside `charts/vinyl-api/values.yaml`:

```yaml
image:
  repository: <your ECR repo>/myvinalcollection
  tag: dev-latest     # or your specific tag/sha
  pullPolicy: Always

service:
  type: LoadBalancer
  targetPort: 8000

env:
  DB_NAME: appdb
  COLL_NAME: records
  SEED_DATA: "true"   # the app seeds the collection on startup when empty

mongo:
  existingSecret: ""  # or "vinyl-api-secrets"
  mongoUrl: "mongodb://appuser:ChangeMeAppPass!@mongo-0.mongo-headless.dev.svc.cluster.local:27017,mongo-1.mongo-headless.dev.svc.cluster.local:27017/appdb?replicaSet=rs0&authSource=appdb"
```

> `DB_NAME` controls which database the app uses in your code. Make sure your Mongo user (`appuser`) has `readWrite` on this DB.

## 4) Install the API

```bash
helm upgrade --install vinyl-api charts/vinyl-api \
  -n dev --set image.tag=dev-latest --wait
```

Check:

```bash
kubectl -n dev get pods -w
kubectl -n dev get svc vinyl-api
```

Grab the external DNS (ELB/ALB) from the `vinyl-api` service and test:

```bash
curl http://<elb-dns>/healthz
# {"status":"ok"}
```

## 5) Upgrade / rollback

```bash
# change image tag (or edit values.yaml), then:
helm upgrade vinyl-api charts/vinyl-api -n dev --set image.tag=<newTag> --wait

# history & rollback
helm history vinyl-api -n dev
helm rollback vinyl-api <REV> -n dev
```

## 6) Uninstall / reset

```bash
helm uninstall vinyl-api -n dev
helm uninstall mongo -n dev

# WARNING: deleting PVCs removes data
kubectl -n dev delete pvc -l app.kubernetes.io/name=mongodb
```

## 7) Troubleshooting

* **API `CrashLoopBackOff` with `Unauthorized` in logs:**
  Your Mongo user lacks permissions on the DB the app is using. Connect to the primary and grant:

  ```javascript
  // in the 'appdb' (or the DB you actually use)
  use appdb
  db.grantRolesToUser("appuser", [{role: "readWrite", db: "appdb"}])

  // if the user doesn't exist:
  db.createUser({user: "appuser", pwd: "ChangeMeAppPass!", roles:[{role:"readWrite", db:"appdb"}]})
  ```

  Ensure the connection string has `authSource=appdb` and `replicaSet=rs0`.

* **`/healthz` works but `/records` returns 500:**
  That’s an **app** error (not Kubernetes). Usually means the app cannot read/write the collection or your seed/DB\_NAME doesn’t match the user’s privileges.

* **Cannot pull image:**
  Check image repo, tag, and ECR permissions. Set `imagePullSecrets` if needed.

* **Stuck pod after tinkering with volumes:**
  If you changed Mongo statefulset storage, remove PVCs (data loss!) or use a fresh namespace.

## 8) Notes about the chart

* The chart exposes the app on port `8000` and configures liveness/readiness probes (`/healthz`).
* Service is `LoadBalancer`. If you’re on EKS, ensure the AWS Load Balancer Controller (or classic ELB support) is present.
* The `secret.yaml` template is **conditional**: it only creates a secret when `mongo.existingSecret` is empty and `mongo.mongoUrl` is set.
* Keep sensitive values out of Git when possible—prefer `existingSecret` in production.

---

### Quick start (copy/paste)

```bash
# namespace
kubectl create ns dev

# mongodb
helm upgrade --install mongo charts/mongodb -n dev -f mongo-values.yml --wait

# api
helm upgrade --install vinyl-api charts/vinyl-api \
  -n dev --set image.tag=dev-latest --wait

# test
kubectl -n dev get svc vinyl-api
curl http://<elb-dns>/healthz
```