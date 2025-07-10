
# Chapter6  Remote Execution & File Transfer

_If you cant reach the node, you cant fix the node._  
This chapter covers robust patterns for SSH, SCP, Rsync, and `sshpass`
bootstrappingplus multiplexing for speed and Expect fallbacks for
legacy Telnet CLI.

---

## 6.1  SSH Fundamentals

### 6.1.1  Option Deep Table

| Flag | Purpose | RealWorld Use |
|------|---------|----------------|
| `-oBatchMode=yes` | Disable any password / keyboardinteractive prompt | CI pipelines; prevents hang |
| `-oConnectTimeout=` | Fail fast on bad IP / ACL | Batch loops wont stall |
| `-oStrictHostKeyChecking=accept-new` | Autolearn keys (lab nets) | Firsttime flash of 500CPE devices |
| `-i key.pem` | Specify identity file | Multiple key sets per POP |
| `-tt` | Force pseudoTTY | Required for `sudo` on some NOS |
| `-J jumphost` | ProxyJump | Access private management VLAN |

**Tip:** Build reusable snippet:

```bash
SSH_OPTS=(-oBatchMode=yes -oConnectTimeout=5 -oStrictHostKeyChecking=accept-new -i "$KEY")
ssh "${SSH_OPTS[@]}" user@$ip "cmd"
```

### 6.1.2  ControlMaster Multiplexing

`~/.ssh/config`

```
Host edge-*
  ControlMaster auto
  ControlPersist 10m
  ControlPath /tmp/ssh-%r@%h:%p
```

*Op impact:* Opening one control socket per host cuts 50node config audit
from **180s  25s** because SCP/SSH reuse the same TCP/crypto session.

---

## 6.2  SCP vs Rsync

| Capability | `scp` | `rsync` |
|------------|-------|---------|
| Delta copy |  |  |
| Resume |  |  `--partial` |
| Bandwidth limit |  |  `--bwlimit` |
| Preserve perms/times |  `-p` |  `-a` |
| Compress |  `-C` |  `-z` |

### Example: Throttle Rsync to 2Mbps on LTE backhaul

```bash
rsync -avz --bwlimit=250 --progress -e "ssh ${SSH_OPTS[*]}" fw.bin admin@$ip:/tmp/
```

---

## 6.3  GNU parallel + SSH FanOut

```bash
export SSH_OPTS KEY
parallel --eta -j100 "
  ssh ${SSH_OPTS[@]} admin@{} 'uptime'"   ::: $(<edges.txt)
```

*ETA* shows estimated completioncritical during a 3am maintenance.

---

## 6.4  `sshpass` Bootstrap (OneTime)

When devices ship with **passwordonly** auth.

```bash
sshpass -p "$PW" ssh -oStrictHostKeyChecking=no admin@$ip 'echo ok'
```

Immediately copy key and disable password:

```bash
sshpass -p "$PW" ssh $ip "mkdir -p ~/.ssh && echo '$PUB' >>~/.ssh/authorized_keys && passwd -l admin"
```

> **Never** store passwords in scriptsread from vault or prompt.

---

## 6.5  Expect Fallback for Telnet CLI

`fw_telnet.exp`:

```expect
#!/usr/bin/expect -f
set ip [lindex $argv 0]
spawn telnet $ip
expect "Username:" { send "admin\r" }
expect "Password:" { send "$env(PASS)\r" }
expect ">" { send "copy tftp://10.0.0.10/fw.bin flash:\r" }
expect ">" { send "reload\r" }
expect eof
```

Called from Bash loop:

```bash
export PASS
parallel -j40 expect fw_telnet.exp ::: $(<old_switches.txt)
```

---

## 6.6  Rsync Inventory Trick

Push *perdevice* config directories:

```bash
for ip in $(<edges.txt); do
  rsync -az config/$ip/ admin@$ip:/flash/config/ &
done; wait
```

Combine with checksum verify:

```bash
ssh $ip 'sha256sum /flash/config/startup.cfg'   >shas/$ip.txt
```

---

## 6.7  SFTP Batch Mode for Vendor NOS

Create `cmds.txt`:

```
put fw.bin /tmp/fw.bin
ls -l /tmp
```

```bash
sftp -b cmds.txt -i $KEY admin@$ip
```

---

## 6.8  Error Handling Patterns

### Timeout + Retry

```bash
retry 3 timeout 15s ssh "${SSH_OPTS[@]}" $ip 'fwupdate status'
```

### Capture stderr

```bash
if ! err=$(scp ... 2>&1); then
   log "SCP failed: $err"
fi
```

---

## 6.9  Exercises

1. Write a script that opens a ControlMaster connection to each host first,
   then parallelexecutes three commands over the mux.  
2. Demonstrate how `--partial-dir` helps resume rsync if LTE link drops.  
3. Replace a nested `for` scp loop with a single rsync **pushdelete** to keep
   node configs in sync.

---

> Next: **Chapter7  CSV / TSV Workflows as Lightweight Databases**.
