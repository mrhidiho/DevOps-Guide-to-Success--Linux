
# Chapter82  WireGuard SmartQueue + Multipath TCP (MPTCP) over LTE Backup

When fiber drops, firmware uploads must failover to LTE without packet loss.
Combining **WireGuard** encrypted tunnels, **Smart Queue Management (CAKE)**,
and **Multipath TCP** enables seamless path migration: WiFi6E primary,
5GLTE backupall under one virtual IP.

_Last updated: 2025-07-10_

---

## 82.1  Stack Overview

```
+----------------------+
|      Application     |
+-----------+----------+
            |
        MPTCP Socket
            |
+-----+-----+-----+---+
|WGUDP|WGUDP|  ...  |  (subflows)
+--+--+--+--+--+--+---+
   |  |  |  |
 CAKE  LTE    WiFi
```

* WireGuard keeps packets encrypted & stateful across IP changes  
* MPTCP splits flows across WG UDP ports (subflows)  
* CAKE qdisc eliminates bufferbloat on LTE

---

## 82.2  Kernel & Packages

* Kernel6.8 with `CONFIG_MPTCP=y`  
* `wireguard-tools` 1.0.2023  
* `mptcpd` daemon

---

## 82.3  WireGuard DualPath Config

Server `/etc/wireguard/wg0.conf`:

```ini
[Interface]
PrivateKey = ...
ListenPort = 51820

[Peer]
PublicKey = <client>
AllowedIPs = 10.55.0.2/32
```

Client:

```ini
[Interface]
Address = 10.55.0.2/32
PrivateKey = ...

[Peer]
PublicKey = <server>
Endpoint = 198.51.100.2:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 15
```

Add second endpoint dynamically:

```bash
wg set wg0 peer <server> endpoint 198.51.100.2:51821
```

(`51821` tunneled via LTE CGNAT).

---

## 82.4  Enable MPTCP with Subflow Limits

```bash
sysctl -w net.mptcp.enabled=1
sysctl -w net.mptcp.pm_max_subflows=4
sysctl -w net.mptcp.pm_max_addr=4
```

Add addresses:

```bash
ip mptcp endpoint add 10.55.0.2 dev wg0 signal
ip mptcp endpoint add 10.55.0.2 dev wg0_4g subflow backup
```

`backup` flag keeps LTE idle until needed.

---

## 82.5  CAKE Smart Queue on LTE

```bash
tc qdisc add dev wwan0 root cake bandwidth 20Mbit diffserv4 ack-filter
```

WiFi path 200Mbit uses `cake bandwidth 220Mbit` on `wlan0`.

---

## 82.6  FailOver Demo

1. Start iperf:

```bash
iperf3 -c 10.55.0.1 -t 120
```

2. Kill WAN NIC:

```bash
ip link set enp3s0 down
```

Observe `iperf` TPS dip <100ms as subflow migrates to LTE.

`ss -ta | grep mptcp` shows `backup=1` flag cleared.

---

## 82.7  Prometheus Metrics

Exporter script:

```bash
active=$(ss -tam | grep -c 'mptcp')
fail=$(ss -tam | grep -c 'backup=1')
echo "mptcp_active_subflows $active" > /var/lib/node_exporter/text/mptcp.prom
echo "mptcp_backup_subflows $fail" >> /var/lib/node_exporter/text/mptcp.prom
```

Alert if active=0 (tunnel down).

---

## 82.8  RealWorld Result

Firmware transfer 1.5GB:

| Event | Drop (pkts) | Throughput |
|-------|------------:|-----------:|
| Fiber up | 0 | 210Mb/s |
| Fiber cut | **0** | 19Mb/s (LTE) |
| Fiber restore | 0 | 210Mb/s |

Seamless; TCP socket persisted.

---

## 82.9  Security Note

* Use separate UDP ports per path to aid firewall rules.  
* WireGuard rekeys every 120s; make sure both endpoints reachable.

---

## 82.10  Exercises

1. Benchmark CAKE vs FQ_CoDel on LTEmeasure RTT during 5Mb upload.  
2. Write `mptcpd` hook to demote LTE subflow when RSSI <-100dBm.  
3. Use `tc netem loss 5%` on WiFi path; plot MPTCP scheduler reaction.

---
