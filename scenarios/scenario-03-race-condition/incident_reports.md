# User Incident Reports & Pattern Analysis

## Timeline of Reports

### Monday 2024-07-15

**08:30 UTC - Report #1**
```
User: alice.johnson@example.com
Subject: Email preferences reset overnight

"I had email notifications turned OFF on Sunday evening.
This morning (Monday), they're ON again. This is the second 
time this month this happened. Is this a bug?"

Support response: "Let me check your account history..."
Findings: Email preference changed from OFF → ON at 2024-07-14 22:47:32 UTC
No user action in audit log for that timestamp
```

**09:15 UTC - Report #2**
```
User: bob.smith@example.com
Subject: Notification settings messed up

"My notification settings show options I never configured.
I never set up SMS notifications, but it shows SMS is ON.
When did this happen?!"

Support findings: SMS notification enabled at 2024-07-14 19:33:18 UTC
No user action, no API call, no admin change
SMS setting was OFF at 19:33:10 UTC, ON at 19:33:22 UTC
```

**09:45 UTC - Report #3 (in support Slack)**
```
User: charlie.williams@example.com
Message: "Is anyone else having issues with their profile 
settings changing randomly? My language preference changed 
from French to English without me doing anything."

Replies: 4 other users confirm similar issues
```

**14:20 UTC - Escalation to Engineering**
```
Support manager: "We're seeing a pattern of user settings 
being randomly changed. Started getting reports this morning.
Can't reproduce, but it's clearly happening."

Eng response: "Probably user confusion or browser cache. 
Our settings API is pretty bulletproof."

Support: "Users are certain they didn't make these changes."
```

### Tuesday 2024-07-16

**03:00 UTC - New batch of reports**
```
Since Monday evening, 87 new complaints received
All follow same pattern: Settings changed without user action
Common settings affected:
- Email preferences (most common)
- Notification types
- Language
- Privacy levels

Support ticket created: URGENT-15234
```

**08:15 UTC - Engineering Investigation Starts**
```
Engineer: "Let me check the code. Settings are updated with
standard REST API. We have database constraints."

Looks at: No errors in logs, no obvious issues
Conclusion: "Probably a small issue, maybe 1-2 edge cases"
```

**12:30 UTC - More Data Arrives**
```
Stats gathered:
- 147 affected users (so far)
- 234 setting change events with no audit trail
- All changes occurred during peak traffic times (evening UTC)
- Users in different geographic regions
- All happened to settings API calls

Engineer: "Wait, all during peak traffic? Let me check 
concurrency patterns..."
```

### Wednesday 2024-07-17

**06:00 UTC - Pattern becomes clear**
```
Stats update:
- 847 affected users now
- 1,340 corruption incidents
- Peak hours correlation identified:
  Peak traffic hours: 18:00-22:00 UTC = 89% of incidents
  Off-peak hours: 01:00-08:00 UTC = 4% of incidents
  
Engineer: "This is a concurrency issue. Something about 
high load is triggering corruptions."
```

**08:45 UTC - Code Review Meeting**
```
Calls Friday's release (v1.8.0) into question

Release notes say: "Optimize settings API with batched updates"

Code change in UserSettingsController:
1. Read current settings
2. Merge with new values
3. Validate
4. Write to database

Reviewer asks: "Is this atomic?"
Author: "Shouldn't need to be, single UPDATE statement"

Reviewer: "Hmm, read-modify-write pattern can be racy..."
Author: "Should be fine at our scale"

Approved anyway (sigh)
```

**14:00 UTC - ROOT CAUSE IDENTIFIED**
```
Race condition found!

Thread A and Thread B both updating same user's settings:

Thread A: 
  1. SELECT settings FROM users WHERE id=123
  2. [gets: {email: ON, sms: OFF}]
  3. Merge with new: {sms: ON}
  4. [prepared update: {email: ON, sms: ON}]
  5. UPDATE users SET settings=? WHERE id=123

Thread B (at same time):
  1. SELECT settings FROM users WHERE id=123
  2. [gets: {email: ON, sms: OFF}] <-- same initial value!
  3. Merge with new: {email: OFF}
  4. [prepared update: {email: OFF, sms: OFF}]
  5. UPDATE users SET settings=? WHERE id=123

Result: Whichever UPDATE executes last wins,
discarding the other thread's changes.
Thread B's update (email: OFF) executes last.
Thread A's (sms: ON) is lost!
```

---

## Statistical Analysis

### Corruption Rate
- **Monday 08:30 - Tuesday 08:00 UTC (24 hours):** 287 incidents
- **Tuesday 08:00 - Wednesday 08:00 UTC (24 hours):** 742 incidents
- **Rate:** 0.8% of all settings updates (1,029 out of 128,625 requests)

### Time Distribution
```
Hour UTC | Incident Count | Traffic Level
---------|----------------|---------------
01:00    | 2              | Low (8%)
02:00    | 1              | Low (7%)
...
12:00    | 18             | Medium (45%)
13:00    | 25             | Medium (52%)
14:00    | 34             | Medium (48%)
15:00    | 42             | High (65%)
16:00    | 51             | High (72%)
17:00    | 68             | High (78%)
18:00    | 142            | PEAK (89%)
19:00    | 156            | PEAK (91%)
20:00    | 168            | PEAK (92%)
21:00    | 134            | PEAK (87%)
22:00    | 89             | High (74%)
23:00    | 52             | Medium (61%)
```

**Correlation:** 89% of incidents occur during peak traffic (18:00-22:00 UTC)

### Affected User Types
- Active users (>5 logins/day): 94% of incidents
- Moderate users (1-5 logins/day): 5% of incidents
- Rare users (<1 login/day): <1% of incidents

**Why:** Active users make concurrent settings API calls

### Affected Settings
```
Setting Type        | Incidents | % of Total
--------------------|-----------|----------
Email preferences   | 412       | 40%
Notification type   | 298       | 29%
Language preference | 186       | 18%
Privacy settings    | 89        | 8%
UI preferences      | 44        | 4%
```

---

## Investigation Findings

### Why Was This Hard to Catch?

1. **Intermittent:** Only 0.8% of operations fail
2. **Under load:** Requires thousands of concurrent users
3. **No error message:** Silent corruption, not crash
4. **Timing issue:** Race window is microseconds
5. **Locally reproducible:** Hard to hit race in tests
6. **No deployment errors:** Code compiles fine
7. **Existing feature:** API worked fine before Friday

### What Changed on Friday?

**v1.8.0 release notes:** "Optimize settings API with batched updates"

**Before (v1.7.9):**
```java
// Each field updated independently
db.update("UPDATE users SET email_enabled = ? WHERE id = ?", 
          emailOn, userId);
db.update("UPDATE users SET sms_enabled = ? WHERE id = ?", 
          smsOn, userId);
```
This was slow but safe (each field locked separately)

**After (v1.8.0):**
```java
// Batched update - read settings, merge, write all at once
Settings current = db.query("SELECT settings FROM users WHERE id = ?", userId);
Settings updated = current.merge(newSettings);
db.update("UPDATE users SET settings = ? WHERE id = ?", 
          updated, userId);
```
This is faster but introduced race condition!

---

## Key Insight

**Race Window:**
```
Time T0:   Thread A reads {email: ON, sms: OFF}
Time T1:   Thread B reads {email: ON, sms: OFF} (same!)
Time T2:   Thread A merges in {sms: ON} → {email: ON, sms: ON}
Time T3:   Thread B merges in {email: OFF} → {email: OFF, sms: OFF}
Time T4:   Thread A writes {email: ON, sms: ON}
Time T5:   Thread B writes {email: OFF, sms: OFF} ← OVERWRITES A's changes!

Result: Thread A's sms change is lost!
```

This is a classic **Read-Modify-Write race condition**.

---

**Next: Review `code_diff.md` to see the exact code change**