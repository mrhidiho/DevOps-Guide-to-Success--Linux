
# Chapter26  Seccomp & AppArmor for Container Sandboxing

Nonroot containers are necessary but **not sufficient**locking down system
calls (seccomp) and filesystem access (AppArmor) adds a defense layer for
your Bash automation containers.

---

## 26.1  Understanding Seccomp Profiles

Docker default profile blocks ~44 syscalls (e.g., `ptrace`, `kexec_load`).

### Generate Baseline

```bash
docker run --rm --security-opt seccomp=unconfined   -it alpine:3.20 sh -c 'apk add -q strace; while true; do sleep 1; done' &
pid=$!
strace -p $pid -c -o strace.txt
```

Feed trace into `oci-seccomp-bpf-hook` to autogenerate minimal profile.

---

## 26.2  Apply Custom Profile

`seccomp.json` snippet:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "archMap": [ { "architecture": "SCMP_ARCH_X86_64"} ],
  "syscalls": [
    { "names": ["read","write","exit","futex","epoll_wait"], "action":"SCMP_ACT_ALLOW" }
  ]
}
```

Run:

```bash
docker run --security-opt seccomp=seccomp.json bash-tool:1.0
```

---

## 26.3  AppArmor Quick Profile

Create `/etc/apparmor.d/bash-tool`:

```
#include <tunables/global>

profile bash-tool flags=(attach_disconnected) {
  network inet,
  capability chown,
  /usr/bin/bash ix,
  /opt/scripts/** r,
  /tmp/** rw,
}
```

Load:

```bash
apparmor_parser -r /etc/apparmor.d/bash-tool
```

Apply:

```bash
docker run --security-opt "apparmor=bash-tool" bash-tool:1.0
```

---

## 26.4  Troubleshooting Denials

Syslog:

```bash
grep DENIED /var/log/kern.log
```

Interactive test:

```bash
docker run --rm --security-opt seccomp=... sh -c 'dd if=/dev/kmem of=/dev/null'
```

Should fail with `Operation not permitted`.

---

## 26.5  CI Pipeline Enforcement

GitLab:

```yaml
seccomp_test:
  image: docker
  services: [docker:dind]
  script:
    - docker run --security-opt seccomp=seccomp.json bash-tool:1.0 --help
```

Fail job if `docker run` exits nonzero.

---

## 26.6  Hardening Podman/Kubernetes

Pod spec:

```yaml
securityContext:
  seccompProfile:
    type: Localhost
    localhostProfile: my-seccomp.json
  appArmorProfile: localhost/bash-tool
```

---

## 26.7  Exercises

1. Generate a strict seccomp profile for `curl` that only allows `socket`,
   `connect`, `sendto`, `recvfrom`, and test file download.  
2. Create an AppArmor profile that prevents network accessverify your Bash
   script errors when it tries to curl.  
3. Build a GitHub Action that builds an image, scans with `dockle`, and fails
   if it reports a critical security score.

---
