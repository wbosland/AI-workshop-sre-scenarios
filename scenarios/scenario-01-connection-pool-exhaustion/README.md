# Scenario 1: The Automated Process Failure

## Incident Overview

**Scenario Title:** Payment Processing Pipeline Failure

**Severity:** CRITICAL

**Impact:** 
- All automated payment processing stopped at 14:32 UTC
- ~2,847 pending transactions stuck in queue
- Customer-facing API returning 503 Service Unavailable
- 45+ minutes of downtime before resolution

**Your Role:** Service Engineer on-call

---

## Initial Situation

At 14:32 UTC on 2024-07-08, your monitoring system fires multiple critical alerts:

```
🚨 CRITICAL ALERT: Payment Service Unavailable
🚨 CRITICAL ALERT: Database Connection Pool Exhausted  
🚨 WARNING: High Error Rate on Payment API
```

A customer support ticket comes in: "Our payments aren't processing. What's wrong?"

Your team is looking to you for answers:
- **What happened?**
- **Why did it happen?**
- **How do we fix it?**
- **How do we prevent it next time?**

---

## Materials Provided

This scenario includes:

1. **Deployment Timeline** (`deployment_timeline.md`) - What was deployed when
2. **Error Logs** (`error_logs.json`) - Application error messages
3. **System Metrics** (`system_metrics.json`) - CPU, memory, connections
4. **Application Logs** (`app_logs.jsonl`) - Detailed activity traces
5. **Git History** (`git_diff.md`) - Recent code changes
6. **Infrastructure Config** (`infrastructure_config.md`) - System setup
7. **Investigation Guide** (`INVESTIGATION_GUIDE.md`) - Step-by-step walkthrough

---

## Investigation Roadmap

### Phase 1: Detection & Triage (5-10 min)
Start here. Answer: What is broken right now?
→ Check `deployment_timeline.md` and `error_logs.json`

### Phase 2: Symptom Collection (10-15 min)
Gather all available data.
→ Review `system_metrics.json` and `app_logs.jsonl`

### Phase 3: Root Cause Analysis (15-20 min)
Use the "5 Whys" method.
→ Correlate timeline, logs, and code changes in `git_diff.md`

### Phase 4: Hypothesis & Validation (10-15 min)
Propose and test theories.

### Phase 5: Remediation (10 min)
Determine immediate and long-term fixes.

### Phase 6: Postmortem (15-20 min)
Document findings.

---

## Tips for Success

- **Don't assume:** Observe in the logs first
- **Use AI:** Parse logs, identify patterns, correlate events
- **Follow the timeline:** Events reveal cause-and-effect
- **Ask "why" repeatedly:** Each answer leads to the next question
- **Look for the change:** What was different before failure?

---

## Suggested AI Prompts

1. **For log analysis:** "Summarize the key error patterns in these logs"
2. **For correlation:** "What happened in the system right before the error rate spike?"
3. **For code review:** "What connection handling changes were made in this diff?"
4. **For hypothesis:** "Based on connection pool exhaustion errors, what could cause this?"

---

**Ready to investigate? Open `INVESTIGATION_GUIDE.md` to begin!**