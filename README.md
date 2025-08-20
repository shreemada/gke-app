---

# 🚀 GKEApp - .NET 8 API with CI/CD using Cloud Build + Helm

This guide explains how to create, containerize, and deploy a simple **.NET 8 Web API** to **Google Kubernetes Engine (GKE)** using **Helm** and **Cloud Build**.

---

## 📌 Prerequisites

* Google Cloud Project with billing enabled
* GKE Cluster created (`my-gke-cluster`)
* Artifact Registry repository (`gke-repo`)
* Installed locally:

  * [Google Cloud SDK](https://cloud.google.com/sdk/docs/install)
  * [Docker](https://docs.docker.com/get-docker/)
  * [Helm](https://helm.sh/docs/intro/install/)
  * [.NET 8 SDK](https://dotnet.microsoft.com/en-us/download/dotnet/8.0)

---

## 🛠 Step 1: Create .NET 8 API

```bash
dotnet new webapi -o app
cd app
```

This generates a sample WeatherForecast API.

---

## 🐳 Step 2: Create Dockerfile (`app/Dockerfile`)

```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY *.csproj .
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release -o /app/publish

# Stage 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "gkeapp.dll"]
```

✅ Replace `gkeapp.dll` if your project name is different.

---

## 🧪 Step 3: Test Locally

```bash
docker build -t gkeapp:v1 .
docker run -p 8080:8080 gkeapp:v1
```

Visit 👉 [http://localhost:8080/swagger](http://localhost:8080/swagger)

---

## ☸️ Step 4: Create Helm Chart

Create structure:

```
helm/gkeapp/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
```

### `helm/gkeapp/Chart.yaml`

```yaml
apiVersion: v2
name: gkeapp
description: A Helm chart for deploying gkeapp on GKE
type: application
version: 0.1.0
appVersion: "1.0"
```

### `helm/gkeapp/values.yaml`

```yaml
replicaCount: 2

image:
  repository: us-central1-docker.pkg.dev/my-project/gke-repo/gkeapp
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: LoadBalancer
  port: 80
  targetPort: 8080
```

### `helm/gkeapp/templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gkeapp
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: gkeapp
  template:
    metadata:
      labels:
        app: gkeapp
    spec:
      containers:
      - name: gkeapp
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: 8080
```

### `helm/gkeapp/templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gkeapp
spec:
  type: {{ .Values.service.type }}
  selector:
    app: gkeapp
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
```

---

## ⚙️ Step 5: Cloud Build Config (`cloudbuild.yaml`)

```yaml
steps:
# Step 1: Build Docker image
- name: "gcr.io/cloud-builders/docker"
  args: ["build", "-t", "$_IMAGE:$SHORT_SHA", "app"]

# Step 2: Push image
- name: "gcr.io/cloud-builders/docker"
  args: ["push", "$_IMAGE:$SHORT_SHA"]

# Step 3: Deploy with Helm
- name: "gcr.io/cloud-builders/gcloud"
  entrypoint: bash
  args:
    - "-c"
    - |
      gcloud container clusters get-credentials $_CLUSTER_NAME --zone $_CLUSTER_ZONE --project $PROJECT_ID
      helm upgrade --install gkeapp ./helm/gkeapp \
        --set image.repository=$_IMAGE \
        --set image.tag=$SHORT_SHA \
        --set service.type=LoadBalancer

substitutions:
  _IMAGE: "us-central1-docker.pkg.dev/$PROJECT_ID/gke-repo/gkeapp"
  _CLUSTER_NAME: "my-gke-cluster"
  _CLUSTER_ZONE: "us-central1-a"

images:
- "$_IMAGE:$SHORT_SHA"
```

---

## 🚀 Step 6: Set Up CI/CD Trigger

1. Go to **Google Cloud Console → Cloud Build → Triggers**
2. Create a trigger:

   * Event: Push to main branch
   * Repo: Your GitHub/GitLab repo
   * Config file: `cloudbuild.yaml`

---

## 🌍 Step 7: Verify Deployment

After pushing code:

* Cloud Build runs
* Docker image builds + pushes to Artifact Registry
* Helm deploys to GKE
* Service creates a **LoadBalancer**

Check service:

```bash
kubectl get svc gkeapp
```

Output:

```
NAME     TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
gkeapp   LoadBalancer   10.96.150.23   35.223.xx.xx     80:32145/TCP   2m
```

👉 Visit `http://<EXTERNAL_IP>/swagger`

---

## 🎉 Done!

You now have a **fully automated CI/CD pipeline** for your .NET 8 API using:

* **Cloud Build** → CI/CD Automation
* **Artifact Registry** → Image Storage
* **Helm** → Kubernetes Deployment
* **GKE** → Hosting

---
