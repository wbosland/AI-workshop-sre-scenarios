# 6-Day Memory Leak Timeline

## Day 1: Tuesday 2024-07-08

### 10:30 UTC - Deployment
**Status:** Successful
**Engineer:** james.Park@company.com
**Version:** v3.2.0
**Change:** "Implement request caching to improve performance"

**What was deployed:**
- New request result caching layer
- Cache stores results in memory for 1 hour
- Bypass cache for admin requests
- Cache key: combination of URL + parameters

**Post-deployment metrics (10:45 UTC):**
- Memory: 2.0 GB (normal)
- p50 latency: 120ms
- p99 latency: 450ms
- Requests/min: 8,400
- Errors: 0.01%

**Status:** ✅ Healthy

---

## Day 2: Wednesday 2024-07-09

### Morning (06:00 UTC)
- Memory: 2.8 GB (slight increase)
- p50 latency: 130ms (normal variation)
- p99 latency: 520ms (slight increase, acceptable)

**Observation:** Nothing alarming

### Evening (18:00 UTC)
- Memory: 3.2 GB (continues increasing)
- p50 latency: 140ms
- p99 latency: 680ms

**Note in logs:** "Cache hit rate 67%, misses 33% - good performance gain"

**Status:** ✅ Acceptable (slight performance increase from caching)

---

## Day 3: Thursday 2024-07-10

### Morning (06:00 UTC)
- Memory: 4.1 GB (doubled in 2 days)
- p50 latency: 160ms
- p99 latency: 1,200ms (degrading)
- GC pause time: 250ms (was 50ms)

**First concern:** "Memory growth is steady, GC pauses getting longer"

### Afternoon (14:00 UTC)
- Memory: 4.6 GB
- p99 latency: 1,800ms

**Slack message from frontend team:** "API feels slower than yesterday. Did something change?"

**Response from ops:** "We'll look into it"

### Action taken: None
**Reason:** "Might be traffic spike, let's monitor"

**Status:** ⚠️ Watch mode

---

## Day 4: Friday 2024-07-11

### Morning (06:00 UTC)
- Memory: 5.5 GB (still growing)
- p50 latency: 220ms (noticeably slower)
- p99 latency: 2,400ms
- GC pause time: 500ms (10x worse than day 1)

**Alert triggered:** "High GC pause time > 500ms"
**Alert threshold:** Only for pauses > 500ms (too late)

### Action taken: Investigation started
**Questions asked:**
- "Did traffic change?" → No, normal levels
- "Did we deploy?" → Yes, Tuesday v3.2.0 (3 days ago)
- "Is it the caching?" → "Let's check if cache is working"

### Afternoon (15:30 UTC)
- Memory: 6.1 GB
- Decision: "Increase heap size from 8GB to 12GB to buy time"
- Action: Restart with new heap size

### Post-restart (16:00 UTC)
- Memory drops to 2.0 GB
- Latency returns to normal (p99: 450ms)
- "That fixed it. Must be something using extra memory today."

**Status:** ✅ Temporarily fixed (by increasing heap size)

---

## Day 5: Saturday 2024-07-12

### Morning (06:00 UTC)
- Memory: 2.1 GB (clean restart from yesterday)
- p99 latency: 480ms (normal)

### Afternoon (14:00 UTC)
- Memory: 6.8 GB (growing FASTER than before)
- p99 latency: 2,100ms
- GC pauses: 800ms

**Observation:** "Growing even faster now. And we just increased heap from 8GB to 12GB."

### Evening (20:00 UTC)
- Memory: 7.8 GB (approaching 8GB limit)
- Alert: "Memory usage > 80% of heap"

**Decision: Emergency restart again**
- Restart service
- Memory drops to 2.1 GB
- Latency returns to normal

**Status:** ⚠️ Issue recurring, pattern emerging

---

## Day 6: Sunday 2024-07-14 (INCIDENT DAY)

### 00:30 UTC
- Memory: 2.0 GB (clean from yesterday's restart)
- p99 latency: 450ms (normal)

### 06:00 UTC
- Memory: 5.2 GB
- p99 latency: 1,500ms
- GC pause: 400ms

### 12:00 UTC
- Memory: 8.5 GB (over 8GB, now hitting actual limit)
- p99 latency: 3,200ms (very slow)
- GC pause: 1,200ms (Full GC happening every minute)

**Alert fired:** "Memory usage > 80% of heap"
**But decision: Don't restart yet, want to investigate**

### 16:00 UTC
- Memory: 9.2 GB
- Requests backed up, queue growing
- GC pause: 2,000ms (50% of time spent in GC)

### 03:47 UTC (Next morning)
- Memory: 11.8 GB (ran out of headroom)
- **OUT OF MEMORY ERROR**
- Service crashes
- Automatic restart triggered

```
java.lang.OutOfMemoryError: Java heap space
  at java.util.Arrays.copyOf(Arrays.java:3236)
  at java.util.ArrayList.grow(ArrayList.java:268)
  at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:252)
  at java.util.ArrayList.add(ArrayList.java:459)
  at cache.CacheManager.put(CacheManager.java:156)
```

### 03:47-03:55 UTC - Downtime
- 8 minutes of complete unavailability
- ~12,000 requests dropped
- Customers see 503 errors

### 03:55 UTC - Service restarts
- Restart triggered by health check failure
- Memory drops to 2.0 GB again
- Service recovers
- Backlog of 12,000 requests starts processing

**Status:** 🚨 CRITICAL INCIDENT

---

## Summary Timeline

| Day | Time | Memory | p99 Latency | Action | Status |
|-----|------|--------|-------------|--------|--------|
| 1 | 10:30 | 2.0 GB | 450ms | Deploy v3.2.0 | ✅ |
| 2 | 18:00 | 3.2 GB | 680ms | Monitor | ✅ |
| 3 | 14:00 | 4.6 GB | 1.8s | Slack question | ⚠️ |
| 4 | 06:00 | 5.5 GB | 2.4s | Alert fires | ⚠️ |
| 4 | 16:00 | 2.0 GB | 450ms | Increase heap + restart | ✅ |
| 5 | 14:00 | 6.8 GB | 2.1s | Growing faster | ⚠️ |
| 5 | 20:00 | 7.8 GB | 2.8s | Emergency restart | ✅ |
| 6 | 03:47 | 11.8 GB | N/A | OOM Crash | 🚨 |
| 6 | 03:55 | 2.0 GB | 450ms | Auto-restart | ✅ |

---

## Key Questions for Investigation

1. **Why didn't we catch this earlier?** (Trend analysis)
2. **Why did restarting fix it temporarily?** (Memory leak keeps growing)
3. **Why did it grow faster on day 5?** (Leak acceleration or extra traffic?)
4. **What changed on day 1?** (Caching feature deployed)
5. **Is the cache the culprit?** (Memory growth = cache growth?)
6. **How much memory is the cache using?** (Heap dump analysis)
7. **Why is cache growing unbounded?** (Code review: is there cleanup?)

---

**Next: Review `memory_metrics.json` to see detailed memory breakdown**