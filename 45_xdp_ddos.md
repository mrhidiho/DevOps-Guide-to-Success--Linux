
# Chapter45  XDP/eXpressDataPath for DDoS Mitigation

XDP attaches BPF programs at the earliest point in the Linux RX pathbefore
iptables, conntrack, or even sk_buff allocationallowing 100Gb mitigation on
commodity NICs.

_Last updated: 2025-07-10_

---

## 45.1  Environment Setup

```bash
apt-get install -y clang llvm libbpf-dev bpftool iproute2
modprobe ixgbe   # ensure driver supports XDP_NATIVE
```

Check capability:

```bash
ethtool -i eth0 | grep native
```

---

## 45.2  Sample Drop Program

`drop_ip_kern.c`:

```c
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_drop(struct xdp_md *ctx){
    void *data_end = (void *)(long)ctx->data_end;
    void *data     = (void *)(long)ctx->data;
    struct ethhdr *eth = data;
    if ((void*)eth + sizeof(*eth) > data_end) return XDP_PASS;

    if (eth->h_proto == __constant_htons(ETH_P_IP)) {
        struct iphdr *ip = data + sizeof(*eth);
        if ((void*)ip + sizeof(*ip) > data_end) return XDP_PASS;
        if (ip->daddr == __constant_htonl(0xC0000201)) /* 192.0.2.1 */
            return XDP_DROP;
    }
    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

Compile & load:

```bash
clang -O2 -g -target bpf -c drop_ip_kern.c -o drop_ip_kern.o
ip link set dev eth0 xdpgeneric obj drop_ip_kern.o sec xdp
```

---

## 45.3  Monitoring Drop Counters

Attach map in code (array map tracking drops). Then:

```bash
bpftool prog list
bpftool map dump id <MAPID>
```

Export to Prometheus:

```bash
echo "xdp_drop_total $(bpftool map dump id $MAPID | wc -l)"   > /var/lib/node_exporter/text/xdp_drop.prom
```

---

## 45.4  Dynamic IP Updates via bpftool

If program uses LPM trie map:

```bash
bpftool map update id $MAPID key 32 0xC0000202 value 0x1
```

Bash daemon ingests RTBH feed and updates map entries on the fly.

---

## 45.5  Offload to NIC Driver

```bash
ip link set dev eth0 xdpdrv obj drop_ip_kern.o sec xdp
```

Kernel dmesg:

```
loaded xdpdrv program on iface eth0(2), id 56
```

Expect ~20Mpps XDP vs ~3Mpps iptables on same hardware.

---

## 45.6  Exercises

1. Modify program to mirror (XDP_TX) first 1:100 dropped packets to `eth1`
   for PCAP capture.  
2. Build a systemd service that reloads the XDP object when its SHA256 hash
   changes (automatic redeploy).  
3. Compare CPU cycles per million packets between iptables DROP and XDP_DROP
   using `perf stat -e cycles` on an `iperf3 --udp` flood.

---
