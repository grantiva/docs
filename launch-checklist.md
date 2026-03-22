# Pre-Show HN Launch Checklist

Launch Date: **March 31, 2026, 9:00-10:00 AM ET**

This checklist ensures Grantiva is ready for the Show HN traffic spike.

## Infrastructure

- [ ] **Railway scaled to 4 replicas** ([GRA-1226](/GRA/issues/GRA-1226))
  - Service: `backend`
  - Command: `railway scale --service backend --replicas 4`
  - Verify: Check Railway dashboard shows 4 active replicas

- [ ] **Database connection pool sized for 4 replicas**
  - Current pool size per replica: 5 connections
  - Total capacity: 20 connections (4 replicas × 5)
  - PostgreSQL max_connections: 100 (Railway default)
  - Recommended: Set `DATABASE_MAX_CONNECTIONS=10` per replica = 40 total
  - Command: `railway variables --service backend --set DATABASE_MAX_CONNECTIONS=10`

- [ ] **Redis maxmemory policy set to `allkeys-lru`** ([GRA-1353](/GRA/issues/GRA-1353))
  - Verify: `railway run --service backend redis-cli CONFIG GET maxmemory-policy`
  - Expected: `allkeys-lru`
  - Set if needed: `railway run --service backend redis-cli CONFIG SET maxmemory-policy allkeys-lru`

- [ ] **Sentry DSN configured** ([GRA-1224](/GRA/issues/GRA-1224))
  - Env var: `SENTRY_DSN`
  - Verify: Check Railway variables for backend service
  - Test: Trigger test error and verify it appears in Sentry dashboard

## Performance & Capacity

- [ ] **Rate limits reviewed and tuned**
  - Current: 100 requests per 60 seconds per IP
  - For launch: Increase to 200 requests per 60 seconds
  - Command: `railway variables --service backend --set RATE_LIMIT_MAX=200`
  - Monitor: Watch for 429 errors in logs

- [ ] **Load test completed** ([GRA-1351](/GRA/issues/GRA-1351))
  - Target: 1000 concurrent users
  - Verify: Backend can handle expected traffic without degradation
  - Document: Results documented in [GRA-1351](/GRA/issues/GRA-1351)

- [ ] **CDN cache settings optimized**
  - Cloudflare caching rules configured
  - Static assets cached at edge
  - API responses cached appropriately (bypass for dynamic endpoints)

## Monitoring & Observability

- [ ] **Railway metrics dashboard open and ready**
  - URL: https://railway.app/project/{project-id}/metrics
  - Metrics to watch:
    - CPU usage (alert if >80%)
    - Memory usage (alert if >85%)
    - Request latency (alert if p95 >500ms)
    - Error rate (alert if >5%)

- [ ] **Sentry dashboard configured** (after GRA-1224)
  - URL: [Sentry Dashboard URL]
  - Alerts configured for:
    - New error types
    - Error rate spikes (>10 errors/min)
    - Performance degradation (>1s transaction time)

- [ ] **Cloudflare analytics ready**
  - URL: https://dash.cloudflare.com/
  - Monitor:
    - Total requests
    - Cache hit rate
    - Bandwidth usage
    - Threat score

- [ ] **Database monitoring**
  - Railway PostgreSQL metrics available
  - Watch:
    - Active connections
    - Query performance
    - Disk usage
    - Replication lag (if applicable)

- [ ] **Redis monitoring**
  - Railway Redis metrics available
  - Watch:
    - Memory usage
    - Hit/miss rate
    - Connected clients
    - Eviction count

## Rollback & Recovery

- [ ] **Rollback tested in dev environment** ([GRA-1352](/GRA/issues/GRA-1352))
  - Test: `railway rollback --service backend` in dev
  - Verify: Dev environment reverts to previous deploy
  - Verify: No data loss or corruption
  - Document: Test results in [GRA-1352](/GRA/issues/GRA-1352)

- [ ] **Database backup verified**
  - Railway automatic backups enabled
  - Latest backup timestamp confirmed
  - Backup restoration procedure documented
  - Test restoration in dev environment (optional)

- [ ] **Incident response runbook created**
  - Location: `docs/incident-response.md`
  - Team reviewed and familiar with procedures
  - Emergency contacts verified

## Team Readiness

- [ ] **Team notified of launch window**
  - Date/Time: March 31, 2026, 9:00-10:00 AM ET
  - All engineers on standby during launch window
  - Slack #incidents channel monitored
  - Phone numbers exchanged for emergency contact

- [ ] **On-call rotation defined**
  - Primary: CTO (Jordan)
  - Secondary: Backend Engineer (Sam)
  - Tertiary: Full Stack Engineer (Kai)

- [ ] **Runbook reviewed by team**
  - All engineers have read `incident-response.md`
  - Questions answered
  - Dry run completed (optional but recommended)

## Application Readiness

- [ ] **Health check endpoint working**
  - URL: https://api.grantiva.io/health
  - Expected: 200 OK with system status
  - Monitors: Database, Redis, memory, disk

- [ ] **Rate limiting working correctly**
  - Test: Send burst of requests
  - Verify: 429 responses after threshold
  - Verify: Rate limit headers present

- [ ] **Attestation flow tested end-to-end**
  - Test device: Real iOS device (not simulator)
  - Steps:
    1. Request challenge
    2. Complete attestation
    3. Receive JWT token
    4. Make authenticated request
  - Verify: No errors, reasonable latency (<500ms)

- [ ] **Dashboard accessible and functional**
  - URL: https://grantiva.io
  - Test: Login, view stats, navigate features
  - Verify: No JavaScript errors in console
  - Verify: Reasonable load time (<2s)

## Content & Communication

- [ ] **Show HN post prepared**
  - Title finalized
  - Description drafted
  - Launch URL ready: https://grantiva.io
  - Submission timing planned (9:00 AM ET)

- [ ] **Documentation up-to-date**
  - README.md current
  - API documentation accurate
  - Quickstart guide tested
  - Example code working

- [ ] **Support channels ready**
  - Email: support@grantiva.io configured and monitored
  - HN comment thread will be monitored
  - Response templates prepared for common questions

## Security

- [ ] **Secrets rotated and secured**
  - Admin API key: 32+ characters, stored in Railway secrets
  - Database password: Strong, rotated recently
  - Redis password: Set and secured
  - No secrets in git history

- [ ] **SSL certificates valid**
  - Verify: `echo | openssl s_client -servername grantiva.io -connect grantiva.io:443 2>/dev/null | openssl x509 -noout -dates`
  - Expected: Valid until at least June 2026
  - Renewal: Cloudflare auto-renews

- [ ] **CORS configured correctly**
  - Allowed origins: `https://grantiva.io`, `https://dev.grantiva.io`
  - No wildcard `*` in production
  - Verify: Test cross-origin requests

## Final Checks (T-1 Hour)

**Completed 1 hour before launch (8:00 AM ET):**

- [ ] All services healthy (green status in Railway)
- [ ] No active incidents or alerts
- [ ] Team in Slack #launch channel
- [ ] Monitoring dashboards open on second screen
- [ ] Coffee ready ☕

## Launch Time (9:00 AM ET)

- [ ] Submit Show HN post
- [ ] Post link in internal Slack
- [ ] Begin active monitoring
- [ ] Respond to HN comments
- [ ] Watch metrics dashboards

## Post-Launch (T+2 Hours)

**Review 2 hours after launch (11:00 AM ET):**

- [ ] Review traffic metrics
- [ ] Check error rates
- [ ] Read HN comment feedback
- [ ] Document any incidents (if applicable)
- [ ] Team debrief scheduled (same day)

---

## Key URLs Reference

### Production
- **Backend API:** https://api.grantiva.io
- **Web Dashboard:** https://grantiva.io
- **Health Check:** https://api.grantiva.io/health

### Development
- **Dev API:** https://dev-api.grantiva.io
- **Dev Web:** https://dev.grantiva.io

### Infrastructure
- **Railway Dashboard:** https://railway.app/project/{project-id}
- **Cloudflare Dashboard:** https://dash.cloudflare.com/
- **Sentry Dashboard:** [URL once GRA-1224 complete]

### Documentation
- **Incident Response:** `docs/incident-response.md`
- **API Docs:** https://grantiva.io/docs

---

**Last Updated:** 2026-03-22
**Owner:** CTO (Jordan)
**Related:** [GRA-1216](/GRA/issues/GRA-1216), [GRA-1350](/GRA/issues/GRA-1350)
