# Complete Deployment Guide - All Scenarios

## Table of Contents
1. [Local Development Environment](#local-development-environment)
2. [Internal/Self-Hosted Deployment](#internalself-hosted-deployment)
3. [Cloud Deployment (Production)](#cloud-deployment-production)
4. [Environment Configuration](#environment-configuration)
5. [Best Practices](#best-practices)

---

## Local Development Environment

### Prerequisites
- Node.js 18+ installed
- Python 3.11+ installed
- Neon PostgreSQL database (or local PostgreSQL)

### Setup Steps

#### 1. Clone Repository
```bash
git clone https://github.com/your-username/your-repo.git
cd your-repo
```

#### 2. Backend Setup
```bash
cd backend

# Create virtual environment (recommended)
python -m venv venv

# Activate virtual environment
# Windows:
venv\Scripts\activate
# Mac/Linux:
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Create .env file
cp .env.example .env
# Edit .env and add your DATABASE_URL
```

#### 3. Frontend Setup
```bash
# From project root
npm install
```

#### 4. Run Development Servers

**Terminal 1 - Backend:**
```bash
cd backend
uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

**Terminal 2 - Frontend:**
```bash
npm run dev
```

Visit: http://localhost:5173

### Development Environment Variables

**backend/.env:**
```env
DATABASE_URL=postgresql://user:password@host.neon.tech/dbname?sslmode=require
```

**For local PostgreSQL:**
```env
DATABASE_URL=postgresql://postgres:password@localhost:5432/todo_db
```

---

## Internal/Self-Hosted Deployment

### Option 1: Docker Deployment (Recommended for Internal)

#### Prerequisites
- Docker and Docker Compose installed
- Server with Ubuntu/Debian/CentOS

#### Step 1: Create Docker Files

**backend/Dockerfile:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Expose port
EXPOSE 8000

# Run application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Dockerfile (Frontend - in root):**
```dockerfile
FROM node:18-alpine as build

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci

# Build app
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**nginx.conf (in root):**
```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**docker-compose.yml (in root):**
```yaml
version: '3.8'

services:
  backend:
    build: ./backend
    container_name: todo-backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=${DATABASE_URL}
    restart: unless-stopped
    networks:
      - todo-network

  frontend:
    build: .
    container_name: todo-frontend
    ports:
      - "80:80"
    depends_on:
      - backend
    restart: unless-stopped
    networks:
      - todo-network

networks:
  todo-network:
    driver: bridge
```

**.env (in root for docker-compose):**
```env
DATABASE_URL=postgresql://user:password@host.neon.tech/dbname?sslmode=require
```

#### Step 2: Deploy with Docker

```bash
# Build and start containers
docker-compose up -d

# View logs
docker-compose logs -f

# Stop containers
docker-compose down

# Rebuild after changes
docker-compose up -d --build
```

Access: http://your-server-ip

#### Step 3: Update Deployment

```bash
# Pull latest code
git pull origin main

# Rebuild and restart
docker-compose down
docker-compose up -d --build
```

### Option 2: Traditional Server Deployment (Ubuntu/Debian)

#### Prerequisites
- Ubuntu 20.04+ or Debian 11+
- Root or sudo access

#### Step 1: Install Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Python
sudo apt install python3 python3-pip python3-venv -y

# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y

# Install Nginx
sudo apt install nginx -y

# Install process manager
sudo npm install -g pm2
```

#### Step 2: Deploy Backend

```bash
# Clone repository
cd /var/www
sudo git clone https://github.com/your-username/your-repo.git todo-app
cd todo-app/backend

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Create .env file
sudo nano .env
# Add DATABASE_URL

# Start with PM2
pm2 start "uvicorn main:app --host 0.0.0.0 --port 8000" --name todo-backend
pm2 save
pm2 startup
```

#### Step 3: Deploy Frontend

```bash
# Build frontend
cd /var/www/todo-app
npm install
npm run build

# Copy to nginx directory
sudo cp -r dist/* /var/www/html/
```

#### Step 4: Configure Nginx

```bash
sudo nano /etc/nginx/sites-available/todo-app
```

**Nginx Configuration:**
```nginx
server {
    listen 80;
    server_name your-domain.com;  # or server IP

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/todo-app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

#### Step 5: Setup SSL (Optional but Recommended)

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Get SSL certificate
sudo certbot --nginx -d your-domain.com
```

#### Step 6: Update Deployment

```bash
# Pull latest code
cd /var/www/todo-app
git pull origin main

# Update backend
cd backend
source venv/bin/activate
pip install -r requirements.txt
pm2 restart todo-backend

# Update frontend
cd ..
npm install
npm run build
sudo cp -r dist/* /var/www/html/
```

---

## Cloud Deployment (Production)

### Option 1: Vercel (Frontend) + Render (Backend) - EASIEST

#### Backend on Render

1. **Sign up**: https://render.com
2. **Create Web Service**:
   - Connect GitHub repository
   - **Root Directory**: `backend`
   - **Build Command**: `pip install -r requirements.txt`
   - **Start Command**: `uvicorn main:app --host 0.0.0.0 --port $PORT`
3. **Environment Variables**:
   - `DATABASE_URL`: Your Neon connection string
   - `PYTHON_VERSION`: `3.11.0`
4. **Deploy** and copy URL

#### Frontend on Vercel

1. **Sign up**: https://vercel.com
2. **Import Project**:
   - Connect GitHub repository
   - **Framework**: Vite (auto-detected)
   - **Build Command**: `npm run build`
   - **Output Directory**: `dist`
3. **Update API URLs** in `src/App.jsx`:
   ```javascript
   const API_URL = 'https://your-backend.onrender.com';
   const response = await fetch(`${API_URL}/api/todos`);
   ```
4. **Deploy**

**Cost**: Free tier available for both

### Option 2: Railway (Full Stack) - SIMPLE

1. **Sign up**: https://railway.app
2. **New Project** → Deploy from GitHub
3. **Backend Service**:
   - Root Directory: `backend`
   - Start Command: `uvicorn main:app --host 0.0.0.0 --port $PORT`
   - Add `DATABASE_URL` variable
   - Generate domain
4. **Frontend Service**:
   - Root Directory: `./`
   - Build Command: `npm run build`
   - Start Command: `npm run preview -- --host 0.0.0.0 --port $PORT`
   - Update API URLs before deploying
5. **Deploy**

**Cost**: $5/month credit free, then pay-as-you-go

### Option 3: AWS (Production Grade) - ADVANCED

#### Prerequisites
- AWS Account
- AWS CLI installed
- Docker installed

#### Architecture
- **Frontend**: S3 + CloudFront
- **Backend**: ECS Fargate or EC2
- **Database**: RDS PostgreSQL or Neon

#### Backend on ECS

1. **Create ECR Repository**:
```bash
aws ecr create-repository --repository-name todo-backend
```

2. **Build and Push Docker Image**:
```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

# Build image
cd backend
docker build -t todo-backend .

# Tag image
docker tag todo-backend:latest YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/todo-backend:latest

# Push image
docker push YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/todo-backend:latest
```

3. **Create ECS Task Definition** (via AWS Console):
   - Container: Use ECR image
   - Port: 8000
   - Environment Variables: Add `DATABASE_URL`
   - CPU: 256, Memory: 512

4. **Create ECS Service**:
   - Launch Type: Fargate
   - Load Balancer: Application Load Balancer
   - Target Group: Port 8000

#### Frontend on S3 + CloudFront

1. **Build Frontend**:
```bash
npm run build
```

2. **Create S3 Bucket**:
```bash
aws s3 mb s3://your-todo-app-frontend
aws s3 website s3://your-todo-app-frontend --index-document index.html
```

3. **Upload Files**:
```bash
aws s3 sync dist/ s3://your-todo-app-frontend --acl public-read
```

4. **Create CloudFront Distribution**:
   - Origin: S3 bucket
   - Default Root Object: index.html
   - Error Pages: 404 → /index.html (for SPA routing)

**Cost**: ~$20-50/month depending on traffic

### Option 4: DigitalOcean App Platform - BALANCED

1. **Sign up**: https://www.digitalocean.com
2. **Create App**:
   - Connect GitHub repository
3. **Backend Component**:
   - Type: Web Service
   - Source Directory: `backend`
   - Build Command: `pip install -r requirements.txt`
   - Run Command: `uvicorn main:app --host 0.0.0.0 --port 8080`
   - Environment Variables: Add `DATABASE_URL`
4. **Frontend Component**:
   - Type: Static Site
   - Build Command: `npm run build`
   - Output Directory: `dist`
5. **Deploy**

**Cost**: $5/month for basic tier

---

## Environment Configuration

### Development (.env)
```env
# Backend
DATABASE_URL=postgresql://user:pass@localhost:5432/todo_dev
DEBUG=True
ENVIRONMENT=development
```

### Staging (.env.staging)
```env
# Backend
DATABASE_URL=postgresql://user:pass@staging-host/todo_staging
DEBUG=False
ENVIRONMENT=staging
ALLOWED_ORIGINS=https://staging.yourdomain.com
```

### Production (.env.production)
```env
# Backend
DATABASE_URL=postgresql://user:pass@prod-host/todo_prod
DEBUG=False
ENVIRONMENT=production
ALLOWED_ORIGINS=https://yourdomain.com
```

### Environment-Specific API URLs

**Option 1: Environment Variables (Vite)**

Create `.env.development`:
```env
VITE_API_URL=http://localhost:8000
```

Create `.env.production`:
```env
VITE_API_URL=https://your-backend.onrender.com
```

Update `src/App.jsx`:
```javascript
const API_URL = import.meta.env.VITE_API_URL;

const response = await fetch(`${API_URL}/api/todos`);
```

**Option 2: Config File**

Create `src/config.js`:
```javascript
const config = {
  development: {
    apiUrl: 'http://localhost:8000'
  },
  production: {
    apiUrl: 'https://your-backend.onrender.com'
  }
};

const environment = import.meta.env.MODE;
export default config[environment];
```

Use in `src/App.jsx`:
```javascript
import config from './config';

const response = await fetch(`${config.apiUrl}/api/todos`);
```

---

## Best Practices

### Security

1. **Never commit .env files**
   - Always use .env.example as template
   - Add .env to .gitignore

2. **Use HTTPS in production**
   - Get SSL certificate (Let's Encrypt is free)
   - Force HTTPS redirects

3. **Secure CORS**
   ```python
   # In production, specify exact origins
   app.add_middleware(
       CORSMiddleware,
       allow_origins=["https://yourdomain.com"],  # Not "*"
       allow_credentials=True,
       allow_methods=["*"],
       allow_headers=["*"],
   )
   ```

4. **Database Security**
   - Use strong passwords
   - Enable SSL mode
   - Restrict IP access
   - Regular backups

### Performance

1. **Frontend Optimization**
   ```bash
   # Build with optimizations
   npm run build
   
   # Analyze bundle size
   npm install -D rollup-plugin-visualizer
   ```

2. **Backend Optimization**
   - Use connection pooling
   - Add caching (Redis)
   - Enable gzip compression
   - Use CDN for static assets

3. **Database Optimization**
   - Add indexes on frequently queried columns
   - Use connection pooling
   - Monitor slow queries

### Monitoring

1. **Application Monitoring**
   - Sentry for error tracking
   - LogRocket for session replay
   - New Relic for APM

2. **Server Monitoring**
   - Uptime monitoring (UptimeRobot)
   - Performance monitoring (Datadog)
   - Log aggregation (Papertrail)

### CI/CD Pipeline

**GitHub Actions** (.github/workflows/deploy.yml):
```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Deploy to Render
        run: |
          curl -X POST ${{ secrets.RENDER_DEPLOY_HOOK }}

  deploy-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Deploy to Vercel
        run: |
          npm install -g vercel
          vercel --prod --token ${{ secrets.VERCEL_TOKEN }}
```

### Backup Strategy

1. **Database Backups**
   - Automated daily backups
   - Keep 30 days of backups
   - Test restore process monthly

2. **Code Backups**
   - Use Git (already done)
   - Multiple remotes (GitHub + GitLab)
   - Tag releases

### Scaling Strategy

1. **Horizontal Scaling**
   - Load balancer
   - Multiple backend instances
   - Database read replicas

2. **Vertical Scaling**
   - Increase server resources
   - Optimize queries
   - Add caching layer

---

## Quick Reference

### Local Development
```bash
# Backend
cd backend && uvicorn main:app --reload

# Frontend
npm run dev
```

### Docker Deployment
```bash
docker-compose up -d
```

### Cloud Deployment
- **Easiest**: Vercel + Render
- **Cheapest**: Railway ($5 credit)
- **Most Control**: AWS/DigitalOcean
- **Best for Teams**: DigitalOcean App Platform

### Update Deployment
```bash
git add .
git commit -m "Update"
git push origin main
# Auto-deploys on most platforms
```

---

## Troubleshooting

### Common Issues

**Issue**: CORS errors
**Solution**: Check backend CORS configuration and allowed origins

**Issue**: Database connection fails
**Solution**: Verify DATABASE_URL format and SSL mode

**Issue**: Frontend can't reach backend
**Solution**: Check API URL configuration and network settings

**Issue**: Build fails
**Solution**: Check Node/Python versions match requirements

### Getting Help

- Check platform documentation
- Review deployment logs
- Test locally first
- Use browser dev tools (F12)

---

## Summary

| Scenario | Best Option | Difficulty | Cost |
|----------|-------------|------------|------|
| Local Dev | Native setup | Easy | Free |
| Internal | Docker Compose | Medium | Server cost |
| Small Project | Vercel + Render | Easy | Free |
| Growing App | Railway | Easy | $5-20/mo |
| Production | DigitalOcean | Medium | $10-50/mo |
| Enterprise | AWS/GCP | Hard | $50+/mo |

Choose based on your needs, budget, and technical expertise!
