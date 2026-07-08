# Deployment Timeline

## Recent Activity (2024-07-08)

### 13:45 UTC - Deployment #2847 Started
**Status:** In Progress
**Engineer:** sarah.chen@company.com
**Change:** Payment Service v2.4.1 Release

```
Deploy start: 13:45:00 UTC
Rolling deployment: 4 instances
- Instance 1: OK (13:47)
- Instance 2: OK (13:48)
- Instance 3: OK (13:49)
- Instance 4: OK (13:51)
Deploy complete: 13:52:00 UTC
```

**What was deployed:**
- New payment retry logic with exponential backoff
- Improved error handling for failed transactions
- Database connection optimization (SEE git_diff.md)
- Performance improvements for high-volume scenarios

---

### 14:15 UTC - Post-Deployment Health Check
**Status:** PASSED ✓
- API response time: 150ms (normal)
- Error rate: 0.02% (normal)
- Database connections: 24/50 (normal)
- Transaction throughput: 847 tx/min (expected)

---

### 14:30 UTC - Promotional Campaign Launch
**Note:** Marketing launched "Flash Sale Friday" promotion
- Expected traffic: 1.5x normal volume
- Campaign email sent to 500k customers
- Social media push active

**System state before crash:**
- API response time: 180ms (slight increase, expected)
- Error rate: 0.03% (normal)
- Database connections: 41/50 (high but within limits)
- Transaction throughput: 1,240 tx/min (expected for campaign)

---

### 14:32:15 UTC - 🚨 FAILURE
**Status:** CRITICAL ERROR
- Payment service returns 503 error
- Database connection pool exhausted
- Queue backlog begins accumulating

---

### 14:32 - 15:17 UTC - Incident Duration (45 minutes)
**Status:** Investigating and remediating
- On-call engineer paged at 14:32
- Incident commander engaged at 14:35
- Root cause identified at 14:55
- Rollback decision made at 15:10
- Deployment #2847 rolled back at 15:12
- Service restored at 15:17

---

## Key Timeline Points

| Time | Event | Note |
|------|-------|------|
| 13:52 | Deployment complete | v2.4.1 live |
| 14:15 | Health check passed | All metrics normal |
| 14:30 | Traffic surge begins | Campaign launch |
| 14:32 | FAILURE | Connections exhausted |
| 14:35 | Incident commander engaged | Investigation starts |
| 14:55 | Root cause identified | Connection leak found |
| 15:10 | Rollback decision | Revert to v2.4.0 |
| 15:12 | Rollback deployed | v2.4.0 active |
| 15:17 | Service restored | Incident closed |

---

## Questions for Investigation

1. **Why did the deployment pass health checks but fail under load?**
2. **What changed in the connection handling code?** (See git_diff.md)
3. **Why does the error rate spike align with the campaign launch?**
4. **Was the issue with the new code or the configuration?**
5. **Could we have detected this earlier?**

---

**Next: Review `error_logs.json` to see what errors appeared**