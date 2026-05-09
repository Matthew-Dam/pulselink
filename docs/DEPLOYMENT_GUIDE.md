# PulseLink Deployment Guide

**Production-Ready Docker, Kubernetes, and CI/CD Setup**

---

## 1. Local Development Setup

### 1.1 Docker Compose (All-in-One)

```yaml
# infrastructure/docker-compose.dev.yml
version: '3.8'

services:
  # PostgreSQL with PostGIS
  postgres:
    image: postgis/postgis:16-3.4
    container_name: pulselink-postgres
    environment:
      POSTGRES_DB: pulselink
      POSTGRES_USER: pulselink_dev
      POSTGRES_PASSWORD: dev_password_change_in_prod
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pulselink_dev"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - pulselink-net

  # Redis
  redis:
    image: redis:7-alpine
    container_name: pulselink-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes --requirepass redis_password_dev
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - pulselink-net

  # FastAPI Backend
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: pulselink-backend
    environment:
      DATABASE_URL: postgresql://pulselink_dev:dev_password_change_in_prod@postgres:5432/pulselink
      REDIS_URL: redis://:redis_password_dev@redis:6379/0
      ENVIRONMENT: development
      DEBUG: "true"
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/app
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    networks:
      - pulselink-net

volumes:
  postgres_data:
  redis_data:

networks:
  pulselink-net:
    driver: bridge
```

### 1.2 Quick Start

```bash
# Start all services
docker-compose -f infrastructure/docker-compose.dev.yml up -d

# Check status
docker-compose -f infrastructure/docker-compose.dev.yml ps

# View logs
docker-compose -f infrastructure/docker-compose.dev.yml logs -f backend

# Stop all services
docker-compose -f infrastructure/docker-compose.dev.yml down

# Remove volumes (full reset)
docker-compose -f infrastructure/docker-compose.dev.yml down -v
```

---

## 2. Production Deployment

### 2.1 Kubernetes Deployment

```yaml
# infrastructure/kubernetes/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: pulselink
---
# infrastructure/kubernetes/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: pulselink-secrets
  namespace: pulselink
type: Opaque
stringData:
  DATABASE_PASSWORD: $(SECURE_DB_PASSWORD)
  REDIS_PASSWORD: $(SECURE_REDIS_PASSWORD)
  JWT_SECRET_KEY: $(SECURE_JWT_KEY)
  ENCRYPTION_KEY: $(SECURE_ENCRYPTION_KEY)
---
# infrastructure/kubernetes/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pulselink-config
  namespace: pulselink
data:
  ENVIRONMENT: production
  LOG_LEVEL: INFO
  DATABASE_HOST: postgres.pulselink.svc.cluster.local
  DATABASE_PORT: "5432"
  DATABASE_NAME: pulselink
  REDIS_HOST: redis.pulselink.svc.cluster.local
  REDIS_PORT: "6379"
---
# infrastructure/kubernetes/api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: pulselink
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - api-service
              topologyKey: kubernetes.io/hostname
      containers:
      - name: api
        image: gcr.io/pulselink/api:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
          name: http
        envFrom:
        - configMapRef:
            name: pulselink-config
        - secretRef:
            name: pulselink-secrets
        resources:
          requests:
            cpu: 1000m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 2Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
---
# infrastructure/kubernetes/api-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: pulselink
  labels:
    app: api-service
spec:
  type: ClusterIP
  selector:
    app: api-service
  ports:
  - port: 80
    targetPort: 8000
    protocol: TCP
    name: http
  sessionAffinity: None
---
# infrastructure/kubernetes/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-service-hpa
  namespace: pulselink
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 5
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```

### 2.2 Ingress Configuration

```yaml
# infrastructure/kubernetes/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pulselink-ingress
  namespace: pulselink
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - api.pulselink.app
    secretName: pulselink-tls
  rules:
  - host: api.pulselink.app
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /ws
        pathType: Prefix
        backend:
          service:
            name: ws-service
            port:
              number: 80
```

---

## 3. CI/CD Pipeline

### 3.1 GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy PulseLink

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgis/postgis:16-3.4
        env:
          POSTGRES_DB: pulselink_test
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      working-directory: ./backend
      run: |
        pip install -r requirements.txt
        pip install pytest pytest-cov pytest-asyncio
    
    - name: Run tests
      working-directory: ./backend
      env:
        DATABASE_URL: postgresql://test_user:test_password@localhost:5432/pulselink_test
        REDIS_URL: redis://localhost:6379/0
      run: pytest tests/ -v --cov=app
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        files: ./backend/coverage.xml

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Login to GCR
      uses: docker/login-action@v2
      with:
        registry: gcr.io
        username: _json_key
        password: ${{ secrets.GCR_JSON_KEY }}
    
    - name: Build and push API image
      uses: docker/build-push-action@v4
      with:
        context: ./backend
        push: true
        tags: gcr.io/pulselink/api:${{ github.sha }}
        cache-from: type=registry,ref=gcr.io/pulselink/api:buildcache
        cache-to: type=registry,ref=gcr.io/pulselink/api:buildcache,mode=max

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'latest'
    
    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > kubeconfig.yaml
        export KUBECONFIG=kubeconfig.yaml
    
    - name: Deploy to staging
      run: |
        kubectl set image deployment/api-service \
          api=gcr.io/pulselink/api:${{ github.sha }} \
          -n pulselink
        kubectl rollout status deployment/api-service -n pulselink

  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && contains(github.event.head_commit.message, '[deploy-prod]')
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > kubeconfig.yaml
        export KUBECONFIG=kubeconfig.yaml
    
    - name: Deploy to production
      run: |
        kubectl set image deployment/api-service \
          api=gcr.io/pulselink/api:${{ github.sha }} \
          -n pulselink
        kubectl rollout status deployment/api-service -n pulselink
    
    - name: Verify deployment
      run: |
        kubectl get pods -n pulselink
        kubectl logs -n pulselink -l app=api-service --tail=20
```

---

## 4. Database Migrations

### 4.1 Alembic Setup

```bash
# Initialize migrations
cd backend
alembic init migrations

# Create migration
alembic revision --autogenerate -m "Add users table"

# Apply migrations
alembic upgrade head

# Rollback
alembic downgrade -1
```

### 4.2 Migration Best Practices

```python
# migrations/versions/001_initial.py
"""Initial schema

Revision ID: 001
Revises:
Create Date: 2026-05-09 12:00:00.000000
"""

from alembic import op
import sqlalchemy as sa

revision = '001'
down_revision = None
branch_labels = None
depends_on = None

def upgrade() -> None:
    # Create extension
    op.execute('CREATE EXTENSION IF NOT EXISTS postgis')
    op.execute('CREATE EXTENSION IF NOT EXISTS pg_trgm')
    
    # Create tables
    op.create_table(
        'users',
        sa.Column('id', sa.UUID(), nullable=False),
        sa.Column('email', sa.String(255), nullable=False, unique=True),
        sa.Column('username', sa.String(100), nullable=False, unique=True),
        sa.Column('password_hash', sa.String(255), nullable=False),
        sa.Column('created_at', sa.DateTime(timezone=True), nullable=False),
        sa.PrimaryKeyConstraint('id')
    )

def downgrade() -> None:
    op.drop_table('users')
```

---

## 5. Monitoring & Observability

### 5.1 Prometheus Setup

```yaml
# infrastructure/kubernetes/prometheus.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: pulselink
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    scrape_configs:
    - job_name: 'api-service'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
          - pulselink
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: api-service
      - source_labels: [__meta_kubernetes_pod_port_number]
        action: keep
        regex: "9090"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: pulselink
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
        - name: storage
          mountPath: /prometheus
      volumes:
      - name: config
        configMap:
          name: prometheus-config
      - name: storage
        emptyDir: {}
```

---

**Next**: See [AI_SYSTEM.md](AI_SYSTEM.md) for machine learning pipelines.
