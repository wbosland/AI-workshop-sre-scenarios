# Investigation Guide: Payment Processing Pipeline Failure

## Overview

This guide walks you through investigating the incident. Use the materials provided, along with AI assistance, to uncover the root cause.

---

## Phase 1: Detection & Triage (5-10 minutes)

### Objective
Quickly understand what is broken right now.

### Your Tasks

1. **Read the README.md** - Get situational awareness
2. **Check deployment_timeline.md**
   - When did the latest deployment happen?
   - When exactly did the failure occur?
   - What changed?

3. **Review error_logs.json**
   - What are the top 3 errors?
   - When did they start?
   - How many errors total?

### Questions to Answer

- [ ] What time did the incident start?
- [ ] What is the primary error message?
- [ ] How many transactions are affected?
- [ ] Which service instances are impacted?

### AI Prompt to Use

```
Summarize these error logs:
- What are the 3 most common errors?
- When did each first appear?
- What component is failing?
```

---

## Phase 2: Symptom Collection (10-15 minutes)

### Objective
Gather all available data points without jumping to conclusions.

### Your Tasks

1. **Examine system_metrics.json**
   - Plot the database connection timeline
   - When did connections reach maximum?
   - How do response times correlate?
   - What happened to transaction throughput?

2. **Review app_logs.jsonl**
   - Follow the log timeline from 14:30 to 14:35
   - What was the system doing before failure?
   - What changed at 14:32?

3. **Create a Timeline** (use deployment_timeline.md as reference)
   ```
   14:30 - Campaign launch
   14:32 - First error
   14:35 - Incident commander engaged
   ...
   ```

### Questions to Answer

- [ ] What was the database connection count before vs. after failure?
- [ ] How quickly did the pool become exhausted?
- [ ] When did queued requests start accumulating?
- [ ] What's the correlation between traffic increase and failure?

### AI Prompt to Use

```
Create a timeline showing:
1. System metrics over time
2. When each metric changed
3. The sequence of events leading to failure

Use this data:
[paste system_metrics.json]
```

---

## Phase 3: Root Cause Analysis - 5 Whys (15-20 minutes)

### Objective
Move from symptoms to root cause.

### The Framework

**Symptom:** Database connection pool is exhausted

**Why #1:** Why are all connections being used?
- Look at: Active connections in metrics
- Evidence: Connection count goes from 41→50 at 14:32
- Hypothesis: Either connections aren't being released, or demand exceeds capacity

**Why #2:** Why did connection usage suddenly increase at 14:32?
- Look at: Deployment timeline (v2.4.1 deployed 13:52) + traffic spike (14:30)
- Evidence: Campaign starts at 14:30, but pool only exhausted at 14:32
- Hypothesis: New code handles traffic differently

**Why #3:** What changed in the code?
- Look at: git_diff.md
- Evidence: Retry logic changed from 3 attempts to 5 with exponential backoff
- Hypothesis: More retry attempts holding connections longer

**Why #4:** Why would more retries exhaust the pool?
- Look at: error_logs.json - errors started at 14:32 exactly
- Evidence: Initial errors show "retrying", suggesting the new retry logic is activating
- Hypothesis: Retry storm from new logic creating temporary connection issues

**Why #5:** Why aren't connections being released?
- Look at: git_diff.md - PaymentRetryHandler.java code
- Evidence: Connection not explicitly closed in retry exception handler
- **ROOT CAUSE IDENTIFIED**: Connection leak in the new retry logic

### Your Tasks

1. **Work through each "Why"**
   - Answer it using evidence from the provided files
   - Document your reasoning

2. **Check the git diff carefully**
   - Find the code that handles connection acquisition
   - Look for connection.close() calls
   - Identify where connections might leak

3. **Trace the retry logic**
   - How many times does it retry?
   - What happens if an exception occurs?
   - Are connections properly cleaned up?

### Questions to Answer

- [ ] When was the problematic code deployed?
- [ ] What's different about the retry logic in v2.4.1?
- [ ] Why would the old retry logic not cause this issue?
- [ ] Where exactly is the connection leak?
- [ ] Why did it take time to surface (14:30-14:32)?

### AI Prompts to Use

```
Analyze this code diff and explain:
1. What changed in the retry logic?
2. What's the difference between v2.4.0 and v2.4.1?
3. Are there any resource leaks?
4. How would this behave under load?

[paste git_diff.md]
```

```
I'm investigating a database connection pool exhaustion. 
This code was deployed just before the failure:
[paste PaymentRetryHandler.java from git_diff.md]

Can you identify:
1. Are connections properly closed?
2. What happens if getConnection() throws an exception?
3. Could this cause a leak under load?
```

---

## Phase 4: Hypothesis & Validation (10-15 minutes)

### Objective
Propose and test theories against the data.

### Hypothesis

**Primary Hypothesis:**
"The v2.4.1 retry logic has a connection leak. When getConnection() fails, the exception is caught but the connection isn't properly closed. Under high load from the campaign, this leak accumulates, exhausting the 50-connection pool."

**Evidence Supporting This:**
1. Failure happened 42 minutes after deployment (time for leak to accumulate)
2. Failure coincided with traffic surge (campaign)
3. Error logs show retry logic activating ("Connection acquisition failed, retrying")
4. Code diff shows connection not explicitly closed on exception
5. Metrics show all 50 connections held simultaneously (typical of a leak)

### Validation Tests

**Test 1: Timeline Alignment**
- [ ] Deployment at 13:52? YES - v2.4.1 deployed
- [ ] Normal operation for 40 minutes? YES - health check at 14:15 passed
- [ ] Failure at traffic spike? YES - 14:32 = 2 min after campaign

**Test 2: Error Pattern Analysis**
- [ ] Errors start exactly at 14:32? YES - error_logs.json shows first error at 14:32:15
- [ ] Error is about connection pool? YES - "all 50 connections are busy"
- [ ] Retry logic is mentioned? YES - "Connection acquisition failed, retrying"

**Test 3: Code Review**
- [ ] Did code change? YES - git_diff.md shows PaymentRetryHandler changed
- [ ] Are connections managed? PARTIALLY - inside try-catch, connection might not close on exception
- [ ] Is it in a try-with-resources? NO - manual resource management

**Test 4: Why Rollback Fixed It**
- [ ] v2.4.0 rolled back at 15:12? YES
- [ ] Service recovered immediately after? YES - connections went from 50 to 22 at 15:13
- [ ] No more errors? YES - app_logs show service recovering

### Alternative Hypotheses Considered

**Hypothesis 2:** "Database hit its max connections limit"
- Counter-evidence: Database was configured for 200 max, payment service only uses 50-200 total
- Unlikely

**Hypothesis 3:** "Campaign traffic just too much for infrastructure"
- Counter-evidence: Campaign was 1.5x normal, infrastructure is sized for peaks
- Would have degradation, not total failure
- Unlikely

**Hypothesis 4:** "Configuration change caused issue"
- Counter-evidence: Connection pool size didn't change (still 50)
- Possible but less likely than code leak

### Questions to Answer

- [ ] Does your hypothesis explain the timeline?
- [ ] Does it explain why rollback fixed it?
- [ ] Does it explain why health checks passed?
- [ ] What evidence proves this is the root cause?

---

## Phase 5: Remediation (10 minutes)

### Immediate Actions (What Was Done)

- [x] Rolled back to v2.4.0
- [x] Service restored at 15:17
- [x] Backlog cleared

### Short-Term Fix (24-48 hours)

**For the development team:**

```java
// WRONG (current code in v2.4.1)
Connection conn = null;
try {
    conn = getConnection();
    return executePayment(txn, conn);
} catch (SQLException e) {
    if (attempt < MAX_RETRIES) {
        Thread.sleep(backoffMs);
        // conn reference lost here - NOT CLOSED!
    }
}

// CORRECT (fix)
try (Connection conn = getConnection()) {
    return executePayment(txn, conn);
} catch (SQLException e) {
    if (attempt < MAX_RETRIES) {
        Thread.sleep(backoffMs);
        // Connection auto-closes here
    }
}
```

**Action Items:**
- [ ] Fix the connection leak in PaymentRetryHandler.java
- [ ] Add explicit try-with-resources for connection management
- [ ] Write unit test for connection cleanup on exception
- [ ] Validate fix locally with load testing

### Long-Term Fixes (1-2 weeks)

1. **Improved Code Review Process**
   - Add checklist for resource management
   - Require manual review of try-catch blocks handling connections

2. **Better Pre-Deployment Testing**
   - Load test new versions at 2x expected traffic
   - Test connection pool under failure scenarios
   - Canary deploy to 1 instance first, monitor for 30+ minutes

3. **Enhanced Health Checks**
   - Health check should stress connection pool, not just ping database
   - Add connection leak detection endpoint
   - Test retry logic under simulated failures

4. **Monitoring Improvements**
   - Lower alert threshold for connection pool (alert at 40/50, not 48/50)
   - Add alert for retry rate anomalies
   - Add alert for connection leak detection (held > 60s)

5. **Runbook Updates**
   - Document this incident scenario
   - Add troubleshooting steps for connection pool exhaustion
   - Include automatic rollback decision criteria

### Questions to Answer

- [ ] What's the minimal fix?
- [ ] What tests should be added?
- [ ] What process changes prevent recurrence?
- [ ] What monitoring improvements help detect earlier?

---

## Phase 6: Postmortem (15-20 minutes)

### Postmortem Template

```markdown
# Postmortem: Payment Processing Pipeline Failure

## Incident Summary
- **Title:** Database Connection Pool Exhaustion
- **Date:** 2024-07-08
- **Duration:** 45 minutes (14:32 - 15:17 UTC)
- **Impact:** 2,847 transactions failed
- **Root Cause:** Connection leak in retry logic

## Timeline
| Time | Event |
|------|-------|
| 13:52 | v2.4.1 deployed |
| 14:15 | Health check passed |
| 14:30 | Campaign traffic starts |
| 14:32 | Connection pool exhausted |
| 14:35 | Incident commander engaged |
| 14:55 | Root cause identified |
| 15:12 | Rollback deployed |
| 15:17 | Service restored |

## Root Cause

The v2.4.1 retry logic uses manual connection management without proper cleanup on exception. When `getConnection()` fails under load, the exception is caught but the connection reference is lost, causing the connection pool to leak. Under the increased traffic from the campaign launch, these leaked connections accumulate, exhausting the 50-connection pool.

## Symptoms vs. Root Cause

**Symptom 1:** Database connection pool exhausted
↓ Why?
**Symptom 2:** Connections not being released
↓ Why?
**Symptom 3:** New code doesn't close connections on exception
↓ Why?
**Symptom 4:** Manual connection management without try-with-resources
↓ Why?
**ROOT CAUSE:** Code review didn't catch resource leak pattern

## Contributing Factors

1. **No load testing for v2.4.1:** Would have revealed the leak
2. **No canary deployment:** Single deploy to all 4 instances
3. **Insufficient pre-deployment testing:** Health checks don't stress connection pool
4. **Aggressive retry logic:** 5 retries instead of 3 amplifies issues
5. **Tight connection pool settings:** Reduced idle-timeout made it worse

## What Went Well

- ✓ Alerts fired immediately when pool was exhausted
- ✓ Incident commander engaged quickly
- ✓ Root cause identified and fixed in 23 minutes
- ✓ Rollback was straightforward and effective
- ✓ Service recovered cleanly

## What Could Be Improved

- ✗ Load testing should have caught this before production
- ✗ Code review should have flagged the resource leak
- ✗ Canary deployment would have limited blast radius
- ✗ Health checks too simplistic
- ✗ Alert threshold (48/50) left little room for error

## Action Items

**Immediate (24 hours):**
- [ ] Deploy fix to v2.4.2 (proper try-with-resources)
- [ ] Add unit test for connection cleanup on exception
- [ ] Notify customers of incident

**Short-term (1 week):**
- [ ] Update code review checklist for resource management
- [ ] Implement load testing in CI/CD
- [ ] Add connection pool stress test to health checks
- [ ] Lower connection pool alert threshold to 40/50

**Long-term (2 weeks):**
- [ ] Implement canary deployment process
- [ ] Add connection leak detection monitoring
- [ ] Update incident playbooks
- [ ] Training for team on connection pooling best practices

## Lessons Learned

1. **Resource leaks under load are subtle:** They pass health checks but fail under stress
2. **Load testing is essential:** Caught this would have prevented incident
3. **Try-with-resources is important:** Automatic cleanup prevents manual errors
4. **Canary deployments save the day:** Would have limited impact to 1 instance
5. **Monitoring thresholds matter:** Too late on exhaustion alert means less recovery time

## Owner/Accountability

- **Incident Commander:** Handled well, escalated appropriately
- **Development Team:** Will implement fixes and testing
- **Ops Team:** Will implement monitoring improvements
- **Leadership Review:** Assessment of code review process
```

### Your Tasks

1. **Fill out the template** using information you gathered
2. **Document timeline** with timestamps and events
3. **Explain the 5 Whys** you worked through
4. **List contributing factors** (why it wasn't caught earlier)
5. **Propose action items** to prevent recurrence
6. **Write lessons learned** specific to your investigation

### Questions to Answer

- [ ] Was this incident preventable?
- [ ] What process broke down?
- [ ] What monitoring alert was missed?
- [ ] What should change to prevent recurrence?
- [ ] Who should be accountable for each preventive measure?

---

## Summary

You've now completed a full incident investigation:
1. ✓ Detected the problem and triaged severity
2. ✓ Collected symptoms and correlated events
3. ✓ Performed root cause analysis using 5 Whys
4. ✓ Developed and validated hypotheses
5. ✓ Proposed immediate and long-term fixes
6. ✓ Documented in a comprehensive postmortem

**Key Takeaway:** The connection leak was subtle but had catastrophic effects under load. This incident demonstrates why resource management, load testing, and code review processes are critical.

---

## Suggested AI Prompts for Each Phase

### Phase 1: Detection
```
Summarize this incident scenario. What failed and when?
[paste README.md and error_logs.json]
```

### Phase 2: Symptoms
```
Create a visual timeline of the incident showing:
1. Database metrics over time
2. Application errors
3. Recovery events
[paste system_metrics.json]
```

### Phase 3: Root Cause
```
Analyze this code change and identify resource management issues:
[paste PaymentRetryHandler.java from git_diff.md]

Specifically:
1. Are connections properly closed in all code paths?
2. What happens if an exception occurs?
3. How would this behave under high load with many retries?
```

### Phase 4: Validation
```
Does this theory explain all the observed symptoms?

Theory: The v2.4.1 retry logic has a connection leak. When getConnection() fails, the connection isn't properly closed. Under campaign traffic, these leak and exhaust the pool.

Evidence:
[paste timeline, error logs, metrics]

Does the theory match all observations?
```

### Phase 5: Remediation
```
What's the proper way to manage database connections in Java retry logic?
Show a corrected version of this code:
[paste broken code from git_diff.md]
```

### Phase 6: Postmortem
```
Write a blameless postmortem for this incident including:
- What happened
- Why it happened
- Contributing factors (not just root cause)
- Action items to prevent recurrence

Use this data:
[paste timeline, root cause analysis, fixes]
```

---

**You're ready to investigate! Start with Phase 1.**