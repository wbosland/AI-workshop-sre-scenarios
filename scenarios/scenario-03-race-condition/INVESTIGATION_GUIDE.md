# Race Condition Investigation Guide - Scenario 3

## Phase 1: Pattern Recognition (10-15 minutes)

### Objective
Prove this isn't random - find the actual pattern.

### Your Tasks

1. **Read the README.md** - Get context
2. **Review incident_reports.md**
   - What are users reporting?
   - Is it truly random?
   - When did it start?

3. **Look for patterns**
   - Are all reports the same type of corruption?
   - What settings are affected?
   - Do they happen at specific times?
   - Do active users have more incidents?

### The Key Insight

**"Random" usually means we haven't found the pattern yet.**

Look for:
- Time pattern (happens during peak hours?)
- User pattern (active users only?)
- Operation pattern (specific API endpoint?)
- Data pattern (certain settings affected?)

### Questions to Answer

- [ ] When did first report come in?
- [ ] How many total incidents by Wednesday?
- [ ] What time of day do most occur?
- [ ] Are active or inactive users affected?
- [ ] What's the actual incident rate (0.8% = how many fail)?
- [ ] Did we deploy anything before reports started?

### AI Prompt

```
Analyze these user reports to find patterns:

1. Are these truly random or is there a pattern?
2. When do most incidents occur?
3. Who is affected (user type)?
4. What specifically is being corrupted?

[paste incident_reports.md]
```

---

## Phase 2: Statistical Analysis (10-15 minutes)

### Objective
Use data to prove the pattern and identify the root.

### Your Tasks

1. **Time correlation**
   ```
   From incident_reports.md:
   Peak hours (18:00-22:00 UTC): 89% of incidents
   Off-peak (01:00-08:00 UTC): 4% of incidents
   
   Question: Why peak hours?
   Answer: More concurrent users = higher chance of race
   ```

2. **Rate calculation**
   ```
   Total incidents: 1,029
   Total requests: 128,625
   Rate: 1,029 / 128,625 = 0.8%
   
   Question: Why 0.8% specifically?
   Answer: Race window is tiny (microseconds)
   Only happens when two threads have EXACT bad timing
   ```

3. **User correlation**
   ```
   Active users: 94% of incidents
   Rare users: <1% of incidents
   
   Question: Why active users?
   Answer: Active users make concurrent API calls
   Rare users make sequential calls (no race possible)
   ```

4. **Deployment correlation**
   ```
   Friday release (v1.8.0): "Optimize settings API with batched updates"
   First report: Monday (3 days later)
   
   Question: Why the delay?
   Answer: Need concurrent traffic to trigger
   Weekend has lower traffic
   Monday is peak again = race manifests
   ```

### The Statistical Signature of a Race Condition

```
✓ Intermittent (0.8% of ops, not 100%)
✓ Correlated with concurrency (peak hours)
✓ Affects concurrent users (active users only)
✓ Unpredictable timing (hard to reproduce)
✓ No error message (silent corruption)
✓ Appeared after code change (v1.8.0)
✓ Happens under load (traffic required)
```

### Questions to Answer

- [ ] What's the incident rate?
- [ ] What's the peak hour vs. off-peak ratio?
- [ ] What user types are affected?
- [ ] When did the deployment that caused this occur?
- [ ] Why did it not manifest until Monday?
- [ ] Is the correlation to traffic conclusive?

### AI Prompt

```
Analyze the statistical patterns:

1. What's the correlation between traffic and incidents?
2. Why only 0.8% of operations fail?
3. Why do active users see more corruption?
4. What deployment caused this?

[paste incident_reports.md - focus on Statistical Analysis section]
```

---

## Phase 3: Code Review - Race Condition (15-20 minutes)

### Objective
Find the concurrent access issue in the code.

### The Read-Modify-Write Pattern

```
BREAKING CODE (v1.8.0):

Thread A (User 123 updating SMS settings):
  1. SELECT settings FROM users WHERE id=123
     → Gets: {email: ON, sms: OFF, language: EN}
  
  2. Merge new values: {sms: ON}
     → Results in: {email: ON, sms: ON, language: EN}
  
  3. UPDATE users SET settings=? WHERE id=123
     → Writes the merged value

THREAD B (User 123 updating email settings) - AT THE SAME TIME:
  1. SELECT settings FROM users WHERE id=123
     → Gets SAME value: {email: ON, sms: OFF, language: EN}
     → (Thread A hasn't written yet)
  
  2. Merge new values: {email: OFF}
     → Results in: {email: OFF, sms: OFF, language: EN}
  
  3. UPDATE users SET settings=? WHERE id=123
     → Writes this value (OVERWRITES Thread A's changes!)

RESULT:
- Thread A wanted SMS: ON (but it's OFF now!)
- Thread B's change succeeded (email: OFF)
- Thread A's change was lost (silent corruption!)
```

### The Fix

**Option 1: Use Database Locking (Pessimistic)**
```java
// Use FOR UPDATE to lock the row
SELECT settings FROM users WHERE id=? FOR UPDATE;
// Now modify
UPDATE users SET settings=? WHERE id=?;
// Row automatically unlocked after transaction
```
Pro: Simple, proven
Con: Slower, can deadlock

**Option 2: Use Atomic Compare-And-Swap**
```java
// Check version hasn't changed
UPDATE users SET settings=?, version=version+1 
WHERE id=? AND version=?
IF no rows updated: RETRY
```
Pro: Lock-free, faster
Con: Must retry on conflict

**Option 3: Use Separate Field Updates (Original)**
```java
// Update each field independently (old way)
UPDATE users SET sms_enabled=? WHERE id=?;
UPDATE users SET email_enabled=? WHERE id=?;
```
Pro: No race possible (each field locked by DB)
Con: Slower (multiple round trips)

### Questions to Answer

- [ ] What pattern causes the race (Read-Modify-Write)?
- [ ] Why does this work fine with 1 user but fail with concurrent users?
- [ ] What changed between v1.7.9 (worked) and v1.8.0 (broken)?
- [ ] How would you fix it?
- [ ] What's the best fix for your system?

### AI Prompt

```
Explain the race condition:

1. What code pattern causes it?
2. Why does it only happen with concurrent requests?
3. How would you fix it?
4. What's the tradeoff of each fix?

Context:
v1.7.9 (works): Multiple UPDATE statements per field
v1.8.0 (broken): Single SELECT + UPDATE for all fields

Problem: "Our settings get corrupted randomly"
Rate: 0.8% of operations
When: During peak traffic (concurrent users)
```

---

## Phase 4: Concurrency Tracing (10-15 minutes)

### Objective
Trace the exact sequence of events in the race window.

### The Timeline (Microseconds Matter)

```
Time    Thread A (SMS update)           Thread B (Email update)
─────────────────────────────────────────────────────────────
T0      SELECT settings                 [waiting]
        → Read: {email:ON, sms:OFF, lang:EN}

T1      [processing]                    SELECT settings
                                        → Read: {email:ON, sms:OFF, lang:EN}

T2      Merge {sms:ON}                  Merge {email:OFF}
        → {email:ON, sms:ON, lang:EN}   → {email:OFF, sms:OFF, lang:EN}

T3      UPDATE users SET settings=...   [processing]
        ✓ Writes successfully
        Now DB has: {email:ON, sms:ON, lang:EN}

T4      [done]                          UPDATE users SET settings=...
                                        ✓ Writes successfully
                                        NOW DB HAS: {email:OFF, sms:OFF, lang:EN}
                                        ↑ OVERWROTE Thread A's changes!

RESULT:
- Thread A's sms change: LOST
- Thread B's email change: KEPT (but overwrote A)
- User sees: email OFF (correct), sms OFF (should be ON) ❌
```

### Why This is Hard to Debug

1. **Race window is tiny** (microseconds)
   - Can't reproduce locally (single-threaded)
   - Only happens under load (concurrent traffic)
   - Timing is non-deterministic

2. **No error message**
   - Both queries succeed
   - Both updates succeed
   - Database doesn't know they race
   - Only result is "unexpected data state"

3. **Looks random**
   - Only happens when timing is exact
   - User A might never hit it
   - User B hits it repeatedly
   - Appears non-reproducible

4. **Audit trail is misleading**
   - Both updates show as successful
   - Only the final state is recorded
   - No log of "this update was overwritten"

### The Diagnostic Question

**"Can the same row be updated by two threads simultaneously?"**
- YES: Race condition possible
- NO: This pattern is safe

For the settings update:
- Yes, same user can update multiple settings at once
- Yes, two API threads can hit the same user's record
- = YES, race is possible
- = Code is broken

### Questions to Answer

- [ ] Trace the exact sequence that causes corruption
- [ ] What's the race window (how long in time)?
- [ ] Why can't we reproduce this locally?
- [ ] Why don't we get an error?
- [ ] How would you test for this?

### AI Prompt

```
Trace this race condition step-by-step:

Scenario: User has {email:ON, sms:OFF}
Thread A: Update SMS to ON (via API call 1)
Thread B: Update email to OFF (via API call 2)
Both called at same time

Show:
1. Timeline of each thread
2. Database state after each operation
3. Why final state is wrong
4. What the correct state should be
```

---

## Phase 5: Fix & Prevention (10 minutes)

### Immediate Fix (Today)

**Option A: Revert to v1.7.9**
```
Rollback v1.8.0
Deploy v1.7.9
- Slower (multiple DB round trips)
- But safe (no races)
- Buys time for proper fix
```

**Option B: Deploy v1.8.1 with database locking**
```
ADD to v1.8.0:
  SELECT settings FROM users WHERE id=? FOR UPDATE;
  // Then do merge and update
- Safe (database ensures atomicity)
- Not much slower (lock held briefly)
- Proper fix
```

**Recommendation:** Option B (proper fix, better long-term)

### Permanent Fix (1 week)

**Comprehensive Approach:**

1. **Code Fix**
   - Use database locks (FOR UPDATE)
   - OR use optimistic locking (version numbers)
   - Test concurrent updates

2. **Code Review Update**
   - Add checklist for concurrent access
   - "Can this code be called simultaneously for same resource?"
   - "Is the Read-Modify-Write atomic?"
   - "Have you tested with concurrent operations?"

3. **Testing**
   - Add tests for concurrent updates
   - Use thread pool, fire concurrent requests
   - Verify no data corruption

4. **Monitoring**
   - Add audit trail for settings changes
   - Alert if setting changes with no user action
   - Would catch this faster next time

### Prevention

**Never allow Read-Modify-Write on shared mutable state without:**
- Database-level locking (FOR UPDATE)
- Application-level locking (synchronized, ReentrantLock)
- Optimistic locking (version checks)
- Event sourcing (immutable append-only)

### Questions to Answer

- [ ] What's the quickest safe fix?
- [ ] Why is this better than rolling back?
- [ ] How do you test for race conditions?
- [ ] What should code review ask?
- [ ] How would you detect this in production?

---

## Phase 6: Postmortem (15-20 minutes)

### The Big Question
**"How did this get past code review?"**

### Root Cause of Root Cause

**What happened:**
- Code review approved batched settings update
- No one asked: "Is this atomic?"
- No one thought about: "What if two users update at same time?"
- Tests only used single-threaded scenarios

**Why it happened:**
- Concurrency bugs are subtle
- Code "looks right" (works single-threaded)
- Performance optimization bias (batching is "better")
- No existing template for concurrent code review

### Contributing Factors

1. **Insufficient Code Review**
   - Looked at: Performance, logic correctness
   - Didn't ask: Concurrent access implications
   - Didn't ask: Is Read-Modify-Write atomic?

2. **Single-threaded Testing**
   - Tests all passed
   - Tests don't use concurrent operations
   - Race condition invisible in test suite

3. **No Concurrency Testing**
   - No load/stress test before deploy
   - Would have triggered race condition
   - Required to verify concurrent code

4. **Silent Failure Mode**
   - Bug doesn't crash system
   - Bug doesn't return errors
   - Just silently corrupts data
   - Took 3 days for users to report

5. **Assumption of Atomicity**
   - Developers assumed: "DB UPDATE is atomic"
   - Truth: Only SINGLE UPDATE is atomic
   - Multiple operations: Not atomic
   - Batching violated this assumption

### Action Items

```
1. Fix the race condition (v1.8.1)
   Owner: engineering team
   Due: Today
   
2. Update code review checklist for concurrent code
   Owner: tech lead
   Due: This week
   Checklist items:
   - Can this be called for same resource concurrently?
   - Is every Read-Modify-Write atomic?
   - Have you used explicit locking (FOR UPDATE, Lock, etc)?
   
3. Implement concurrent operation testing
   Owner: QA team
   Due: Before next release
   - Fire N concurrent threads at each API endpoint
   - Verify no data corruption
   - Required for any state-modifying code
   
4. Add audit trail for settings changes
   Owner: platform team
   Due: 1 week
   - Log who changed what when
   - Use to detect unexpected changes
   - Use to recover from corruption
   
5. Create incident runbook for data corruption
   Owner: ops team
   Due: This week
   - How to detect
   - How to investigate
   - How to recover
```

### Lessons Learned

1. **Concurrency is a multiplier**
   - Single-threaded code: Works or fails clearly
   - Multi-threaded code: Barely works most of the time
   - Race conditions: Only fail under load, non-deterministic

2. **Performance optimizations must maintain safety**
   - Batching is good for performance
   - But if it breaks atomicity: Bad trade-off
   - Ask: "Am I trading correctness for performance?"

3. **Testing must match production conditions**
   - Single-threaded tests: Insufficient
   - Need concurrent operation testing
   - Need load/stress testing
   - Need realistic transaction patterns

4. **Code review must consider concurrency**
   - Add concurrency questions to checklist
   - Train reviewers in concurrency patterns
   - Require evidence of concurrent testing

5. **Atomicity is a contract**
   - Database atomicity: Only for single statement
   - Application atomicity: Must be enforced
   - Violation: Silent data corruption
   - Cost of mistake: High (data integrity)

---

**You've completed the Race Condition investigation!**

**Key Takeaway:** Concurrency bugs are subtle because:
1. They work most of the time (race window is tiny)
2. They look correct locally (single-threaded)
3. They fail silently (no error message)
4. They're timing-dependent (non-deterministic)

**The fix:** Always ask:
- "Can this be concurrent?"
- "Is it atomic?"
- "Have I tested with concurrent operations?"
- "What's my locking/synchronization strategy?"