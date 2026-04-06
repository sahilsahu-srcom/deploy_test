# Google Cloud Platform (GCP) Deployment Guide

## Table of Contents
1. [GCP Overview & Free Tier](#gcp-overview--free-tier)
2. [Prerequisites & Setup](#prerequisites--setup)
3. [Option 1: Cloud Run (Serverless)](#option-1-cloud-run-serverless)
4. [Option 2: App Engine](#option-2-app-engine)
5. [Option 3: Compute Engine (VM)](#option-3-compute-engine-vm)
6. [Option 4: GKE (Kubernetes)](#option-4-gke-kubernetes)
7. [Cost Comparison](#cost-comparison)
8. [CI/CD Setup](#cicd-setup)

---

## GCP Overview & Free Tier

### Free Trial
- **$300 credit** for 90 days
- No automatic charges after trial
- Must manually upgrade to paid

### Always Free Tier (After Trial)
- **Cloud Run**: 2 million requests/month
- **Cloud Functions**: 2 million invocations/month
- **Compute Engine**: 1 f1-micro instance (US regions)
- **Cloud Storage**: 5GB storage
- **Cloud Build**: 120 build-minutes/day
- **Firestore**: 1GB storage, 50K reads/day

### Key Differences from AWS
✅ More generous free tier
✅ Simpler pricing
✅ Better for containers
✅ Excellent for Kubernetes
❌ Smaller ecosystem than AWS
❌ Fewer regions

---

## Prerequisites & Setup

### 1. Create GCP Account
1. Go to https://cloud.google.com
2. Click "Get started for free"
3. Sign in with Google account
4. Add payment method (for $300 credit)
5. Create new project

### 2. Install Google Cloud SDK

**Windows:**
```powershell
# Download installer
(New-Object Net.WebClient).DownloadFile("https://dl.google.com/dl/cloudsdk/channels/rapid/GoogleCloudSDKInstaller.exe", "$env:Temp\GoogleCloudSDKInstaller.exe")
& $env:Temp\GoogleCloudSDKInstaller.exe
```

**Mac:**
```bash
brew install --cask google-cloud-sdk
```

**Linux:**
```bash
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
```

### 3. Initialize gcloud

```bash
# Login
gcloud auth login

# Set project
gcloud config set project YOUR_PROJECT_ID

# Set region
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
```

### 4. Enable Required APIs

```bash
# Enable APIs
gcloud services enable cloudbuild.googleapis.com
gcloud services enable run.googleapis.com
gcloud services enable compute.googleapis.com
gcloud services enable sqladmin.googleapis.com
gcloud services enable storage.googleapis.com
```

---


## Option 1: Cloud Run (Serverless) - RECOMMENDED

**Best for**: Most projects, auto-scaling, pay-per-use
**Cost**: FREE tier covers most small apps
**Difficulty**: ⭐⭐ Easy

### Architecture
```
User → Cloud CDN → Cloud Storage (Frontend)
     → Cloud Run (Backend) → Cloud SQL / Neon
```

### Step 1: Prepare Backend for Cloud Run

Verify `backend/Dockerfile`:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Cloud Run uses PORT environment variable
ENV PORT=8080
EXPOSE 8080

CMD exec uvicorn main:app --host 0.0.0.0 --port ${PORT}
```

### Step 2: Build and Deploy Backend

```bash
cd backend

# Build container image
gcloud builds submit --tag gcr.io/YOUR_PROJECT_ID/todo-backend

# Deploy to Cloud Run
gcloud run deploy todo-backend \
  --image gcr.io/YOUR_PROJECT_ID/todo-backend \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars DATABASE_URL="your-neon-url"

# Get service URL
gcloud run services describe todo-backend \
  --platform managed \
  --region us-central1 \
  --format 'value(status.url)'
```

**Your backend URL**: https://todo-backend-xxxxx-uc.a.run.app

### Step 3: Deploy Frontend to Cloud Storage

```bash
# Build frontend
npm run build

# Create bucket
gsutil mb -l us-central1 gs://your-todo-app-frontend

# Enable website configuration
gsutil web set -m index.html -e index.html gs://your-todo-app-frontend

# Upload files
gsutil -m cp -r dist/* gs://your-todo-app-frontend/

# Make public
gsutil iam ch allUsers:objectViewer gs://your-todo-app-frontend
```

**Frontend URL**: https://storage.googleapis.com/your-todo-app-frontend/index.html

### Step 4: Setup Cloud CDN (Optional)

```bash
# Create backend bucket
gcloud compute backend-buckets create todo-frontend-backend \
  --gcs-bucket-name=your-todo-app-frontend \
  --enable-cdn

# Create URL map
gcloud compute url-maps create todo-url-map \
  --default-backend-bucket=todo-frontend-backend

# Create HTTP proxy
gcloud compute target-http-proxies create todo-http-proxy \
  --url-map=todo-url-map

# Reserve IP address
gcloud compute addresses create todo-ip --global

# Get IP address
gcloud compute addresses describe todo-ip --global --format="value(address)"

# Create forwarding rule
gcloud compute forwarding-rules create todo-http-rule \
  --address=todo-ip \
  --global \
  --target-http-proxy=todo-http-proxy \
  --ports=80
```

### Step 5: Update Frontend API URLs

Update `src/App.jsx`:
```javascript
const API_URL = 'https://todo-backend-xxxxx-uc.a.run.app';

const response = await fetch(`${API_URL}/api/todos`);
```

Rebuild and redeploy:
```bash
npm run build
gsutil -m rsync -r -d dist/ gs://your-todo-app-frontend/
```

### Cloud Run Features

**Auto-scaling:**
```bash
# Set min/max instances
gcloud run services update todo-backend \
  --min-instances=0 \
  --max-instances=10
```

**Custom domain:**
```bash
# Map custom domain
gcloud run domain-mappings create \
  --service=todo-backend \
  --domain=api.yourdomain.com
```

**View logs:**
```bash
gcloud run logs read todo-backend --limit=50
```

---


## Option 2: App Engine

**Best for**: Simple deployment, no Docker needed
**Cost**: ~$10-30/month
**Difficulty**: ⭐ Very Easy

### Step 1: Create app.yaml

Create `backend/app.yaml`:
```yaml
runtime: python311

env_variables:
  DATABASE_URL: "your-neon-url"

handlers:
- url: /.*
  script: auto

automatic_scaling:
  min_instances: 0
  max_instances: 5
  target_cpu_utilization: 0.65
```

### Step 2: Deploy Backend

```bash
cd backend

# Deploy
gcloud app deploy

# View logs
gcloud app logs tail -s default

# Get URL
gcloud app browse
```

**Backend URL**: https://YOUR_PROJECT_ID.appspot.com

### Step 3: Deploy Frontend

Same as Cloud Run option - use Cloud Storage + CDN.

### App Engine vs Cloud Run

| Feature | App Engine | Cloud Run |
|---------|-----------|-----------|
| Deployment | Easier | Requires Docker |
| Cold start | Slower | Faster |
| Pricing | More expensive | Cheaper |
| Flexibility | Less | More |
| Free tier | Limited | Better |

---

## Option 3: Compute Engine (VM)

**Best for**: Full control, traditional hosting
**Cost**: FREE f1-micro or $10-30/month
**Difficulty**: ⭐⭐⭐ Medium

### Step 1: Create VM Instance

```bash
# Create firewall rule
gcloud compute firewall-rules create allow-http \
  --allow tcp:80,tcp:443,tcp:8000 \
  --source-ranges 0.0.0.0/0 \
  --target-tags http-server

# Create instance (f1-micro is free tier)
gcloud compute instances create todo-server \
  --machine-type=f1-micro \
  --zone=us-central1-a \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=10GB \
  --tags=http-server

# Get external IP
gcloud compute instances describe todo-server \
  --zone=us-central1-a \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)'
```

### Step 2: SSH and Setup

```bash
# SSH into instance
gcloud compute ssh todo-server --zone=us-central1-a

# Update system
sudo apt update && sudo apt upgrade -y

# Install dependencies
sudo apt install python3 python3-pip python3-venv nginx nodejs npm -y

# Install PM2
sudo npm install -g pm2
```

### Step 3: Deploy Application

```bash
# Clone repository
cd /home/$USER
git clone https://github.com/your-username/your-repo.git todo-app
cd todo-app

# Setup backend
cd backend
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Create .env
nano .env
# Add DATABASE_URL

# Start with PM2
pm2 start "uvicorn main:app --host 0.0.0.0 --port 8000" --name todo-backend
pm2 save
pm2 startup

# Build frontend
cd ..
npm install
npm run build

# Copy to nginx
sudo cp -r dist/* /var/www/html/
```

### Step 4: Configure Nginx

```bash
sudo nano /etc/nginx/sites-available/todo-app
```

Add:
```nginx
server {
    listen 80;
    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/todo-app /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

### Step 5: Setup SSL with Let's Encrypt

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Get certificate (requires domain)
sudo certbot --nginx -d yourdomain.com
```

### Compute Engine Management

**Start/Stop instance:**
```bash
gcloud compute instances stop todo-server --zone=us-central1-a
gcloud compute instances start todo-server --zone=us-central1-a
```

**Create snapshot (backup):**
```bash
gcloud compute disks snapshot todo-server \
  --snapshot-names=todo-backup-$(date +%Y%m%d) \
  --zone=us-central1-a
```

**Resize instance:**
```bash
gcloud compute instances set-machine-type todo-server \
  --machine-type=e2-small \
  --zone=us-central1-a
```

---


## Option 4: GKE (Kubernetes)

**Best for**: Microservices, large scale, learning K8s
**Cost**: ~$70+/month (cluster costs)
**Difficulty**: ⭐⭐⭐⭐⭐ Advanced

### Step 1: Create GKE Cluster

```bash
# Create cluster (autopilot mode - easier)
gcloud container clusters create-auto todo-cluster \
  --region=us-central1

# Get credentials
gcloud container clusters get-credentials todo-cluster \
  --region=us-central1

# Verify
kubectl get nodes
```

### Step 2: Create Kubernetes Manifests

Create `k8s/backend-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: todo-backend
  template:
    metadata:
      labels:
        app: todo-backend
    spec:
      containers:
      - name: backend
        image: gcr.io/YOUR_PROJECT_ID/todo-backend:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
---
apiVersion: v1
kind: Service
metadata:
  name: todo-backend-service
spec:
  type: LoadBalancer
  selector:
    app: todo-backend
  ports:
  - port: 80
    targetPort: 8000
```

Create `k8s/secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  url: "your-neon-database-url"
```

### Step 3: Deploy to GKE

```bash
# Build and push image
cd backend
gcloud builds submit --tag gcr.io/YOUR_PROJECT_ID/todo-backend

# Create secret
kubectl apply -f k8s/secret.yaml

# Deploy backend
kubectl apply -f k8s/backend-deployment.yaml

# Get external IP
kubectl get service todo-backend-service
```

### Step 4: Deploy Frontend

Create `k8s/frontend-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: todo-frontend
  template:
    metadata:
      labels:
        app: todo-frontend
    spec:
      containers:
      - name: frontend
        image: gcr.io/YOUR_PROJECT_ID/todo-frontend:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: todo-frontend-service
spec:
  type: LoadBalancer
  selector:
    app: todo-frontend
  ports:
  - port: 80
    targetPort: 80
```

Build frontend Docker image:
```dockerfile
# Dockerfile in root
FROM node:18-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Deploy:
```bash
# Build frontend
gcloud builds submit --tag gcr.io/YOUR_PROJECT_ID/todo-frontend

# Deploy
kubectl apply -f k8s/frontend-deployment.yaml

# Get IP
kubectl get service todo-frontend-service
```

### GKE Management

**Scale deployment:**
```bash
kubectl scale deployment todo-backend --replicas=5
```

**Update image:**
```bash
kubectl set image deployment/todo-backend backend=gcr.io/YOUR_PROJECT_ID/todo-backend:v2
```

**View logs:**
```bash
kubectl logs -l app=todo-backend --tail=100
```

**Delete cluster:**
```bash
gcloud container clusters delete todo-cluster --region=us-central1
```

---


## Cost Comparison

### Monthly Costs (Estimated)

#### Option 1: Cloud Run + Cloud Storage
```
Cloud Run:
- First 2M requests: FREE
- After: $0.40 per million requests
- CPU: $0.00002400 per vCPU-second
- Memory: $0.00000250 per GiB-second

Cloud Storage:
- First 5GB: FREE
- After: $0.020 per GB/month

Estimated: $0-5/month for small apps
```

#### Option 2: App Engine
```
App Engine Standard:
- First 28 instance hours/day: FREE
- After: $0.05 per instance hour

Estimated: $10-30/month
```

#### Option 3: Compute Engine
```
f1-micro (FREE tier):
- 1 instance in US regions: FREE
- 30GB storage: FREE

e2-micro (paid):
- $6.11/month (preemptible)
- $7.11/month (standard)

e2-small:
- $13.23/month

Estimated: $0-30/month
```

#### Option 4: GKE
```
GKE Autopilot:
- $0.10 per vCPU/hour
- $0.011 per GB memory/hour
- Minimum ~$70/month

GKE Standard:
- Cluster management: $0.10/hour ($73/month)
- Plus node costs

Estimated: $70-200/month
```

### With Cloud SQL (if not using Neon)
```
db-f1-micro: $7.67/month
db-g1-small: $25/month
db-n1-standard-1: $50/month

Storage: $0.17 per GB/month
Backups: $0.08 per GB/month
```

### Data Transfer Costs
```
First 1GB/month: FREE
1-10TB: $0.12 per GB
10-150TB: $0.11 per GB

Within same region: FREE
```

---

## Complete Cost Examples

### Example 1: Todo App (Low Traffic)
**Setup**: Cloud Run + Cloud Storage + Neon
```
Cloud Run: $0 (within free tier)
Cloud Storage: $0 (within free tier)
Neon Database: $0 (free tier)
Data Transfer: $0 (minimal)

Total: $0/month ✅
```

### Example 2: Small Business App
**Setup**: Cloud Run + Cloud Storage + Cloud SQL
```
Cloud Run: $5 (moderate traffic)
Cloud Storage: $1
Cloud SQL (db-f1-micro): $8
Data Transfer: $2

Total: $16/month
```

### Example 3: Production App
**Setup**: GKE + Cloud SQL + Cloud CDN
```
GKE Autopilot: $100
Cloud SQL (db-n1-standard-1): $50
Cloud Storage + CDN: $10
Data Transfer: $20

Total: $180/month
```

---

## CI/CD with Cloud Build

### Create cloudbuild.yaml

Create `cloudbuild.yaml` in root:
```yaml
steps:
  # Build backend
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/todo-backend', './backend']
  
  # Push backend image
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/todo-backend']
  
  # Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'todo-backend'
      - '--image'
      - 'gcr.io/$PROJECT_ID/todo-backend'
      - '--region'
      - 'us-central1'
      - '--platform'
      - 'managed'
      - '--allow-unauthenticated'
  
  # Build frontend
  - name: 'node:18'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        npm install
        npm run build
  
  # Deploy frontend to Cloud Storage
  - name: 'gcr.io/cloud-builders/gsutil'
    args: ['-m', 'rsync', '-r', '-d', 'dist/', 'gs://your-todo-app-frontend/']

timeout: '1200s'
```

### Setup Cloud Build Trigger

```bash
# Connect GitHub repository
gcloud builds triggers create github \
  --repo-name=your-repo \
  --repo-owner=your-username \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml

# Manual trigger
gcloud builds submit --config=cloudbuild.yaml
```

### GitHub Actions Alternative

Create `.github/workflows/gcp-deploy.yml`:
```yaml
name: Deploy to GCP

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v2
      
      - name: Setup Cloud SDK
        uses: google-github-actions/setup-gcloud@v0
        with:
          service_account_key: ${{ secrets.GCP_SA_KEY }}
          project_id: ${{ secrets.GCP_PROJECT_ID }}
      
      - name: Configure Docker
        run: gcloud auth configure-docker
      
      - name: Build and Push Backend
        run: |
          cd backend
          docker build -t gcr.io/${{ secrets.GCP_PROJECT_ID }}/todo-backend .
          docker push gcr.io/${{ secrets.GCP_PROJECT_ID }}/todo-backend
      
      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy todo-backend \
            --image gcr.io/${{ secrets.GCP_PROJECT_ID }}/todo-backend \
            --platform managed \
            --region us-central1 \
            --allow-unauthenticated
      
      - name: Build Frontend
        run: |
          npm install
          npm run build
      
      - name: Deploy Frontend
        run: |
          gsutil -m rsync -r -d dist/ gs://your-todo-app-frontend/
```

---


## Monitoring & Logging

### Cloud Logging

**View logs:**
```bash
# Cloud Run logs
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=todo-backend" --limit=50

# Compute Engine logs
gcloud logging read "resource.type=gce_instance" --limit=50

# Filter by severity
gcloud logging read "severity>=ERROR" --limit=20
```

### Cloud Monitoring

**Create uptime check:**
```bash
gcloud monitoring uptime-checks create todo-backend-check \
  --display-name="Todo Backend Uptime" \
  --resource-type=uptime-url \
  --monitored-resource=https://todo-backend-xxxxx-uc.a.run.app/api/todos
```

**Create alert policy:**
```bash
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="High Error Rate" \
  --condition-display-name="Error rate > 5%" \
  --condition-threshold-value=5 \
  --condition-threshold-duration=300s
```

### Cloud Trace

Enable tracing in `backend/main.py`:
```python
from google.cloud import trace_v1

# Add to FastAPI app
@app.middleware("http")
async def add_trace(request: Request, call_next):
    tracer = trace_v1.TraceServiceClient()
    # Trace logic here
    response = await call_next(request)
    return response
```

---

## Security Best Practices

### 1. Use Secret Manager

```bash
# Create secret
echo -n "your-database-url" | gcloud secrets create db-url --data-file=-

# Grant access to Cloud Run
gcloud secrets add-iam-policy-binding db-url \
  --member="serviceAccount:YOUR_PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Use in Cloud Run
gcloud run deploy todo-backend \
  --image gcr.io/YOUR_PROJECT_ID/todo-backend \
  --update-secrets=DATABASE_URL=db-url:latest
```

### 2. Enable Cloud Armor (DDoS Protection)

```bash
# Create security policy
gcloud compute security-policies create todo-security-policy \
  --description "Security policy for todo app"

# Add rate limiting rule
gcloud compute security-policies rules create 1000 \
  --security-policy todo-security-policy \
  --expression "true" \
  --action "rate-based-ban" \
  --rate-limit-threshold-count 100 \
  --rate-limit-threshold-interval-sec 60 \
  --ban-duration-sec 600
```

### 3. Enable VPC Service Controls

```bash
# Create service perimeter
gcloud access-context-manager perimeters create todo-perimeter \
  --title="Todo App Perimeter" \
  --resources=projects/YOUR_PROJECT_NUMBER \
  --restricted-services=storage.googleapis.com,run.googleapis.com
```

### 4. IAM Best Practices

```bash
# Create custom service account
gcloud iam service-accounts create todo-backend-sa \
  --display-name="Todo Backend Service Account"

# Grant minimal permissions
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:todo-backend-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

# Use in Cloud Run
gcloud run deploy todo-backend \
  --service-account=todo-backend-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

---

## Terraform Configuration

Create `terraform/gcp-main.tf`:
```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 4.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# Cloud Storage bucket for frontend
resource "google_storage_bucket" "frontend" {
  name          = "${var.project_id}-frontend"
  location      = var.region
  force_destroy = true

  website {
    main_page_suffix = "index.html"
    not_found_page   = "index.html"
  }

  uniform_bucket_level_access = true
}

# Make bucket public
resource "google_storage_bucket_iam_member" "public_read" {
  bucket = google_storage_bucket.frontend.name
  role   = "roles/storage.objectViewer"
  member = "allUsers"
}

# Cloud Run service for backend
resource "google_cloud_run_service" "backend" {
  name     = "todo-backend"
  location = var.region

  template {
    spec {
      containers {
        image = "gcr.io/${var.project_id}/todo-backend:latest"
        
        env {
          name  = "DATABASE_URL"
          value = var.database_url
        }

        resources {
          limits = {
            cpu    = "1000m"
            memory = "512Mi"
          }
        }
      }
    }

    metadata {
      annotations = {
        "autoscaling.knative.dev/minScale" = "0"
        "autoscaling.knative.dev/maxScale" = "10"
      }
    }
  }

  traffic {
    percent         = 100
    latest_revision = true
  }
}

# Allow unauthenticated access
resource "google_cloud_run_service_iam_member" "public_access" {
  service  = google_cloud_run_service.backend.name
  location = google_cloud_run_service.backend.location
  role     = "roles/run.invoker"
  member   = "allUsers"
}

# Outputs
output "backend_url" {
  value = google_cloud_run_service.backend.status[0].url
}

output "frontend_bucket" {
  value = google_storage_bucket.frontend.url
}

# Variables
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "region" {
  description = "GCP Region"
  type        = string
  default     = "us-central1"
}

variable "database_url" {
  description = "Database connection URL"
  type        = string
  sensitive   = true
}
```

Deploy with Terraform:
```bash
cd terraform
terraform init
terraform plan -var="project_id=YOUR_PROJECT_ID" -var="database_url=your-neon-url"
terraform apply -var="project_id=YOUR_PROJECT_ID" -var="database_url=your-neon-url"
```

---

