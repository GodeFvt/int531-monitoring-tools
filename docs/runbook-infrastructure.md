# Infrastructure Alert Runbooks

This document provides troubleshooting steps and resolution procedures for infrastructure-related alerts.

---

## Table of Contents

- [InstanceDown](#instancedown)
- [HighCpuUsage](#highcpuusage)
- [CriticalCpuUsage](#criticalcpuusage)
- [HighMemoryUsage](#highmemoryusage)
- [CriticalMemoryUsage](#criticalmemoryusage)
- [LowDiskSpace](#lowdiskspace)
- [CriticalDiskSpace](#criticaldiskspace)
- [HighDiskIO](#highdiskio)

---

## InstanceDown

**Severity:** Critical  
**Threshold:** Service down for more than 1 minute

### Description
A monitored service or instance is not responding to Prometheus health checks, indicating it is either stopped, crashed, or unreachable.

### Impact
- Service unavailability
- Loss of monitoring visibility
- Potential data loss
- User-facing errors

### Diagnosis Steps

1. **Check which instance is down:**
   ```promql
   up == 0
   ```

2. **Identify the service:**
   ```bash
   # Check Docker containers
   docker ps -a | grep <instance_name>
   
   # Check system services
   systemctl status <service_name>
   ```

3. **Check container/service logs:**
   ```bash
   # Docker logs
   docker logs <container_name> --tail 100
   
   # System logs
   journalctl -u <service_name> -n 100
   ```

4. **Check resource availability:**
   ```bash
   # CPU and Memory
   top
   htop
   
   # Disk space
   df -h
   ```

### Resolution

1. **For Docker containers:**
   ```bash
   # Check status
   docker ps -a | grep <container>
   
   # Restart container
   docker restart <container_name>
   
   # If restart fails, check logs
   docker logs <container_name>
   
   # Recreate if needed
   docker compose up -d <service_name>
   ```

2. **For system services:**
   ```bash
   # Restart service
   sudo systemctl restart <service_name>
   
   # Check status
   sudo systemctl status <service_name>
   
   # Enable if disabled
   sudo systemctl enable <service_name>
   ```

3. **For Node Exporter:**
   ```bash
   # Check if running
   ps aux | grep node_exporter
   
   # Restart
   sudo systemctl restart node_exporter
   ```

4. **For application services:**
   ```bash
   cd /home/sysadmin/int531-demo
   docker compose restart app
   ```

5. **Check network connectivity:**
   ```bash
   # Test connectivity to Prometheus
   curl -I http://localhost:9090
   
   # Test service endpoint
   curl -I http://<service>:<port>/metrics
   ```

### Prevention
- Implement automatic restart policies
- Set up proper health checks
- Monitor resource usage proactively
- Use container orchestration with self-healing
- Regular service maintenance windows

---

## HighCpuUsage

**Severity:** Warning  
**Threshold:** CPU usage > 80% for 5 minutes

### Description
System CPU utilization is consistently high, which may impact application performance and response times.

### Impact
- Degraded application performance
- Increased response times
- Risk of service unavailability
- Potential system instability

### Diagnosis Steps

1. **Check CPU usage:**
   ```bash
   # Overall CPU usage
   top
   mpstat 1 5
   
   # Per-process CPU
   ps aux --sort=-%cpu | head -20
   ```

2. **Identify CPU-intensive processes:**
   ```bash
   # Top CPU consumers
   top -b -n 1 | head -20
   
   # With Docker
   docker stats --no-stream
   ```

3. **Check system load:**
   ```bash
   uptime
   cat /proc/loadavg
   ```

4. **View CPU metrics:**
   ```promql
   100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
   ```

### Resolution

1. **Identify and optimize processes:**
   ```bash
   # Find CPU hogs
   ps aux --sort=-%cpu | head -10
   
   # Check Docker containers
   docker stats
   ```

2. **For runaway processes:**
   ```bash
   # Nice down CPU-intensive processes
   renice +10 <PID>
   
   # Kill if necessary
   kill -15 <PID>
   ```

3. **Scale resources:**
   - Add more CPU cores
   - Implement horizontal scaling
   - Optimize application code

4. **Optimize Docker containers:**
   ```bash
   # Set CPU limits
   docker update --cpus="2" <container_name>
   ```

5. **Check for mining/malware:**
   ```bash
   # Check suspicious processes
   ps aux | grep -E 'crypto|miner'
   
   # Check connections
   netstat -tupn
   ```

### Prevention
- Set appropriate resource limits
- Implement auto-scaling
- Regular performance profiling
- Code optimization
- Load testing before deployment
- CPU quota management for containers

---

## CriticalCpuUsage

**Severity:** Critical  
**Threshold:** CPU usage > 95% for 2 minutes

### Description
System CPU is at critical levels, system may become unresponsive or crash.

### Impact
- Severe performance degradation
- System unresponsiveness
- Service outages
- Risk of system crash

### Diagnosis Steps

1. **Immediate check:**
   ```bash
   top -b -n 1 | head -20
   ps aux --sort=-%cpu | head -5
   ```

2. **Check system responsiveness:**
   ```bash
   uptime
   w
   ```

### Resolution

**IMMEDIATE ACTION REQUIRED:**

1. **Kill non-essential processes:**
   ```bash
   # Identify and kill top CPU consumer
   kill -9 <PID>
   ```

2. **Restart problematic services:**
   ```bash
   docker restart <heavy_container>
   ```

3. **Emergency resource allocation:**
   - Reduce container CPU limits temporarily
   - Stop non-critical services
   - Clear system caches if safe

4. **Scale up immediately:**
   - Add CPU resources
   - Distribute load to other instances

### Prevention
- Critical CPU alert at 90%
- Automated scaling policies
- Circuit breakers in applications
- Regular capacity planning

---

## HighMemoryUsage

**Severity:** Warning  
**Threshold:** Memory usage > 80% for 5 minutes

### Description
System memory utilization is high, which may lead to swapping and performance degradation.

### Impact
- Performance degradation
- Increased disk I/O (swapping)
- Risk of OOM killer
- Application slowdowns

### Diagnosis Steps

1. **Check memory usage:**
   ```bash
   free -h
   vmstat 1 5
   cat /proc/meminfo
   ```

2. **Identify memory-intensive processes:**
   ```bash
   # Top memory consumers
   ps aux --sort=-%mem | head -20
   
   # With Docker
   docker stats --no-stream
   ```

3. **Check for memory leaks:**
   ```bash
   # Monitor memory over time
   watch -n 5 'free -h'
   
   # Check OOM events
   dmesg | grep -i "out of memory"
   ```

4. **View memory metrics:**
   ```promql
   (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / 
   node_memory_MemTotal_bytes * 100
   ```

### Resolution

1. **Free up memory:**
   ```bash
   # Clear caches (safe)
   sync; echo 3 > /proc/sys/vm/drop_caches
   
   # Check swap usage
   swapon --show
   ```

2. **Restart memory-heavy services:**
   ```bash
   docker restart <container_name>
   ```

3. **Set memory limits:**
   ```bash
   # Docker memory limit
   docker update --memory="512m" <container_name>
   ```

4. **Kill memory-intensive processes:**
   ```bash
   # Identify and kill
   kill -15 <PID>
   ```

5. **Increase swap if needed:**
   ```bash
   # Check swap
   free -h
   
   # Add swap (if needed)
   fallocate -l 2G /swapfile
   chmod 600 /swapfile
   mkswap /swapfile
   swapon /swapfile
   ```

### Prevention
- Set appropriate memory limits
- Fix memory leaks in applications
- Regular memory profiling
- Implement memory monitoring
- Use memory-efficient algorithms

---

## CriticalMemoryUsage

**Severity:** Critical  
**Threshold:** Memory usage > 95% for 2 minutes

### Description
System memory is critically low, OOM killer may start terminating processes.

### Impact
- Imminent OOM (Out of Memory) condition
- Processes will be killed
- System instability
- Data loss risk

### Diagnosis Steps

1. **Immediate check:**
   ```bash
   free -h
   ps aux --sort=-%mem | head -10
   ```

2. **Check OOM events:**
   ```bash
   dmesg | grep -i "killed process"
   journalctl -k | grep -i "out of memory"
   ```

### Resolution

**IMMEDIATE ACTION REQUIRED:**

1. **Free memory immediately:**
   ```bash
   # Drop caches
   sync; echo 3 > /proc/sys/vm/drop_caches
   
   # Kill largest memory consumer
   kill -9 <PID>
   ```

2. **Restart critical services:**
   ```bash
   docker restart <container>
   ```

3. **Emergency measures:**
   ```bash
   # Stop non-critical services
   docker stop <non_critical_container>
   
   # Enable emergency swap
   fallocate -l 1G /emergency_swap
   chmod 600 /emergency_swap
   mkswap /emergency_swap
   swapon /emergency_swap
   ```

4. **Scale up:**
   - Add more RAM
   - Distribute load
   - Increase swap space

### Prevention
- Set memory alerts at 85%
- Implement OOM guards
- Regular memory leak testing
- Proper resource allocation

---

## LowDiskSpace

**Severity:** Warning  
**Threshold:** Available disk space < 20% for 5 minutes

### Description
File system is running low on available space, which can impact system operations and data writes.

### Impact
- Cannot write new data
- Log rotation issues
- Application errors
- Database write failures

### Diagnosis Steps

1. **Check disk usage:**
   ```bash
   df -h
   df -i  # Check inodes too
   ```

2. **Find largest directories:**
   ```bash
   du -h --max-depth=1 / | sort -hr | head -20
   du -h --max-depth=1 /var | sort -hr
   ```

3. **Find large files:**
   ```bash
   find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null
   ```

4. **Check Docker disk usage:**
   ```bash
   docker system df
   docker ps -s
   ```

### Resolution

1. **Clean Docker resources:**
   ```bash
   # Remove unused images
   docker image prune -a
   
   # Remove stopped containers
   docker container prune
   
   # Remove unused volumes
   docker volume prune
   
   # Complete cleanup
   docker system prune -a --volumes
   ```

2. **Clean system logs:**
   ```bash
   # Clean journal logs (keep 1 day)
   journalctl --vacuum-time=1d
   
   # Clean old logs
   find /var/log -type f -name "*.log" -mtime +30 -delete
   ```

3. **Clean package caches:**
   ```bash
   # Ubuntu/Debian
   apt-get clean
   apt-get autoclean
   
   # Remove old kernels
   apt-get autoremove --purge
   ```

4. **Remove temporary files:**
   ```bash
   rm -rf /tmp/*
   rm -rf /var/tmp/*
   ```

5. **Compress old logs:**
   ```bash
   find /var/log -type f -name "*.log" -exec gzip {} \;
   ```

### Prevention
- Set up log rotation
- Regular cleanup automation
- Monitor disk usage trends
- Implement disk quotas
- Use separate partitions for data

---

## CriticalDiskSpace

**Severity:** Critical  
**Threshold:** Available disk space < 10% for 2 minutes

### Description
File system is critically low on space, system may become unstable or unusable.

### Impact
- System instability
- Cannot write data
- Service failures
- Potential data corruption

### Diagnosis Steps

1. **Immediate check:**
   ```bash
   df -h
   df -i
   ```

2. **Quick large file scan:**
   ```bash
   du -sh /*
   docker system df
   ```

### Resolution

**IMMEDIATE ACTION REQUIRED:**

1. **Emergency cleanup:**
   ```bash
   # Docker cleanup
   docker system prune -a -f --volumes
   
   # Clear logs
   journalctl --vacuum-time=1h
   truncate -s 0 /var/log/*.log
   
   # Clear temp
   rm -rf /tmp/* /var/tmp/*
   ```

2. **Stop services if needed:**
   ```bash
   docker stop <non_critical_services>
   ```

3. **Move large files:**
   ```bash
   # Move to external storage
   mv /large/directory /external/storage/
   ```

4. **Emergency expansion:**
   - Attach additional disk
   - Extend existing volume
   - Mount external storage

### Prevention
- Disk space alert at 20%
- Automated cleanup scripts
- Proper capacity planning
- Separate data partitions

---

## HighDiskIO

**Severity:** Warning  
**Threshold:** Disk I/O utilization > 80% for 5 minutes

### Description
Disk I/O utilization is high, which can cause performance bottlenecks and increased latency.

### Impact
- Slow application response
- Database performance issues
- System sluggishness
- Increased latency

### Diagnosis Steps

1. **Check I/O statistics:**
   ```bash
   iostat -x 5 5
   iotop -o
   ```

2. **Identify I/O-intensive processes:**
   ```bash
   iotop
   pidstat -d 5
   ```

3. **Check disk metrics:**
   ```bash
   vmstat 1 5
   dstat -d
   ```

4. **Docker I/O:**
   ```bash
   docker stats
   ```

### Resolution

1. **Optimize I/O-heavy processes:**
   ```bash
   # Nice down I/O priority
   ionice -c3 -p <PID>
   ```

2. **Tune database:**
   - Adjust buffer pool size
   - Optimize queries
   - Add indexes

3. **Reduce writes:**
   ```bash
   # Reduce log levels
   # Disable unnecessary logging
   # Batch write operations
   ```

4. **Upgrade storage:**
   - Use SSD instead of HDD
   - Increase IOPS
   - Use faster storage tier

5. **Docker volume optimization:**
   ```bash
   # Use tmpfs for temporary data
   # Optimize volume drivers
   ```

### Prevention
- Use appropriate storage types
- Implement caching strategies
- Optimize database queries
- Regular performance testing
- Monitor I/O trends

---

## General Troubleshooting Commands

### System Information
```bash
# Overall system status
htop
top

# Memory info
free -h
vmstat 1

# Disk info
df -h
iostat -x

# Process info
ps aux
systemctl status
```

### Docker Commands
```bash
# Container status
docker ps -a
docker stats

# Logs
docker logs <container> --tail 100 --follow

# Resource usage
docker system df
docker inspect <container>

# Restart
docker restart <container>
docker compose restart
```

### Monitoring
```bash
# Check Prometheus
curl http://localhost:9090/-/healthy

# Check Node Exporter
curl http://localhost:9100/metrics

# Check service metrics
curl http://localhost:3100/api/metrics
```

---

## Escalation

If issues cannot be resolved:

1. **Document the problem:**
   - Current metrics/screenshots
   - Actions taken
   - Error messages
   - Timeline

2. **Gather diagnostics:**
   ```bash
   # System info
   uname -a
   df -h
   free -h
   top -b -n 1
   
   # Logs
   journalctl -n 100
   docker logs <container> --tail 100
   ```

3. **Contact:**
   - Infrastructure team
   - On-call engineer
   - System administrator

---

**Last Updated:** December 18, 2025  
**Maintained By:** SRE Team
