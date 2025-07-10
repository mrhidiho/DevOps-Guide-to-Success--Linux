
# Chapter55  Fabric & Invoke for Parallel SSH Orchestration

`Fabric` (v2/3) and its core library **Invoke** bring Pythonic task runners,
rich SSH abstractions, and threaded fanoutwithout rewriting all your Bash.
Think of them as a glue layer: Bash still stages firmware, Fabric handles
parallelism, result objects, and retry logic.

_Last updated: 2025-07-10_

---

## 55.1  Quick Install

```bash
pip install fabric invoke
```

Requires Python 3.8 on your jumpbox.

---

## 55.2  Basic Fabric Task File (`fabfile.py`)

```python
from fabric import Connection, task
from invoke import Responder

@task
def uptime(c, hostfile="edges.txt"):
    with open(hostfile) as f:
        hosts = [h.strip() for h in f]
    for h in hosts:
        c = Connection(host=h, user="admin", connect_kwargs={"key_filename":"/opt/key"})
        c.run("uptime -p")
```

Run:

```bash
fab uptime --hostfile devices.txt
```

---

## 55.3  Parallel Execution with ThreadPool

```python
from concurrent.futures import ThreadPoolExecutor
@task
def parallel_fw(c, img="fw.bin", hostfile="edges.txt"):
    hosts = [h.strip() for h in open(hostfile)]
    def worker(h):
        conn = Connection(h, user="admin", connect_kwargs={"key_filename":"/opt/key"})
        conn.put(img, remote="/tmp/fw.bin")
        res = conn.run("sha256sum /tmp/fw.bin", hide=True)
        return h, res.ok
    with ThreadPoolExecutor(max_workers=50) as exe:
        results = list(exe.map(worker, hosts))
    print(results)
```

Handles 50concurrent SSH sessions, tracks failures.

---

## 55.4  Bridging to Bash Scripts

```python
@task
def bash_flash(c, host):
    conn = Connection(host, user="admin", connect_kwargs={"key_filename":"/opt/key"})
    conn.put("apply.sh", remote="/tmp/")
    conn.run("bash /tmp/apply.sh", pty=True)
```

Bash remains authoritative; Fabric just ships & executes.

---

## 55.5  Invoke Standalone Task Runner (No SSH)

`tasks.py`:

```python
from invoke import task, Collection

@task
def lint(ctx):
    ctx.run("shellcheck -x scripts/*.sh")

@task
def build(ctx):
    ctx.run("docker build -t fwtool .")

ns = Collection(lint, build)
```

CLI:

```bash
inv lint build
```

Great replacement for Makefile + Bash combos.

---

## 55.6  Using Responders for Interactive Prompts

```python
sudo_pass = Responder(pattern=r"\[sudo\] password:", response="netops\n")
conn.run("sudo systemctl restart snmpd", pty=True, watchers=[sudo_pass])
```

Avoids brittle `expect` for simple password/sudo cases.

---

## 55.7  Error Handling & Retries

```python
from tenacity import retry, stop_after_attempt
@retry(stop=stop_after_attempt(3))
def safe_run(conn, cmd):
    return conn.run(cmd, hide=True)
```

Combine Fabric connection with `tenacity` for robust loops.

---

## 55.8  Exercises

1. Write a Fabric task that distributes `flowspec.yaml` to every route reflector and reloads ExaBGPlog failures to a CSV.  
2. Refactor your Chapter24 kernel tuning loop into Fabric tasks, measuring total runtime vs Bash GNUparallel.  
3. Build an Invoke pipeline (`inv test deploy`) that lints Bash, runs pytest for Expect scripts, then calls Fabric deploy.

---
