# Network Automation Patterns

Quick reference for production-grade network automation patterns. Use alongside the `net-devops` agent.

## Table of Contents

- [Nornir Multi-Device Orchestration](#nornir-multi-device-orchestration)
- [NAPALM Config Management](#napalm-config-management)
- [Netmiko Patterns](#netmiko-patterns)
- [NETCONF / RESTCONF](#netconf--restconf)
- [gNMI Streaming Telemetry](#gnmi-streaming-telemetry)
- [Ansible Network Patterns](#ansible-network-patterns)
- [Secrets Management](#secrets-management)
- [Error Handling Patterns](#error-handling-patterns)
- [Anti-Patterns](#anti-patterns)

---

## Nornir Multi-Device Orchestration

Prefer Nornir for anything touching more than one device. It handles parallelism, inventory, and result aggregation natively.

```python
from nornir import InitNornir
from nornir.core.task import Task, Result
from nornir_netmiko.tasks import netmiko_send_command, netmiko_send_config
from nornir_utils.plugins.functions import print_result
from nornir.core.filter import F

nr = InitNornir(config_file="nornir.yml")

# Filter to a subset of devices
spines = nr.filter(F(role="spine") & F(site="dc1"))

# Run a task across all spines in parallel (default: 20 workers)
result = spines.run(
    task=netmiko_send_command,
    command_string="show bgp summary",
)

print_result(result)

# Check for failures
for host, multi_result in result.items():
    if multi_result.failed:
        print(f"[FAIL] {host}: {multi_result.exception}")
```

### Nornir inventory (hosts.yml)

```yaml
spine1:
  hostname: 192.0.2.1
  platform: eos
  groups:
    - spine
    - dc1

spine2:
  hostname: 192.0.2.2
  platform: eos
  groups:
    - spine
    - dc1
```

### Nornir config push with diff

```python
from nornir_napalm.plugins.tasks import napalm_configure

def push_config(task: Task) -> Result:
    r = task.run(
        task=napalm_configure,
        filename=f"configs/{task.host.name}.cfg",
        dry_run=False,       # set True for --check mode
        replace=False,       # merge, not replace
    )
    return Result(host=task.host, result=r.result)

result = nr.run(task=push_config)
print_result(result)
```

---

## NAPALM Config Management

Use NAPALM when you need a vendor-agnostic layer for config get/set and structured data.

```python
import napalm

def get_driver(platform: str, host: str, username: str, password: str):
    driver = napalm.get_network_driver(platform)  # eos, ios, junos, nxos, iosxr
    device = driver(hostname=host, username=username, password=password)
    device.open()
    return device

# Config diff before committing
device = get_driver("eos", "spine1", "admin", "secret")
device.load_merge_candidate(filename="changes/spine1.conf")

diff = device.compare_config()
if not diff:
    print("No changes to apply")
    device.discard_config()
else:
    print(diff)
    device.commit_config(revert_in=300)  # Auto-rollback in 5 min
    # After verifying:
    device.confirm_commit()

device.close()
```

### NAPALM getters (structured data)

```python
# Retrieve structured data — vendor-agnostic
bgp    = device.get_bgp_neighbors()          # BGP peer state
ifaces = device.get_interfaces()              # Interface up/down, speed
ipcfg  = device.get_interfaces_ip()          # IP addresses per interface
routes = device.get_route_to("10.0.0.0/8")   # Route lookup
facts  = device.get_facts()                   # Hostname, OS, uptime, serial
lldp   = device.get_lldp_neighbors()         # CDP/LLDP neighbor table
arp    = device.get_arp_table()               # ARP table
```

### NAPALM validation

```python
# Validate current state against a compliance spec
# compliance.yml defines expected values for getters
compliance_report = device.compliance_report(validation_file="compliance.yml")
assert compliance_report["complies"], compliance_report
```

---

## Netmiko Patterns

Use Netmiko for direct device access when NAPALM doesn't support the platform or operation.

```python
from netmiko import ConnectHandler, NetmikoTimeoutException, NetmikoAuthenticationException

device_params = {
    "device_type": "cisco_ios",
    "host": "192.0.2.10",
    "username": "admin",
    "password": "secret",
    "timeout": 30,
    "session_log": f"logs/{host}.log",  # Always log sessions
}

try:
    with ConnectHandler(**device_params) as net_connect:
        # Read operations
        output = net_connect.send_command(
            "show ip bgp summary",
            use_textfsm=True,   # Returns parsed list of dicts
        )

        # Config operations — always check for errors
        result = net_connect.send_config_set(
            ["router bgp 65000", " neighbor 10.0.0.2 shutdown"],
            error_pattern=r"% Invalid|% Error|% Ambiguous",
        )

        net_connect.save_config()

except NetmikoTimeoutException as e:
    print(f"[ERROR] Timeout connecting to {host}: {e}")
except NetmikoAuthenticationException as e:
    print(f"[ERROR] Auth failure on {host}: {e}")
```

### TextFSM parsing

```python
# TextFSM returns structured data when use_textfsm=True
bgp_peers = net_connect.send_command("show ip bgp summary", use_textfsm=True)
# bgp_peers is a list of dicts: [{neighbor, version, as, state, ...}, ...]

down_peers = [p for p in bgp_peers if p["state"] not in ("Established", "Up")]
if down_peers:
    print(f"[WARN] {len(down_peers)} BGP peer(s) not established: {down_peers}")
```

---

## NETCONF / RESTCONF

Prefer NETCONF/RESTCONF over CLI scraping when the platform and YANG model support it.

### NETCONF with ncclient

```python
from ncclient import manager
import xmltodict, json

with manager.connect(
    host="192.0.2.1",
    port=830,
    username="admin",
    password="secret",
    hostkey_verify=False,
    device_params={"name": "iosxe"},
) as m:
    # Get BGP config via YANG filter
    filter_xml = """
    <filter>
      <bgp xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-bgp">
        <as-no-range/>
      </bgp>
    </filter>
    """
    result = m.get_config(source="running", filter=filter_xml)
    data = xmltodict.parse(result.xml)
    print(json.dumps(data, indent=2))
```

### RESTCONF with requests

```python
import requests, json

BASE = "https://192.0.2.1/restconf/data"
HEADERS = {
    "Content-Type": "application/yang-data+json",
    "Accept": "application/yang-data+json",
}
AUTH = ("admin", "secret")

# GET: read interface config
resp = requests.get(
    f"{BASE}/ietf-interfaces:interfaces/interface=GigabitEthernet1",
    headers=HEADERS, auth=AUTH, verify=False,
)
resp.raise_for_status()
print(json.dumps(resp.json(), indent=2))

# PATCH: update interface description
payload = {
    "ietf-interfaces:interface": {
        "name": "GigabitEthernet1",
        "description": "Uplink to spine1",
    }
}
resp = requests.patch(
    f"{BASE}/ietf-interfaces:interfaces/interface=GigabitEthernet1",
    headers=HEADERS, auth=AUTH, verify=False,
    json=payload,
)
resp.raise_for_status()
```

---

## gNMI Streaming Telemetry

```python
# gNMIc CLI — subscribe to interface counters
# gNMIc: https://gnmic.openconfig.net/
```

```bash
# Subscribe to interface stats (streaming, 10s interval)
gnmic -a spine1:6030 -u admin -p secret --insecure \
  subscribe \
  --path "interfaces/interface[name=*]/state/counters" \
  --mode stream \
  --stream-mode sample \
  --sample-interval 10s \
  --encoding json_ietf

# One-shot GET via gNMI
gnmic -a spine1:6030 -u admin -p secret --insecure \
  get --path "network-instances/network-instance[name=default]/protocols/protocol[name=BGP]/bgp/neighbors"
```

### pygnmi (Python gNMI client)

```python
from pygnmi.client import gNMIclient

with gNMIclient(
    target=("spine1", 6030),
    username="admin",
    password="secret",
    insecure=True,
) as gc:
    # Get BGP neighbor state
    result = gc.get(
        path=["network-instances/network-instance[name=default]/protocols/protocol[name=BGP]/bgp/neighbors"],
        datatype="state",
        encoding="json_ietf",
    )
    print(result)

    # Subscribe to interface counters
    for update in gc.subscribe(
        subscribe={
            "subscription": [
                {
                    "path": "interfaces/interface[name=*]/state/counters",
                    "mode": "sample",
                    "sample_interval": 10_000_000_000,  # 10s in nanoseconds
                }
            ],
            "mode": "stream",
            "encoding": "json_ietf",
        }
    ):
        print(update)
```

---

## Ansible Network Patterns

```yaml
# group_vars/all.yml — connection settings
ansible_connection: network_cli
ansible_network_os: "{{ platform }}"   # ios, eos, nxos, junos, iosxr
ansible_user: "{{ vault_ansible_user }}"
ansible_password: "{{ vault_ansible_password }}"
ansible_become: true
ansible_become_method: enable
```

```yaml
# Playbook: deploy BGP config from template
- name: Deploy BGP configuration
  hosts: spines
  gather_facts: false
  tasks:
    - name: Generate config from template
      ansible.builtin.template:
        src: templates/bgp.j2
        dest: "/tmp/{{ inventory_hostname }}_bgp.cfg"
      delegate_to: localhost

    - name: Diff config before applying
      cisco.ios.ios_config:
        src: "/tmp/{{ inventory_hostname }}_bgp.cfg"
      check_mode: true        # dry-run
      register: config_diff

    - name: Show diff
      ansible.builtin.debug:
        var: config_diff

    - name: Apply config
      cisco.ios.ios_config:
        src: "/tmp/{{ inventory_hostname }}_bgp.cfg"
        save_when: changed
      when: not ansible_check_mode
```

### Jinja2 config template pattern

```jinja2
{# templates/bgp.j2 #}
router bgp {{ bgp.asn }}
  bgp router-id {{ bgp.router_id }}
  bgp log-neighbor-changes
{% for peer in bgp.peers %}
  neighbor {{ peer.ip }} remote-as {{ peer.asn }}
  neighbor {{ peer.ip }} description {{ peer.description }}
  neighbor {{ peer.ip }} update-source {{ peer.update_source | default('Loopback0') }}
{% if peer.route_map_in is defined %}
  neighbor {{ peer.ip }} route-map {{ peer.route_map_in }} in
{% endif %}
{% if peer.route_map_out is defined %}
  neighbor {{ peer.ip }} route-map {{ peer.route_map_out }} out
{% endif %}
{% endfor %}
```

---

## Secrets Management

**Never hardcode credentials. Always use one of these patterns.**

```python
# Environment variables (minimum viable)
import os
USERNAME = os.environ["NET_USERNAME"]
PASSWORD = os.environ["NET_PASSWORD"]

# HashiCorp Vault (preferred for production)
import hvac
client = hvac.Client(url="https://vault.internal", token=os.environ["VAULT_TOKEN"])
secret = client.secrets.kv.v2.read_secret_version(path="network/spine1")
creds = secret["data"]["data"]  # {"username": ..., "password": ...}

# Ansible Vault for playbooks
# ansible-vault encrypt_string 'secretpassword' --name 'ansible_password'
# Store vault-encrypted vars in group_vars/all/vault.yml
```

---

## Error Handling Patterns

```python
from nornir.core.exceptions import NornirExecutionError
from netmiko import NetmikoTimeoutException, NetmikoAuthenticationException
import logging

logger = logging.getLogger(__name__)

def safe_run(nr, task, **kwargs):
    """Run a Nornir task and surface failures clearly."""
    result = nr.run(task=task, **kwargs)

    failed_hosts = [h for h, r in result.items() if r.failed]
    if failed_hosts:
        logger.error("Task failed on: %s", failed_hosts)
        for host in failed_hosts:
            logger.error("[%s] %s", host, result[host].exception)

    # Raise if all hosts failed — likely a systemic issue
    if len(failed_hosts) == len(result):
        raise NornirExecutionError(result)

    return result
```

---

## Anti-Patterns

| Anti-Pattern | Risk | Better Approach |
|---|---|---|
| Screen-scraping when YANG/API available | Brittle to output changes, locale-sensitive | Use NETCONF/RESTCONF/gNMI with structured YANG data |
| Hardcoded credentials | Credential leak, no rotation | Environment variables, Vault, or Ansible Vault |
| No session logging | No audit trail when things go wrong | Always set `session_log` in Netmiko |
| Non-idempotent config pushes | Re-runs accumulate duplicate stanzas | Test idempotency in lab; use NAPALM `replace` for full config |
| Applying to all devices simultaneously | One error takes down the whole fabric | Sequential or small batches with verification between |
| No timeout set | Script hangs indefinitely | Always set `timeout` and `auth_timeout` on connections |
| Catching bare `Exception` | Hides root cause | Catch specific exceptions (Timeout, Auth, NornirExecutionError) |
| Config in automation repo without encryption | Secrets in plaintext in git | Use Ansible Vault or reference Vault for all credential data |
