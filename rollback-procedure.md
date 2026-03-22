# Railway Rollback Procedure

**Last Updated:** 2026-03-22
**Status:** ✅ Dashboard rollback verified in dev (1m 55s) | ⚠️ CLI rollback does not exist

## Critical Discovery

**The incident response plan assumed `railway rollback` exists — it does not.**

Railway CLI (as of 2026-03-22) has **no rollback command**. Rollback to a previous deployment can only be done via:

1. **Railway Web Dashboard** (recommended for production emergencies)
2. **CLI workaround** (slower, requires rebuild)

## Method 1: Web Dashboard Rollback (Recommended)

**Duration:** ~2 minutes (verified in dev: 1m 55s, includes healthchecks, no rebuild)

### Prerequisites
- Railway dashboard access at railway.app
- Team member with project permissions

### Steps

1. Navigate to Railway dashboard: https://railway.app
2. Select project: **grantiva**
3. Select environment: **production** or **development**
4. Select service: **super-duper-disco** (backend)
5. Click **Deployments** tab (default view)
6. Scroll to find the last known-good deployment (marked as "REMOVED")
7. Click the **"..."** (three dots) menu on that deployment
8. Select **"Rollback"** from the dropdown menu
9. Confirm in the modal dialog: "Are you sure you want to revert to this deployment?"
10. Click **"Rollback"** button (red) to confirm

### Expected Behavior
- Railway creates a new deployment using the previous build artifact
- No rebuild phase (build artifact is reused)
- Deployment status: Initialization → Deploy → Healthcheck (~50-60s) → Active
- Zero-downtime switch (healthcheck-based cutover)
- Previous environment variables are restored automatically
- Total time: ~2 minutes (majority is healthcheck phase)
- Previous deployment changes from "REMOVED" to "ACTIVE" status

## Method 2: CLI Workaround (Not Recommended)

**Duration:** 3-5 minutes (requires rebuild)

### Option A: Remove + Redeploy from Git

```bash
# 1. Remove the bad deployment (CAUSES DOWNTIME)
railway down --service super-duper-disco --environment production

# 2. Checkout known-good commit
git checkout <known-good-commit-sha>

# 3. Redeploy
railway up --service super-duper-disco --environment production

# 4. Return to main branch
git checkout main
```

⚠️ **Downtime:** Service is offline between `down` and new deployment health checks passing.

### Option B: Manual Redeploy from Known-Good Commit

```bash
# 1. Checkout known-good commit
git checkout <known-good-commit-sha>

# 2. Deploy
railway up --service super-duper-disco --environment production

# 3. Return to main branch
git checkout main
```

⚠️ **Risk:** Two active deployments briefly (old + new). Railway may route traffic to either until old is manually removed.

## Railway CLI Commands Reference

### List Recent Deployments

```bash
railway deployment list \
  --service super-duper-disco \
  --environment production \
  --limit 10 \
  --json
```

**Output fields:**
- `id` - Deployment ID
- `status` - SUCCESS, REMOVED, FAILED, BUILDING, etc.
- `createdAt` - Timestamp

### Remove Latest Deployment

```bash
railway down --service super-duper-disco --environment production
```

⚠️ **Causes downtime** - service goes offline until next deployment.

### Redeploy Latest

```bash
railway deployment redeploy \
  --service super-duper-disco \
  --environment production \
  --yes
```

**Note:** Only redeploys the _latest_ deployment. Cannot target a specific previous deployment ID.

## Database Migration Rollback

### Safe Migration Pattern

All migrations should be **additive and reversible**:

✅ **Safe:**
- Add nullable column
- Add new table
- Add index
- Add enum value (at end of list)

❌ **Unsafe (breaks rollback):**
- Remove column
- Rename column
- Change column type
- Remove enum value

### Rollback with Migration Compatibility

**Scenario:** Deployment N+1 added a new nullable column via migration.

1. Rollback deployment to N via dashboard
2. **Old code (N) still works** - ignores the new column
3. Migration stays applied in database
4. No manual migration revert needed

**Scenario:** Deployment N+1 removed a column.

1. Rollback deployment to N via dashboard
2. **Old code (N) breaks** - expects the removed column
3. ❌ **Manual intervention required:**
   - Manually restore the column via SQL
   - OR accept downtime and wait for re-migration

### Migration Revert Procedure (Emergency Only)

If you must manually revert a migration:

```bash
# Connect to production database
railway connect --service postgres --environment production

# Inside psql
\d table_name  -- inspect current schema
ALTER TABLE table_name DROP COLUMN new_column;  -- revert additive change
```

⚠️ **This is destructive.** Only do this if rollback is blocked by migration incompatibility.

## Post-Rollback Verification Checklist

After any rollback, verify:

- [ ] Health checks passing: `curl https://api.grantiva.io/health`
- [ ] Login works (OAuth flow)
- [ ] Attestation flow works (test device)
- [ ] Database queries succeed (check logs for errors)
- [ ] Redis sessions work (login persists)
- [ ] No errors in Railway logs: `railway logs --service super-duper-disco --environment production`

## Incident Response Playbook Integration

**During a production incident:**

1. **Assess:** Is the issue in the latest deployment?
2. **Decision:** Rollback or forward-fix?
   - Rollback if root cause unclear or fix requires investigation
   - Forward-fix if bug is trivial and fix is ready
3. **Rollback:** Use **Method 1 (Web Dashboard)** for speed
4. **Verify:** Run post-rollback checklist
5. **Investigate:** Once stable, determine root cause
6. **Fix:** Create PR with fix + tests
7. **Deploy:** Normal deploy process (dev → QA → production)

## Development Environment Testing

### Test Procedure (Manual - Requires Dashboard)

1. Deploy canary to dev: `railway up --service super-duper-disco --environment development`
2. Verify canary: `curl https://dev-api.grantiva.io/health-canary`
3. Open Railway dashboard → grantiva → development → super-duper-disco → Deployments
4. Click "..." menu on previous deployment → "Rollback"
5. Confirm rollback in modal dialog
6. Wait for deployment to complete
7. Verify correct commit deployed: `curl https://dev-api.grantiva.io/health`
8. Measure duration (target: < 2 minutes)

**Status:** ✅ **Verified on 2026-03-22**

### Test Results (2026-03-22)

**Rollback Details:**
- Environment: `development`
- From: [GRA-1386] Fix health endpoint (commit: 0d72a019)
- To: [GRA-1343] Set ticket status to awaitingReply (commit: e956e7d5)
- Method: Railway dashboard → Deployment actions → Rollback

**Timeline:**
- Start: 20:09:52 UTC
- Healthcheck phase: Started within 17 seconds
- Completion: 20:11:47 UTC
- **Total Duration: 1 minute 55 seconds** ✅ (under 2-minute requirement)

**Verification:**
- ✅ Health endpoint responding: `{"status":"healthy","commit":"e956e7d5..."}`
- ✅ Correct commit deployed (e956e7d5 matches GRA-1343)
- ✅ API status: ok
- ✅ Database connectivity: ok (75ms latency)
- ✅ No rebuild required (reused existing build artifact)
- ✅ Zero downtime (healthcheck-based cutover)

**Key Observations:**
1. Railway dashboard rollback is **instant** - no rebuild phase
2. Rollback reuses the existing build artifact from the target deployment
3. Deployment phases: Initialization → Deploy → Healthcheck → Post-deploy
4. Build phase was **skipped** (pre-built artifact used)
5. Healthcheck phase took ~50-60 seconds (majority of rollback time)
6. Confirmation dialog prevents accidental rollbacks
7. Environment variables are preserved from the target deployment

**Conclusion:** Dashboard rollback works as documented. Duration well under the 2-minute incident response requirement.

## Recommendations

1. **Document in runbooks:** Replace all references to `railway rollback` with "Railway dashboard rollback"
2. **Practice:** Monthly rollback drill in dev environment
3. **Dashboard access:** Ensure 2+ team members have Railway project access
4. **Monitoring:** Set up alerting so we detect bad deploys before users report
5. **Alternative:** Consider using Railway's GitHub integration rollback feature (needs investigation)

## Related

- Incident Response Runbook: `docs/incident-response.md` (if exists)
- Railway Status: https://railway.statuspage.io
- Railway Docs: https://docs.railway.app/reference/deployments

---

**Action Required:** Update all incident response documentation to reflect that rollback requires dashboard access, not CLI.
