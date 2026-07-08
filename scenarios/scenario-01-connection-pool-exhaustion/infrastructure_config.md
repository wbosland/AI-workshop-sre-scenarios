# Infrastructure Configuration

## Database Configuration

### Connection Pool Settings (HikariCP)

**File:** `config/database.properties`

```properties
# Connection Pool Configuration (as of v2.4.1)
datasource.hikari.maximum-pool-size=50
datasource.hikari.minimum-idle-connections=5
datasource.hikari.connection-timeout=30000
datasource.hikari.idle-timeout=300000
datasource.hikari.max-lifetime=900000
datasource.hikari.leak-detection-threshold=60000
```

**Parameters:**
- `maximum-pool-size`: 50 connections max
- `minimum-idle-connections`: Keep 5 idle at minimum
- `connection-timeout`: Wait max 30 seconds for a connection
- `idle-timeout`: Close connections idle for 300s (5 min)
- `max-lifetime`: Close connections after 900s (15 min) regardless
- `leak-detection-threshold`: Log warning if connection held > 60s

**Issue:** With connection leak, `leak-detection-threshold` should have triggered warnings at 14:32, but the threshold may have been missed due to retry storms overwhelming the logs.

---

## Payment Service Deployment

### Instances

```yaml
payment-api-prod-1:
  environment: production
  region: us-east-1
  instance_type: t3.xlarge
  threads:
    core_threads: 20
    max_threads: 50
    queue_size: unbounded
  
payment-api-prod-2:
  environment: production
  region: us-east-1
  instance_type: t3.xlarge
  threads:
    core_threads: 20
    max_threads: 50
    queue_size: unbounded
    
payment-api-prod-3:
  environment: production
  region: us-east-1
  instance_type: t3.xlarge
  threads:
    core_threads: 20
    max_threads: 50
    queue_size: unbounded
    
payment-api-prod-4:
  environment: production
  region: us-east-1
  instance_type: t3.xlarge
  threads:
    core_threads: 20
    max_threads: 50
    queue_size: unbounded
```

### Database Setup

```yaml
Database:
  engine: PostgreSQL 14
  instance: db.r5.2xlarge
  max_connections: 200
  shared_buffers: 16GB
  effective_cache_size: 64GB
  
Connection Distribution:
  payment-api-prod-1: 50 connections
  payment-api-prod-2: 50 connections
  payment-api-prod-3: 50 connections
  payment-api-prod-4: 50 connections
  monitoring/backup: 0-5 connections
  headroom: 0-5 connections
  
  Total allocated: 200 connections max
  Total usable: ~195 connections
```

---

## Load Balancer Configuration

```yaml
Load Balancer: AWS ALB
Target Group: payment-api-prod
Health Check:
  interval: 30s
  timeout: 5s
  healthy_threshold: 2
  unhealthy_threshold: 2
  path: /health
  protocol: HTTP
  port: 8080
```

**Issue:** Health check endpoint doesn't stress the connection pool, so it passed even though the application was failing under load.

---

## Monitoring and Alerting

### Key Metrics (Prometheus)

```yaml
Metrics Scraped:
  - hikari_connections_active
  - hikari_connections_idle
  - hikari_connections_pending
  - http_requests_total
  - http_request_duration_seconds
  - payment_transactions_total
  - payment_retry_attempts
  - payment_retry_exhausted_total
```

### Alert Rules

```yaml
Alerts:
  - name: DatabaseConnectionPoolExhausted
    expr: hikari_connections_active{pool="payment",service="payment-api"} >= 48
    for: 30s
    severity: critical
    
  - name: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
    for: 1m
    severity: critical
    
  - name: ApiResponseTimeHigh
    expr: histogram_quantile(0.99, http_request_duration_seconds) > 2
    for: 5m
    severity: warning
    
  - name: PaymentRetryRateHigh
    expr: rate(payment_retry_attempts[5m]) > 500
    for: 2m
    severity: warning
```

**Issue:** Alert for connection pool (48/50) triggered at 14:32, but with only 30s delay, the incident cascaded before remediation.

---

## Deployment Pipeline

### Version 2.4.0 (Previous - Stable)

```yaml
Deployment: 2024-07-06 08:00 UTC
Version: v2.4.0
Status: Stable (running for 30+ hours)
Retry Logic: 3 attempts, fixed 1s delay
Connection Pool: Original config
```

### Version 2.4.1 (Failed Release)

```yaml
Deployment: 2024-07-08 13:52 UTC
Version: v2.4.1
Status: Failed in production (45 min)
Changes:
  - New exponential backoff retry logic (5 attempts)
  - Aggressive connection pool timeout settings
  - Enhanced error logging
Issue: Connection leak in retry handler
Resolution: Rolled back to v2.4.0 at 15:12 UTC
```

---

## Pre-Deployment Checklist (What Should Have Caught This)

- [ ] **Load Testing:** v2.4.1 should have been tested with 2x traffic before release
- [ ] **Connection Pool Stress Test:** Should have simulated connection failures
- [ ] **Code Review:** Connection management in retry logic should have been flagged
- [ ] **Canary Deployment:** Should have deployed to 1 instance first, monitored for 30+ min
- [ ] **Enhanced Health Checks:** Should have included endpoint that stresses connection pool

---

## Questions for Improvement

1. **Why was the connection leak not caught in code review?**
2. **Why wasn't v2.4.1 canary-deployed to 1 instance first?**
3. **Why didn't load testing catch the issue?**
4. **Can we add better pre-deployment validation?**
5. **Should we reduce the threshold for connection pool exhaustion alert?**
6. **Should we implement circuit breakers to fail fast rather than retry indefinitely?**