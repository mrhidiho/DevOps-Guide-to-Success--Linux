
# Chapter12  Network Port & Service Exposure Audits

Unplanned open ports are a prime foothold for attackers and misconfigurations
survive firmware upgrades unless you verify **every release**. This chapter
builds a complete Bashbased auditing pipeline that:

1. **Discovers** open TCP/UDP ports with Nmap or Masscan.  
2. **Compares** findings to an _expected_ port list stored in CSV.  
3. **Triages & remediates** via `systemctl`, firewalld, or iptables.  
4. **Reports** violations to Slack / PagerDuty.  

Every section contains realworld Spectrumstyle examples.

---

## 12.1  InventoryDriven Scan Design

Why inventorydriven? Scanning a /16 blindly yields noise and wasted time.
Your `devices.csv` already lists _which_ nodes and _which_ ports should be
exposedtreat it as the **source of truth**.

devices.csv:

```
ip,role,site,expected_tcp,expected_udp
10.10.0.1,edge,den,"80,443",""
10.10.0.2,edge,den,"80,443",""
10.20.0.5,cmts,kc,"22,161","161"
```

* Commaseparated port lists stored as _strings_.  
* Separate TCP and UDPNmap uses different scan techniques.

---

## 12.2  Generating the Target List

```bash
mapfile -t targets < <(tail -n +2 devices.csv | cut -d, -f1)
printf "Scanning %d devices\n" "${#targets[@]}"
```

If CSV includes thousands of rows, split by POP:

```bash
awk -F, '$3=="den"{print $1}' devices.csv >den.txt
```

---

## 12.3  Fast Sweep with Nmap

### Recommended Command

```bash
nmap -T4 -sS -sU --top-ports 1000 \
     -oX scan_$(date +%F).xml \
     "${targets[@]}"
```

| Flag | Rationale |
|------|-----------|
| `-T4` | Aggressive timing; safe on local POP LAN |
| `-sS` | TCPSYN scan (stealth, fast) |
| `-sU` | UDP scan (takes longer) |
| `--top-ports 1000` | 99% of services live here; adjust for full sweep |
| `-oX` | XML for structured postprocessing |

**Runtime**: ~90s for 1k nodes on 1Gbps lab VLAN.

---

## 12.4  Parsing Nmap XML in Bash

### Utility Function

```bash
extract_ports() {
  # $1 = IP, $2 = xml file
  xmllint --xpath     "//host[address/@addr='$1']/ports/port[state/@state='open']/@portid"     "$2" 2>/dev/null | tr ' ' '\n' | sed 's/[^0-9]*//g' | sort -n | paste -sd, -
}
```

`xmllint` is part of `libxml2`; tiny footprint.

---

## 12.5  Comparing to Expected List

```bash
violations=()
while IFS=, read -r ip role site tcp udp; do
  [[ $ip == ip ]] && continue
  open_tcp=$(extract_ports "$ip" "$xml" )
  if [[ $open_tcp != "$tcp" ]]; then
     violations+=("$ip TCP $open_tcp (exp $tcp)")
  fi
done <devices.csv
```

Stores violations in array for bulk reporting.

---

## 12.6  Automated Remediation Workflow

```bash
for v in "${violations[@]}"; do
  set -- $v            # positional split
  ip=$1 proto=$2 ports=$3
  ssh "$ip" "systemctl list-sockets --state=LISTEN" >logs/$ip.sockets

  # Example: mask unexpected ftp.service
  if grep -q ':21' logs/$ip.sockets; then
     ssh "$ip" 'systemctl mask --now vsftpd'
  fi
done
```

### WorldReadable Port on BusyBox

If `vsftpd` isnt present but port 21 still open (BusyBox `inetd`), fallback:

```bash
ssh $ip "sed -i '/^ftp/d' /etc/inetd.conf && kill -HUP $(pidof inetd)"
```

---

## 12.7  HighSpeed Masscan for Edge VLANs

Masscan can scan 65k ports on /24 in seconds:

```bash
masscan -iL den.txt -p1-65535 --rate 100000 -oX masscan.xml
```

Convert to Nmap XML to reuse parser:

```bash
masscan --readscan masscan.xml -oX scan.xml
```

---

## 12.8  Continuous Scanning with Cron & Slack

`/usr/local/sbin/scan_ports.sh`:

```bash
#!/usr/bin/env bash
xml=/var/log/scan/scan_$(date +%F).xml
nmap -T4 -sS -p- -oX "$xml" $(<targets.txt)

if grep -q '<port ' "$xml"; then
  jq -n --arg text "Ports open $(hostname)" '{text:$text}' |      curl -XPOST -H 'Content-type: application/json' \
          -d @- $SLACK_WEBHOOK
fi
```

Cron entry:

```
30 3 * * * root /usr/local/sbin/scan_ports.sh
```

---

## 12.9  ss + firewallcmd OnDevice Check

Quick inline audit:

```bash
ssh $ip 'ss -lntp | awk "$4 ~ /:0\.0\.0\.0/"'
```

Block with firewalld:

```bash
ssh $ip 'firewall-cmd --permanent --add-rich-rule="rule family=ipv4 source address='$mgmt'/24 port port=21 protocol=tcp reject" && firewall-cmd --reload'
```

---

## 12.10  Exercises

1. Modify the comparison logic to accept **port ranges** (e.g., `60000-61000`
   for passive RTP).  
2. Build a Masscan wrapper that scans only UDP SNMP (161) across /16 and
   reports nodes where SNMP is down.  
3. Integrate scan results into Prometheus by writing
   `open_ports_total{port="21"} 3` metrics.  

---

> Next security chapter: **User / Group & Sudo Auditing**.
