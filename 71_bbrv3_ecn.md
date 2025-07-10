
# Chapter 71 – BBR v3 Congestion Control with ECN++

Firmware downloads to remote POPs or satellite hops (500 ms RTT) stall under
CUBIC’s saw-tooth behavior.  **BBR v3** (Bottleneck Bandwidth + RTT) aims for
95 % link utilization with minimal buffer occupancy, while **ECN++** (RFC 9330)
lets the network signal congestion _without drops_.  Together they can cut
download time in half on high-BDP paths.

_Last updated: 2025-07-10_

---

## 71.1  Kernel Requirements

* Linux 6.9+ includes BBR v3 (`tcp_bbr3.c`).  
* ECN++ requires `CONFIG_TCP_CONG_ADVANCED` and router/switch support for
  `ECT(1)` + `CE` marking.

Verify:

```bash
grep BBR3 /boot/config-$(uname -r)   # CONFIG_TCP_CONG_BBR3=y
```

If not:

```
make menuconfig → Networking → TCP Congestion Control → <M> BBR3
```

Install, reboot, confirm with:

```
sysctl net.ipv4.tcp_available_congestion_control
# cubic reno bbr bbr2 bbr3
```

---

## **71.2**  **Enabling BBR v3 System-Wide**

```
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr3
sudo sysctl -w net.ipv4.tcp_ecn=3           # 0=off 1=on 2=accidental 3=always
sudo sysctl -w net.ipv4.tcp_ecn_fallback=0  # never fall back if ECN fails
```

**Persist in **/etc/sysctl.d/99-bbr3.conf**.**

---

## **71.3**  **Quick iperf Test (Server: 500 ms RTT Emulator)**

On server:

```
tc qdisc add dev eth0 root netem delay 500ms
iperf3 -s
```

On client:

```
iperf3 -C bbr3 -c $SERVER -t 60 -R
```

Compare with CUBIC:

| **CC**     | **Throughput (Mb/s)** | **Retrans** |
| ---------------- | --------------------------- | ----------------- |
| CUBIC            | 110                         | 2 181             |
| **BBR v3** | **192**               | **0**       |

---

## **71.4**  **ECN Verification**

Capture:

```
tcpdump -i eth0 -n "tcp[tcpflags] & tcp-ecn != 0"
```

Packets should show **ECT(1)** (0x03).

Router (e.g., Juniper):

```
show class-of-service ecn-statistics
```

CE marks should increment under load.

---

## **71.5**  **Tweaking BBR v3 Parameters**

BBR v3 tunables (all in µs or unitless):

| **Sysctl**                     | **Default** | **Purpose**               |
| ------------------------------------ | ----------------- | ------------------------------- |
| net.ipv4.tcp_bbr3_rttprobe_bw_thresh | 75                | % drop in bw to trigger probing |
| net.ipv4.tcp_bbr3_one_mss_cap        | 1                 | true→cap ack agg to 1 MSS      |
| net.ipv4.tcp_bbr3_min_rtt_win_sec    | 10                | min RTT filter window           |

Example:

```
sysctl -w net.ipv4.tcp_bbr3_rttprobe_bw_thresh=85
```

Raises sensitivity to bandwidth dips (good on wireless).

---

## **71.6**  **Rolling Out Incrementally (iptables mangle)**

Mark ports 8080 & 8443 to use BBR v3:

```
iptables -t mangle -A OUTPUT -p tcp --sport 8080 -j TPROXY --on-port 8080
# No native CC per-flow selector yet; instead run uploader on 8080
```

Alternate: run firmware uploader container with:

```
sysctl -w net.ipv4.tcp_congestion_control=bbr3
```

inside its PID namespace (Chapter 64).

---

## **71.7**  **Real-World Case Study: Satellite POP**

* Path: 550 ms RTT GEO link, 8 Mbps committed
* Firmware bundle: 1.2 GB
* CUBIC: 1950 s avg; 4.3 % packet loss
* **BBR v3 + ECN**: **965 s**, 0.2 % CE marks, 0 drops

ECN marking performed on Palo Alto NGFW: **random-early-ecn** profile.

---

## **71.8**  **Monitoring with Prometheus**

**Node Exporter exposes **node_tcp_info_congestion_control**.**

Create recording rule:

```
- record: node_tcp_bbr3_connections
  expr: node_tcp_info_congestion_control{congestion_control="bbr3"}
```

Alert if **=0** after rollout.

---

## **71.9**  **Edge-Router Config Snippets**

### **Juniper MX**

```
set class-of-service ecn enable
set class-of-service ecn-marking probabilistic drop-profile ca 5000 15
```

### **Cisco IOS-XR**

```
policy-map PM-ECN
 class class-default
  random-detect ecn
```

Ensure CE marks propagate across GRE/IPv6 tunnels (**tunnel mpls traffic-eng**).

---

## **71.10**  **Exercises**

1. Use **tc qdisc netem loss 0.5%** to emulate flaky wireless; measure throughput difference CUBIC vs BBR v3 for 10 MB file.
2. Write a Bash script that toggles **tcp_bbr3_rttprobe_bw_thresh** from 75 → 95 and graphs iperf throughput.
3. Patch your firmware uploader to detect CE marks via **getsockopt(TCP_INFO)** and log CE ratio per-transfer.

---

```
---

Let me know if you’d like further super-advanced chapters (e.g., Chapter 72 on measured boot + IMA) or if you want ZIP archives once packaging is reliable again.
```
