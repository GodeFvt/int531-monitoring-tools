# Backend Alert Runbooks (Golden Signals)

This document provides troubleshooting steps and resolution procedures for backend application alerts following the Golden Signals methodology: **Traffic**, **Errors**, **Latency**, and **Saturation**.

---

## Table of Contents

### Traffic
- [HighRequestRate](#highrequestrate)
- [LowRequestRate](#lowrequestrate)

### Errors
- [HighErrorRate](#higherrorrate)
- [CriticalErrorRate](#criticalerrorrate)
- [BackendDbErrors](#backenderrors)

### Latency
- [HighLatencyP95](#highlowlatencyp95)
- [CriticalLatencyP95](#criticallatencyp95)
- [HighLatencyP99](#highlatencyp99)

### Saturation
- [HighActiveConnections](#highactiveconnections)

---

## Golden Signals Overview

**The Four Golden Signals:**
1. **Traffic** - How much demand is being placed on your system
2. **Errors** - The rate of requests that fail
3. **Latency** - How long it takes to serve requests
4. **Saturation** - How "full" your service is

---

# Traffic Signals

## HighRequestRate

**Severity:** Warning  
**Threshold:** Request rate > 1000 requests/minute for 5 minutes  
**Golden Signal:** Traffic

### Description
Backend is receiving an unusually high volume of requests, which may indicate a traffic spike, bot activity, or a potential DDoS attack.

### Impact
- Increased resource consumption
- Potential service degradation
- Risk of reaching capacity limits
- Higher infrastructure costs

### Diagnosis Steps

1. **Check current request rate:**
   ```promql
   sum(rate(http_requests_total[5m])) * 60
   ```

2. **Check per-endpoint breakdown:**
   ```promql
   sum by(endpoint) (rate(http_requests_total[5m])) * 60
   ```

3. **Identify request sources:**
   ```bash
   # Check access logs
   docker logs int531-demo-app-1 | grep "GET\|POST" | tail -100
   
   # Check for patterns
   docker logs int531-demo-app-1 | awk '{print $1}' | sort | uniq -c | sort -nr
   ```

4. **Check response times:**
   ```promql
   rate(http_request_duration_seconds_sum[5m]) / 
   rate(http_request_duration_seconds_count[5m])
   ```

5. **Verify system resources:**
   ```bash
   docker stats
   ```

### Resolution

1. **Verify if traffic is legitimate:**
   - Check for marketing campaigns
   - Verify with business team
   - Check analytics for user patterns

2. **If legitimate traffic:**
   ```bash
   # Scale horizontally
   docker compose up -d --scale app=3
   
   # Or increase container resources
   docker update --cpus="2" --memory="2g" int531-demo-app-1
   ```

3. **If suspicious/bot traffic:**
   ```bash
   # Implement rate limiting (in application code)
   # Block suspicious IPs (if using reverse proxy)
   # Enable CAPTCHA for suspicious patterns
   ```

4. **Optimize application:**
   - Enable caching for static content
   - Optimize database queries
   - Add CDN for static assets

5. **Monitor for impact:**
   ```promql
   # Check error rates
   rate(http_requests_total{status=~"5.."}[5m])
   
   # Check latency
   histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
   ```

### Prevention
- Implement rate limiting
- Use CDN and caching
- Set up auto-scaling
- Configure DDoS protection
- Regular load testing

---

## LowRequestRate

**Severity:** Warning  
**Threshold:** Request rate < 10 requests/minute for 10 minutes  
**Golden Signal:** Traffic

### Description
Backend is receiving significantly fewer requests than normal, which may indicate a problem with upstream services, network issues, or user-facing errors.

### Impact
- Potential revenue loss
- User dissatisfaction
- Indicates upstream problems
- May mask other issues

### Diagnosis Steps

1. **Verify request rate:**
   ```promql
   sum(rate(http_requests_total[5m])) * 60
   ```

2. **Check if service is accessible:**
   ```bash
   # Test locally
   curl -I http://localhost:3100/
   curl -I http://localhost:3100/api/students
   
   # Test from outside
   curl -I http://<public_ip>:3100/
   ```

3. **Check upstream services:**
   ```bash
   # Check if reverse proxy/load balancer is running
   # Check DNS resolution
   # Check network connectivity
   ```

4. **Check error rates:**
   ```promql
   rate(http_requests_total{status=~"5.."}[5m])
   ```

5. **Check application logs:**
   ```bash
   docker logs int531-demo-app-1 --tail 100
   ```

### Resolution

1. **If service is down/unreachable:**
   ```bash
   # Restart service
   docker compose restart app
   
   # Check health
   docker ps
   curl http://localhost:3100/api/students
   ```

2. **If upstream issues:**
   - Verify load balancer configuration
   - Check DNS records
   - Verify firewall rules
   - Check reverse proxy logs

3. **If frontend issues:**
   ```bash
   # Check frontend logs
   # Verify API endpoint URLs
   # Check CORS configuration
   ```

4. **Check for scheduled maintenance:**
   - Verify no maintenance windows active
   - Check for deployments in progress

5. **Notify relevant teams:**
   - Alert frontend team
   - Check with operations team
   - Verify with monitoring team

### Prevention
- Implement health checks
- Set up synthetic monitoring
- Configure alerting for frontend
- Regular endpoint testing
- Proper maintenance communication

---

# Error Signals

## HighErrorRate

**Severity:** Warning  
**Threshold:** Error rate (5xx) > 5% for 5 minutes  
**Golden Signal:** Errors

### Description
Backend is returning HTTP 5xx errors at an elevated rate, indicating server-side issues affecting user requests.

### Impact
- User-facing errors
- Data operations failing
- Degraded user experience
- Potential data integrity issues

### Diagnosis Steps

1. **Check error rate:**
   ```promql
   sum(rate(http_requests_total{status=~"5.."}[5m])) /
   sum(rate(http_requests_total[5m])) * 100
   ```

2. **Identify error types:**
   ```promql
   sum by(status) (rate(http_requests_total{status=~"5.."}[5m]))
   ```

3. **Check which endpoints are failing:**
   ```promql
   sum by(endpoint, status) (rate(http_requests_total{status=~"5.."}[5m]))
   ```

4. **Check application logs:**
   ```bash
   docker logs int531-demo-app-1 --tail 200 | grep -i "error\|exception\|fail"
   ```

5. **Check error details:**
   ```bash
   # Trigger test error endpoint
   curl http://localhost:3100/api/mock-error
   
   # Check response
   docker logs int531-demo-app-1 --tail 50
   ```

### Resolution

1. **For database connection errors:**
   ```bash
   # Check database connectivity
   docker exec int531-demo-mariadb-1 mariadb -u root -prootpassword -e "SELECT 1"
   
   # Restart database if needed
   docker compose restart mariadb
   
   # Check connection pool
   # Verify DATABASE_URL configuration
   ```

2. **For application errors:**
   ```bash
   # Check application health
   docker logs int531-demo-app-1 --tail 100
   
   # Restart application
   docker compose restart app
   
   # Check for recent deployments
   git log -1
   ```

3. **For resource exhaustion:**
   ```bash
   # Check resources
   docker stats int531-demo-app-1
   
   # Increase if needed
   docker update --memory="1g" int531-demo-app-1
   ```

4. **For specific error codes:**
   - **500 Internal Server Error:** Application bug, check logs
   - **502 Bad Gateway:** Upstream service down
   - **503 Service Unavailable:** Service overloaded
   - **504 Gateway Timeout:** Request timeout

5. **Rollback if needed:**
   ```bash
   # If due to recent deployment
   git revert HEAD
   docker compose build app
   docker compose up -d app
   ```

### Prevention
- Comprehensive error handling
- Circuit breakers for external dependencies
- Proper logging and monitoring
- Regular code reviews
- Automated testing (unit, integration)
- Gradual rollouts with rollback capability

---

## CriticalErrorRate

**Severity:** Critical  
**Threshold:** Error rate (5xx) > 15% for 2 minutes  
**Golden Signal:** Errors

### Description
Backend is experiencing a critical failure rate, majority of requests are failing.

### Impact
- Severe service degradation
- Mass user impact
- Potential data loss
- Business operation disruption

### Diagnosis Steps

1. **Immediate error check:**
   ```bash
   docker logs int531-demo-app-1 --tail 50
   curl -I http://localhost:3100/api/students
   ```

2. **Check service status:**
   ```bash
   docker ps | grep app
   docker stats int531-demo-app-1 --no-stream
   ```

### Resolution

**IMMEDIATE ACTION REQUIRED:**

1. **If recent deployment:**
   ```bash
   # Immediate rollback
   git log -5  # Identify last good commit
   git checkout <last_good_commit>
   docker compose build app
   docker compose up -d app
   ```

2. **If service crashed:**
   ```bash
   # Restart immediately
   docker compose restart app
   
   # Check if it stays up
   watch -n 1 'docker ps | grep app'
   ```

3. **If database issues:**
   ```bash
   # Check database
   docker exec int531-demo-mariadb-1 mariadb -u root -prootpassword -e "SHOW PROCESSLIST"
   
   # Restart if needed
   docker compose restart mariadb
   docker compose restart app
   ```

4. **Emergency communication:**
   - Notify on-call team
   - Update status page
   - Prepare incident report

5. **Post-incident:**
   - Gather all logs
   - Create incident timeline
   - Identify root cause
   - Implement preventive measures

### Prevention
- Staged deployments
- Automated health checks
- Automatic rollback on failure
- Load testing before production
- Chaos engineering practices

---

## BackendDbErrors

**Severity:** Warning  
**Threshold:** Database error rate > 2% for 5 minutes  
**Golden Signal:** Errors

### Description
Backend is experiencing errors when communicating with the database, indicating connectivity or query execution problems.

### Impact
- Failed data operations
- Inconsistent application state
- User transaction failures
- Potential data corruption

### Diagnosis Steps

1. **Check database error rate:**
   ```promql
   sum(rate(db_errors_total[5m])) /
   sum(rate(db_operations_total[5m])) * 100
   ```

2. **Check database connectivity:**
   ```bash
   # Test connection
   docker exec int531-demo-mariadb-1 mariadb -u root -prootpassword -e "SELECT 1"
   
   # Check from app container
   docker exec int531-demo-app-1 sh -c 'echo "SELECT 1" | mariadb -h mariadb -u appuser -ppassword students_db'
   ```

3. **Check database status:**
   ```bash
   docker logs int531-demo-mariadb-1 --tail 100
   
   docker exec int531-demo-mariadb-1 mariadb -u root -prootpassword -e "SHOW STATUS"
   ```

4. **Check for connection pool issues:**
   ```bash
   # Check application logs for connection errors
   docker logs int531-demo-app-1 | grep -i "connection\|pool\|timeout"
   ```

5. **Check query performance:**
   ```bash
   docker exec int531-demo-mariadb-1 mariadb -u root -prootpassword -e "SHOW FULL PROCESSLIST"
   ```

### Resolution

1. **For connection pool exhaustion:**
   ```typescript
   // Increase pool size in db/index.ts
   connection: {
     max: 20,  // Increase from default
     idleTimeoutMillis: 30000
   }
   ```

2. **For slow queries:**
   ```bash
   # Enable slow query log
   docker exec int531-demo-mariadb-1 mariadb -u root -prootpassword -e "
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1;
   "
   
   # Check slow queries
   docker exec int531-demo-mariadb-1 tail /var/log/mysql/slow.log
   ```

3. **For connection failures:**
   ```bash
   # Restart database
   docker compose restart mariadb
   
   # Wait for healthy
   sleep 10
   
   # Restart app
   docker compose restart app
   ```

4. **For deadlocks:**
   ```bash
   # Check for deadlocks
   docker exec int531-demo-mariadb-1 mariadb -u root -prootpassword -e "
   SHOW ENGINE INNODB STATUS\G
   " | grep -A 50 "LATEST DETECTED DEADLOCK"
   ```

5. **Optimize queries:**
   - Add missing indexes
   - Rewrite inefficient queries
   - Use connection pooling properly
   - Implement retry logic with exponential backoff

### Prevention
- Proper connection pool configuration
- Query optimization and indexing
- Database health monitoring
- Regular performance reviews
- Implement circuit breakers
- Connection retry logic

---

# Latency Signals

## HighLatencyP95

**Severity:** Warning  
**Threshold:** P95 latency > 500ms for 5 minutes  
**Golden Signal:** Latency

### Description
95th percentile of request latency is elevated, indicating that a significant portion of users are experiencing slow responses.

### Impact
- Degraded user experience
- Increased page load times
- User frustration
- Potential timeouts

### Diagnosis Steps

1. **Check P95 latency:**
   ```promql
   histogram_quantile(0.95, 
     rate(http_request_duration_seconds_bucket[5m])
   )
   ```

2. **Check per-endpoint latency:**
   ```promql
   histogram_quantile(0.95, 
     rate(http_request_duration_seconds_bucket[5m])
   ) by (endpoint)
   ```

3. **Check database query time:**
   ```promql
   histogram_quantile(0.95,
     rate(db_query_duration_seconds_bucket[5m])
   )
   ```

4. **Check system resources:**
   ```bash
   docker stats
   ```

5. **Profile slow requests:**
   ```bash
   # Check application logs for slow requests
   docker logs int531-demo-app-1 | grep "slow\|timeout"
   ```

### Resolution

1. **For database-related latency:**
   ```bash
   # Check slow queries
   docker exec int531-demo-mariadb-1 mariadb -u root -prootpassword -e "
   SELECT * FROM information_schema.processlist 
   WHERE time > 1 ORDER BY time DESC;
   "
   
   # Check for missing indexes
   # Optimize queries
   ```

2. **For resource constraints:**
   ```bash
   # Increase container resources
   docker update --cpus="2" --memory="2g" int531-demo-app-1
   
   # Scale horizontally
   docker compose up -d --scale app=2
   ```

3. **For application code:**
   - Profile application code
   - Optimize hot paths
   - Add caching where appropriate
   - Reduce external API calls

4. **For external dependencies:**
   - Implement timeouts
   - Add circuit breakers
   - Use async operations
   - Cache external responses

5. **Enable performance monitoring:**
   ```typescript
   // Add detailed timing metrics
   // Implement distributed tracing
   // Add performance profiling
   ```

### Prevention
- Regular performance testing
- Query optimization
- Proper indexing
- Caching strategy
- Code profiling
- Load testing

---

## CriticalLatencyP95

**Severity:** Critical  
**Threshold:** P95 latency > 2000ms for 2 minutes  
**Golden Signal:** Latency

### Description
95th percentile latency is critically high, most users are experiencing severe slowdowns.

### Impact
- Severe user experience degradation
- High timeout rates
- User abandonment
- Business impact

### Diagnosis Steps

1. **Immediate check:**
   ```bash
   # Test endpoint
   time curl http://localhost:3100/api/students
   
   # Check system
   docker stats --no-stream
   ```

2. **Check for blocking operations:**
   ```bash
   docker logs int531-demo-app-1 --tail 100
   ```

### Resolution

**IMMEDIATE ACTION REQUIRED:**

1. **Identify bottleneck:**
   ```bash
   # Database
   docker exec int531-demo-mariadb-1 mariadb -u root -prootpassword -e "SHOW FULL PROCESSLIST"
   
   # Kill slow queries if needed
   docker exec int531-demo-mariadb-1 mariadb -u root -prootpassword -e "KILL <id>"
   ```

2. **Emergency scaling:**
   ```bash
   docker compose up -d --scale app=3
   ```

3. **Restart services:**
   ```bash
   docker compose restart app
   ```

4. **Enable maintenance mode if severe:**
   - Return 503 with Retry-After header
   - Display maintenance page
   - Fix underlying issue

### Prevention
- Critical latency alerts at lower threshold
- Automatic scaling policies
- Circuit breakers
- Request timeout enforcement

---

## HighLatencyP99

**Severity:** Warning  
**Threshold:** P99 latency > 2000ms for 5 minutes  
**Golden Signal:** Latency

### Description
99th percentile latency is high, indicating that the slowest 1% of requests are experiencing significant delays.

### Impact
- Poor experience for subset of users
- May indicate edge cases or bugs
- Potential timeout issues
- Can indicate resource saturation

### Diagnosis Steps

1. **Check P99 latency:**
   ```promql
   histogram_quantile(0.99,
     rate(http_request_duration_seconds_bucket[5m])
   )
   ```

2. **Compare with P95 and P50:**
   ```promql
   # P50
   histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))
   # P95
   histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
   # P99
   histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
   ```

3. **Identify outlier patterns:**
   - Large requests
   - Complex queries
   - Specific user actions
   - Edge cases

### Resolution

1. **Optimize edge cases:**
   - Add pagination for large datasets
   - Implement query limits
   - Optimize complex operations
   - Add request size limits

2. **Implement timeouts:**
   ```typescript
   // Add request timeouts
   const timeout = setTimeout(() => {
     // Handle timeout
   }, 30000);  // 30 seconds
   ```

3. **Add caching for expensive operations:**
   ```typescript
   // Cache expensive queries
   // Use memoization
   // Implement Redis cache
   ```

4. **Profile slow requests:**
   - Enable detailed logging
   - Add performance tracing
   - Identify bottlenecks

### Prevention
- Handle edge cases properly
- Implement proper pagination
- Set request limits
- Regular performance profiling
- Distributed tracing

---

# Saturation Signals

## HighActiveConnections

**Severity:** Warning  
**Threshold:** Active HTTP connections > 100 for 5 minutes  
**Golden Signal:** Saturation

### Description
Number of active HTTP connections is high, indicating the service is approaching or at capacity.

### Impact
- Service saturation
- New requests may be rejected
- Increased latency
- Risk of service unavailability

### Diagnosis Steps

1. **Check active connections:**
   ```promql
   http_active_connections
   ```

2. **Check connection distribution:**
   ```bash
   # System connections
   netstat -an | grep :3100 | wc -l
   
   # Docker stats
   docker stats int531-demo-app-1
   ```

3. **Check for connection leaks:**
   ```bash
   # Monitor over time
   watch -n 1 'netstat -an | grep :3100 | wc -l'
   ```

4. **Check request rate vs connections:**
   ```promql
   rate(http_requests_total[5m]) * 60
   ```

### Resolution

1. **If at capacity limit:**
   ```bash
   # Scale horizontally
   docker compose up -d --scale app=3
   
   # Or increase resources
   docker update --cpus="2" int531-demo-app-1
   ```

2. **If connection leaks:**
   - Check for long-running requests
   - Verify proper connection closure
   - Implement connection timeouts
   - Review application code for leaks

3. **Implement connection limits:**
   ```typescript
   // Add connection pooling
   // Implement max connections limit
   // Add connection timeout
   ```

4. **Load balancing:**
   - Distribute load across instances
   - Implement proper load balancing
   - Use connection pooling

5. **Check for abuse:**
   ```bash
   # Check for connection flooding
   netstat -an | grep :3100 | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr
   ```

### Prevention
- Implement connection limits
- Use load balancing
- Set up auto-scaling
- Monitor connection trends
- Implement rate limiting
- Proper connection pool configuration

---

## General Backend Debugging Commands

### Application Health
```bash
# Check containers
docker ps
docker compose ps

# Application logs
docker logs int531-demo-app-1 --tail 100 --follow

# Test endpoints
curl -I http://localhost:3100/
curl http://localhost:3100/api/students
curl http://localhost:3100/api/metrics

# Check metrics
curl http://localhost:3100/api/metrics | grep http_requests
```

### Database Health
```bash
# Database connection test
docker exec int531-demo-mariadb-1 mariadb -u root -prootpassword -e "SELECT 1"

# Check processes
docker exec int531-demo-mariadb-1 mariadb -u root -prootpassword -e "SHOW FULL PROCESSLIST"

# Check status
docker exec int531-demo-mariadb-1 mariadb -u root -prootpassword -e "SHOW STATUS LIKE '%connection%'"

# Database logs
docker logs int531-demo-mariadb-1 --tail 100
```

### Performance Analysis
```bash
# Resource usage
docker stats

# Network connections
netstat -tulpn | grep :3100

# Application metrics
curl http://localhost:3100/api/metrics

# Request timing
time curl http://localhost:3100/api/students
```

---

## Escalation

If issues cannot be resolved:

1. **Gather diagnostics:**
   ```bash
   # Application logs
   docker logs int531-demo-app-1 > app-logs.txt
   
   # Database logs
   docker logs int531-demo-mariadb-1 > db-logs.txt
   
   # Metrics snapshot
   curl http://localhost:3100/api/metrics > metrics.txt
   curl http://localhost:9090/api/v1/query?query=up > prometheus-status.txt
   
   # System state
   docker ps > containers.txt
   docker stats --no-stream > stats.txt
   ```

2. **Document incident:**
   - Timeline of events
   - Actions taken
   - Current metrics
   - Error messages

3. **Contact:**
   - Backend development team
   - Database administrator
   - On-call engineer
   - Platform/SRE team

---

**Last Updated:** December 18, 2025  
**Maintained By:** Backend Engineering Team  
**Golden Signals Reference:** [Google SRE Book - Chapter 6](https://sre.google/sre-book/monitoring-distributed-systems/)
