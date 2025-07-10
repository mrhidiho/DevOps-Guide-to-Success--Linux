
# Chapter30  Python+Bash Hybrid: Netmiko & Paramiko Bridges

Pure Bash scales far, but Python libraries like **Paramiko**, **Netmiko**, and
**pexpect** offer richer SSH handling.  This chapter shows how to embed
small Python snippets inside Bash to extend functionality without rewriting
entire pipelines.

---

## 30.1  Pattern: HereDoc Python Block

```bash
hosts=( $(<edges.txt) )
python - <<'PY'
import sys, json, paramiko, concurrent.futures
hosts = json.loads(sys.argv[1])
def uptime(ip):
    ssh = paramiko.SSHClient(); ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh.connect(ip, username="admin", key_filename="/opt/key")
    _, stdout, _ = ssh.exec_command("uptime -p")
    print(f"{ip},{stdout.read().decode().strip()}")
with concurrent.futures.ThreadPoolExecutor(max_workers=50) as ex:
    ex.map(uptime, hosts)
PY "$(printf '%s' "${hosts[@]}" | jq -R . | jq -s .)"
```

Bash passes JSON array; Python prints CSV, Bash captures.

---

## 30.2  Netmiko Bulk Config Push

`push_cfg.py`:

```python
from netmiko import ConnectHandler
import csv, sys, concurrent.futures
cfg = open('golden.cfg').read().splitlines()
def worker(row):
    ip, role = row['ip'], row['role']
    dev = dict(device_type='cisco_ios', host=ip,
               username='admin', use_keys=True, key_file='/opt/key')
    with ConnectHandler(**dev) as c:
        c.send_config_set(cfg)
        out = c.save_config()
    print(f"{ip},OK")
with open('devices.csv') as f:
    rows = list(csv.DictReader(f))
with concurrent.futures.ThreadPoolExecutor(max_workers=30) as ex:
    ex.map(worker, rows)
```

Invoke from Bash: `python push_cfg.py >results.csv`

---

## 30.3  Returning Exit Status

`python - <<'PY' ... PY`; at end:

```python
import sys; sys.exit(1 if failures else 0)
```

Bash `echo $?` picks up status.

---

## 30.4  pexpect Wrapper for Weird CLI

Bash calls small Python:

```bash
python - <<'PY'
import pexpect, sys, os
ip = sys.argv[1]
child = pexpect.spawn(f'telnet {ip}')
child.expect('Username:'); child.sendline('admin')
child.expect('Password:'); child.sendline(os.environ['PASS'])
child.expect('>'); child.sendline('show ver')
child.expect('>'); print(child.before.decode())
PY "$ip"
```

---

## 30.5  Virtualenv Bundling in CI

```bash
python -m venv venv
source venv/bin/activate
pip install netmiko paramiko pexpect
tar czf venv.tgz venv
```

CI caches `venv.tgz` to speed future jobs.

---

## 30.6  Exercises

1. Extend `push_cfg.py` to consume firmware target from `devices.csv` and
   flash only if version mismatched (use Netmiko `send_command`).  
2. Write a Bash wrapper that splits `devices.csv` by role and runs separate
   Python processes, collating CSV outputs.  
3. Use Paramikos `SFTPClient` inside the Python block to verify checksum
   of `/tmp/fw.bin` before Bash proceeds to apply stage.

---
