
# Chapter35  NETCONF/YANG & RESTCONF Automation

Modern routers expose modeldriven APIs.  With NETCONF/YANG and RESTCONF you
get **structured configs**, diff support, and transactional commitsideal for
largescale, idempotent provisioning.

---

## 35.1  Quick NETCONF Primer

* Transport: SSH (port830) with `subsystem netconf`.  
* Data: XML (`rpc`, `edit-config`, `get-config`).  
* Schema: YANG models (IETF or vendor).

---

## 35.2  Bash + `ncclient` HereDoc

```bash
python - <<'PY'
from ncclient import manager
import sys, json
dev = {"host": sys.argv[1], "username":"netops", "key_filename":"/opt/key"}
with manager.connect(**dev, hostkey_verify=False) as m:
    config = m.get_config("running").data_xml
    print(config)
PY "$ip"
```

Bash loops over IPs and pipes XML to `xmllint` for XPath query.

---

## 35.3  Push Interface Description via YANG Patch

`iface.xml`:

```xml
<interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
  <interface>
    <name>GigabitEthernet0/0</name>
    <description>uplink-den</description>
  </interface>
</interfaces>
```

Send:

```bash
cat iface.xml | python - "$ip" <<'PY'
import sys, ncclient
xml = sys.stdin.read()
ip = sys.argv[1]
with ncclient.manager.connect(host=ip, username="netops", key_filename="/opt/key", hostkey_verify=False) as m:
    m.edit_config(target='running', config=xml)
PY
```

---

## 35.4  RESTCONF Curl Example

```bash
curl -k -u netops:pass   -H 'Content-Type: application/yang-data+json'   https://$ip/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet0%2F0   -d '{"description":"uplink-den"}'
```

Bash wrapper iterates devices.csv.

---

## 35.5  Candidate  Confirmed Commit

```bash
m.edit_config(target='candidate', config=xml)
m.commit()
```

Rollback: `m.discard_changes()` if healthcheck fails.

---

## 35.6  YANG Validator

Install `pyang`:

```bash
pyang -f tree -Werror vendor.yang
```

CI job validates any change to `yang/` directory.

---

## 35.7  Exercises

1. Write a Python snippet inside Bash that fetches interface counters via
   NETCONF `<get>` and outputs CSV.  
2. Build a RESTCONF patch script that sets MTU and verify diff returns empty
   via `--dry-run`.  
3. Add a pyang CI job that fails if modified YANG model introduces
   leaf/node name that conflicts with IETF core.

---
