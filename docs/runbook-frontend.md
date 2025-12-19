# Frontend Alert Runbooks (Golden Signals)

This document provides troubleshooting steps and resolution procedures for frontend application alerts following the Golden Signals methodology: **Traffic**, **Errors**, **Latency**, and **Saturation**.

---

## Table of Contents

### Traffic
- [HighPageLoadRate](#highpageloadrate)

### Errors
- [HighFrontendErrorRate](#highfrontenderrorrate)
- [CriticalFrontendErrorRate](#criticalfrontenderrorrate)

### Latency
- [HighPageLoadTime](#highpageloadtime)
- [CriticalPageLoadTime](#criticalpageloadtime)
- [HighFrontendAPILatency](#highfrontendapilatency)

### Saturation
- [HighBrowserMemoryUsage](#highbrowsermemoryusage)

---

## Golden Signals Overview

**The Four Golden Signals for Frontend:**
1. **Traffic** - Page views, user sessions, interactions
2. **Errors** - JavaScript errors, failed requests, console errors
3. **Latency** - Page load time, API response time, interaction delays
4. **Saturation** - Browser memory usage, CPU usage, resource loading

---

# Traffic Signals

## HighPageLoadRate

**Severity:** Warning  
**Threshold:** Page load rate > 500 loads/minute for 5 minutes  
**Golden Signal:** Traffic

### Description
Frontend is experiencing an unusually high number of page loads, which may indicate a traffic spike, bot activity, or user behavior changes.

### Impact
- Increased server load
- Higher bandwidth usage
- Potential performance degradation
- Increased infrastructure costs

### Diagnosis Steps

1. **Check page load metrics:**
   ```promql
   sum(rate(frontend_page_loads_total[5m])) * 60
   ```

2. **Check per-page breakdown:**
   ```promql
   sum by(page) (rate(frontend_page_loads_total[5m])) * 60
   ```

3. **Check browser distribution:**
   ```promql
   sum by(browser) (rate(frontend_page_loads_total[5m])) * 60
   ```

4. **Analyze user sessions:**
   ```promql
   frontend_active_sessions
   ```

5. **Check Grafana Frontend Dashboard:**
   - Navigate to Frontend Dashboard
   - Check "Page Views" panel
   - Review "Traffic by Page" breakdown
   - Check "User Sessions" metrics

### Resolution

1. **Verify if traffic is legitimate:**
   - Check for marketing campaigns
   - Review social media mentions
   - Verify with marketing/business team
   - Check analytics for user patterns

2. **If bot/crawler traffic:**
   ```bash
   # Check user agents in logs
   docker logs int531-demo-app-1 | grep "User-Agent" | sort | uniq -c | sort -nr
   
   # Implement bot detection
   # Add robots.txt restrictions
   # Use CAPTCHA for suspicious patterns
   ```

3. **If legitimate high traffic:**
   - Scale backend services
   - Enable CDN caching
   - Optimize static asset delivery
   - Implement edge caching

4. **Optimize frontend performance:**
   - Enable asset compression
   - Implement code splitting
   - Use lazy loading
   - Optimize images and resources

5. **Monitor backend impact:**
   ```promql
   # Check backend request rate
   sum(rate(http_requests_total[5m])) * 60
   
   # Check backend latency
   histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
   ```

### Prevention
- Implement CDN
- Use caching strategies
- Set up auto-scaling for backend
- Configure rate limiting
- Regular load testing
- Bot detection and management

---

# Error Signals

## HighFrontendErrorRate

**Severity:** Warning  
**Threshold:** Frontend error rate > 2% for 5 minutes  
**Golden Signal:** Errors

### Description
Frontend is experiencing an elevated rate of JavaScript errors or failed operations, affecting user experience.

### Impact
- Degraded user experience
- Feature malfunctions
- User frustration
- Potential data loss

### Diagnosis Steps

1. **Check error rate:**
   ```promql
   sum(rate(frontend_errors_total[5m])) /
   sum(rate(frontend_page_loads_total[5m])) * 100
   ```

2. **Check error types:**
   ```promql
   sum by(error_type) (rate(frontend_errors_total[5m]))
   ```

3. **Check affected pages:**
   ```promql
   sum by(page) (rate(frontend_errors_total[5m]))
   ```

4. **Check browser distribution:**
   ```promql
   sum by(browser) (rate(frontend_errors_total[5m]))
   ```

5. **Check browser console:**
   - Open browser DevTools (F12)
   - Check Console tab for errors
   - Check Network tab for failed requests
   - Review Application tab for storage issues

### Resolution

1. **For JavaScript errors:**
   ```javascript
   // Check browser console
   // Typical errors to look for:
   - TypeError: Cannot read property 'x' of undefined
   - ReferenceError: variable is not defined
   - SyntaxError: Unexpected token
   ```

2. **For API call failures:**
   ```bash
   # Check backend availability
   curl -I http://localhost:3100/api/students
   
   # Check backend logs
   docker logs int531-demo-app-1 --tail 100
   
   # Test specific API endpoints
   curl http://localhost:3100/api/students
   ```

3. **For browser compatibility issues:**
   - Check if errors are browser-specific
   - Test in multiple browsers
   - Review browser support matrix
   - Update polyfills if needed

4. **For network failures:**
   ```bash
   # Check CORS configuration
   # Verify API endpoints
   # Check network connectivity
   # Review Content Security Policy
   ```

5. **For state management errors:**
   - Check Redux/state management errors
   - Verify data flow
   - Review component lifecycle
   - Check for race conditions

6. **Rollback if needed:**
   ```bash
   # If errors started after deployment
   cd /home/sysadmin/int531-demo
   git log -5
   git checkout <last_good_commit>
   docker compose build app
   docker compose up -d app
   ```

### Prevention
- Comprehensive error handling
- Error boundary components (React)
- Browser compatibility testing
- Automated frontend testing
- Error tracking (Sentry, LogRocket)
- Code reviews and linting
- Type checking (TypeScript)

---

## CriticalFrontendErrorRate

**Severity:** Critical  
**Threshold:** Frontend error rate > 10% for 2 minutes  
**Golden Signal:** Errors

### Description
Frontend is experiencing critical failure rates, affecting majority of users.

### Impact
- Severe user experience impact
- Feature unavailability
- Potential application crash
- Business disruption

### Diagnosis Steps

1. **Immediate check:**
   ```bash
   # Test application
   curl -I http://localhost:3100/
   
   # Open browser and check console
   # F12 -> Console tab
   ```

2. **Check recent deployments:**
   ```bash
   cd /home/sysadmin/int531-demo
   git log -1
   ```

### Resolution

**IMMEDIATE ACTION REQUIRED:**

1. **If recent deployment:**
   ```bash
   # Immediate rollback
   cd /home/sysadmin/int531-demo
   git log -5
   git checkout <last_good_commit>
   docker compose build app
   docker compose up -d app
   
   # Verify fix
   curl http://localhost:3100/
   ```

2. **If backend issues:**
   ```bash
   # Check backend health
   curl http://localhost:3100/api/students
   
   # Restart backend if needed
   docker compose restart app
   ```

3. **If build/bundle issues:**
   ```bash
   # Rebuild application
   cd /home/sysadmin/int531-demo
   docker compose build app --no-cache
   docker compose up -d app
   ```

4. **Emergency communication:**
   - Notify on-call team immediately
   - Update status page
   - Prepare incident communication
   - Document all actions taken

5. **Enable maintenance mode:**
   - Display user-friendly error page
   - Return 503 status with Retry-After
   - Provide status updates

### Prevention
- Staged deployments (canary, blue-green)
- Automated rollback on high error rates
- Comprehensive testing before deployment
- Feature flags for gradual rollouts
- Real-time error monitoring
- Automated alerts to on-call

---

# Latency Signals

## HighPageLoadTime

**Severity:** Warning  
**Threshold:** Average page load time > 3 seconds for 5 minutes  
**Golden Signal:** Latency

### Description
Pages are loading slowly, negatively impacting user experience and potentially increasing bounce rates.

### Impact
- Poor user experience
- Increased bounce rates
- Lower conversion rates
- User frustration
- SEO impact

### Diagnosis Steps

1. **Check page load time:**
   ```promql
   rate(frontend_page_load_duration_seconds_sum[5m]) /
   rate(frontend_page_load_duration_seconds_count[5m])
   ```

2. **Check per-page breakdown:**
   ```promql
   rate(frontend_page_load_duration_seconds_sum[5m]) /
   rate(frontend_page_load_duration_seconds_count[5m])
   by (page)
   ```

3. **Use browser DevTools:**
   - Open DevTools (F12)
   - Go to Network tab
   - Reload page
   - Check waterfall view
   - Identify slow resources

4. **Check backend API latency:**
   ```promql
   rate(frontend_api_duration_seconds_sum[5m]) /
   rate(frontend_api_duration_seconds_count[5m])
   ```

5. **Check resource timing:**
   ```javascript
   // In browser console
   performance.getEntriesByType('navigation')[0]
   performance.getEntriesByType('resource')
   ```

### Resolution

1. **For slow API calls:**
   ```bash
   # Check backend latency
   time curl http://localhost:3100/api/students
   
   # Check backend metrics
   curl http://localhost:3100/api/metrics | grep http_request_duration
   
   # See backend runbook if needed
   ```

2. **For large bundle sizes:**
   ```bash
   # Check bundle size
   cd /home/sysadmin/int531-demo
   npm run build
   
   # Analyze bundle
   # Implement code splitting
   # Use dynamic imports
   # Remove unused dependencies
   ```

3. **For slow resources:**
   - Optimize images (compression, WebP format)
   - Enable lazy loading
   - Use CDN for static assets
   - Implement resource hints (preload, prefetch)

4. **For rendering issues:**
   ```javascript
   // Check in browser DevTools -> Performance tab
   // Look for:
   - Long tasks (> 50ms)
   - Layout thrashing
   - Excessive re-renders
   ```

5. **Optimize React components:**
   ```typescript
   // Use React.memo for expensive components
   // Implement useMemo and useCallback
   // Optimize re-renders
   // Use virtual scrolling for large lists
   ```

6. **Enable caching:**
   ```typescript
   // Implement service worker
   // Use browser caching
   // Cache API responses
   // Use SWR or React Query
   ```

### Prevention
- Regular performance audits
- Lighthouse CI integration
- Performance budgets
- Code splitting and lazy loading
- Image optimization pipeline
- CDN usage
- Monitoring Core Web Vitals

---

## CriticalPageLoadTime

**Severity:** Critical  
**Threshold:** Average page load time > 10 seconds for 2 minutes  
**Golden Signal:** Latency

### Description
Pages are loading extremely slowly, causing severe user experience degradation and high abandonment.

### Impact
- Severe UX degradation
- High bounce rate
- User abandonment
- Business impact
- Revenue loss

### Diagnosis Steps

1. **Immediate check:**
   ```bash
   # Test page load
   time curl http://localhost:3100/
   
   # Check backend
   time curl http://localhost:3100/api/students
   ```

2. **Browser test:**
   - Open page in browser
   - Check DevTools Network tab
   - Identify blocking resources

### Resolution

**IMMEDIATE ACTION REQUIRED:**

1. **Check backend performance:**
   ```bash
   # Check backend latency
   curl http://localhost:3100/api/metrics | grep http_request_duration
   
   # Check backend health
   docker stats int531-demo-app-1
   
   # Restart if needed
   docker compose restart app
   ```

2. **Check for network issues:**
   ```bash
   # Test connectivity
   ping localhost
   
   # Check Docker network
   docker network inspect int531-demo_default
   ```

3. **Emergency optimizations:**
   - Enable aggressive caching
   - Disable non-critical features
   - Reduce bundle size
   - Defer non-critical resources

4. **Scale backend:**
   ```bash
   docker compose up -d --scale app=2
   ```

5. **Consider maintenance mode:**
   - If unable to resolve quickly
   - Display lightweight maintenance page
   - Provide ETA for resolution

### Prevention
- Performance monitoring alerts
- Automated performance testing
- Load testing infrastructure
- CDN failover
- Performance SLOs

---

## HighFrontendAPILatency

**Severity:** Warning  
**Threshold:** P95 API latency > 1 second for 5 minutes  
**Golden Signal:** Latency

### Description
Frontend API calls are taking longer than expected, impacting user interactions and data operations.

### Impact
- Slow user interactions
- Degraded feature performance
- Increased timeouts
- Poor UX

### Diagnosis Steps

1. **Check API latency:**
   ```promql
   histogram_quantile(0.95,
     rate(frontend_api_duration_seconds_bucket[5m])
   )
   ```

2. **Check per-endpoint breakdown:**
   ```promql
   histogram_quantile(0.95,
     rate(frontend_api_duration_seconds_bucket[5m])
   ) by (endpoint)
   ```

3. **Test API directly:**
   ```bash
   # Test endpoints
   time curl http://localhost:3100/api/students
   time curl http://localhost:3100/api/students/1
   ```

4. **Check browser Network tab:**
   - Open DevTools (F12)
   - Go to Network tab
   - Filter by XHR/Fetch
   - Check timing for API calls

5. **Check backend metrics:**
   ```bash
   curl http://localhost:3100/api/metrics | grep http_request_duration
   ```

### Resolution

1. **If backend latency:**
   ```bash
   # See backend runbook for resolution
   # Check backend alerts
   # Verify database performance
   ```

2. **Optimize API calls:**
   ```typescript
   // Implement request debouncing
   // Use pagination
   // Batch API requests
   // Cache responses
   ```

3. **Implement client-side optimization:**
   ```typescript
   // Use SWR or React Query for caching
   import useSWR from 'swr'
   
   const { data } = useSWR('/api/students', fetcher, {
     revalidateOnFocus: false,
     dedupingInterval: 10000
   })
   
   // Implement optimistic updates
   // Use local state when possible
   ```

4. **Add timeout handling:**
   ```typescript
   const fetchWithTimeout = async (url: string, timeout = 5000) => {
     const controller = new AbortController()
     const id = setTimeout(() => controller.abort(), timeout)
     
     try {
       const response = await fetch(url, { signal: controller.signal })
       clearTimeout(id)
       return response
     } catch (error) {
       // Handle timeout
     }
   }
   ```

5. **Implement retry logic:**
   ```typescript
   const fetchWithRetry = async (url: string, retries = 3) => {
     for (let i = 0; i < retries; i++) {
       try {
         return await fetch(url)
       } catch (error) {
         if (i === retries - 1) throw error
         await new Promise(r => setTimeout(r, 1000 * Math.pow(2, i)))
       }
     }
   }
   ```

### Prevention
- API response caching
- Request optimization
- Backend performance monitoring
- Timeout and retry policies
- Progressive loading
- Skeleton screens

---

# Saturation Signals

## HighBrowserMemoryUsage

**Severity:** Warning  
**Threshold:** Browser memory usage > 500MB for 5 minutes  
**Golden Signal:** Saturation

### Description
Browser is consuming excessive memory, which may lead to slowdowns, crashes, or tab crashes.

### Impact
- Browser slowdowns
- Potential tab crashes
- Poor performance on low-end devices
- Increased memory pressure
- User frustration

### Diagnosis Steps

1. **Check memory metrics:**
   ```promql
   frontend_browser_memory_bytes / 1024 / 1024
   ```

2. **Browser DevTools Memory Profiling:**
   ```javascript
   // In browser console
   console.log(performance.memory)
   
   // Check:
   // - usedJSHeapSize
   // - totalJSHeapSize
   // - jsHeapSizeLimit
   ```

3. **Take heap snapshot:**
   - Open DevTools (F12)
   - Go to Memory tab
   - Take heap snapshot
   - Analyze retained objects
   - Look for detached DOM nodes

4. **Check for memory leaks:**
   - Take multiple snapshots over time
   - Compare snapshots
   - Identify growing objects
   - Look for event listeners not cleaned up

5. **Profile memory allocation:**
   - DevTools -> Performance tab
   - Record with memory checkbox
   - Perform actions
   - Check memory timeline

### Resolution

1. **Identify memory leaks:**
   ```javascript
   // Common causes:
   // - Event listeners not removed
   // - Closures holding references
   // - Detached DOM nodes
   // - Global variables
   // - Timers not cleared
   ```

2. **Fix React component leaks:**
   ```typescript
   useEffect(() => {
     const handler = () => { /* ... */ }
     window.addEventListener('resize', handler)
     
     // Cleanup!
     return () => {
       window.removeEventListener('resize', handler)
     }
   }, [])
   
   // Clear timers
   useEffect(() => {
     const timer = setInterval(() => { /* ... */ }, 1000)
     return () => clearInterval(timer)
   }, [])
   ```

3. **Optimize component rendering:**
   ```typescript
   // Use React.memo
   const ExpensiveComponent = React.memo(({ data }) => {
     // Component logic
   })
   
   // Cleanup refs
   useEffect(() => {
     return () => {
       // Clear refs
     }
   }, [])
   ```

4. **Reduce memory footprint:**
   - Implement virtualization for large lists
   - Unload images when not visible
   - Clear caches periodically
   - Remove unused data from state

5. **For immediate relief:**
   ```javascript
   // Suggest page reload to users
   // Clear local storage/session storage
   localStorage.clear()
   sessionStorage.clear()
   ```

### Prevention
- Regular memory leak testing
- Proper cleanup in useEffect
- Avoid memory leaks in event listeners
- Use weak references where appropriate
- Implement data pagination
- Virtual scrolling for large lists
- Regular code reviews for leaks
- Memory profiling in development

---

## General Frontend Debugging

### Browser DevTools

**Console Tab:**
```javascript
// Check for errors
// View console.log output
// Test JavaScript code

// Useful commands:
console.table(data)
console.time('operation')
console.timeEnd('operation')
performance.now()
```

**Network Tab:**
```
// Check API calls
// View timing waterfall
// Check request/response
// Monitor WebSocket connections

// Filter by:
- XHR/Fetch (API calls)
- JS (JavaScript files)
- CSS (Stylesheets)
- Img (Images)
```

**Performance Tab:**
```
// Record performance profile
// Check FPS
// Identify long tasks
// Check memory usage
// View paint events
```

**Application Tab:**
```
// Check Local Storage
// Check Session Storage
// Check Cookies
// Check Service Workers
// Check Cache Storage
```

### Performance Metrics

```javascript
// In browser console

// Navigation Timing
const perfData = performance.getEntriesByType('navigation')[0]
console.log('DNS:', perfData.domainLookupEnd - perfData.domainLookupStart)
console.log('TCP:', perfData.connectEnd - perfData.connectStart)
console.log('Request:', perfData.responseStart - perfData.requestStart)
console.log('Response:', perfData.responseEnd - perfData.responseStart)
console.log('DOM Processing:', perfData.domComplete - perfData.domLoading)

// Core Web Vitals (if available)
new PerformanceObserver((entryList) => {
  for (const entry of entryList.getEntries()) {
    console.log('LCP:', entry.renderTime || entry.loadTime)
  }
}).observe({entryTypes: ['largest-contentful-paint']})

// Memory
if (performance.memory) {
  console.log('Memory:', {
    used: Math.round(performance.memory.usedJSHeapSize / 1048576) + ' MB',
    total: Math.round(performance.memory.totalJSHeapSize / 1048576) + ' MB'
  })
}
```

### Testing Frontend

```bash
# Access application
curl -I http://localhost:3100/

# Check frontend metrics endpoint
curl http://localhost:3100/api/frontend-metrics

# Test API from frontend perspective
curl http://localhost:3100/api/students

# Check Docker container
docker logs int531-demo-app-1 --tail 50

# Restart if needed
docker compose restart app
```

### Common Issues

**Page Won't Load:**
1. Check backend is running
2. Check browser console for errors
3. Check network tab for failed requests
4. Verify API endpoints

**JavaScript Errors:**
1. Check browser console
2. Verify browser compatibility
3. Check for syntax errors
4. Review recent code changes

**Slow Performance:**
1. Check Network tab timing
2. Profile with Performance tab
3. Check backend API latency
4. Review bundle size

**Memory Issues:**
1. Take heap snapshot
2. Check for memory leaks
3. Profile memory allocation
4. Review component cleanup

---

## Escalation

If issues cannot be resolved:

1. **Gather diagnostics:**
   ```bash
   # Screenshots of:
   - Browser DevTools Console
   - Browser DevTools Network tab
   - Browser DevTools Performance profile
   - Grafana Frontend Dashboard
   
   # Export:
   - Heap snapshot (.heapsnapshot file)
   - Performance profile (.json file)
   - HAR file (Network -> Export HAR)
   
   # Logs:
   docker logs int531-demo-app-1 > app-logs.txt
   
   # Metrics:
   curl http://localhost:3100/api/frontend-metrics > frontend-metrics.txt
   ```

2. **Document incident:**
   - User reports and screenshots
   - Browser and OS information
   - Steps to reproduce
   - Timeline of events
   - Actions taken
   - Current impact

3. **Contact:**
   - Frontend development team
   - Backend team (if API issues)
   - On-call engineer
   - UX team (for user impact assessment)

---

## Additional Resources

### Performance Tools
- Chrome DevTools
- Lighthouse
- WebPageTest
- GTmetrix
- React DevTools Profiler

### Monitoring
- Real User Monitoring (RUM)
- Sentry for error tracking
- LogRocket for session replay
- Google Analytics for user behavior

### Best Practices
- [Web Vitals](https://web.dev/vitals/)
- [React Performance](https://react.dev/learn/render-and-commit)
- [Next.js Performance](https://nextjs.org/docs/app/building-your-application/optimizing)

---

**Last Updated:** December 18, 2025  
**Maintained By:** Frontend Engineering Team  
**Golden Signals Reference:** [Google SRE Book - Chapter 6](https://sre.google/sre-book/monitoring-distributed-systems/)
