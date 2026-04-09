# Network Change Checklist

Quick reference for validating network changes through lab, non-prod, and staging before touching production. Use alongside the `net-devops`, `net-proto`, and `net-docs` agents.

## Table of Contents

- [Pre-Change Validation](#pre-change-validation)
- [Lab / Non-Prod Testing](#lab--non-prod-testing)
- [Staging Gates](#staging-gates)
- [Production Change Execution](#production-change-execution)
- [Post-Change Verification](#post-change-verification)
- [Rollback Procedure](#rollback-procedure)
- [Common Anti-Patterns](#common-anti-patterns)

---

## Pre-Change Validation

Complete before any lab or production work begins.

- [ ] Change peer-reviewed by a second engineer
- [ ] Config generated from source of truth (NetBox / Nautobot) — no hand-edited production configs
- [ ] Config diff reviewed (`napalm.compare_config()` or `show | compare` on Junos / `show archive config differences` on IOS-XR)
- [ ] Intended impact documented: which prefixes, sessions, flows, or services are affected
- [ ] Maintenance window booked; stakeholders notified
- [ ] Rollback plan written and reviewed (not just "reverse the change" — specific commands)
- [ ] Change record raised (CAB ticket or equivalent) with pre-checks, steps, verification, backout

```bash
# Batfish pre-flight: parse configs and check for reachability regressions
python3 - <<'EOF'
from pybatfish.client.session import Session
from pybatfish.datamodel import *

bf = Session(host="localhost")
bf.set_network("my-network")
bf.init_snapshot("./configs", name="pre-change", overwrite=True)

# Check for undefined references
print(bf.q.undefinedReferences().answer().frame())

# Reachability baseline
result = bf.q.reachability(
    pathConstraints=PathConstraints(startLocation="/edge.*/"),
    headers=HeaderConstraints(dstIps="10.0.0.0/8"),
    actions="SUCCESS"
).answer().frame()
print(result)
EOF
```

---

## Lab / Non-Prod Testing

**All changes must pass lab validation before staging or production.**

### Containerlab topology

Spin up a representative lab topology that mirrors the production segment being changed.

```yaml
# containerlab topology example: spine-leaf with eBGP
name: spine-leaf-lab
topology:
  nodes:
    spine1:
      kind: ceos
      image: ceos:4.30.0F
      startup-config: configs/spine1.cfg
    spine2:
      kind: ceos
      image: ceos:4.30.0F
      startup-config: configs/spine2.cfg
    leaf1:
      kind: ceos
      image: ceos:4.30.0F
      startup-config: configs/leaf1.cfg
    leaf2:
      kind: ceos
      image: ceos:4.30.0F
      startup-config: configs/leaf2.cfg
  links:
    - endpoints: ["spine1:eth1", "leaf1:eth1"]
    - endpoints: ["spine1:eth2", "leaf2:eth1"]
    - endpoints: ["spine2:eth1", "leaf1:eth2"]
    - endpoints: ["spine2:eth2", "leaf2:eth2"]
```

```bash
# Deploy lab
containerlab deploy -t topology.yml

# Tear down after testing
containerlab destroy -t topology.yml --cleanup
```

### Lab validation checklist

- [ ] Lab topology mirrors the affected production segment (same platform, same OS version where possible)
- [ ] Change applied to lab nodes using the **exact same automation** that will run in production
- [ ] Control-plane convergence verified (BGP sessions up, OSPF adjacencies, EVPN routes)
- [ ] Data-plane forwarding verified (end-to-end ping / traceroute across affected paths)
- [ ] Failure scenarios tested: link failure, node reload, control-plane restart
- [ ] Convergence time measured and acceptable (document actual values)
- [ ] Rollback procedure tested in lab and confirmed to work
- [ ] Lab test results documented (show commands, counters, convergence timers)

### pyATS / Genie validation

```python
# Snapshot before and diff after applying change
from genie.testbed import load
from genie.utils.diff import Diff

testbed = load("testbed.yml")
device = testbed.devices["spine1"]
device.connect()

# Before snapshot
before = device.parse("show bgp summary")

# Apply change here ...

# After snapshot
after = device.parse("show bgp summary")

# Diff
diff = Diff(before, after)
diff.findDiff()
print(diff)
```

### Netmiko idempotency check

```python
from netmiko import ConnectHandler

def apply_and_verify(device_params: dict, config_lines: list[str]) -> bool:
    with ConnectHandler(**device_params) as net_connect:
        # Dry-run: check current state
        output_before = net_connect.send_command("show running-config")

        # Apply config
        net_connect.send_config_set(config_lines)
        net_connect.save_config()

        # Re-apply to confirm idempotency (should produce no diff)
        net_connect.send_config_set(config_lines)
        diff = net_connect.send_command("show archive config differences")

        if diff.strip():
            print(f"[WARN] Non-idempotent — diff on second apply:\n{diff}")
            return False
        return True
```

---

## Staging Gates

**Nothing moves to production until every gate is green.**

| Gate | Requirement | Tool |
|------|-------------|------|
| Config syntax | Zero parse errors | Batfish `initSnapshot` |
| Reference integrity | No undefined interfaces or ACLs | Batfish `undefinedReferences` |
| Reachability | No critical paths broken vs baseline | Batfish `reachability` / `differentialReachability` |
| Control plane | All expected sessions/adjacencies up | pyATS / NAPALM getters |
| Data plane | Bidirectional forwarding confirmed | Ping / traceroute from lab nodes |
| Idempotency | Second apply produces no diff | Netmiko / NAPALM `compare_config` |
| Rollback | Rollback tested and produces clean state | Lab test run |
| Peer review | Second engineer has reviewed diff and lab results | Manual sign-off |

```python
# Batfish differential reachability: catch regressions vs snapshot baseline
from pybatfish.client.session import Session
from pybatfish.datamodel import *

bf = Session(host="localhost")
bf.set_network("my-network")

# Load baseline (before change) and candidate (after change) snapshots
result = bf.q.differentialReachability(
    pathConstraints=PathConstraints(startLocation="/edge.*/"),
    headers=HeaderConstraints(dstIps="0.0.0.0/0"),
).answer(
    snapshot="candidate",
    reference_snapshot="baseline"
).frame()

if not result.empty:
    print("[FAIL] Reachability regressions detected:")
    print(result)
else:
    print("[PASS] No reachability regressions")
```

---

## Production Change Execution

- [ ] Confirmed maintenance window is active; NOC/on-call notified
- [ ] Pre-change `show` outputs captured and saved (BGP summary, route counts, interface status, OSPF neighbours)
- [ ] Automation script dry-run executed first (`--check` / `--diff` mode)
- [ ] Change applied device-by-device, not to all devices simultaneously
- [ ] Each device verified before proceeding to the next
- [ ] Traffic / telemetry dashboards monitored throughout

```bash
# Ansible dry-run before committing
ansible-playbook site.yml --check --diff -l spine1

# Apply to one device first, verify, then proceed
ansible-playbook site.yml -l spine1
# verify spine1 ...
ansible-playbook site.yml -l spine2
```

```python
# NAPALM commit with confirmation timer (auto-rollback if not confirmed)
import napalm

driver = napalm.get_network_driver("eos")
device = driver(hostname="spine1", username="admin", password="secret")
device.open()

device.load_merge_candidate(filename="changes/spine1.conf")
print(device.compare_config())  # Review diff before committing

# Commit with 5-minute confirmation window — device rolls back if not confirmed
device.commit_config(revert_in=300)

# After verifying traffic/control-plane is healthy:
device.confirm_commit()
device.close()
```

---

## Post-Change Verification

Run immediately after the change window, before closing the maintenance window.

- [ ] BGP sessions: expected peer count matches, no sessions in `Idle` or `Active`
- [ ] OSPF / IS-IS: adjacency count unchanged, no unexpected LSA floods
- [ ] Route table: prefix counts within expected range (±5% baseline)
- [ ] Interface status: no unexpected `down` interfaces
- [ ] Traffic: flows confirmed (bidirectional ping / traceroute on critical paths)
- [ ] Telemetry: no anomalous spikes in CPU, memory, or error counters
- [ ] Application / service health confirmed by service owner

```python
# NAPALM getters for post-change verification
import napalm, json

driver = napalm.get_network_driver("eos")
device = driver(hostname="spine1", username="admin", password="secret")
device.open()

bgp = device.get_bgp_neighbors()
facts = device.get_facts()
interfaces = device.get_interfaces()

# Assert all expected BGP peers are established
for peer, data in bgp["global"]["peers"].items():
    assert data["is_up"], f"BGP peer {peer} is DOWN"
    assert data["is_enabled"], f"BGP peer {peer} is disabled"

print("[PASS] All BGP peers established")
device.close()
```

---

## Rollback Procedure

- [ ] Rollback commands prepared and tested in lab before the change window
- [ ] Rollback can be executed within the maintenance window time remaining
- [ ] Rollback triggered if **any** post-change verification step fails
- [ ] After rollback: re-run full post-change verification to confirm clean state
- [ ] Incident raised if rollback was required; root-cause analysis scheduled

```python
# NAPALM rollback (uses device's candidate config / rollback buffer)
device.rollback()

# For Junos devices with rollback history:
# net_connect.send_command("rollback 1")
# net_connect.commit()
```

---

## Common Anti-Patterns

| Anti-Pattern | Risk | Better Approach |
|---|---|---|
| Skipping lab testing | Untested change breaks production | Always test in Containerlab or equivalent first |
| Hand-editing production configs | Config drift, no audit trail | Generate configs from source of truth; apply via automation |
| Applying to all devices simultaneously | Single mistake takes down entire fabric | Roll out device-by-device with verification between each |
| No pre-change snapshots | No baseline for comparison post-change | Capture `show` outputs and parse with pyATS before touching anything |
| Treating "show | compare" as verification | Syntax correct ≠ functionally correct | Run Batfish reachability and end-to-end traffic tests |
| Rollback plan of "reverse the change" | Reversal may not restore state cleanly | Write and lab-test explicit rollback commands |
| Committing without a timer | No auto-recovery if you lose access | Use commit confirmed / `revert_in` wherever the platform supports it |
| Closing maintenance window before verifying | Leaves broken state undetected | Keep window open until all verification steps pass |
