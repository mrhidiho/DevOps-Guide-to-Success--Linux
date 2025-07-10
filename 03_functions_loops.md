
# Chapter3  Functions, Libraries & Loop Patterns

_A 10line function reused 5,000 times beats a 50line copypaste script
every day, especially when those 5,000 runs are parallel across an ISP
backbone._

This chapter shows how to build **idempotent**, testable Bash functions and
robust loop constructsincluding advanced CSVdriven inventories and GNU
parallel fanout.

---

## 3.1  Anatomy of a ProductionGrade Function

```bash
flash_fw() {
  local ip=$1 fw=$2               # positional args -> locals
  log "Staging $fw  $ip"

  scp -q -i "$KEY" "$fw" admin@$ip:/tmp/fw.bin ||
     { log "scp failed $ip"; return 1; }

  ssh -i "$KEY" admin@$ip 'fwupdate apply /tmp/fw.bin && reboot' ||
     { log "fwupdate failed on $ip"; return 2; }
}
```

| Bestpractice | Why |
|---------------|-----|
| `local ip fw` | Prevents globalvar collisions in large libs |
| `return N` codes | Callers can tally success/fail buckets |
| Earlyreturn on error | Avoids cascading commands (still inside `-e`) |
| Quoted variables | Protect spaces in paths/hostnames |

### Idempotency Principle

The function reuploads **only** if `/tmp/fw.bin` differs from SHA256 of
`$fw`preventing needless flashes when rerunning playbooks.

```bash
remote_sha=$(ssh $ip 'sha256sum /tmp/fw.bin 2>/dev/null || true' | cut -d' ' -f1)
local_sha=$(sha256sum "$fw" | cut -d' ' -f1)
[[ $remote_sha == $local_sha ]] && { log "Already staged"; return 0; }
```

---

## 3.2  Building a Reusable Library

Directory structure:

```
lib/
  logging.sh
  retry.sh
  fw_ops.sh    (imports logging & retry)
```

`fw_ops.sh`:

```bash
#!/usr/bin/env bash
source "${BASH_SOURCE%/*}/logging.sh"
source "${BASH_SOURCE%/*}/retry.sh"

flash() { ... }
verify() { ... }
```

Scripts then **import** the lib:

```bash
#!/usr/bin/env bash
set -euo pipefail
source "$(dirname "$0")/lib/fw_ops.sh"

flash "$@"
```

Version control keeps functions audited and unittested in isolation.

---

## 3.3  Loop Patterns

### 3.3.1  `for  in` vs `while read`

| Pattern | Pros | Cons |
|---------|------|------|
| `for x in $(cat file)` | Simple | Breaks on spaces, forks `cat`, loses empty lines |
| `while IFS= read -r line; do  done <file` | Safe, preserves whitespace | Slightly more typing |
| `mapfile -t arr <file` + array loop | Fast (C code), keeps NUL | Bash4+ only |

### Example: Reboot Only Down Nodes

```bash
mapfile -t edges <edges.txt
for ip in "${edges[@]}"; do
  if ! ping -c1 -W1 "$ip" &>/dev/null; then
     ssh "$ip" reboot &
  fi
done; wait
```

---

### 3.3.2  CSV Inventory Loop  Advanced

`devices.csv`:

```
ip,role,site,fw_target,reboot_window
10.0.0.1,edge,den,FW2025,02:00-04:00
```

```bash
now=$(date +%H:%M)
while IFS=, read -r ip role site fw win; do
  [[ $ip == ip ]] && continue       # skip header
  [[ $now < ${win%-*} || $now > ${win#*-} ]] && continue

  flash_fw "$ip" "firmware/$fw.bin" &   # parallel background
done < devices.csv
wait
```

*Window logic* ensures maintenance only during approved slots per node.

---

## 3.4  GNU parallel FanOut

```bash
export KEY; export -f flash_fw
parallel --bar -j50 flash_fw ::: $(cut -d, -f1 devices.csv | tail -n+2) ::: FW2025
```

*Placeholders:* first `:::` feeds list to `$1`, second to `$2` (**matrix
expansion**).  The `--bar` gives a live progress bar with ETA.

### Why not xargs?

* xargs reorder output; `parallel` preserves onelineperhost logs  
* dynamic job throttling; can drop to 10 on packet loss conditions.

---

## 3.5  Job Control & Timeouts

```bash
for ip in "${edges[@]}"; do
  ( timeout 90s flash_fw "$ip" "$fw" && touch ok/$ip ) &
done
wait

fail=$(comm -23 <(printf '%s
' "${edges[@]}" | sort) <(ls ok | sort))
echo "FAILED: $fail"
```

Uses background subshell with `timeout` to prevent hung SSH sessions.

---

## 3.6  Profiling Loop Performance

```bash
SECONDS=0
parallel -j50 verify_fw ::: "${edges[@]}"
log "verify took ${SECONDS}s"
```

Add `set -x; PS4='+${SECONDS}s ' bash -x script.sh` for linetimed trace.

---

## 3.7  Testing Functions with Bats

`test/flash.bats`:

```bash
@test "flash returns 0 when scp and ssh succeed" {
  function scp() { return 0; }
  function ssh() { return 0; }
  run bash -c 'source lib/fw_ops.sh; flash_fw 1.1.1.1 fw.bin'
  [ "$status" -eq 0 ]
}
```

Mocks out external commands so unit tests run without network.

---

## 3.8  Exercises

1. Write `parallel_retry` that wraps GNU parallel but retries failed hosts up
   to N times.  
2. Convert `devices.csv` into persite associative array
   `targets[site]="ip1 ip2"`.  
3. Profile how long it takes to SHA256 a 50MB firmware on 500 nodes using
   `parallel -j100` vs `xargs -P100`.  

---

> Next: **Chapter4  Text Processing DeepDive**.
