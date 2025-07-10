
# Chapter25  eBPF & `strace` for RealTime Debugging

When a new firmware suddenly spikes CPU or silently drops packets, classic
logging isnt enough. Modern kernels provide **eBPF** and oldfaithful
`strace` for zeroreboot tracing. This chapter shows how to attach
lowoverhead probes, gather metrics, and interpret outputentirely orchestrated
from Bash.

---

## 25.1  Installing bpftrace / bcctools

Ubuntu:

```bash
apt-get install -y bpftrace bpfcc-tools linux-headers-$(uname -r)
```

Quick capability check:

```bash
bpftrace -e 'BEGIN { printf("eBPF ready\n"); exit(); }'
```

---

## 25.2  Tracing New TCP Connections

```bash
sudo bpftrace -e 'tcpconnect { printf("%s %d -> %s %d\n",
 str(ipv4->saddr), tcp->sport, str(ipv4->daddr), tcp->dport); }'
```

Use in Bash wrapper:

```bash
timeout 60s bpftrace -e 'tcpconnect { @[comm] = count(); }' >tcp.bt
```

Produces perprocess connection countgreat for detecting runaway CLI loops.

---

## 25.3  Packet Drop Debug (`tracepoint:skb:kfree_skb`)

```bash
bpftrace -e 'tracepoint:skb:kfree_skb /args->reason==1/
  { @[comm] = count(); }'
```

`reason==1` is `SKB_DROP_REASON_TCP_CSUM`. Identify if firmware disables
offload incorrectly.

---

## 25.4  `execsnoop` for Unexpected Binaries

```bash
execsnoop -t -l 0 | grep -E 'wget|curl|nc'
```

Alerts when a process launches shellout downloads on routers.

---

## 25.5  Using `strace` with Filters

Attach to PID:

```bash
strace -f -e trace=network -p $PID -tt -o net.strace
```

Filters:

| Filter | What |
|--------|------|
| `-e trace=file` | Only file syscalls |
| `-e trace=%process` | fork/exec/clone |
| `-e status=failed` | Only failed calls (strace6+) |

---

## 25.6  Combining eBPF + Prometheus

`tcpbytes.bt`:

```bpftrace
kprobe:tcp_cleanup_rbuf
{
  @bytes += args->copied;
}
END
{
  printf("tcp_read_bytes %d\n", @bytes);
}
```

Run hourly cron and write output to `tcp_read_bytes.prom`.

---

## 25.7  Ring Buffer Overrun Protection

Use `-c`  `bpftrace -c "cmd" 'program'` to run limited time. Or set
`--buffer-size 8` (MB).

---

## 25.8  Exercises

1. Build a `bpftrace` script that counts TCP retransmits per interface and
   alerts if >100/min.  
2. Write a Bash wrapper that `strace`s `fwupdate` and aborts if any `open()`
   fails with `ENOENT`.  
3. Combine `execsnoop` output with Prometheus `process_start_total` metric
   via textfile exporter.

---

> Next: **Chapter26  Seccomp & AppArmor for Container Sandboxing**
