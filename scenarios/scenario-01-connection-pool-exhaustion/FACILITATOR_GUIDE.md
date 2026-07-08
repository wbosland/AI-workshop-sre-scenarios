# SRE Incident Investigation Workshop - Facilitator Guide

## Overview

This guide helps you run the AI-assisted incident investigation workshop for your service engineers. The workshop teaches real-world troubleshooting skills by walking trainees through a realistic incident scenario.

**Workshop Duration:** 60-90 minutes  
**Ideal Group Size:** 5-12 participants  
**Materials Required:** AI tool access (ChatGPT, GitHub Copilot, Claude, etc.)

---

## Before the Workshop

### 1. Prepare the Environment (15 minutes)

**Setup for trainees:**
- [ ] Clone/access the repository: `https://github.com/wbosland/AI-workshop-sre-scenarios`
- [ ] Ensure all trainees have access to an AI tool (ChatGPT, Copilot, Claude)
- [ ] Have the scenario files open in their editor or browser
- [ ] Test that trainees can access all 8 files

**Setup for facilitator:**
- [ ] Read through all scenario files
- [ ] Prepare printed copies of the investigation guide (optional)
- [ ] Have the "Answer Key" (see end of this guide) available
- [ ] Prepare a whiteboard or shared screen for timeline visualization
- [ ] Set up a timer for each phase

### 2. Participant Prerequisites

**This workshop is best for:**
- Service engineers with 6+ months experience
- On-call responders
- Junior engineers wanting to learn troubleshooting
- Teams preparing for incident response exercises

**Ideal group composition:**
- Mix of frontend, backend, and infrastructure engineers
- Include 1-2 junior and 1-2 senior members per group
- Teams of 3-4 work well for discussion

### 3. Pre-Workshop Communication

**Send to participants 24 hours before:**

```
Subject: AI Workshop - Incident Investigation Tomorrow

Hi team,

Tomorrow we're running an incident investigation workshop using AI tools to help troubleshoot a real-world production scenario.

What to prepare:
- Laptop with internet access
- Access to an AI tool (ChatGPT, GitHub Copilot, or Claude)
- ~90 minutes of focused time

What we'll cover:
- Using logs and metrics to diagnose problems
- Root cause analysis using the 5 Whys method
- How AI can accelerate troubleshooting
- Writing effective postmortems

No preparation needed - we'll walk through everything together.

See you tomorrow!
```

---

## Workshop Agenda

### Opening (5 minutes)

**What to say:**

```
"Welcome to our incident investigation workshop. 

Today you're a service engineer who just got paged at 2:32 PM. 
Payments aren't processing, customers are calling, and nobody knows why.

You have logs, metrics, and code changes. Your job: figure out what happened, why it happened, and how to fix it.

We'll work through this together using AI tools to help us analyze data and spot patterns.

By the end, you'll have practiced:
- Correlating events from multiple data sources
- Using root cause analysis (5 Whys)
- Proposing effective fixes
- Writing a postmortem

Ready? Let's go."
```

### Phase 1: Detection & Triage (5-10 minutes)

**Facilitator Role:** Guide and time-box

**Run the phase:**

1. **Set the stage (1 min)**
   - Project the README.md on screen
   - Read aloud: "At 14:32 UTC, these alerts fired..."
   - Show the 3 critical alerts

2. **Have trainees read (2 min)**
   - Ask them to quickly read `deployment_timeline.md`
   - Ask: "When did things break? What changed?"

3. **Review error logs (3-5 min)**
   - Display `error_logs.json`
   - Ask a trainee: "What's the top error?"
   - Facilitate discussion: "How many transactions affected?"

4. **Debrief (1 min)**
   - Confirm they identified:
     - Timing: 14:32 UTC
     - Error: Connection pool exhausted
     - Impact: 2,847 transactions

**Common Mistakes to Correct:**
- ❌ "The database crashed" → ✅ "The connection pool is exhausted"
- ❌ "It's a traffic spike" → ✅ "Traffic spiked, but why did pool exhaust?"
- ❌ "We need to increase resources" → ✅ "Resources are fine, need to find root cause"

**AI Prompt to Share:**
```
Here's the first error log from our incident. 
Summarize for me:
1. What's the most common error?
2. When did it start?
3. How many total errors?

[paste error_logs.json]
```

---

### Phase 2: Symptom Collection (10-15 minutes)

**Facilitator Role:** Coach correlation thinking

**Run the phase:**

1. **Metrics review (5 min)**
   - Have trainees look at `system_metrics.json`
   - Draw on whiteboard:
     ```
     Connections:  24 → 41 → 46 → 50 → 50 → 22
     Time:        14:15 14:25 14:30 14:32 14:45 15:13
     ```
   - Ask: "What does this pattern tell us?"

2. **Timeline construction (5 min)**
   - Have trainees build a timeline:
     ```
     13:52 - v2.4.1 deployed
     14:15 - Health check OK
     14:30 - Campaign starts (traffic 1.5x)
     14:32 - Pool exhausted, errors start
     14:35 - IC engaged
     14:55 - Root cause found
     15:12 - Rollback deployed
     15:17 - Recovered
     ```
   - Point out: "40 minutes between deployment and failure"

3. **Log analysis (5 min)**
   - Have trainees read `app_logs.jsonl`
   - Ask: "What do you see at 14:32?"
   - Point out: "See the 'retrying' messages? New code is retrying."

**Debrief (1 min)**
- Confirm they see the pattern:
  - ✅ Deployment happened first
  - ✅ System ran fine for 40 minutes
  - ✅ Traffic spike triggered failures
  - ✅ Connections never released (stayed at 50)
  - ✅ Rollback fixed it instantly

**AI Prompt to Share:**
```
Create a timeline of this incident showing:
1. Key events and their times
2. What the system was doing at each point
3. When things started going wrong

Here's the data:
[paste deployment_timeline.md]
[paste system_metrics.json]
[paste app_logs.jsonl]
```

**Common Mistakes to Correct:**
- ❌ "The campaign was too big" → ✅ "Campaign was normal load, system should handle it"
- ❌ "Database was slow" → ✅ "Database was fine, we couldn't connect"
- ❌ "We need better monitoring" → ✅ "Monitoring showed the problem, issue is in code"

---

### Phase 3: Root Cause Analysis - 5 Whys (15-20 minutes)

**Facilitator Role:** Socratic method - ask questions, don't give answers

**Run the phase:**

1. **Set up the framework (1 min)**
   - Explain the 5 Whys method
   - Write on board:
     ```
     Why #1: Why are connections exhausted?
     Why #2: Why aren't they being released?
     Why #3: Why?
     Why #4: Why?
     Why #5: ROOT CAUSE
     ```

2. **Walk through Whys (12-15 min)**

   **Why #1: Why are all 50 connections being used?**
   - Facilitate: "Look at metrics. Connections go from 41 to 50. Why?"
   - Guide to: "Either demand increased or connections aren't released"
   - Point out: "Campaign caused demand to increase, but normal systems handle this"

   **Why #2: Why aren't connections being released?**
   - Ask: "Something changed at deployment. What code changed?"
   - Have them look at `git_diff.md`
   - Highlight: "Retry logic changed from 3 attempts to 5"
   - Ask: "Could that cause connections to stay open?"

   **Why #3: How would more retries cause connection exhaustion?**
   - Ask: "Under normal load, would 5 retries vs 3 matter?"
   - Guide: "Not really. But when connections START failing, what happens?"
   - Point out in logs: "See the 'Connection acquisition failed, retrying' messages?"
   - Facilitate: "So we have MORE retries × connection failures = more connection attempts"

   **Why #4: Why would retries not cause failures with v2.4.0?**
   - Ask: "Look at the code diff. In v2.4.0, how were connections handled?"
   - Point out: "v2.4.0 had 3 retries with fixed delay"
   - Ask: "In v2.4.1, what's different?"
   - Guide to: "The connection is never explicitly closed in the exception handler"

   **Why #5: ROOT CAUSE - Why isn't the connection closed?**
   - Have them read the code in `git_diff.md`
   - Ask: "After the exception, where does the connection reference go?"
   - Point out: "It's lost, but not closed. That's a leak."
   - Confirm: "Under high load with retries, these leaks accumulate and exhaust the pool"

3. **Show the fix (2-3 min)**
   - Display the corrected code from `git_diff.md`
   - Explain try-with-resources
   - Show: "This ensures the connection is ALWAYS closed"

**Debrief (1 min)**
- Confirm the root cause chain:
  ```
  Leak in code → Accumulates under load → Pool exhausted → System fails
  Campaign triggered this by providing the load
  ```

**AI Prompt to Share:**
```
I found these code changes deployed before the incident. 
Analyze them for resource management issues:

1. What changed?
2. Are connections properly closed?
3. What happens if an exception occurs?
4. Could this cause a pool exhaustion under load?

[paste git_diff.md focusing on PaymentRetryHandler.java]
```

**Common Mistakes to Correct:**
- ❌ "The campaign caused this" → ✅ "Campaign exposed a pre-existing bug"
- ❌ "We need bigger pools" → ✅ "We need to fix the leak, not increase pool size"
- ❌ "The database settings are wrong" → ✅ "The code is leaking connections, settings fine"

---

### Phase 4: Hypothesis & Validation (10-15 minutes)

**Facilitator Role:** Encourage critical thinking

**Run the phase:**

1. **State the hypothesis (1 min)**
   - Write on board:
     ```
     HYPOTHESIS:
     The v2.4.1 retry logic has a connection leak.
     When getConnection() fails, the exception is caught
     but the connection isn't closed. Under campaign load,
     these leak and exhaust the pool.
     ```

2. **Validate against evidence (8-10 min)**
   - Ask trainees: "Does this hypothesis explain everything we observed?"
   - Walk through validation checklist:

   **Test 1: Timeline**
   - Question: "Why didn't it break at deployment (13:52)?"
   - Answer: "Leak takes time to accumulate"
   - Question: "Why did it break at 14:32?"
   - Answer: "Campaign traffic provided enough load for leaks to accumulate"
   - ✅ Timeline fits

   **Test 2: Error pattern**
   - Question: "Do the error logs show retry logic?"
   - Answer: "Yes - 'Connection acquisition failed, retrying'"
   - Question: "Do we see the 5 retries in the logs?"
   - Answer: "Yes - retry_count goes 0,1,2,3,4,5"
   - ✅ Error pattern fits

   **Test 3: Metrics**
   - Question: "Do connections stay at 50 because they're leaked?"
   - Answer: "Yes - all 50 are held, none released"
   - Question: "Why did rollback fix it?"
   - Answer: "v2.4.0 doesn't have the leak, connections released"
   - ✅ Metrics fit

   **Test 4: Rollback effect**
   - Question: "When rollback deployed, what happened?"
   - Answer: "Connections went from 50 to 22 immediately"
   - Question: "What does that tell us?"
   - Answer: "The leak was in the new code, old code works fine"
   - ✅ Rollback effect confirms it

3. **Consider alternatives (2 min)**
   - Ask: "Could the campaign just be too much traffic?"
   - Counter: "Campaign is 1.5x normal, system sized for peaks"
   - Ask: "Could the database settings be wrong?"
   - Counter: "Settings didn't change, pool size still 50"
   - Conclusion: "Connection leak is the only explanation that fits all data"

**Debrief (1 min)**
- Reinforce: "Good root cause analysis checks ALL the data"
- Point out: "We didn't assume, we validated"

**AI Prompt to Share:**
```
Does this hypothesis explain everything we observed?

Hypothesis: Connection leak in v2.4.1 retry logic

Evidence:
- Deployment at 13:52, failure at 14:32 (40 min gap)
- Errors show retry logic: "Connection acquisition failed, retrying"
- Connections stuck at 50/50, never released
- Rollback at 15:12 → connections drop to 22 immediately
- Campaign traffic 1.5x normal (provides load to trigger)

[paste error_logs.json]
[paste system_metrics.json]
[paste app_logs.jsonl]

Does the hypothesis match all observations?
Are there any contradictions?
```

---

### Phase 5: Remediation (10 minutes)

**Facilitator Role:** Lead discussion on immediate vs. long-term fixes

**Run the phase:**

1. **Immediate actions (2 min)**
   - Ask: "What do we do RIGHT NOW?"
   - Guide to:
     - ✅ Rollback to v2.4.0 (DONE in scenario)
     - ✅ Verify service is healthy
     - ✅ Process backlog
   - Point out: "In real incident, this was done at 15:12"

2. **Short-term fix (3 min)**
   - Display the code fix from `git_diff.md`
   - Show broken version:
     ```java
     Connection conn = null;
     try { conn = getConnection(); ... }
     catch { /* conn not closed */ }
     ```
   - Show fixed version:
     ```java
     try (Connection conn = getConnection()) { ... }
     ```
   - Explain: "Try-with-resources ensures automatic cleanup"
   - Ask: "How long to deploy this fix?"
   - Answer: "24-48 hours (code review, test, deploy)"

3. **Long-term improvements (5 min)**
   - Ask: "How do we prevent this next time?"
   - Facilitate discussion:

   **Prevention #1: Code Review**
   - Question: "Should this have been caught in code review?"
   - Answer: "Yes - connection leak is obvious if you look for it"
   - Action: "Add resource management checklist to code review"

   **Prevention #2: Load Testing**
   - Question: "Would load testing have caught this?"
   - Answer: "Yes - stress test would show the leak"
   - Action: "Test new versions at 2x expected traffic"

   **Prevention #3: Canary Deployment**
   - Question: "If we deployed to 1 instance first, what happens?"
   - Answer: "That 1 instance fails, but 3 others serve traffic"
   - Action: "Implement canary deployment for risky changes"

   **Prevention #4: Better Health Checks**
   - Question: "Why did health checks pass?"
   - Answer: "Health checks don't stress the connection pool"
   - Action: "Add health check that exercises connection pool under load"

   **Prevention #5: Monitoring**
   - Question: "Alert fired at 48/50 connections. Too late?"
   - Answer: "Yes - alert at 40/50 gives more time to respond"
   - Action: "Lower alert threshold and improve leak detection"

**Debrief (1 min)**
- Reinforce: "Good incident response has 3 phases:"
  1. Immediate mitigation (rollback)
  2. Quick fix (deploy corrected code)
  3. Prevention (process improvements)

**AI Prompt to Share:**
```
How should we fix this connection leak?

Broken code:
[paste PaymentRetryHandler from git_diff.md]

Provide:
1. Minimal fix to stop the leak
2. Explanation of why it works
3. Best practices for connection handling in Java
```

---

### Phase 6: Postmortem (15-20 minutes)

**Facilitator Role:** Guide documentation, model blamelessness

**Run the phase:**

1. **Set tone (1 min)**
   - Say: "We're writing a BLAMELESS postmortem"
   - Explain: "Our goal is to understand what happened and why the systems allowed it"
   - Clarify: "NOT to blame individuals, but to improve processes"

2. **Fill out template together (10-12 min)**
   - Have trainees work in their groups
   - Project the template from `INVESTIGATION_GUIDE.md`
   - Have someone read each section aloud as they fill it out:

   **Incident Summary Section:**
   - Title: "Database Connection Pool Exhaustion"
   - Date: 2024-07-08
   - Duration: 45 minutes
   - Impact: 2,847 failed transactions
   - Root Cause: Connection leak in retry logic

   **Timeline Section:**
   - Facilitate using the times from `deployment_timeline.md`
   - Show how clear timeline helps understanding

   **Root Cause Explanation:**
   - Have trainees write in their own words
   - Show how the 5 Whys explains the cause chain

   **Contributing Factors (NOT ROOT CAUSE):**
   - Emphasize: "These are system weaknesses, not failures"
   - Examples:
     - No load testing for v2.4.1
     - No canary deployment (all 4 instances at once)
     - Health checks too simplistic
     - Code review didn't catch resource leak pattern

   **What Went Well:**
   - Point out positive things:
     - Alerts fired quickly
     - IC engaged rapidly
     - Rollback was straightforward
     - Root cause found in 23 min

   **What Could Be Better:**
   - NOT "We should blame the engineer"
   - YES "Our process didn't catch this before production"

   **Action Items:**
   - For each contributing factor, propose an action:
     ```
     Contributing Factor: No load testing
     Action: Require load testing at 2x traffic for risky changes
     Owner: DevOps team
     Timeline: Before next major release
     ```

3. **Discuss lessons learned (3-4 min)**
   - Ask each trainee: "What's one thing you'll do differently?"
   - Examples:
     - "Check for resource leaks in code review"
     - "Stress test connection pools"
     - "Always use try-with-resources"
     - "Canary deploy risky changes"
     - "Lower alert thresholds for resource limits"

4. **Publish postmortem (1 min)**
   - Explain: "Real postmortems are shared with the team"
   - Share with:
     - Engineering team (what to learn)
     - Product (how long we were down)
     - Customers (transparency)
     - Oncall training (for future reference)

**Sample Postmortem Excerpt:**

```markdown
## What Went Well
- Alerts fired immediately (14:32)
- Incident commander engaged by 14:35
- Root cause identified in 23 minutes (14:55)
- Rollback was straightforward
- Service recovered cleanly (15:17)

## What Could Be Better
- Load testing didn't stress connection pool
- Deployed to all 4 instances at once (no canary)
- Code review missed resource leak pattern
- Health checks were too simplistic
- Alert threshold left no margin (48/50)

## Action Items
1. Load test at 2x traffic before production (Owner: Eng, Due: Next sprint)
2. Implement canary deployment (Owner: DevOps, Due: 2 weeks)
3. Add resource management to code review checklist (Owner: Tech Lead, Due: This week)
4. Add connection pool stress test to health check (Owner: Ops, Due: 1 week)
5. Lower alert threshold to 40/50 (Owner: Monitoring, Due: Today)
```

---

## Closing (5 minutes)

**What to say:**

```
"Great work today. You just did what our on-call engineers do every day:

1. Stayed calm under pressure
2. Collected data before jumping to conclusions
3. Asked 'why' repeatedly until finding the real cause
4. Proposed fixes at multiple levels
5. Documented to help the team learn

The tools you used today - logs, metrics, code diffs - are your best friends
when things break in production.

And AI tools like ChatGPT can help you work faster:
- Parsing logs to find patterns
- Correlating events across systems
- Reviewing code for issues
- Writing clear explanations

Your next incident? You'll handle it better.

Any questions?"
```

**What to ask at the end:**

1. "What was the most surprising thing you learned?"
2. "What would you do differently if you hit this in production?"
3. "How would you have caught this earlier?"

---

## Answer Key - Phase by Phase

### Phase 1: Detection & Triage
**Expected answers:**
- When: 14:32:15 UTC
- What: Database connection pool exhausted
- How many: 2,847 affected transactions
- Why: v2.4.1 deployed 13:52, campaign traffic started 14:30

### Phase 2: Symptom Collection
**Expected timeline:**
```
13:52 - v2.4.1 deployed (4 instances)
14:15 - Health check passed (24/50 connections)
14:30 - Campaign traffic starts (41/50 connections)
14:31 - High utilization warning (46/50)
14:32 - FAILURE (50/50 - all connections held)
14:35 - IC engaged
14:45 - Situation worsening (3,401 waiting requests)
14:55 - Root cause identified
15:10 - Rollback decision
15:12 - v2.4.0 deployed
15:13 - Connections drop to 22, recovery begins
15:17 - Backlog cleared, incident resolved
```

### Phase 3: Root Cause Analysis
**Expected 5 Whys:**
1. Why exhausted? → Connections not released
2. Why not released? → New code doesn't close them on exception
3. Why doesn't new code close them? → Manual management without try-with-resources
4. Why manual management? → Try-with-resources not used in retry logic
5. ROOT CAUSE → Code review didn't catch resource leak pattern

### Phase 4: Hypothesis Validation
**Expected validation:**
- ✅ Timeline: Leak accumulates over 40 min, traffic spike triggers
- ✅ Errors: See "Connection acquisition failed, retrying"
- ✅ Metrics: All 50 connections held continuously
- ✅ Rollback: Immediate recovery when v2.4.0 deployed
- ✅ Rollback effect: Connections drop from 50 to 22 instantly

### Phase 5: Remediation
**Expected fixes:**
- Immediate: Rollback (done at 15:12)
- Short-term: Deploy v2.4.2 with try-with-resources fix
- Long-term: 
  - Load testing in CI/CD
  - Canary deployment process
  - Code review checklist for resource management
  - Better health checks
  - Lower alert thresholds

### Phase 6: Postmortem
**Expected components:**
- Title: Database Connection Pool Exhaustion
- Duration: 45 minutes
- Impact: 2,847 transactions
- Root cause: Connection leak in retry logic
- Contributing factors: 5 items (no load test, no canary, etc.)
- Action items: 5+ items with owners and timelines
- Lessons learned: Blameless perspective on what to improve

---

## Facilitation Tips

### Do's ✅
- ✅ Ask questions instead of giving answers
- ✅ Let trainees struggle a bit - learning happens in confusion
- ✅ Validate good thinking: "That's good pattern recognition"
- ✅ Point out when they miss something: "Look at the timestamps again"
- ✅ Use the whiteboard to visualize timelines and patterns
- ✅ Encourage AI tool usage - that's realistic in real incidents
- ✅ Emphasize blamelessness in the postmortem
- ✅ Celebrate good detective work

### Don'ts ❌
- ❌ Tell them the answer immediately
- ❌ Let them make big assumptions without evidence
- ❌ Ignore when they blame people instead of systems
- ❌ Skip the postmortem - it's where learning happens
- ❌ Rush through phases - reflection is important
- ❌ Pretend you don't know the answer (be ready to reveal it)
- ❌ Make it feel like a test with "right" answers
- ❌ Forget to point out what went well

### Managing Group Dynamics

**If someone dominates:**
- "Great thinking. Let's hear from someone else."

**If someone is quiet:**
- "What do YOU think is happening in this log?"

**If group disagrees:**
- "Both theories are possible. What evidence would prove one right?"

**If group gets stuck:**
- "Let's use the AI tool. Ask it to parse these logs and find patterns."

**If group is going wrong direction:**
- "That's reasonable, but let's look at the timeline again."

---

## Variations & Extensions

### Difficulty Levels

**Level 1: Easy (45 minutes)**
- Provide the timeline already highlighted
- Point out the connection pool error
- Help with git diff interpretation
- Focus on: triage → root cause → postmortem

**Level 2: Medium (60 minutes)**
- Full scenario as designed
- Trainees find timeline themselves
- No hints on root cause
- Full 6 phases

**Level 3: Hard (90+ minutes)**
- Scenario with more mixed data
- Red herrings in the logs
- Need to correlate multiple services
- Unclear timeline, must reconstruct

### Competitive Version

**Teams compete to:**
- Find root cause fastest ⏱️
- Best analysis depth 📊
- Most comprehensive postmortem 📝

**Scoring:**
- Detection speed: 10 points
- Correct root cause: 20 points
- Hypothesis validation: 15 points
- Fix quality: 15 points
- Postmortem completeness: 20 points
- **Total: 80 points**

### Live Variation

**Instead of static logs:**
- Deploy a broken application to staging
- Let teams SSH in and investigate live
- Provide actual system commands (logs, metrics, etc.)
- Introduce new "events" in real-time
  - "It's now 14:45... situation getting worse"
  - "It's now 15:10... decision to rollback made"

---

## Troubleshooting Common Issues

### "People are spending too long on Phase 3"
**Solution:**
- Set a 15-minute timer for Phase 3
- After 10 min, start nudging them toward root cause
- Use a "hint ladder":
  1. "Look at the code changes in git_diff.md"
  2. "Focus on connection management"
  3. "Look for close() or try-with-resources"
  4. "The connection isn't closed when exception occurs"

### "People aren't using the AI tool"
**Solution:**
- Model it yourself: "Let me ask Copilot to parse these logs"
- Give them specific prompts: "Ask it: 'What errors happened before the failure?'"
- Show them the time savings

### "People are blaming individuals in the postmortem"
**Solution:**
- Gently redirect: "How could the PROCESS have prevented this?"
- Rephrase: "Instead of 'Engineer didn't review code', say 'Code review process didn't catch resource leaks'"
- Explain blameless postmortems: "We improve systems, not blame people"

### "The group found a different root cause"
**Solution:**
- Validate their thinking: "That's logical, let's test it"
- Ask: "What evidence proves this?"
- Guide back: "Look at what happened when we rolled back..."
- If they're right about something else, note it: "Good catch!"

---

## Post-Workshop Follow-Up

### Within 24 hours:
- Send summary of postmortem to team
- Highlight key lessons learned
- Share any action items that require follow-up

### Within 1 week:
- Check if any action items were implemented
- Follow up with participants: "Have you seen this pattern in real incidents?"
- Gather feedback: "What could make this workshop better?"

### For next time:
- Update scenario if same issue happens in production
- Create new scenarios based on recent incidents
- Vary difficulty levels
- Run it for new team members

---

## Creating Additional Scenarios

Once you've mastered this one, consider creating scenarios for:

1. **Memory Leak (Gradual Degradation)**
   - Slow increase in memory over days
   - Eventually OOM kill
   - Need to correlate metrics over longer timeframe

2. **Race Condition (Intermittent Issues)**
   - Errors happen randomly, ~1% of requests
   - Hard to reproduce locally
   - Need to read concurrent code carefully

3. **Configuration Error (Deployment Goes Wrong)**
   - Typo in config file
   - Service starts but fails silently
   - Need to diff configuration files

4. **Dependency Issue (Cascading Failure)**
   - External service returns 503
   - Our service doesn't handle gracefully
   - Need to understand failure propagation

5. **Rate Limiting (Unexpected Threshold)**
   - New code adds retry logic
   - Triggers rate limiting from dependency
   - Need to think about second-order effects

---

## Final Tips

1. **Enjoy it** - Incident investigation is detective work. It's fun!
2. **Learn from it** - Every trainee experiences help the team
3. **Iterate on it** - Update the scenario based on feedback
4. **Share it** - Other teams likely have similar training needs
5. **Remember** - This scenario is based on real incidents. It happens.

---

**Questions or feedback? Update this guide as you learn what works best for your team!**
