# Copilot Instructions

## General

This repository supports various SONiC networking workflows. When assisting, use `az cli` and Kusto (Azure Data Explorer) for querying test data, logs, and telemetry. The self-hosted runner on this repo has access to Azure resources.

---

## SONiC DualToR Nightly Triage

### Context

DualToR (Dual Top-of-Rack) is a high-availability topology in SONiC where two ToR switches are connected to the same set of servers, providing redundancy and seamless failover. Below are instructions for triaging nightly test failures in this topology.

### Triage Workflow

1. **Identify failing tests**: Review the nightly test run results and identify tests that have failed. Focus on tests in the `dualtor` test module.

2. **Classify failures**:
   - **Infrastructure issues**: Testbed connectivity problems, link flaps, or environment setup failures.
   - **Known issues**: Match failures against known bugs in the backlog (Azure DevOps / GitHub Issues).
   - **New regressions**: Failures not matching any known issue that appeared after a recent code change.

3. **Analyze logs**: For each failure:
   - Check the pytest log for the assertion or error message.
   - Review the `syslog` and `sairedis.rec` from the DUT for relevant errors.
   - Check `mux_cable` status and `mux_simulator` logs for mux-related failures.
   - Look at `portchannel`, `bgp`, and `arp` state for connectivity issues.

4. **Common DualToR failure patterns**:
   - **Mux state mismatch**: `mux_cable` shows unexpected standby/active state. Check orchagent logs for mux state transitions.
   - **Traffic disruption during switchover**: Verify CRM counters and nexthop programming in `ASIC_DB`.
   - **Heartbeat/health monitor failures**: Check `linkmgrd` and `xcvrd` logs for link prober issues.
   - **Tunnel traffic failures**: Inspect IPinIP tunnel counters and `APP_DB` tunnel entries.
   - **Server-facing link issues**: Verify NiC simulator state and LACP status.

5. **Resolution actions**:
   - For infrastructure issues: File a testbed maintenance ticket.
   - For known issues: Link the failure to the existing work item and note the occurrence.
   - For new regressions: File a new bug with logs, attach the test run link, and identify the likely offending commit via `git log`.

### Key Tools and Queries

- Use `az cli` to query Kusto for historical test results and flakiness data.
- Query pattern: `TestResults | where TestName contains "dualtor" | where Result == "Failed" | summarize count() by TestName | order by count_ desc`
- Use the Azure Data Explorer (Kusto) database to correlate failures across multiple nightly runs.

### Priorities

- **P0**: All tests in a topology failing (likely infra issue) — escalate immediately.
- **P1**: New regression affecting core functionality (mux switchover, traffic forwarding).
- **P2**: Flaky test (passes on retry) — track frequency and investigate root cause if > 30% flake rate.
- **P3**: Cosmetic or non-blocking issues.
