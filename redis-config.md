# Redis Configuration

Grantiva uses Railway-managed Redis for session storage, rate limiting, and live device count caching.

## Configuration Baseline

Audited 2026-03-22 (Morgan, DevOps).

| Parameter          | Value         | Notes                                           |
| ------------------ | ------------- | ----------------------------------------------- |
| `maxmemory`        | 0 (unlimited) | Railway enforces container limits externally     |
| `maxmemory-policy` | `volatile-lru` | Set at startup by configure.swift on every boot |
| Current usage      | ~1.88 MB      | Well under typical Railway container limits      |
| Peak (7 days)      | ~1.93 MB      | No pressure at current traffic levels           |
| Key count          | ~875          | All `vrs-` session keys, all have 24h TTL       |

## Eviction Policy

**Policy**: `volatile-lru` — evicts least-recently-used keys that have a TTL set.

This ensures:
- Expired/idle sessions are evicted before active ones
- Rate limit counters (60s TTL) are evicted first
- Keys without TTL are never silently evicted

Railway Redis runs without a `redis.conf` so `CONFIG REWRITE` is unavailable. The backend re-applies the policy at every startup via configure.swift (line ~88). This means a Redis restart followed by a backend restart will restore the correct policy.

## Session Keys

All session keys use the prefix `vrs-` (Vapor Redis Sessions default).

- **TTL**: 24 hours (86400 seconds)
- **Key count**: ~875 as of 2026-03-22 baseline
- **TTL applied to existing keys**: 2026-03-22 via Lua script hotfix
- **Permanent fix**: `TTLSessionsDelegate` in configure.swift uses `SETEX` — all new sessions receive 24h TTL automatically (landed in [GRA-1353](/GRA/issues/GRA-1353))

## Monitoring Procedures

### Daily (before launch — until March 31)

```bash
REDIS_PUBLIC_URL="redis://default:<password>@shinkansen.proxy.rlwy.net:54164"

# Memory usage
redis-cli -u "$REDIS_PUBLIC_URL" INFO memory | grep used_memory_human

# Eviction policy check
redis-cli -u "$REDIS_PUBLIC_URL" CONFIG GET maxmemory-policy

# Key count
redis-cli -u "$REDIS_PUBLIC_URL" DBSIZE

# Keys without TTL (should return empty)
redis-cli -u "$REDIS_PUBLIC_URL" EVAL \
  "local keys = redis.call('KEYS', '*') local no_ttl = {} for _, key in ipairs(keys) do if redis.call('TTL', key) == -1 then table.insert(no_ttl, key) end end return no_ttl" 0
```

Alert if:
- Memory usage > 80% of Railway container limit
- Eviction policy is not `volatile-lru`
- Keys without TTL appear (run hotfix below)

### During Launch (March 31, 9–11am ET)

Monitor from Railway dashboard:
- Redis → Metrics → Memory usage
- Watch for `evicted_keys` spike in `INFO stats`

```bash
# Watch eviction count
redis-cli -u "$REDIS_PUBLIC_URL" INFO stats | grep evicted_keys
```

## Emergency Procedures

### If eviction policy reverts to `noeviction`

```bash
redis-cli -u "$REDIS_PUBLIC_URL" CONFIG SET maxmemory-policy volatile-lru
```

The backend will also re-apply this on the next restart.

### If session keys appear with TTL=-1

```bash
# Set 24h TTL on all session keys
redis-cli -u "$REDIS_PUBLIC_URL" EVAL \
  "local keys = redis.call('KEYS', 'vrs-*') local count = 0 for _, key in ipairs(keys) do redis.call('EXPIRE', key, 86400) count = count + 1 end return count" 0
```

### If Redis fills up and writes are rejected

1. Check `INFO memory` — confirm usage is at limit
2. Flush old sessions (safe — users will be logged out):
   ```bash
   redis-cli -u "$REDIS_PUBLIC_URL" EVAL \
     "return redis.call('DEL', unpack(redis.call('KEYS', 'vrs-*')))" 0
   ```
3. Users will re-authenticate on next dashboard visit (sessions fall back to DB)

### If Redis is completely down

- Session fallback: configure.swift lines 88–92 automatically fall back to database sessions (slower but functional)
- Rate limiting: fails open (no rate limiting) — manually enable Cloudflare rate limiting rules if needed

## Related

- [GRA-1353](/GRA/issues/GRA-1353) — Redis configuration audit + permanent session TTL fix
- [GRA-1350](/GRA/issues/GRA-1350) — Incident response runbook
