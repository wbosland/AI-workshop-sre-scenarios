# Scenario 3: The Silent Killer

## Incident Overview

**Scenario Title:** Race Condition Causing Intermittent Data Corruption

**Severity:** HIGH (intermittent, hard to track)

**Impact:**
- ~0.8% of user profiles have corrupted data
- Errors are intermittent (not reproducible locally)
- Affects only specific operations under high concurrency
- Takes 3 days to identify the pattern
- Users report: "Sometimes my settings reset randomly"
- 1,247 affected users, ~2,100 data corruption incidents

**Your Role:** On-call engineer investigating "random user data corruption"

---

## Initial Situation

You receive reports starting Monday:

```
User report #1 (Monday 08:30 UTC):
"My email preferences reset overnight. I had email OFF, 
now it's back ON. This is the 2nd time this week."

User report #2 (Monday 09:15 UTC):
"My notification settings got messed up. Shows settings I never changed."

User report #3 (Monday 09:45 UTC):
"Is anyone else getting random profile resets?"
```

Support team says: "We're getting a steady stream of these reports. But it's random - we can't reproduce it."

Your questions:
- **What causes the corruption?** (Only happens sometimes)
- **Why is it intermittent?** (Can't reproduce locally)
- **How widespread is it?** (How many users affected?)
- **When did it start?** (New code? Configuration change?)
- **Can we stop it without understanding it?** (How urgent?)

---

## Key Differences from Scenarios 1 & 2

✓ **Intermittent, not consistent** (hard to diagnose)
✓ **Requires concurrency analysis** (not just logs)
✓ **Needs statistical thinking** (0.8% of operations)
✓ **Correlates with high load** (happens under stress)
✓ **Data corruption** (integrity issue, not availability)

---

## Materials Provided

1. **incident_reports.md** - User complaints with timestamps
2. **error_logs.json** - Rare errors in logs (easy to miss)
3. **database_audit.json** - Before/after state of corrupted records
4. **concurrency_analysis.md** - Transaction log showing race condition
5. **code_diff.md** - Code changes from Friday release
6. **load_patterns.json** - When corruption happens (correlates with traffic)
7. **profiling_data.jsonl** - Thread traces showing race window
8. **application_config.md** - Database transaction isolation levels

---

## Investigation Roadmap

### Phase 1: Pattern Recognition (10-15 min)
→ Is this really random? What's the actual pattern?

### Phase 2: Statistical Analysis (10-15 min)
→ When does it happen? Who's affected? What's the rate?

### Phase 3: Code Review - Race Condition (15-20 min)
→ Find the concurrent access issue in the code

### Phase 4: Concurrency Tracing (10-15 min)
→ Trace the exact race window between threads

### Phase 5: Fix & Prevention (10 min)
→ Synchronization, transactions, or code restructuring?

### Phase 6: Postmortem (15-20 min)
→ Why did code review miss a race condition?

---

## Difficulty Differences

**Compared to Scenarios 1 & 2:**
- **Harder:** Requires understanding of concurrency and race conditions
- **Harder:** Need to reason about thread timing (microseconds matter)
- **Harder:** Statistical pattern recognition (not obvious)
- **Different:** Corruption is data integrity issue, not availability
- **Different:** Can't just restart - need to fix the bug

---

**Start with Phase 1: Read `incident_reports.md` to identify patterns**