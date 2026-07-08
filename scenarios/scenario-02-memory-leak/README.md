# Scenario 2: The Gradual Slowdown

## Incident Overview

**Scenario Title:** Memory Leak Causing Progressive Degradation

**Severity:** HIGH (degrading to CRITICAL)

**Impact:**
- Service response times gradually increase over 6 days
- API p99 latency: 200ms → 5000ms
- Memory usage grows steadily: 2GB → 7.2GB
- Out-of-Memory (OOM) crash on day 6 at 03:47 UTC
- 8 minutes of complete downtime before automatic restart
- ~12,000 requests dropped during crash window

**Your Role:** On-call engineer responding to "our service is getting slower"

---

## Initial Situation

At 03:47 UTC on 2024-07-14, your monitoring alerts:

```
🚨 CRITICAL ALERT: Service Out of Memory
🚨 CRITICAL ALERT: Service Restarting
⚠️  WARNING: High response times (last 6 days)
⚠️  WARNING: Memory growth trend
```

Your manager says: "Performance has been degrading all week. Why didn't we catch this?"

Questions to answer:
- **What's causing the memory growth?**
- **Why didn't alerts fire earlier?**
- **Is the crash the problem, or just a symptom?**
- **How do we stop it from happening again?**

---

## Key Differences from Scenario 1

✓ **Long-term issue** (6 days vs. 45 minutes)
✓ **Subtle symptoms** (gradual degradation vs. sudden failure)
✓ **Trending data** (memory over time vs. snapshot)
✓ **Prevention vs. recovery** (could have caught it earlier)
✓ **Resource leak** (memory vs. connections)

---

## Materials Provided

1. **timeline.md** - 6-day timeline with daily metrics
2. **memory_metrics.json** - Memory usage over time, GC logs
3. **response_time_metrics.json** - Latency growth day-by-day
4. **error_logs.json** - OOM error and crashes
5. **application_logs.jsonl** - Request processing logs
6. **git_history.md** - Code deployed 6 days ago
7. **infrastructure_config.md** - JVM settings, heap size
8. **heap_dump_analysis.md** - Memory profiler output (from crash)

---

## Investigation Roadmap

### Phase 1: Symptom Detection (5-10 min)
→ What's broken and when? Is it sudden or gradual?

### Phase 2: Trending Analysis (10-15 min)
→ Plot metrics over time. What's the pattern?

### Phase 3: Root Cause - The Leak (15-20 min)
→ Use heap dump and code review to find memory leak

### Phase 4: Hypothesis Testing (10-15 min)
→ Does the leak pattern match the code changes?

### Phase 5: Immediate & Long-term Fixes (10 min)
→ Restart vs. code fix vs. monitoring improvement

### Phase 6: Postmortem (15-20 min)
→ Why wasn't this caught during the week?

---

## Difficulty Differences

**Compared to Scenario 1 (Connection Pool):**
- **Harder:** Need to read heap dumps and understand GC logs
- **Harder:** Correlate trends over multiple days
- **Easier:** Less cascading (gradual, not sudden)
- **Different:** Focus on prevention (alert thresholds, monitoring)

---

**Start with Phase 1: Read `timeline.md` to understand the 6-day progression**