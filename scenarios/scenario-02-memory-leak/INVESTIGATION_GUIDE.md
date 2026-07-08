# Memory Leak Investigation Guide - Scenario 2

## Phase 1: Symptom Detection (5-10 minutes)

### Objective
Quickly understand: Is this a sudden failure or gradual degradation?

### Your Tasks

1. **Read the README.md** - Get context
2. **Review timeline.md**
   - When did the service deploy?
   - When did problems start?
   - Was it gradual or sudden?

3. **Look at memory_metrics.json**
   - Day 1 memory: ?
   - Day 6 memory at crash: ?
   - What's the growth pattern?

### Questions to Answer

- [ ] When did v3.2.0 deploy?
- [ ] When did OOM crash occur?
- [ ] How long between deployment and crash?
- [ ] Is memory usage growing linearly or exponentially?
- [ ] At what point should we have acted?

### AI Prompt

```
Analyze this 6-day incident timeline:
1. When did symptoms start?
2. What was the progression?
3. Why didn't we catch it on day 3?

[paste timeline.md]
```

---

## Phase 2: Trending Analysis (10-15 minutes)

### Objective
Use data to understand the pattern, not gut feeling.

### Your Tasks

1. **Plot the memory growth**
   ```
   Memory over 6 days:
   Day 1: 2.0 GB
   Day 2: 3.2 GB (+1.2)
   Day 3: 4.6 GB (+1.4)
   Day 4: 5.5 GB → restart → 2.0 GB
   Day 5: 6.8 GB (+4.8 after restart!)
   Day 6: CRASH at 11.8 GB
   ```
   - What do you notice?
   - Why faster on Day 5?

2. **Analyze GC behavior**
   - Day 1 GC pause: 50ms
   - Day 4 GC pause: 500ms (10x worse)
   - Day 6 GC pause: 2000ms (40x worse)
   - What does this tell you?

3. **Response time correlation**
   - As memory grows, latency grows
   - Why?
   - What's the relationship?

4. **The restart paradox**
   - After restart on Day 4, memory goes from 6.1 GB → 2.0 GB
   - Why did it go back down?
   - But then grew FASTER on Day 5?
   - What does this tell us?

### Key Insight

**The restart paradox is crucial:**
- Problem goes away (memory drops)
- But returns FASTER (grows 4.8 GB/day vs. 1.4 GB/day)
- This proves: **Leak is in code, not configuration**
- If it was config, restart wouldn't make leak return
- If it was traffic, why would it grow faster?

### Questions to Answer

- [ ] What's the memory growth rate (GB/day)?
- [ ] Why did growth rate increase after restart?
- [ ] What deployed on Day 1?
- [ ] Could the memory growth BE the deployment?
- [ ] Why did GC pauses increase proportionally?

### AI Prompt

```
Analyze this memory growth pattern:

[paste memory_metrics.json]

Answer:
1. What's the growth rate per day?
2. Why did growth accelerate after the restart on Day 4?
3. What does "grows faster after restart" tell us about the root cause?
4. When should we have alerted on memory?
```

---

## Phase 3: Root Cause - The Leak (15-20 minutes)

### Objective
Find what code is leaking memory.

### Your Tasks

1. **What changed on Day 1?**
   - Look at timeline.md
   - Version deployed: v3.2.0
   - Change: "Implement request caching to improve performance"
   - Cache stores results in memory for 1 hour
   - **BINGO: NEW CODE that uses memory**

2. **Connect the dots**
   - Memory starts growing after caching deployed
   - Cache stores results in memory
   - Results grow over time
   - **Hypothesis: Cache never empties (unbounded growth)**

3. **The 5 Whys for Memory Leak**
   - **Why #1: Why is memory growing?**
     - Answer: Something is allocating memory that never gets released
   
   - **Why #2: What allocates memory?**
     - Answer: The new caching feature stores results
   
   - **Why #3: Why doesn't the cache empty?**
     - Look at git_history.md (code review)
     - Does it show cleanup logic?
     - Does it implement expiration?
     - Does it have size limits?
   
   - **Why #4: Why did code review miss this?**
     - Answer: Code review didn't check for unbounded growth
     - No one asked: "What happens after 1 week of requests?"
   
   - **Why #5: Why is Day 5 growth faster?**
     - Answer: Larger heap (8GB → 12GB) allows more concurrent requests
     - More concurrent requests = more cache entries = faster leak rate
     - **Math: 1.4 GB/day with 8GB heap = limited concurrency**
     - **Math: 4.8 GB/day with 12GB heap = more concurrency possible**

### The Leak Mechanism

```
Request 1:
1. Process request
2. Store result in cache[key] = result
3. Cache size: 120 MB
4. Scheduled cleanup: "Remove entries older than 1 hour"

After 1 week:
1. Millions of requests
2. Cache never actually empties (why?)
   - Cleanup logic never runs?
   - Cleanup broken?
   - Keys never match (cache key includes timestamp?)
3. Cache grows unbounded
4. Memory exhausted
5. GC tries harder (longer pauses)
6. Eventually: OOM
```

### Questions to Answer

- [ ] What feature deployed on Day 1?
- [ ] What memory structure does it use?
- [ ] Does it have cleanup/expiration logic?
- [ ] Is the cleanup logic working?
- [ ] How would you fix it?

### AI Prompt

```
Explain the memory leak:

1. What code allocates the memory?
2. What code should free it?
3. Why isn't the freeing happening?
4. How would you verify this is the leak?

Context:
- v3.2.0 deployed with "request caching"
- Memory grows 1.4 GB/day initially
- After restart (bigger heap), grows 4.8 GB/day
- Crash happens when hitting heap limit
```

---

## Phase 4: Hypothesis Testing (10-15 minutes)

### Objective
Validate that caching is indeed the culprit.

### Test 1: Timeline Alignment
```
✓ v3.2.0 deployed: 2024-07-08 10:30 UTC
✓ Memory starts growing immediately (Day 1 afternoon: 3.2 GB)
✓ Cache feature introduced
✓ Before this: stable memory (2.0 GB for weeks)
```
**Result: Timing aligns perfectly**

### Test 2: Growth Rate vs. Load
```
✓ Memory grows steadily (not based on traffic spikes)
✓ Growth continues even on weekends (steady load)
✓ Growth is consistent (1.4 GB/day)
✓ Not correlated with peak hours
```
**Result: Consistent leak, not load-dependent**

### Test 3: Restart Effect
```
✓ After restart on Day 4:
  - Memory drops instantly to 2.0 GB
  - Cache cleared (restarted)
✓ But then grows FASTER (4.8 GB/day)
  - Why faster with bigger heap?
  - Bigger heap = more concurrent requests can fill it
  - = faster leak accumulation
  - This PROVES leak is in the code, not config
```
**Result: Restart pattern confirms code leak**

### Test 4: Cache Growth Matches Memory Growth
```
Day 1: Cache 120 MB, Memory 2.0 GB total
Day 2: Cache 680 MB, Memory 3.2 GB total
Day 3: Cache 1,580 MB, Memory 4.6 GB total
Day 4: Cache 2,340 MB, Memory 5.5 GB total

Observation:
- Cache size ≈ memory growth
- If we subtract baseline memory (1.5 GB for app code)
- Remaining ≈ cache

Example Day 3:
- Total memory: 4.6 GB
- Baseline: ~1.5 GB (app code, libraries)
- Free: 4.6 - 1.5 = 3.1 GB
- Cache: 1.58 GB
- Other (objects, temp): 1.52 GB
- Math checks out!
```
**Result: Memory growth = cache growth**

### Questions to Answer

- [ ] Does timing align with v3.2.0 deployment?
- [ ] Is growth consistent (leak) or variable (traffic)?
- [ ] Does restart prove it's code (not config)?
- [ ] Does cache size growth match memory growth?
- [ ] Are there any contradictions to the hypothesis?

### AI Prompt

```
Validate this hypothesis:
"v3.2.0 caching feature has an unbounded memory leak"

Evidence:
[paste timeline.md focusing on Day 1 deployment]
[paste memory_metrics.json showing daily growth]

Questions:
1. Does timing align?
2. Does the restart pattern make sense?
3. Why does growth accelerate after restart?
4. Is there any evidence contradicting this?
```

---

## Phase 5: Immediate & Long-term Fixes (10 minutes)

### Immediate Actions (Already Done)
- [x] Identified memory is growing
- [x] Restarted service (temporary fix)
- [x] Increased heap size (bought time)

### Short-Term Fix (24 hours)

**Option 1: Disable caching (quickest)**
```
Rollback to v3.2.0-pre-caching
Pro: Immediate relief
Con: Lose performance gain
```

**Option 2: Fix the leak (better)**
```
1. Find why cache cleanup isn't working
2. Options:
   a. Implement cache eviction (LRU with max size)
   b. Fix expiration timer (1-hour cleanup)
   c. Reduce cache scope (don't cache everything)
3. Deploy v3.2.1
```

**Recommendation:** Option 2 - fix and deploy v3.2.1

### Long-Term Fixes (1-2 weeks)

**Prevention #1: Memory Monitoring**
- Alert when memory grows > 100 MB/day
- Alert when GC pause > 200ms
- Would have caught this on Day 2

**Prevention #2: Load Testing**
- Run new code for 1 week of simulated traffic
- Check memory trend
- Required for any caching features

**Prevention #3: Code Review**
- For any memory-allocating feature:
  - "What's the max size?"
  - "How does it get cleaned up?"
  - "What happens after 1 week of traffic?"
  - "Have you tested with 1M entries?"

**Prevention #4: Automated Testing**
- Memory leak detection test
- Run 1M requests, verify memory returned to baseline
- Fail if memory > baseline + 100MB

### Questions to Answer

- [ ] What's the quickest fix?
- [ ] What's the best fix?
- [ ] Why should we fix vs. rollback?
- [ ] How do we prevent this in future?
- [ ] What alert would have caught this on Day 1?

---

## Phase 6: Postmortem (15-20 minutes)

### The Big Question
**"Why didn't we catch this for 6 days?"**

### Postmortem Analysis

**What went well:**
- ✓ Issue was noticed (Monday, Day 1)
- ✓ Users reported it (user-driven observability)
- ✓ Quick investigation (Monday afternoon)
- ✓ Temporary workaround (increase heap)

**What went wrong:**
- ✗ No memory trend alerting
- ✗ Code review didn't check for unbounded growth
- ✗ No pre-deployment memory leak testing
- ✗ Alert fired only at >80% (too late)
- ✗ No validation of cache cleanup logic
- ✗ Weekend escalation slow (no on-call for memory)

### Contributing Factors

1. **Insufficient Monitoring**
   - Alert threshold: 80% of heap (too late)
   - Should alert: 50% (gives time to investigate)
   - Should alert: Memory growth > 200MB/day (trend-based)

2. **Code Review Gap**
   - Caching feature merged without verification of:
     - Max cache size
     - Eviction policy
     - Memory bounds testing
   - Questions asked: performance
   - Questions NOT asked: resource limits

3. **No Load Testing**
   - Feature tested locally (small dataset)
   - Not tested with week-long traffic simulation
   - Would have shown 1.4 GB/day growth trend

4. **No Memory Profiling**
   - Heap dump only taken at crash (too late)
   - Should be taken proactively
   - Memory profiler should run continuously

5. **Alert Fatigue Avoidance**
   - Didn't want to alert on normal activity
   - Resulted in deaf to abnormal activity
   - Need smarter alerts (trend-based, not threshold-based)

### Action Items

```
1. Fix the cache (v3.2.1)
   Owner: james.Park@company.com
   Due: Today
   
2. Add memory trend alerts (alert when memory grows >200MB/day)
   Owner: ops-team
   Due: This week
   
3. Add pre-deployment memory leak test
   Owner: qa-team
   Due: Before next release
   
4. Update code review checklist for memory-allocating features
   Owner: tech-lead
   Due: Tomorrow
   
5. Implement continuous memory profiling (sample hourly)
   Owner: observability-team
   Due: 2 weeks
```

### Lessons Learned

1. **Gradual degradation is harder to catch than sudden failures**
   - Sudden: Alarms everywhere
   - Gradual: Looks like normal variation
   - Need different monitoring approach

2. **Resource limits must be baked into code, not monitoring**
   - Don't rely on alert to catch resource leak
   - Code should enforce: cache.maxSize = 500MB
   - Monitoring should verify: cache.size < maxSize

3. **Load testing matters**
   - Local tests don't show leaks
   - Multi-day simulations reveal trends
   - Required for any feature using resources

4. **Restart is a symptom, not a fix**
   - Each restart buys time
   - But problem returns (and faster)
   - Must fix the root cause

5. **Monitoring alerting thresholds matter**
   - 80% is too late
   - Need trend-based alerts
   - Need baseline + growth rate alerts

---

**You've completed the Memory Leak investigation!**

**Key Takeaway:** Long-term issues require different investigation mindset:
- Focus on trends, not snapshots
- Check for gradual change, not sudden failure
- Use memory profilers and heap dumps
- Correlate memory with deployment time
- Think about "what happens after N days of traffic?"