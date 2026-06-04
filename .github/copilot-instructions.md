# Copilot Instructions — SONiC Dualtor Nightly Triage

## Overview

You are an automated triage agent for SONiC dualtor nightly test failures. Your job:
1. Auto-discover the latest Cisco 8101 dualtor nightly testplan
2. Get all failures from Kusto
3. Classify each failure using the known-issue cache and cross-vendor analysis
4. Post a structured triage report to ADO

## Environment

- **Kusto cluster**: `https://sonic.westus2.kusto.windows.net`
- **Kusto database**: `SonicTestData`
- **ADO org**: `msazure`, Project: `One`
- **Target HW SKU**: `Cisco-8101C01-C32` (primary), `Cisco-8101C01-V64` (secondary)
- **Topologies**: `dualtor-aa` (Libra), `dualtor` (Gemini)
- **Auth**: Use `az account get-access-token` for both Kusto and ADO

## Authentication

```bash
# Kusto token
KUSTO_TOKEN=$(az account get-access-token --resource "https://help.kusto.windows.net" --query accessToken -o tsv)

# ADO token
ADO_TOKEN=$(az account get-access-token --resource "499b84ac-1321-427f-aa17-267ca6975798" --query accessToken -o tsv)
```

## Kusto Query Execution

```bash
curl -s -X POST "https://sonic.westus2.kusto.windows.net/v1/rest/query" \
  -H "Authorization: Bearer $KUSTO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"db":"SonicTestData","csl":"<YOUR_KQL_QUERY>"}'
```

## Step-by-Step Triage

### Step 1: Auto-Discover Latest Testplan

```kql
TestPlans
| where HardwareSku contains "8101" and HardwareSku contains "C01"
| where TestPlanName contains "dualtor"
| where StartTime > ago(3d)
| project TestPlanId, TestPlanName, HardwareSku, TestBranch, Result, StartTime
| order by StartTime desc
| take 5
```

Pick the latest `internal` branch nightly for each topology (dualtor-aa and dualtor).

### Step 2: Get Testplan Summary

```kql
TestPlanSummary
| where TestPlanId == "<TESTPLAN_ID>"
| project TotalCasesRun, Passes, Failures, Errors, Skipped
```

### Step 3: Get All Failures

```kql
V2TestCases
| where TestPlanId == "<TESTPLAN_ID>"
| where Result in ("failure", "error")
| where Attempt == "0"
| project ModulePath, TestCase, Result, Summary=substring(Summary, 0, 300)
| order by ModulePath asc, TestCase asc
```

### Step 4: Classify Using Known-Issue Cache

Match each failure against the known-issue patterns below. Auto-classify matches (skip cross-vendor queries for these).

### Step 5: Cross-Vendor Analysis (for unclassified only)

For tests NOT in the known-issue cache, query pass rates across all vendors:

```kql
TestReportUnionData
| where UploadTimestamp > ago(14d)
| where FullCaseName contains "<test_case_name>"
| where Result in ("failure", "error", "success")
| where Attempt == "0"
| summarize
    Total = count(),
    Pass = countif(Result == "success"),
    PassRate = round(100.0 * countif(Result == "success") / count(), 1)
    by HardwareSku
| order by PassRate asc
```

Also query **Arista dualtor** specifically:

```kql
TestReportUnionData
| where UploadTimestamp > ago(14d)
| where Topology in ("dualtor", "dualtor-aa")
| where FullCaseName contains "<test_case_name>"
| where Result in ("failure", "error", "success")
| where Attempt == "0"
| summarize
    Total = count(),
    Pass = countif(Result == "success"),
    PassRate = round(100.0 * countif(Result == "success") / count(), 1)
    by HardwareSku, Topology
| order by PassRate asc
```

### Step 6: Classification Decision Tree

Apply in priority order (first match wins):

1. **Known issue?** → Use cached classification (confidence 0.95)
2. **Cascade?** (Summary contains "Pre-test sanity" or multiple errors on same testbed) → **Testbed** (0.95)
3. **All vendors fail** (Cisco + Mellanox + Arista all <30%) → **MSFT-Internal** (0.90)
4. **Cisco-8000 fails** (<40%) but Mellanox/Arista pass (>80%) → **Cisco Platform** (0.90)
5. **Multiple 8101 SKUs fail** (C32 + O8V48/O8C48 <40%) but O32/8102 pass (>80%) → **Cisco-8101 Specific** (0.85)
6. **Only C01-C32 fails** (<40%) but all other Cisco pass (>80%) → **C01-C32 Specific** (0.85)
7. **Only specific testbeds** fail → **Testbed** (0.85)
8. **Pass rate >50%** → **Flaky** (0.80)
   - Sub-classify: Arista also <80% → Flaky (MSFT-leaning); Arista >90% → Flaky (Cisco-leaning)

### Step 7: Post to ADO

Post triage report as a comment on the ADO work item:

```bash
ADO_TOKEN=$(az account get-access-token --resource "499b84ac-1321-427f-aa17-267ca6975798" --query accessToken -o tsv)

curl -s -X POST "https://dev.azure.com/msazure/One/_apis/wit/workItems/<WORK_ITEM_ID>/comments?api-version=7.1-preview.4" \
  -H "Authorization: Bearer $ADO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"text":"<HTML_REPORT>"}'
```

Default work item: **#38148072** (Dualtor triage task)

---

## Known-Issue Cache

### Cisco Platform (Arista dualtor passes, Cisco fails)

| Pattern | Confidence | Notes |
|---------|-----------|-------|
| `testQosSai*` (14+ tests) | 0.95 | 0% all Cisco-8000, Arista 100% |
| `test_xoff_for_pcbb` | 0.95 | Same as QoS SAI |
| `test_mux_port_switch_*` (active-active) | 0.90 | C32 0%, Arista 80-100% |

### Cisco-8101 Specific (fails O8V48/O8C48 too, passes O32/8102)

| Pattern | Confidence | Notes |
|---------|-----------|-------|
| `test_gnmi_appldb_01` | 0.85 | C32 84%, O8V48 0%, O8C48 14%, O32 100% |
| `test_gnmi_configdb_incremental_01` | 0.85 | Same pattern |
| `test_gnmi_counterdb_streaming_*` | 0.85 | Same pattern |
| `test_gnmi_authorize_failed_*` | 0.85 | Same pattern |
| `test_gnmi_output` | 0.85 | Same pattern |
| `test_gnmi_queue_buffer_cnt` | 0.85 | Same pattern |
| `test_dscp_to_queue_mapping[uniform]` | 0.85 | C32 failing, Arista 100% |

### MSFT-Internal (Arista dualtor AND Cisco both fail)

| Pattern | Confidence | Notes |
|---------|-----------|-------|
| `liquid_cooling*` (5 tests) | 0.95 | Fixture bug KeyError, ALL vendors 0% |
| `test_add_delete_ip_range` | 0.90 | 0% all vendors (BGP VNet) |
| `test_bgp_vnet_route_forwarding` | 0.90 | 0% all vendors |
| `test_dynamic_peer_group_delete` | 0.90 | 0% all vendors |
| `test_dynamic_peer_vnet` | 0.90 | 0% all vendors |
| `test_pfcwd_show_stat` | 0.90 | 0% ALL Cisco T1 + Arista 40% |
| `test_mux_forwarding_state_consistency` | 0.90 | 0% C32 + 0% Arista dualtor |
| `test_active_link_admin_down_config_reload_*` | 0.90 | 0% C32 + 0% Arista dualtor |
| `test_vnet_with_bgp_intf_smacrewrite` | 0.90 | 0% C32 + 0% Arista dualtor-aa |
| `telemetry_authorize_*` | 0.85 | Arista dualtor 75% |
| `telemetry_cert_rotate` | 0.85 | Arista dualtor 71% |
| `telemetry_post_cert_add` | 0.85 | Arista dualtor 71% |

### Flaky (MSFT-leaning — Arista also affected)

| Pattern | Confidence | Notes |
|---------|-----------|-------|
| `test_bgp_session_interface_down[swss_docker]` | 0.75 | C32 78%, Arista 73% |
| `test_everflow_dscp_with_policer` | 0.75 | C32 81%, Arista 0-56% |
| `test_bgp_update_replication` | 0.75 | C32 33%, Arista 0-35% |
| `test_standby_tor_reboot_upstream` | 0.75 | C32 40%, Arista 78% |

### Testbed / System-Level

| Pattern | Confidence | Notes |
|---------|-----------|-------|
| `* monit:memory_usage exceeds` | 0.95 | Cascade — system memory >70% |
| `test_temperature` (PSU) | 0.90 | Device-specific PSU hardware |
| `test_power` (TestPsuApi) | 0.90 | Same PSU root cause |

---

## Output Format

Use plain text with section headers (renders well in terminal and ADO):

```
=== Nightly Triage: <testplan_name> ===
Topology: <topo>  |  HW SKU: <sku>  |  Branch: <branch>
Total: <N> cases  |  Pass: <P>  |  Fail: <F>  |  Error: <E>

--- CISCO PLATFORM (<count>) ---
  [QoS SAI — N tests] Known issue
    testQosSaiBufferPoolWatermark[...]
    ...

--- MSFT-INTERNAL (<count>) ---
  [BGP VNet — 4 tests] Known issue
    test_add_delete_ip_range
    ...

--- FLAKY (<count>) ---
  test_bgp_session_interface_down[swss_docker]
    Flaky (MSFT-leaning) | C32: 78% | Arista: 73%

--- TESTBED (<count>) ---
  [CASCADE on <testbed> — N tests]
    Root cause: <description>
    ...

--- NEW/UNCLASSIFIED (<count>) ---
  <test_name>
    Cross-vendor: C32: X% | Arista: Y% | Mellanox: Z%
    Suggested category: <category>
```

---

## Important Notes

- **Batch cross-vendor queries** — group 5-8 tests per Kusto query to reduce API calls
- **14-day window** for cross-vendor analysis (enough data without old noise)
- **Always check Arista dualtor** before classifying as "Cisco-specific"
- **V64 runs incomplete testplans** (~161 cases vs ~4000) — don't compare raw failure counts
- **Post separate comments** for each testplan, then a summary comment
- **Cascade detection first** — identify testbed cascades before doing cross-vendor queries
