
# Chapter13  User / Group & Sudo Auditing

Unauthorized local accounts or excessive sudo rights can lead to privilege
escalation and pivot attacks across an ISPs management network.  
This chapter builds Bash tooling to **enumerate**, **validate**, and
**remediate** user and group configurations on routers, switches, and Linux
jumpboxes.

---

## 13.1  Enumerating Accounts

### 13.1.1  Local `/etc/passwd`

```bash
awk -F: '{print $1,$3,$7}' /etc/passwd | column -t
```

| Field | Meaning |
|-------|---------|
| `$1` | Username |
| `$3` | UID (UID<1000 = system on modern distros) |
| `$7` | Login shell (`/sbin/nologin` = noninteractive) |

### 13.1.2  NSSAware Enumeration

```bash
getent passwd | cut -d: -f1
```

Includes LDAP/SSSD domain userscritical on shared jump boxes.

---

## 13.2  Detecting Rogue Accounts

Approved list example (`approved_users.txt`):

```
root
admin
netops
```

Audit script:

```bash
#!/usr/bin/env bash
set -euo pipefail
comm -23 <(getent passwd | cut -d: -f1 | sort) <(sort approved_users.txt)
```

Any usernames printed are **unauthorized**.

---

## 13.3  Group and Sudo Policy Checks

### 13.3.1  List Sudo Users

```bash
getent group sudo | cut -d: -f4 | tr ',' '\n'
```

Audit against `allowed_sudo.txt`.

### 13.3.2  Parse `/etc/sudoers` for ALL=NOPASSWD

```bash
grep -E '^[^#].*NOPASSWD' /etc/sudoers /etc/sudoers.d/* || true
```

CI pipeline fails if NOPASSWD detected outside automation account.

---

## 13.4  Aging & Lock Status

```bash
lastlog -b 90
chage -l username
```

Lock inactive users:

```bash
userdel -r oldacct      # or
passwd -l tempuser
```

---

## 13.5  Automating Remediation

`user_audit.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
mapfile -t allowed <approved_users.txt
while read -r user _; do
  if ! printf '%s\n' "${allowed[@]}" | grep -qx "$user"; then
     echo "Removing $user"
     userdel -r "$user"
  fi
done < <(getent passwd)
```

Run via Ansible or SSH loop on network devices with BusyBox `deluser`.

---

## 13.6  CI Security Gate

GitLab example:

```yaml
user_audit:
  stage: security
  script:
    - ./user_audit.sh
```

Fails build if new account slips into container image.

---

## 13.7  Exercises

1. Extend `user_audit.sh` to check that every interactive user's home dir
   exists and is owned by the user.  
2. Create a Bash script that diffs current sudoers against a Gittracked
   golden copy and patches drift automatically.  
3. Write an awk oneliner that prints all groups with **no** members (orphan
   groups).

---

> Next security chapter: **Filesystem Permissions & ACL Enforcement**.
