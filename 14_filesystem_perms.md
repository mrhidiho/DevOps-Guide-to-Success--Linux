
# Chapter14  Filesystem Permissions & ACL Enforcement

Misset permissionsworldwritable config files, stray SUID binariesare a
pentesters doorway into privilege escalation.  This chapter equips you with
Bash+ coreutils to **scan**, **report**, and **fix** filesystem permission
issues on routers, switches, and Linux servers.

---

## 14.1  Linux Permission Basics Refresher

| Symbolic | Octal | Meaning |
|----------|-------|---------|
| `rwxr-xr-x` | `755` | Owner full; group/world read+exec |
| `rw-r-----` | `640` | Owner read/write; group read |
| `rwsr-xr-x` | `4755` | SUID root (danger for custom bins) |

Special bits:

| Flag | Octal | Purpose |
|------|-------|---------|
| **SUID** | 4### | Exec runs with owner UID |
| **SGID** | 2### | Exec runs with group GID / dir inherits group |
| **Sticky** | 1### | Only owner can delete in worldwritable dir (e.g., /tmp) |

---

## 14.2  Scanning for WorldWritable & SUID Files

### WorldWritable (nonsticky)

```bash
find / -xdev -type f -perm -0002 ! -perm -1000 -print >world_writable.txt
```

* `-xdev` stay on same filesystem.  
* `-perm -0002` any write bit for others.  
* `! -perm -1000` exclude sticky.

### SetUID / SetGID

```bash
find / -xdev \( -perm -4000 -o -perm -2000 \) -type f -print >suid_sgid.txt
```

Whitelist expected binaries:

```bash
comm -23 suid_sgid.txt whitelist_suid.txt >unknown_suid.txt
```

---

## 14.3  ACL (Access Control List) Audit

List ACLs:

```bash
getfacl -R /etc | grep -v '^#' >acl.txt
```

Look for unexpected user entries:

```bash
grep -E 'user:.*:rwx' acl.txt
```

Remove rogue ACL:

```bash
setfacl -x u:tester /etc/secret.conf
```

---

## 14.4  Fixing Permissions Recursively

### Config Trees

```bash
chmod -R go-rwx /flash/config
chown -R root:netops /flash/config
```

### Remove SUID Bit

```bash
chmod u-s /usr/local/bin/old_backup
```

---

## 14.5  Immutable Flag for Critical Files

Prevent even root overwrite without chattr clear:

```bash
chattr +i /etc/ssh/sshd_config
```

Clear flag for updates:

```bash
chattr -i /etc/ssh/sshd_config && vi ...
```

---

## 14.6  File Integrity Monitoring (FIM) with SHA256

Generate baseline:

```bash
find /flash/config -type f -exec sha256sum {} + >baseline.sha256
```

Verify:

```bash
sha256sum -c baseline.sha256 --quiet | grep -v 'OK' >drift.txt
```

Alert if `drift.txt` nonempty.

---

## 14.7  Automating in Bash Across Fleet

`perm_audit.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail; IFS=$'\n\t'
mapfile -t hosts <hosts.txt

audit_node(){
  ssh "$1" "find / -xdev -perm -4000 -o -perm -2000" |      grep -v -f whitelist_suid.txt
}
export -f audit_node

parallel -j30 audit_node ::: "${hosts[@]}" >suspect_suid.csv
```

---

## 14.8  Integrating with CI Container Builds

Dockerfile snippet:

```dockerfile
RUN chmod 600 /etc/ssh/ssh_host_* &&     find / -xdev -perm -0002 -exec chmod o-w {} +
```

CI stage:

```yaml
script:
  - ./scan_permissions.sh || { echo "Bad perms"; exit 2; }
```

---

## 14.9  Exercises

1. Write a Bash function `fix_world_writable <dir>` that removes otherwrite
   bit but preserves sticky when present.  
2. Extend `perm_audit.sh` to output JSON suitable for ingestion by Elastic
   Security (`jq -R -s`).  
3. Create a systemd timer that verifies SUID whitelist every 15minutes and
   masks any package installing unexpected SUID bins.

---

> Next security chapter: **Log Analysis & Intrusion Indicators**.
