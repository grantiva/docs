# Incident Response Runbook

This document provides procedures for responding to production incidents during the Show HN launch (March 31, 2026) and beyond.

## Detection & Triage

### How to Detect Outages

1. **Railway Metrics Dashboard**
   - URL: https://railway.app/project/{project-id}/metrics
   - Monitor: CPU usage, memory, request latency, error rates
   - Alert threshold: >500ms p95 latency or >5% error rate

2. **Sentry Alerts** (once GRA-1224 is complete)
   - Real-time error tracking and performance monitoring
   - Alert on: new error types, error rate spikes, performance degradation

3. **User Reports**
   - HN comment thread monitoring
   - Email support@grantiva.io
   - Twitter/social media mentions

### Severity Classification

- **P0 (Critical)** — Site completely down, attestation API non-functional, database unreachable
  - Response time: Immediate
  - All hands on deck

- **P1 (High)** — Degraded performance, intermittent failures, some features broken
  - Response time: Within 15 minutes
  - CTO notified

- **P2 (Minor)** — UI glitches, non-critical features affected, minor performance issues
  - Response time: Within 1 hour
  - Can be addressed during normal business hours

### First Responder Actions

When an incident is detected:

1. **Assess severity** using classification above
2. **Check Railway logs**
   ```bash
   railway logs --service backend --tail 100
   ```
3. **Check database connections**
   ```bash
   railway run --service backend psql $DATABASE_URL -c "SELECT count(*) FROM pg_stat_activity;"
   ```
4. **Check Redis status**
   ```bash
   railway run --service backend redis-cli INFO memory
   ```
5. **Post in Slack #incidents** with initial status

## Common Failure Scenarios

### Database Connection Pool Exhausted

**Symptoms:**
- "Connection timeout" errors in logs
- `pg_stat_activity` shows many idle connections
- Requests hang or time out

**Immediate Actions:**
1. Check active connections:
   ```sql
   SELECT count(*), state FROM pg_stat_activity GROUP BY state;
   ```
2. Kill long-running queries:
   ```sql
   SELECT pg_terminate_backend(pid) FROM pg_stat_activity
   WHERE state = 'active' AND query_start < now() - interval '5 minutes';
   ```
3. Increase connection pool size via env var:
   ```bash
   railway variables --service backend --set DATABASE_MAX_CONNECTIONS=20
   railway restart --service backend
   ```

**Root Cause Investigation:**
- Check for connection leaks in code
- Review slow query logs
- Consider adding connection pool monitoring

### Redis Memory Full

**Symptoms:**
- "OOM command not allowed" errors
- Session management failures
- Cache misses

**Immediate Actions:**
1. Check memory usage:
   ```bash
   railway run --service backend redis-cli INFO memory
   ```
2. Check eviction policy (should be `allkeys-lru`):
   ```bash
   railway run --service backend redis-cli CONFIG GET maxmemory-policy
   ```
3. If policy is wrong, set it:
   ```bash
   railway run --service backend redis-cli CONFIG SET maxmemory-policy allkeys-lru
   ```
4. Flush old sessions if necessary (only as last resort):
   ```bash
   railway run --service backend redis-cli FLUSHDB
   ```

**Root Cause Investigation:**
- Review session TTL settings (should be 24h)
- Check for memory leaks in session data
- Consider increasing Redis memory allocation

### Rate Limit Too Aggressive

**Symptoms:**
- "Rate limit exceeded" errors
- Legitimate traffic being blocked
- Spike in 429 responses

**Immediate Actions:**
1. Temporarily increase rate limits via env var:
   ```bash
   railway variables --service backend --set RATE_LIMIT_WINDOW=60 RATE_LIMIT_MAX=200
   railway restart --service backend
   ```
2. Monitor traffic patterns
3. Review IP-based vs token-based rate limiting

**Long-term Fix:**
- Implement tiered rate limits based on user type
- Add burst allowance for legitimate traffic spikes
- Consider using Redis-based distributed rate limiting

### Out of Memory (OOM)

**Symptoms:**
- Replicas crashing and restarting
- "Cannot allocate memory" errors
- Railway shows memory at 100%

**Immediate Actions:**
1. Restart affected replicas:
   ```bash
   railway restart --service backend
   ```
2. Check for memory leaks in logs (look for increasing memory over time)
3. Scale horizontally if needed:
   ```bash
   railway scale --service backend --replicas 6
   ```
4. Review recent code changes for memory-intensive operations

**Root Cause Investigation:**
- Profile memory usage with Instruments or similar
- Check for unbounded data structures
- Review caching strategies

### SSL/CDN Issues

**Symptoms:**
- Certificate errors
- Cloudflare errors (522, 524)
- Slow page loads

**Immediate Actions:**
1. Check Cloudflare status: https://www.cloudflarestatus.com/
2. Verify SSL certificate expiry:
   ```bash
   echo | openssl s_client -servername grantiva.io -connect grantiva.io:443 2>/dev/null | openssl x509 -noout -dates
   ```
3. Check Cloudflare DNS settings
4. Bypass Cloudflare temporarily if needed (update DNS to point directly to Railway)

**Escalation:**
- Contact Cloudflare support for infrastructure issues
- Check Cloudflare dashboard for configuration issues

## Rollback Procedures

### Railway Rollback

To revert to the previous deploy:

```bash
# List recent deployments
railway deployments --service backend

# Rollback to previous deployment
railway rollback --service backend
```

**Important:** Rollback only affects application code, NOT database schema.

### Database Rollback (CAUTION)

**Never auto-rollback database migrations.** Migrations may contain data transforms that cannot be safely reversed.

If a migration causes issues:
1. Assess whether data has been modified
2. If only schema changes (no data loss), you may manually revert:
   ```bash
   railway run --service backend swift run App migrate --revert
   ```
3. If data has been transformed, restore from backup:
   ```bash
   # Restore from most recent backup
   railway backup restore --service postgresql --backup <backup-id>
   ```

### Feature Flag Rollback

For broken features behind feature flags:
1. Access admin dashboard: https://api.grantiva.io/admin
2. Navigate to feature flags
3. Disable the problematic flag
4. Changes take effect immediately

## Communication

### Internal Communication

**Slack #incidents Channel:**
- Post initial incident notification
- Regular status updates (every 15-30 minutes for P0/P1)
- Root cause analysis when resolved
- Post-mortem scheduling

**Template:**
```
🚨 P0 Incident: Backend API down
Status: Investigating
Impact: All attestation requests failing
Started: 2026-03-31 09:15 ET
Responders: @jordan

Updates:
- 09:16 - Checked Railway logs, seeing database connection errors
- 09:18 - Database connection pool exhausted, terminating long queries
- 09:20 - Service recovered, monitoring
```

### External Communication

**Status Page** (if implemented):
- Post incident notification
- Regular updates
- Resolution notification

**Hacker News** (for prolonged outages during Show HN):
- Post update as comment on Show HN thread
- Be transparent about issues
- Provide ETA for resolution

**Example HN Comment:**
```
Hi everyone - we're experiencing some performance issues due to
higher than expected traffic (great problem to have!). We're
working on scaling up and should be back to normal in ~15 minutes.
Thanks for your patience!
```

## Escalation

### P0 Incidents
- **CTO (Jordan)** alerted immediately via Slack DM + phone call
- All available engineers join #incidents
- Cancel non-critical meetings

### Infrastructure Issues
- **Railway Support:** https://railway.app/support
- Include: project ID, service name, error logs, time of incident

### DNS/CDN Issues
- **Cloudflare Support:** https://support.cloudflare.com/
- Include: domain, error codes, screenshot of issue

### Database Issues
- Check Railway PostgreSQL metrics first
- If persistent, contact Railway support with query logs
- Consider failover to read replica if implemented

## Post-Incident

After resolving any P0 or P1 incident:

1. **Immediate Debrief** (within 1 hour)
   - What happened
   - What was the impact
   - What fixed it

2. **Post-Mortem** (within 24 hours)
   - Timeline of events
   - Root cause analysis
   - Action items to prevent recurrence
   - Document in `docs/post-mortems/YYYY-MM-DD-incident-name.md`

3. **Follow-up Tickets**
   - Create Paperclip tickets for action items
   - Assign owners and due dates
   - Track in #incidents channel

## Quick Reference

### Key URLs
- Railway Backend: https://railway.app/project/{project-id}/service/backend
- Railway Database: https://railway.app/project/{project-id}/service/postgresql
- Railway Redis: https://railway.app/project/{project-id}/service/redis
- Cloudflare Dashboard: https://dash.cloudflare.com/
- Production API: https://api.grantiva.io
- Production Web: https://grantiva.io
- Dev API: https://dev-api.grantiva.io
- Dev Web: https://dev.grantiva.io

### Key Commands
```bash
# View logs
railway logs --service backend --tail 100

# Restart service
railway restart --service backend

# Scale replicas
railway scale --service backend --replicas 6

# Rollback
railway rollback --service backend

# Database query
railway run --service backend psql $DATABASE_URL

# Redis commands
railway run --service backend redis-cli INFO
```

### Emergency Contacts
- CTO Jordan: [Slack @jordan]
- Railway Support: support@railway.app
- Cloudflare Support: https://support.cloudflare.com/

---

**Last Updated:** 2026-03-22
**Owner:** CTO (Jordan)
**Related:** [GRA-1216](/GRA/issues/GRA-1216), [GRA-1350](/GRA/issues/GRA-1350)
