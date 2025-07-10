
# Chapter41  Bash Performance & Debugging DeepDive

Large fleet loops can hit minutelong runtimes. This chapter shows profiling
and optimization tools to cut execution time and memory.

---

## 41.1  `set -x` with Timestamps

```bash
PS4='+${SECONDS}s ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
set -x
```

Outputs runtime per command.

---

## 41.2  `time` vs builtin `time`

Measure pipeline:

```bash
/usr/bin/time -f '%e real' bash -c 'parallel -j100 ping -c1 {} ::: $(<ips)'
```

---

## 41.3  `coproc` for Async I/O

```bash
coproc dig @8.8.8.8 example.com
read -r line <&"${COPROC[0]}"
```

Avoids subshell fork per DNS query.

---

## 41.4  Subshell vs Grouping

Subshell adds fork:

```bash
( var=1 )
{ var=1; }
```

Prefer grouping unless isolation required.

---

## 41.5  Avoid UUOC (Useless Use of Cat)

```bash
while read -r; do ... done <file   # good
cat file | while read; do ...      # spawns cat
```

---

## 41.6  Arrays vs ForSplits

`mapfile -t arr <file` loads in C10 faster than `for i in $(...)`. Memory
tradeoff acceptable for <1M lines.

---

## 41.7  perf trace & FlameGraphs

```bash
perf trace -e sched:sched_process_fork -p $$ -o perf.txt
```

Convert to FlameGraph with `flamegraph.pl` for CPU hotspots.

---

## 41.8  Exercises

1. Rewrite loop using `xargs -P` and measure speedup with `/usr/bin/time`.  
2. Use `coproc` to maintain persistent SSH control socket.  
3. Generate FlameGraph for `grep -R` vs `ripgrep` and compare syscalls.

---
