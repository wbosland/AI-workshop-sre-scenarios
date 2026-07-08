# Additional Scenario Ideas

Once you've run Scenarios 1-3, consider creating these additional incident scenarios:

---

## Scenario 4: The Cascading Failure

**Scenario Title:** Dependency Failure Propagation

**Difficulty:** Medium-High

**Key Concepts:**
- Timeout behavior
- Failure propagation
- Circuit breakers
- Graceful degradation
- Dependency management

**Incident Description:**
- Payment service depends on fraud-check service
- Fraud service goes down (database connectivity issue)
- Payment service doesn't have circuit breaker
- Payment requests timeout waiting for fraud response
- Timeouts pile up, exhausting thread pool
- Payment service becomes unavailable
- Cascading failure across platform

**Duration:** 3 hours
**Impact:** 15,000 failed transactions, 2 hours unavailability
**Investigation Complexity:** High (multiple services, trace the chain)

**What trainees learn:**
- How to trace failures across services
- Timeout and circuit breaker patterns
- Graceful degradation
- Dependency management
- How external failures propagate

---

## Scenario 5: The Configuration Disaster

**Scenario Title:** Typo in Production Config

**Difficulty:** Easy-Medium

**Key Concepts:**
- Configuration management
- Feature flags
- Deployment validation
- Secrets management
- Configuration drift

**Incident Description:**
- New database credentials deployed
- DevOps engineer typo in environment variable
- Service starts but can't connect to database
- Silent failure (health check too lenient)
- Only discovered when requests start failing
- 15 minutes of degraded service
- Config rollback required

**Duration:** 1 hour
**Impact:** Intermittent failures during rollout
**Investigation Complexity:** Low (good for newer engineers)

**What trainees learn:**
- Configuration validation
- Deployment safety checks
- Health check rigor
- Configuration rollback procedures
- Secrets management best practices

---

## Scenario 6: The Third-Party Integration Failure

**Scenario Title:** API Rate Limit Cascade

**Difficulty:** Medium

**Key Concepts:**
- Rate limiting
- Backoff strategies
- Retry storms
- Queue management
- Dependency API limits

**Incident Description:**
- Email service depends on third-party SMS API
- SMS API has 100 requests/second limit
- New feature increases SMS usage 5x
- System hits rate limit, starts getting 429 errors
- Default retry (immediate) creates retry storm
- Hits rate limit even harder
- SMS deliveries completely blocked
- Takes 2 hours to realize the issue

**Duration:** 2 hours
**Impact:** 50,000 SMS messages delayed/failed
**Investigation Complexity:** Medium (need to trace third-party)

**What trainees learn:**
- Rate limiting behavior
- Exponential backoff
- Queue-based handling
- Third-party dependency monitoring
- Graceful degradation with rate limits

---

## Scenario 7: The Data Migration Gone Wrong

**Scenario Title:** Production Data Corruption During Migration

**Difficulty:** High

**Key Concepts:**
- Data migration strategies
- Schema changes
- Backward compatibility
- Dual-write patterns
- Verification strategies

**Incident Description:**
- Database schema migration: Add NOT NULL column
- Null check forgotten in application
- Application writes NULLs to new column
- Constraint violation crashes requests
- Some data already migrated, some not
- Partial data corruption
- Rollback requires data recovery

**Duration:** 4 hours
**Impact:** 1,200 corrupted records, 30 min unavailability
**Investigation Complexity:** High (data integrity, multiple systems)

**What trainees learn:**
- Safe migration patterns
- Backward compatibility
- Data verification
- Rollback procedures
- Data integrity checks

---

## Scenario 8: The Observer Pattern Failure

**Scenario Title:** Observability Gaps Hide Real Problem

**Difficulty:** Medium

**Key Concepts:**
- Metrics vs. symptoms
- Observability gaps
- False alerts
- Correlation analysis
- Unknown unknowns

**Incident Description:**
- Metrics show everything fine (CPU, memory, latency)
- But users report: "API requests rejected with 'quota exceeded'"
- Team doesn't have quota usage metrics
- Quota system has different thresholds than expected
- No alerts on quota depletion
- Investigation takes 1 hour because metrics don't show issue
- Eventually discover: quota tracking code is broken

**Duration:** 1.5 hours
**Impact:** 30% of users get quota exceeded
**Investigation Complexity:** Medium-High (missing metrics)

**What trainees learn:**
- Importance of instrumentation
- Finding observability gaps
- When metrics can be misleading
- Correlation of symptoms to metrics
- Business metrics vs. technical metrics

---

## How to Create a New Scenario

### Step 1: Choose Your Incident Type
Pick from:
- Resource exhaustion (connection pools, memory, threads, disk)
- Performance degradation (latency, throughput)
- Data corruption (race conditions, concurrent access)
- Cascading failures (dependency failures, timeouts)
- Configuration errors (typos, version mismatches)
- Third-party issues (API failures, rate limits)
- Observability gaps (missing metrics, insufficient logging)

### Step 2: Gather the Materials
Create these 8 files for each scenario:
1. **README.md** - Scenario overview and roadmap
2. **timeline.md** - Event timeline with key dates/times
3. **[type]_logs.json** - Error logs with details
4. **metrics.json** - System metrics over time
5. **app_logs.jsonl** - Application activity logs
6. **code_diff.md** - Code changes (if applicable)
7. **config.md** - System configuration details
8. **INVESTIGATION_GUIDE.md** - 6-phase investigation walkthrough

### Step 3: Create the Timeline
- Start with the impact (what broke?)
- Work backward (when did it start?)
- Add deployment/change events (what changed?)
- Add mitigation/discovery events (when was it found?)
- Add resolution events (how was it fixed?)

### Step 4: Generate the Data
- **Logs:** Write realistic error messages
- **Metrics:** Create graphs showing the issue
- **Timeline:** Include key decision points
- **Code diff:** Show exactly what changed
- **Config:** Document the system setup

### Step 5: Create the Investigation Guide
Structure it as:
1. **Phase 1 - Detection:** How to see the problem
2. **Phase 2 - Diagnosis:** Gathering evidence
3. **Phase 3 - Analysis:** Finding root cause
4. **Phase 4 - Validation:** Testing the hypothesis
5. **Phase 5 - Resolution:** Fixing the issue
6. **Phase 6 - Postmortem:** Learning from it

Include:
- Objectives for each phase
- Specific tasks
- Questions to answer
- Common mistakes to avoid
- AI prompts to use
- Expected answers/insights

### Step 6: Test the Scenario
- Have someone work through it
- Time each phase
- Check if investigation is clear
- Verify all materials are consistent
- Adjust difficulty if needed

---

## Progression Path for Workshop

**Session 1 (90 min): Scenario 1**
- Connection pool exhaustion (sudden failure)
- Best for: First training session
- Difficulty: Medium
- Learns: Troubleshooting fundamentals

**Session 2 (90 min): Scenario 2**
- Memory leak (gradual degradation)
- Best for: After mastering Scenario 1
- Difficulty: Medium-High
- Learns: Trend analysis, monitoring

**Session 3 (90 min): Scenario 3**
- Race condition (intermittent)
- Best for: After Scenarios 1 & 2
- Difficulty: High
- Learns: Concurrency, statistics

**Session 4+ (choose based on team):**
- Use Scenarios 4-8 for specialized training
- Or create custom scenarios from real incidents

---

## Tips for Creating Great Scenarios

1. **Base on real incidents** - Use your incident history
2. **Vary difficulty** - Mix easy and hard
3. **Focus on learning goals** - What should trainees learn?
4. **Include red herrings** - Make investigation interesting
5. **Use realistic data** - Real logs, real timings
6. **Tell a story** - Timeline should feel like real incident
7. **Include multiple perspectives** - Logs, metrics, code, config
8. **Make investigation multi-step** - No obvious answers
9. **Document well** - Facilitator guide is crucial
10. **Test first** - Work through it yourself

---

**Next:** Create a scenario specific to your team's technology stack and common issues!