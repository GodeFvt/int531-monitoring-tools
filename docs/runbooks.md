# Database Alert Runbooks

This document provides troubleshooting steps and resolution procedures for database-related alerts.

---

## Table of Contents

- [HighDbErrorRate](#highdberrorrate)
- [MySQLDown](#mysqldown)
- [MySQLHighConnectionUsage](#mysqlhighconnectionusage)
- [MySQLConnectionUsageCritical](#mysqlconnectionusagecritical)
- [MySQLHighSlowQueries](#mysqlhighslowqueries)
- [MySQLHighAbortedConnections](#mysqlhighabortedconnections)
- [MySQLLowBufferPoolHitRate](#mysqllowbufferpoolhitrate)
- [MySQLHighRowLockTime](#mysqlhighrowlocktime)
- [MySQLHighTableLockWaits](#mysqlhightablelockwaits)
- [MySQLReplicationLag](#mysqlreplicationlag)
- [MySQLHighThreadsRunning](#mysqlhighthreadsrunning)

---

## HighDbErrorRate

**Severity:** Critical  
**Threshold:** Any database query errors detected from the application

### Description
The application is experiencing database query errors, which could indicate connection issues, query syntax errors, permission problems, or database availability issues.

### Impact
- Users may experience failed operations
- Data may not be saved or retrieved correctly
- Application functionality may be degraded

### Diagnosis Steps

1. **Check application logs:**
   ```bash
   docker logs x-nextjs-app --tail 100
   ```

2. **Check database connectivity:**
   ```bash
   docker exec x-mariadb mysql -u app -pint53102 -e "SELECT 1"
   ```

3. **View recent error metrics:**
   ```promql
   rate(db_query_errors_total[5m])
   sum by (operation, table, error_type) (rate(db_query_errors_total[5m]))
   ```

4. **Check database error logs:**
   ```bash
   docker logs x-mariadb --tail 100
   ```

### Resolution

1. **If database is down:**
   - Restart the database container: `docker restart x-mariadb`
   - Check for disk space issues
   - Review database error logs for crash information

2. **If connection pool exhausted:**
   - Check connection settings in application
   - Look for connection leaks in application code
   - Consider increasing connection pool size

3. **If query errors:**
   - Review recent code deployments
   - Check for schema migration issues
   - Verify query syntax in application logs

4. **If permission errors:**
   - Verify database user permissions
   - Check grants: `SHOW GRANTS FOR 'app'@'%';`

### Prevention
- Implement proper connection pooling
- Add query timeout settings
- Use prepared statements to prevent SQL syntax errors
- Monitor connection pool metrics
- Implement circuit breaker patterns

---

## MySQLDown

**Severity:** Critical  
**Threshold:** MySQL/MariaDB down for more than 1 minute

### Description
The MySQL/MariaDB database is not responding to health checks, indicating it is either stopped, crashed, or unreachable.

### Impact
- Complete application outage
- All database operations will fail
- Data cannot be read or written

### Diagnosis Steps

1. **Check if container is running:**
   ```bash
   docker ps | grep mariadb
   ```

2. **Check container logs:**
   ```bash
   docker logs x-mariadb --tail 100
   ```

3. **Check mysqld_exporter logs:**
   ```bash
   docker logs x-mysqld-exporter --tail 50
   ```

4. **Check resource usage:**
   ```bash
   docker stats x-mariadb --no-stream
   ```

5. **Check disk space:**
   ```bash
   df -h
   ```

### Resolution

1. **If container is stopped:**
   ```bash
   docker start x-mariadb
   ```

2. **If container is running but unresponsive:**
   ```bash
   docker restart x-mariadb
   ```

3. **If out of disk space:**
   - Clean up old logs and temporary files
   - Remove unnecessary Docker images: `docker system prune`
   - Increase disk space

4. **If database crashed:**
   - Check for InnoDB corruption in logs
   - Attempt recovery: Set `innodb_force_recovery` if needed
   - Restore from backup if corruption is severe

5. **If network issues:**
   - Check Docker network: `docker network inspect sre-network`
   - Verify firewall rules
   - Check DNS resolution

### Prevention
- Set up automated backups
- Monitor disk space proactively
- Configure resource limits appropriately
- Implement health checks
- Use persistent volumes for data

---

## MySQLHighConnectionUsage

**Severity:** Warning  
**Threshold:** Connection usage > 80% for 5 minutes

### Description
MySQL is using more than 80% of its maximum allowed connections, which may lead to connection exhaustion.

### Impact
- New connections may be refused soon
- Application performance may degrade
- Risk of complete connection exhaustion

### Diagnosis Steps

1. **Check current connections:**
   ```promql
   mysql_global_status_threads_connected
   mysql_global_variables_max_connections
   ```

2. **View connection details:**
   ```sql
   SHOW PROCESSLIST;
   SHOW STATUS LIKE 'Threads_connected';
   SHOW VARIABLES LIKE 'max_connections';
   ```

3. **Identify long-running queries:**
   ```sql
   SELECT * FROM information_schema.processlist 
   WHERE command != 'Sleep' 
   ORDER BY time DESC;
   ```

4. **Check for connection leaks:**
   ```bash
   docker logs x-nextjs-app | grep -i "connection"
   ```

### Resolution

1. **Kill idle connections:**
   ```sql
   -- Kill connections idle > 300 seconds
   SELECT CONCAT('KILL ', id, ';') 
   FROM information_schema.processlist 
   WHERE command = 'Sleep' AND time > 300;
   ```

2. **Increase max_connections (temporary):**
   ```sql
   SET GLOBAL max_connections = 200;
   ```

3. **Optimize application connection pooling:**
   - Reduce connection timeout
   - Implement connection pool limits
   - Ensure connections are properly closed

4. **Restart application if connection leak suspected:**
   ```bash
   docker restart x-nextjs-app
   ```

### Prevention
- Set appropriate connection pool size in application
- Implement connection timeout settings
- Use connection pooling middleware
- Monitor connection usage regularly
- Set up alerts at lower thresholds (e.g., 70%)

---

## MySQLConnectionUsageCritical

**Severity:** Critical  
**Threshold:** Connection usage > 95% for 2 minutes

### Description
MySQL connection usage is critically high. The database may start refusing new connections at any moment.

### Impact
- Imminent connection exhaustion
- New connection attempts will fail
- Application errors will increase rapidly
- Service degradation or outage

### Diagnosis Steps

1. **Immediate connection check:**
   ```sql
   SHOW STATUS LIKE 'Threads_connected';
   SHOW VARIABLES LIKE 'max_connections';
   ```

2. **List all connections:**
   ```sql
   SELECT * FROM information_schema.processlist;
   ```

3. **Check connection by user:**
   ```sql
   SELECT user, host, COUNT(*) as connections 
   FROM information_schema.processlist 
   GROUP BY user, host;
   ```

### Resolution

**IMMEDIATE ACTION REQUIRED:**

1. **Emergency connection cleanup:**
   ```sql
   -- Kill all sleeping connections
   SELECT GROUP_CONCAT(CONCAT('KILL ', id) SEPARATOR ';') 
   FROM information_schema.processlist 
   WHERE command = 'Sleep';
   ```

2. **Increase max_connections immediately:**
   ```bash
   docker exec x-mariadb mysql -u root -pint53102 \
     -e "SET GLOBAL max_connections = 300;"
   ```

3. **Restart application to reset connection pools:**
   ```bash
   docker restart x-nextjs-app
   ```

4. **Monitor recovery:**
   ```promql
   (mysql_global_status_threads_connected / mysql_global_variables_max_connections) * 100
   ```

### Prevention
- Implement connection pool limits in application
- Set up auto-scaling for application instances
- Configure aggressive connection timeouts
- Regular connection pool monitoring
- Load testing to determine optimal max_connections

---

## MySQLHighSlowQueries

**Severity:** Warning  
**Threshold:** More than 1 slow query per second for 5 minutes

### Description
MySQL is experiencing a high rate of slow queries, which can impact overall database performance and application responsiveness.

### Impact
- Degraded application performance
- Increased response times
- Resource contention in database
- Potential for query queue buildup

### Diagnosis Steps

1. **Check slow query log:**
   ```sql
   SHOW VARIABLES LIKE 'slow_query%';
   SHOW VARIABLES LIKE 'long_query_time';
   ```

2. **View slow query statistics:**
   ```promql
   rate(mysql_global_status_slow_queries[5m])
   ```

3. **Identify slow queries:**
   ```sql
   SELECT * FROM information_schema.processlist 
   WHERE time > 5 
   ORDER BY time DESC;
   ```

4. **Check query execution plans:**
   ```sql
   EXPLAIN [slow_query];
   ```

### Resolution

1. **Identify problematic queries:**
   - Review slow query log
   - Use `EXPLAIN` to analyze query plans
   - Look for missing indexes

2. **Optimize queries:**
   - Add appropriate indexes
   - Rewrite inefficient queries
   - Use query caching where appropriate

3. **Check table statistics:**
   ```sql
   ANALYZE TABLE table_name;
   ```

4. **Optimize tables:**
   ```sql
   OPTIMIZE TABLE table_name;
   ```

5. **Increase buffer pool size if needed:**
   ```sql
   SET GLOBAL innodb_buffer_pool_size = 536870912; -- 512MB
   ```

### Prevention
- Regular query performance review
- Implement query monitoring
- Use indexes appropriately
- Regular ANALYZE TABLE maintenance
- Set appropriate slow_query_time threshold
- Code review for N+1 query problems

---

## MySQLHighAbortedConnections

**Severity:** Warning  
**Threshold:** More than 1 aborted connection per second for 5 minutes

### Description
MySQL is experiencing a high rate of aborted connections, which may indicate network issues, timeouts, or application problems.

### Impact
- Wasted connection resources
- Potential connection pool exhaustion
- May indicate underlying application or network issues

### Diagnosis Steps

1. **Check aborted connection metrics:**
   ```promql
   rate(mysql_global_status_aborted_connects[5m])
   rate(mysql_global_status_aborted_clients[5m])
   ```

2. **View connection status:**
   ```sql
   SHOW STATUS LIKE 'Aborted%';
   ```

3. **Check error logs:**
   ```bash
   docker logs x-mariadb | grep -i "abort\|disconnect"
   ```

4. **Review application logs:**
   ```bash
   docker logs x-nextjs-app | grep -i "connection\|timeout"
   ```

### Resolution

1. **Check connection timeout settings:**
   ```sql
   SHOW VARIABLES LIKE '%timeout%';
   ```

2. **Increase timeouts if too aggressive:**
   ```sql
   SET GLOBAL wait_timeout = 600;
   SET GLOBAL interactive_timeout = 600;
   ```

3. **Check for authentication failures:**
   ```bash
   docker logs x-mariadb | grep -i "access denied"
   ```

4. **Verify network connectivity:**
   ```bash
   docker exec x-nextjs-app ping -c 3 mariadb
   ```

5. **Review application connection handling:**
   - Check for proper connection closure
   - Verify connection pool configuration
   - Look for connection timeout issues

### Prevention
- Implement proper connection error handling
- Use connection pooling with retry logic
- Configure appropriate timeout values
- Monitor network latency
- Implement health checks

---

## MySQLLowBufferPoolHitRate

**Severity:** Warning  
**Threshold:** Buffer pool hit rate < 95% for 10 minutes

### Description
InnoDB buffer pool hit rate is low, meaning more disk reads are required, which significantly impacts database performance.

### Impact
- Slower query execution
- Increased disk I/O
- Higher latency
- Reduced throughput

### Diagnosis Steps

1. **Check buffer pool statistics:**
   ```promql
   (1 - (rate(mysql_global_status_innodb_buffer_pool_reads[5m]) / 
         rate(mysql_global_status_innodb_buffer_pool_read_requests[5m]))) * 100
   ```

2. **View buffer pool status:**
   ```sql
   SHOW STATUS LIKE 'Innodb_buffer_pool%';
   ```

3. **Check buffer pool size:**
   ```sql
   SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
   ```

4. **Monitor disk I/O:**
   ```bash
   docker stats x-mariadb --no-stream
   ```

### Resolution

1. **Increase buffer pool size:**
   ```sql
   -- Allocate 512MB (adjust based on available RAM)
   SET GLOBAL innodb_buffer_pool_size = 536870912;
   ```
   
   Or permanently in docker-compose.yml:
   ```yaml
   command:
     - --innodb-buffer-pool-size=512M
   ```

2. **Optimize queries to use indexes:**
   ```sql
   SHOW INDEX FROM table_name;
   ANALYZE TABLE table_name;
   ```

3. **Check for table scans:**
   ```sql
   SELECT * FROM information_schema.processlist 
   WHERE command != 'Sleep';
   ```

4. **Monitor improvement:**
   ```promql
   rate(mysql_global_status_innodb_buffer_pool_read_requests[5m])
   rate(mysql_global_status_innodb_buffer_pool_reads[5m])
   ```

### Prevention
- Allocate 50-70% of available RAM to buffer pool
- Regular performance testing
- Monitor working set size
- Optimize queries to reduce data scanned
- Use appropriate indexes

**Recommended Buffer Pool Sizes:**
- 1GB RAM: 512MB buffer pool
- 2GB RAM: 1GB buffer pool
- 4GB+ RAM: 2-3GB buffer pool

---

## MySQLHighRowLockTime

**Severity:** Warning  
**Threshold:** Average row lock time > 1 second for 5 minutes

### Description
InnoDB row lock wait time is high, indicating contention on specific rows or tables.

### Impact
- Query slowdowns
- Transaction timeouts
- Degraded concurrency
- Potential deadlocks

### Diagnosis Steps

1. **Check lock metrics:**
   ```promql
   rate(mysql_global_status_innodb_row_lock_time[5m]) / 
   rate(mysql_global_status_innodb_row_lock_waits[5m]) / 1000
   ```

2. **View current locks:**
   ```sql
   SELECT * FROM information_schema.innodb_locks;
   SELECT * FROM information_schema.innodb_lock_waits;
   ```

3. **Check for blocking queries:**
   ```sql
   SELECT 
     r.trx_id waiting_trx_id,
     r.trx_mysql_thread_id waiting_thread,
     r.trx_query waiting_query,
     b.trx_id blocking_trx_id,
     b.trx_mysql_thread_id blocking_thread,
     b.trx_query blocking_query
   FROM information_schema.innodb_lock_waits w
   INNER JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
   INNER JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;
   ```

4. **Monitor lock waits:**
   ```sql
   SHOW ENGINE INNODB STATUS\G
   ```

### Resolution

1. **Identify and kill blocking queries:**
   ```sql
   -- Find blocking thread
   SELECT * FROM information_schema.innodb_lock_waits;
   -- Kill blocking thread
   KILL [thread_id];
   ```

2. **Optimize transactions:**
   - Keep transactions short
   - Commit frequently
   - Access rows in consistent order

3. **Review isolation level:**
   ```sql
   SHOW VARIABLES LIKE 'transaction_isolation';
   -- Consider using READ COMMITTED if appropriate
   SET GLOBAL transaction_isolation = 'READ-COMMITTED';
   ```

4. **Add appropriate indexes:**
   ```sql
   -- Ensure WHERE clauses use indexes
   SHOW INDEX FROM table_name;
   ```

### Prevention
- Design transactions to be short
- Access resources in consistent order
- Use appropriate isolation levels
- Implement optimistic locking where possible
- Add proper indexes to reduce lock scope
- Avoid long-running transactions

---

## MySQLHighTableLockWaits

**Severity:** Warning  
**Threshold:** More than 10 table lock waits per second for 5 minutes

### Description
High rate of table lock waits, typically caused by MyISAM tables or explicit table locks.

### Impact
- Query queueing
- Reduced concurrency
- Degraded performance
- Potential query timeouts

### Diagnosis Steps

1. **Check table lock metrics:**
   ```promql
   rate(mysql_global_status_table_locks_waited[5m])
   rate(mysql_global_status_table_locks_immediate[5m])
   ```

2. **View lock status:**
   ```sql
   SHOW STATUS LIKE 'Table_locks%';
   ```

3. **Check for MyISAM tables:**
   ```sql
   SELECT table_schema, table_name, engine 
   FROM information_schema.tables 
   WHERE engine = 'MyISAM' AND table_schema NOT IN ('mysql', 'information_schema');
   ```

4. **Check for LOCK TABLES statements:**
   ```sql
   SELECT * FROM information_schema.processlist 
   WHERE info LIKE '%LOCK TABLES%';
   ```

### Resolution

1. **Convert MyISAM to InnoDB:**
   ```sql
   ALTER TABLE table_name ENGINE=InnoDB;
   ```

2. **Remove explicit table locks:**
   - Review code for `LOCK TABLES` statements
   - Replace with InnoDB row-level locks

3. **Optimize bulk operations:**
   - Use batch inserts instead of individual inserts
   - Disable indexes during bulk loads
   - Use `LOAD DATA INFILE` for large imports

4. **Check for ALTER TABLE operations:**
   ```sql
   SHOW PROCESSLIST;
   ```

### Prevention
- Use InnoDB instead of MyISAM
- Avoid LOCK TABLES statements
- Minimize ALTER TABLE during peak hours
- Use online DDL features
- Implement proper indexing strategy

---

## MySQLReplicationLag

**Severity:** Warning  
**Threshold:** Replication lag > 30 seconds for 5 minutes

### Description
MySQL replication is lagging behind the master, which can cause stale data reads and potential data inconsistency.

### Impact
- Stale data on replica
- Inconsistent read results
- Potential data loss if failover occurs
- Backup delays

### Diagnosis Steps

1. **Check replication status:**
   ```sql
   SHOW SLAVE STATUS\G
   ```

2. **Check lag metric:**
   ```promql
   mysql_slave_lag_seconds
   ```

3. **View replication threads:**
   ```sql
   SELECT * FROM information_schema.processlist 
   WHERE user = 'system user';
   ```

4. **Check for errors:**
   ```sql
   SHOW SLAVE STATUS\G
   -- Look for Last_SQL_Error and Last_IO_Error
   ```

### Resolution

1. **Check for slow queries on replica:**
   ```sql
   SELECT * FROM information_schema.processlist 
   WHERE time > 5 ORDER BY time DESC;
   ```

2. **Optimize replica performance:**
   - Increase `innodb_buffer_pool_size`
   - Enable parallel replication
   - Optimize slow queries

3. **Check network issues:**
   ```bash
   # Test network latency to master
   ping -c 10 master-host
   ```

4. **Temporary skip error (use carefully):**
   ```sql
   STOP SLAVE;
   SET GLOBAL sql_slave_skip_counter = 1;
   START SLAVE;
   ```

5. **Rebuild replica if needed:**
   - Stop replica
   - Take fresh backup from master
   - Restore and restart replication

### Prevention
- Monitor replication lag continuously
- Use semi-synchronous replication
- Implement parallel replication
- Ensure replica hardware matches master
- Regular replication health checks
- Automated failover testing

---

## MySQLHighThreadsRunning

**Severity:** Warning  
**Threshold:** More than 50 threads running for 5 minutes

### Description
High number of threads actively running queries, which may indicate query slowness, insufficient resources, or application issues.

### Impact
- Database overload
- Increased query queue
- Higher response times
- Potential resource exhaustion

### Diagnosis Steps

1. **Check threads running:**
   ```promql
   mysql_global_status_threads_running
   ```

2. **View active queries:**
   ```sql
   SELECT * FROM information_schema.processlist 
   WHERE command != 'Sleep' 
   ORDER BY time DESC;
   ```

3. **Check query distribution:**
   ```sql
   SELECT command, COUNT(*) as count 
   FROM information_schema.processlist 
   GROUP BY command;
   ```

4. **Monitor CPU and I/O:**
   ```bash
   docker stats x-mariadb --no-stream
   ```

### Resolution

1. **Identify slow queries:**
   ```sql
   SELECT * FROM information_schema.processlist 
   WHERE time > 10 
   ORDER BY time DESC;
   ```

2. **Kill long-running queries:**
   ```sql
   KILL QUERY [process_id];
   ```

3. **Optimize slow queries:**
   - Add indexes
   - Rewrite queries
   - Use EXPLAIN to analyze

4. **Increase resources if needed:**
   - Add more CPU
   - Increase RAM
   - Optimize buffer pool

5. **Check for application issues:**
   - Connection pool size
   - Query patterns
   - Transaction handling

### Prevention
- Set max_connections appropriately
- Implement query timeouts
- Regular query optimization
- Monitor query performance
- Load testing
- Implement connection pooling limits
- Use read replicas for reporting queries

---

## General Troubleshooting Tips

### Useful MySQL Commands

```sql
-- Check all status variables
SHOW GLOBAL STATUS;

-- Check all system variables
SHOW VARIABLES;

-- View InnoDB status
SHOW ENGINE INNODB STATUS\G

-- Check processlist
SHOW FULL PROCESSLIST;

-- Check open tables
SHOW OPEN TABLES;

-- Check table status
SHOW TABLE STATUS FROM database_name;
```

### Docker Commands

```bash
# Check container status
docker ps | grep mariadb

# View logs
docker logs x-mariadb --tail 100 --follow

# Execute MySQL commands
docker exec x-mariadb mysql -u root -pint53102 -e "SHOW STATUS"

# Check resource usage
docker stats x-mariadb --no-stream

# Restart services
docker restart x-mariadb
docker restart x-mysqld-exporter
```

### Prometheus Queries

```promql
# MySQL availability
mysql_up

# Query rate
rate(mysql_global_status_queries[5m])

# Connection usage
(mysql_global_status_threads_connected / mysql_global_variables_max_connections) * 100

# Buffer pool hit rate
(1 - (rate(mysql_global_status_innodb_buffer_pool_reads[5m]) / 
      rate(mysql_global_status_innodb_buffer_pool_read_requests[5m]))) * 100

# Slow queries rate
rate(mysql_global_status_slow_queries[5m])
```

---

## Escalation

If the issue cannot be resolved using this runbook:

1. **Gather diagnostics:**
   - Container logs
   - MySQL error logs
   - Metrics screenshots from Grafana
   - Output of `SHOW ENGINE INNODB STATUS`

2. **Document actions taken:**
   - What was tried
   - Results of each action
   - Current system state

3. **Contact:**
   - Database team
   - On-call engineer
   - System administrator

4. **Emergency contacts:**
   - DBA team: [contact info]
   - Infrastructure team: [contact info]
   - Manager on-call: [contact info]

---

## Additional Resources

- [MySQL Documentation](https://dev.mysql.com/doc/)
- [MariaDB Documentation](https://mariadb.com/kb/en/)
- [Percona Blog](https://www.percona.com/blog/)
- [MySQL Performance Blog](https://www.percona.com/blog/category/mysql/)
- [InnoDB Troubleshooting](https://dev.mysql.com/doc/refman/8.0/en/innodb-troubleshooting.html)

---

**Last Updated:** December 18, 2025  
**Maintained By:** SRE Team
