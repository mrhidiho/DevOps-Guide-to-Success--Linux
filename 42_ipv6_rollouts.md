
# Chapter42  IPv6Only Rollouts & DualStack Caveats

ISPs increasingly run IPv6only access networks with NAT64/DNS64 for legacy
traffic. Bash scripts must handle linklocal pings, IPv6 formatting, and
dualstack quirks.

---

## 42.1  Ping & Curl IPv6

```bash
ping6 -c3 2001:db8::1
curl -g -6 http://[2001:db8::1]/health
```

Avoid `curl -6 http://2001:db8::1` (needs brackets).

---

## 42.2  LinkLocal Addresses

Include interface zone:

```bash
ssh root@fe80::1%eth0
ping6 fe80::1%eth0
```

---

## 42.3  iproute2 Differences

```bash
ip -6 route
ip -6 addr add 2001:db8::2/64 dev eth0
```

VRRPv2 uses Address Family IPv4 onlyswitch to VRRPv3.

---

## 42.4  NAT64 Testing

```bash
ping64(){
  ping -c3 $(dig AAAA $1 +short | head -1)
}
ping64 www.google.com
```

---

## 42.5  DualStack Pitfalls

`ping` picks IPv6 if `/etc/gai.conf` default. Force IPv4 with `-4` during
legacy checks.

---

## 42.6  Exercises

1. Bash function `best_ping host` that picks faster v4/v6 path.  
2. Script to detect SLAAC failure by checking `ip -6 addr tentative`.  
3. Use `tracepath6` to validate MTU path discovery after firmware update.

---
