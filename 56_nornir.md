# Chapter56  Nornir Framework for InventoryDriven Automation

When Bash + Netmiko loops reach thousands of nodes, error handling and
structured inventory become painful. **Nornir** is a Python automation
framework that gives you pluggable inventory backends, multithreaded
executors, and task abstractionswhile still letting you call Bash or legacy
Expect scripts when needed.

_Last updated: 2025-07-10_

---

## 56.1  Quick Install

```bash
pip install nornir nornir-netmiko nornir-utils
```

---

## 56.2  Minimal Inventory (YAML)

```
hosts:
  edge-01:
    hostname: 203.0.113.11
    platform: cisco_ios
    data:
      site: den
  edge-02:
    hostname: 203.0.113.12
    platform: cisco_ios
    data:
      site: lax
groups:
  defaults:
    username: admin
    connection_options:
      netmiko:
        extras:
          key_file: /opt/key
```

Project tree:

```
./inventory/{hosts.yaml,groups.yaml}
./tasks/
```

---

## 56.3  Nornir Script  Show Version

```python
from nornir import InitNornir
from nornir_netmiko import netmiko_send_command
from nornir_utils.plugins.functions import print_result

nr = InitNornir(config_file="config.yaml")

def show_ver(task):
    task.run(netmiko_send_command, command_string="show version")

result = nr.run(task=show_ver)
print_result(result)
```

Runs in threads (default=#CPU cores).

---

## 56.4  Filtering Hosts

```python
den = nr.filter(site="den")
den.run(task=show_ver)
```

---

## 56.5  Integrating with Bash Firmware Script

```python
from nornir.core.task import Task, Result
import subprocess, shlex

def bash_flash(task: Task):
    cmd = f"ssh admin@{task.host.hostname} 'bash -s' < apply.sh"
    proc = subprocess.run(shlex.split(cmd), capture_output=True, text=True)
    return Result(host=task.host, result=proc.stdout, failed=proc.returncode!=0)

nr.run(task=bash_flash)
```

Nornir captures perhost stdout/stderr and failure.

---

## 56.6  Parallelism & DryRun

```python
nr = InitNornir(config_file="config.yaml", dry_run=True)
```

Dryrun prints planned commands without execution (when plugin supports).

Set worker count:

```python
nr = InitNornir(config_file="config.yaml", num_workers=100)
```

---

## 56.7  Error Handling

```python
from nornir_utils.plugins.functions import print_result, print_title
bad = nr.run(task=show_ver)
failed_hosts = bad.failed_hosts
print("Failed:", failed_hosts)
```

Retry:

```python
for host in failed_hosts:
    nr.filter(name=host).run(task=show_ver)
```

---

## 56.8  Inventory from NetBox API

```python
from nornir.plugins.inventory.netbox import NetBoxInventory
nr = InitNornir(
    runner={"plugin": "threaded", "options": {"num_workers": 150}},
    inventory={
      "plugin": "NetBoxInventory",
      "options": {
        "nb_url":"https://netbox.lab",
        "nb_token": os.environ["NB_TOKEN"],
        "ssl_verify": False
      }
    })
```

No YAML to maintain.

---

## 56.9  Prometheus Export

Convert failures to textfile metric:

```python
failed = len(bad.failed_hosts)
with open("/var/lib/node_exporter/text/nornir_fail.prom","w") as f:
    f.write(f"nornir_failed_hosts {failed}\n")
```

---

## 56.10  Exercises

1. Write a Nornir task that checks firmware version and only flashes nodes running mismatched builds.  
2. Add a custom filter for `device_role="core"` then run a `ping 8.8.8.8` command via Netmiko plugin.  
3. Integrate Nornir into GitLab CI: inventory from NetBox, execute tasks, and fail the job if any host errors.

---
