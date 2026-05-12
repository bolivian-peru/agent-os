# Swarm-Predict Example: Deploy PostgreSQL 17 from Source

This document provides a runnable example of the swarm-predict skill, demonstrating the full workflow from scenario selection to risk report generation.

## Quick Reference

- **Scenario:** Deploy PostgreSQL 17 from source on a Debian 12 server
- **Personas:** 6 experts (Ops Engineer, Security Analyst, Performance Engineer, End User, Cost Analyst, Chaos Engineer)
- **Expected Outcome:** GO WITH CONDITIONS (Confidence: 72%)

## When to Use This Example

Use this as a template when:

- First learning swarm-predict
- Testing the skill on a new osModa installation
- Validating that all 6 phases produce expected output format

## Prerequisites

A running osModa server with:

- `skills/swarm-predict/SKILL.md` deployed
- Access to the agent via web chat or Telegram
- Credentials configured for the `osmoda` agent

## Step-by-step Scenario

### Step 1: Activate the Skill

Connect to the osModa agent and submit the prompt:

```
I need to deploy PostgreSQL 17 from source on our Debian 12 server.
We currently run PostgreSQL 15 from apt.
Run a swarm-predict risk analysis first.
```

### Step 2: Phase 1 Output — Context Gathering

The agent gathers system state:

```
[system_health]
CPU: 38%, RAM: 51%, Disk: 67% (180GB free)
Load: 0.42, Uptime: 23 days
OS: Debian 12.5, Kernel 6.1.0

[system_query({ query: "services" })]
Running: postgresql-15 (port 5432, 12 active connections)
nginx (port 80/443)
docker (port 9423)

[journal_logs({ unit: "postgresql-15", lines: 30 })]
Apr 28 06:12:42 server postgres[1234]: checkpoint starting: time
Apr 28 06:27:51 server postgres[1234]: checkpoint complete: wrote 847 buffers
Apr 29 14:33:18 server postgres[1234]: could not receive data from client: Connection reset by peer
Apr 30 02:15:09 server postgres[1234]: could not connect to GSSAPI layer: Mechanism not available

[file_read({ path: "/etc/postgresql/15/main/postgresql.conf" })]
max_connections = 100
shared_buffers = 256MB
effective_cache_size = 768MB
wal_level = replica
```

**Situation Briefing:**
> Current server runs PostgreSQL 15 on Debian 12 with 180GB disk free, 12 active connections, and no high-load issues. We need to deploy PostgreSQL 17 from source while maintaining the existing PostgreSQL 15 instance for backward compatibility. The proposed change involves compiling from source with custom flags and managing two versions via systemd.

---

### Step 3: Phase 2 Output — Persona Selection

The agent selects 6 personas based on the scenario:

| Name | Role | Optimizes For | Blind Spot |
|------|------|---------------|-------------|
| Sarah Chen | Ops Engineer | Reliability, zero-downtime | Over-conservative, blocks progress |
| Viktor Petrov | Security Analyst | Attack surface, CVE history | Paranoid, sees threats everywhere |
| Priya Sharma | Performance Engineer | Latency, throughput | Optimistic about gains, ignores stability |
| Alex Thompson | End User | Response time, zero disruption | No technical context |
| James Park | Junior Dev | Simplicity, documentation | Naive questions reveal assumptions |
| Maya Rodriguez | Chaos Engineer | Failure modes, blast radius | Over-indexes on unlikely scenarios |

---

### Step 4: Phase 3 Output — Three-Round Debate

#### Round 1: Initial Reactions

**Sarah (Ops Engineer):**
> Biggest concern: Running two PostgreSQL versions simultaneously could conflict on port 5432. We need distinct ports or careful service management. Risk others miss: Source install means no apt package manager tracking for security updates. Initial position: CONDITIONAL - requires port isolation plan.

**Viktor (Security Analyst):**
> Biggest concern: PostgreSQL 17 has 47 new security features but also new attack surface. Source compile means we handle our own security patching. Risk others miss: PostgreSQL 17 changes GSSAPI behavior - the journal log already shows GSSAPI issues. Initial position: CONDITIONAL - needs security review of compile flags.

**Priya (Performance Engineer):**
> Biggest concern: PostgreSQL 17 promises 2x throughput on analytics queries but we need pg_config flags to enable. Risk others miss: Our shared_buffers=256MB is conservative for 8GB RAM server. Initial position: SUPPORT - this is a clear upgrade path.

**Alex (End User):**
> Biggest concern: Will my existing connections to the database work after migration? Risk others miss: I don't understand the difference between versions. Initial position: CONDITIONAL - needs clear migration guide.

**James (Junior Dev):**
> Biggest concern: How do we roll back if PostgreSQL 17 breaks? The apt version has automatic security updates but source doesn't. Risk others miss: Is there documentation for the new features in pg17? Initial position: NO-GO - too many unknowns.

**Maya (Chaos Engineer):**
> Biggest concern: Two postgres instances = two attack surfaces. If one gets compromised, does the other stay safe? Risk others miss: Source compile on Debian might pull dependencies with known CVEs. Initial position: CONDITIONAL - needs containerization or isolation.

---

#### Round 2: Challenge

**Sarah challenges Priya:**
> Your 2x throughput claim requires specific compile flags. Without `-DFORCE_ASSERTIONS` and proper `CFLAGS`, we get stock performance. The upgrade won't deliver what you promise.

**Priya responds:**
> Fair point. My analysis assumed optimal compile flags. The agent should use `./configure CFLAGS="-O2 -march=native"` and `--enable-cassert` for production. Without these, I'm reducing my confidence.

**Viktor challenges James:**
> Your NO-GO is based on "unknown unknowns." PostgreSQL has excellent rollback documentation - we can use `pg_upgrade` for data migration and keep pg15 as fallback. Rollback is well-tested.

**James responds:**
> The `pg_upgrade` path requires both versions installed simultaneously. My concern is service management - which one starts on boot? If systemd gets confused, we lose both databases.

**Maya challenges Sarah:**
> Your port isolation solution is correct but incomplete. What happens to the 12 existing connections during switchover? If we just change ports, applications need reconnect logic.

**Sarah responds:**
> Valid point. We need a migration window with: (1) keep pg15 on 5432, (2) start pg17 on 5433, (3) migrate apps one-by-one, (4) switch pg17 to 5432 after testing. This is doable but needs coordination.

---

#### Round 3: Final Position

**Sarah:**
> Recommendation: GO WITH CONDITIONS
> Condition: Port migration plan must be documented and tested before production cutover
> What breaks first: Application connection strings not updated to new port

**Viktor:**
> Recommendation: GO WITH CONDITIONS
> Condition: Compile with `-DFORCE_ASSERTIONS` disabled in production, GSSAPI behavior verified
> What breaks first: GSSAPI authentication fails for existing tunnel connections

**Priya:**
> Recommendation: GO WITH CONDITIONS
> Condition: Use optimized compile flags (`-O2 -march=native`) and run `pg Benchmark` before production
> What breaks first: Performance not meeting expectations if compile flags are wrong

**Alex:**
> Recommendation: GO WITH CONDITIONS
> Condition: Clear documentation of what features changed and how to use them
> What breaks first: Users confused by new query syntax or removed deprecated features

**James:**
> Recommendation: GO WITH CONDITIONS
> Condition: Full backup of pg15 data before any changes, rollback procedure documented
> What breaks first: Data corruption during upgrade if preconditions not met

**Maya:**
> Recommendation: GO WITH CONDITIONS
> Condition: Run pg17 in isolated environment first (Docker or separate VM) for 48 hours
> What breaks first: Side effects from new version affecting existing pg15 through shared resources

---

### Step 5: Phase 4 Output — Risk Report

```
## Risk Analysis Report

### Change
Deploy PostgreSQL 17 from source while maintaining existing PostgreSQL 15 for migration period

### System Context
- CPU: 38%, RAM: 51%, Disk: 180GB free
- Current: PostgreSQL 15 on port 5432, 12 active connections
- OS: Debian 12.5

### Consensus Risks
- Port conflict during dual-operation — Severity: HIGH
- Application connection string updates required — Severity: MEDIUM
- GSSAPI authentication behavior changes — Severity: MEDIUM

### Contested Risks
- Compile flag optimization for performance
  - Concerned: Priya (needs native flags)
  - Dismisses: Sarah (flags don't affect stability)
- Security patching responsibility
  - Concerned: Viktor (self-managed security)
  - Dismisses: Priya (security updates less frequent than major versions)

### Verdict: GO WITH CONDITIONS
Confidence: 72%

### Conditions
1. Install pg17 on port 5433, keep pg15 on 5432 during migration
2. Use optimized compile flags: `CFLAGS="-O2 -march=native" --enable-cassert`
3. Test GSSAPI authentication before production cutover
4. Document connection string changes for applications
5. Full `pg_dumpall` backup before migration window
6. Run 48-hour isolated test on non-production data first

### SafeSwitch Plan
- Pre-flight: Verify pg17 binary with `pg17 --version`, check disk space (>5GB for source build)
- TTL: 3600 seconds (1 hour for migration window)
- Health checks: `pg_isready -h localhost -p 5433`, connection count < current + 10
- Rollback trigger: pg17 process dies, connections drop to 0, disk space < 1GB
```

---

### Step 6: Phase 5 Output — Execution Confirmation

```
[User Response]
Approved for execution with conditions. Proceed with SafeSwitch deployment.

[Agent Response]
Executing swarm-predict deployment via SafeSwitch.

safe_switch_begin({
  plan: "Deploy PostgreSQL 17 from source on port 5433",
  ttl_secs: 3600,
  health_checks: ["pg_isready -h localhost -p 5433", "connections < 22"]
})

[Execution Log]
1. Downloaded postgresql-17.2.tar.gz
2. Compiled with flags: CFLAGS="-O2 -march=native"
3. Installed to /opt/pgsql-17/
4. Initialized new cluster on port 5433
5. Started pg17 service
6. Verified: pg_isready returns OK
7. Current connections: 12 on pg15, 0 on pg17

Status: GO FOR PHASE 2 - Application Migration
```

---

## Expected Output Shape Summary

A successful swarm-predict run produces:

| Phase | Key Outputs |
|-------|-------------|
| Phase 1 | system_health JSON, service_status list, journal_logs excerpt, situation briefing paragraph |
| Phase 2 | 6 persona cards with Name/Role/Optimizes/Blind-spot |
| Phase 3 | 3 rounds with labeled responses per persona |
| Phase 4 | Structured report with Consensus/Contested risks, Verdict, Confidence %, Conditions list |
| Phase 5 | safe_switch_begin call, execution log, status update |

## Common Failure Modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| No system data in Phase 1 | Agent skipped context gathering | Remind to run system_health first |
| All personas agree | Shallow analysis | Request Chaos Engineer persona specifically |
| Confidence doesn't match debate | Model over-estimating | Add round 4 (Red Team) to surface disagreement |
| Vague risks ("something might break") | Missing system data | Return to Phase 1 with more specific queries |

## Related Documentation

- [SWARM-PREDICT.md](../docs/SWARM-PREDICT.md) — Testing & handoff guide
- [SKILL.md](./SKILL.md) — Full skill specification
- [SafeSwitch Documentation](../docs/SAFESWITCH.md) — Deployment with auto-rollback