# PulseLink Scalability Strategy

**Enterprise-Scale Architecture for 10M+ DAU**

---

## 1. Horizontal Scaling Architecture

### 1.1 Stateless Microservices (Kubernetes)

All backend services are stateless and horizontally scalable:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 3  # Initial replicas
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
      containers:
      - name: api
        image: gcr.io/pulselink/api:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 2Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 3
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
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

### 1.2 Load Balancing Strategy

```nginx
# NGINX configuration for API routing
upstream api_backend {
    least_conn;  # Least connections algorithm
    server api-1:8000 max_fails=3 fail_timeout=30s;
    server api-2:8000 max_fails=3 fail_timeout=30s;
    server api-3:8000 max_fails=3 fail_timeout=30s;
    # More servers auto-added by Kubernetes
}

upstream ws_backend {
    # WebSocket requires sticky sessions
    hash $http_x_forwarded_for consistent;
    server ws-1:8000 max_fails=3 fail_timeout=30s;
    server ws-2:8000 max_fails=3 fail_timeout=30s;
    server ws-3:8000 max_fails=3 fail_timeout=30s;
}

server {
    listen 443 ssl http2;
    server_name api.pulselink.app;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;
    limit_req zone=api_limit burst=200 nodelay;
    
    location /api/ {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 5s;
        proxy_send_timeout 10s;
        proxy_read_timeout 10s;
    }
    
    location /ws/ {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # WebSocket timeouts (longer)
        proxy_connect_timeout 5s;
        proxy_send_timeout 3600s;
        proxy_read_timeout 3600s;
    }
}
```

---

## 2. Database Scaling

### 2.1 PostgreSQL Sharding Strategy

```python
# Database sharding by region and user ID
class ShardingService:
    SHARDS = {
        'us-east': {'host': 'db-us-east-1.internal', 'port': 5432},
        'us-west': {'host': 'db-us-west-1.internal', 'port': 5432},
        'eu-west': {'host': 'db-eu-west-1.internal', 'port': 5432},
        'asia-pac': {'host': 'db-ap-southeast-1.internal', 'port': 5432},
    }
    
    @staticmethod
    def get_shard_for_user(user_id: str, region: str = 'us-east') -> str:
        """Determine which shard a user belongs to"""
        # Primary key: region
        # Secondary: user_id hash for distribution within region
        user_hash = int(user_id.replace('-', ''), 16)
        return SHARDS[region]
    
    @staticmethod
    def get_connection(user_id: str, region: str = 'us-east'):
        """Get database connection for user's shard"""
        shard_config = ShardingService.get_shard_for_user(user_id, region)
        return asyncpg.create_pool(
            host=shard_config['host'],
            port=shard_config['port'],
            database='pulselink',
            min_size=10,
            max_size=100,
            ssl=True
        )

# Usage
async def get_user_location(user_id: str, region: str = 'us-east'):
    pool = await ShardingService.get_connection(user_id, region)
    async with pool.acquire() as conn:
        location = await conn.fetchrow(
            'SELECT * FROM locations WHERE user_id = $1 ORDER BY created_at DESC LIMIT 1',
            user_id
        )
        return location
```

### 2.2 Connection Pooling

```python
# PgBouncer configuration for connection pooling
# pgbouncer.ini

[databases]
pulselink = host=db-primary.internal port=5432 dbname=pulselink

[pgbouncer]
pool_mode = transaction  # Transaction pooling for best compatibility
max_client_conn = 10000
default_pool_size = 25
min_pool_size = 10
reserve_pool_size = 5
reserve_pool_timeout = 3
max_db_connections = 1000
max_user_connections = 500
server_lifetime = 3600
server_idle_timeout = 600
query_timeout = 30000

# Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1

# Statistics
stats_period = 60
```

### 2.3 Read Replicas & Failover

```yaml
# Kubernetes StatefulSet for PostgreSQL replicas
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
spec:
  serviceName: postgresql
  replicas: 3  # Primary + 2 replicas
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
        role: primary  # or replica
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: pulselink
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: 2000m
            memory: 4Gi
          limits:
            cpu: 4000m
            memory: 8Gi
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 500Gi
```

---

## 3. Caching Strategy

### 3.1 Redis Cluster Setup

```python
# Redis clustering for distributed caching
from redis.cluster import RedisCluster

class CacheService:
    def __init__(self):
        self.redis = RedisCluster(
            startup_nodes=[
                {"host": "redis-1.internal", "port": 6379},
                {"host": "redis-2.internal", "port": 6379},
                {"host": "redis-3.internal", "port": 6379},
            ],
            decode_responses=True,
            skip_full_coverage_check=True,
            max_connections=500,
        )
    
    async def get_user_location(self, user_id: str) -> dict:
        """Get user location from cache, fallback to DB"""
        # Try cache first
        cached = await self.redis.get(f'location:{user_id}:current')
        if cached:
            return json.loads(cached)
        
        # Fallback to database
        location = await self.db.get_user_location(user_id)
        
        # Update cache (5 min TTL)
        await self.redis.setex(
            f'location:{user_id}:current',
            300,
            json.dumps(location)
        )
        
        return location
    
    async def publish_location_update(self, user_id: str, location: dict):
        """Publish location update to all subscribers"""
        await self.redis.publish(
            f'location_updates:{user_id}',
            json.dumps(location)
        )
```

### 3.2 Cache Invalidation Strategy

```python
class CacheInvalidation:
    async def invalidate_user_cache(self, user_id: str):
        """Invalidate all user-related caches"""
        keys_to_delete = [
            f'location:{user_id}:*',
            f'friends:user:{user_id}',
            f'consent:user:{user_id}:*',
            f'geofences:user:{user_id}',
        ]
        
        for pattern in keys_to_delete:
            cursor = 0
            while True:
                cursor, keys = await self.redis.scan(
                    cursor, match=pattern, count=1000
                )
                if keys:
                    await self.redis.delete(*keys)
                if cursor == 0:
                    break
    
    async def invalidate_friendship_cache(self, user_id_1: str, user_id_2: str):
        """Invalidate friendship-related caches"""
        await self.redis.delete(
            f'friends:user:{user_id_1}',
            f'friends:user:{user_id_2}',
        )
```

---

## 4. Location Data Optimization

### 4.1 Time-Series Partitioning

```sql
-- Automatic partition creation for locations
CREATE OR REPLACE FUNCTION create_monthly_location_partition()
RETURNS void AS $$
DECLARE
    partition_date DATE;
    partition_name TEXT;
    next_partition_date DATE;
BEGIN
    FOR i IN 0..12 LOOP
        partition_date := DATE_TRUNC('month', NOW()) + (i || ' months')::INTERVAL;
        partition_name := 'locations_' || TO_CHAR(partition_date, 'YYYYMM');
        next_partition_date := partition_date + INTERVAL '1 month';
        
        EXECUTE format(
            'CREATE TABLE IF NOT EXISTS %I PARTITION OF locations
             FOR VALUES FROM (%L) TO (%L)',
            partition_name,
            partition_date,
            next_partition_date
        );
        
        -- Create indexes on partition
        EXECUTE format(
            'CREATE INDEX IF NOT EXISTS idx_%s_gist ON %I USING GIST(location)',
            partition_name || '_gist',
            partition_name
        );
        
        EXECUTE format(
            'CREATE INDEX IF NOT EXISTS idx_%s_user ON %I (user_id)',
            partition_name || '_user',
            partition_name
        );
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Schedule partition creation
SELECT cron.schedule(
    'create_monthly_partitions',
    '0 0 1 * *',  -- First day of month
    'SELECT create_monthly_location_partition()'
);
```

### 4.2 Geospatial Query Optimization

```python
# Optimized geospatial queries with proper indexing
class GeospatialService:
    async def find_nearby_users(
        self,
        user_id: str,
        latitude: float,
        longitude: float,
        radius_meters: int = 1000,
        limit: int = 20
    ) -> List[dict]:
        """Find nearby users efficiently"""
        
        # Use GIST index for spatial search
        query = """
        SELECT 
            u.id,
            u.username,
            l.location,
            ST_Distance(l.location, ST_MakePoint($3, $2)::geography) as distance_meters
        FROM locations l
        JOIN users u ON l.user_id = u.id
        WHERE l.user_id IN (
            SELECT CASE 
                WHEN user_id_1 = $1 THEN user_id_2 
                ELSE user_id_1 
            END
            FROM friendships 
            WHERE status = 'accepted' 
            AND ($1 = user_id_1 OR $1 = user_id_2)
        )
        AND ST_DWithin(
            l.location,
            ST_MakePoint($3, $2)::geography,
            $4
        )
        AND l.created_at > NOW() - INTERVAL '5 minutes'
        ORDER BY distance_meters ASC
        LIMIT $5
        """
        
        results = await self.db.fetch(
            query,
            user_id,
            latitude,
            longitude,
            radius_meters,
            limit
        )
        
        return results
```

---

## 5. WebSocket Scaling

### 5.1 Redis Pub/Sub for Cross-Pod Communication

```python
# WebSocket gateway with Redis pub/sub for clustering
class WebSocketGateway:
    def __init__(self):
        self.active_connections: Dict[str, WebSocket] = {}
        self.redis_pubsub = None
    
    async def startup(self):
        """Initialize Redis connection on startup"""
        self.redis_pubsub = await aioredis.create_redis_pool(
            'redis://redis-cluster:6379'
        )
    
    async def connect(self, user_id: str, websocket: WebSocket):
        """Accept WebSocket connection"""
        await websocket.accept()
        self.active_connections[user_id] = websocket
        
        # Subscribe to user's channel
        await self.subscribe_to_events(user_id)
    
    async def subscribe_to_events(self, user_id: str):
        """Subscribe to user events from Redis"""
        channel = f'user:{user_id}:events'
        (ch,) = await self.redis_pubsub.subscribe(channel)
        
        while await ch.wait_message():
            message = await ch.get_json()
            
            # Send to connected WebSocket
            if user_id in self.active_connections:
                try:
                    await self.active_connections[user_id].send_json(message)
                except Exception as e:
                    logger.error(f"Failed to send to {user_id}: {e}")
    
    async def broadcast_to_nearby_users(
        self,
        source_user_id: str,
        location: dict,
        target_user_ids: List[str]
    ):
        """Broadcast location to nearby users (Redis pub/sub)"""
        message = {
            'type': 'location_update',
            'user_id': source_user_id,
            'location': location,
            'timestamp': datetime.utcnow().isoformat()
        }
        
        # Publish to Redis for all pods
        for target_id in target_user_ids:
            await self.redis_pubsub.publish(
                f'user:{target_id}:events',
                json.dumps(message)
            )
```

---

## 6. Performance Monitoring

### 6.1 Prometheus Metrics

```python
from prometheus_client import Counter, Histogram, Gauge

# Define metrics
location_updates_total = Counter(
    'location_updates_total',
    'Total location updates',
    ['source']
)

location_update_latency = Histogram(
    'location_update_latency_seconds',
    'Location update latency',
    buckets=(0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0)
)

active_websocket_connections = Gauge(
    'active_websocket_connections',
    'Active WebSocket connections'
)

db_query_latency = Histogram(
    'db_query_latency_seconds',
    'Database query latency',
    ['operation', 'table']
)

# Usage
@app.post('/location/update')
async def update_location(request: LocationRequest):
    with db_query_latency.labels('insert', 'locations').time():
        await db.insert_location(request)
    
    location_updates_total.labels(source=request.source).inc()
```

### 6.2 Grafana Dashboards

**Key Metrics to Monitor:**
- API response times (P50, P95, P99)
- WebSocket latency
- Database query times
- Cache hit rates
- CPU/memory usage
- Error rates
- Active connections
- Location update throughput

---

## 7. Capacity Planning

### Scalability Estimates

| Metric | 1M Users | 10M Users | 100M Users |
|--------|----------|-----------|------------|
| API Pods | 5-10 | 30-50 | 100-200 |
| WebSocket Pods | 5-10 | 30-50 | 100-200 |
| Redis Nodes | 3 | 9-12 | 30+ |
| DB Shards | 1 | 4-8 | 16-32 |
| Monthly Cost | $50K | $200K | $500K+ |

### Growth Plan

1. **Months 1-3**: Single-region, 10-20 API pods, 1 DB shard
2. **Months 4-6**: Multi-region preparation, 50-100 API pods, 4 DB shards
3. **Months 7-12**: Global deployment, auto-scaling, 200+ pods, 16 DB shards
4. **Year 2+**: Edge computing, serverless functions, 500+ pods globally

---

**Next**: See [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md) for Docker & Kubernetes setup.
