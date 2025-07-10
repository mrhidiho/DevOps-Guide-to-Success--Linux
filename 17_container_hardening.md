
# Chapter17  Container Hardening & Minimal Base Images

Many Bash automation tasksscanning firmware, running parallel SSH, exporting
metricsare now packaged as **OCI containers**. Hardened images reduce
surfacearea, ease CI pipelines, and satisfy security audits.

---

## 17.1  Why Minimal Images?

| Benefit | Detail |
|---------|--------|
| Smaller attack surface | Fewer binaries  fewer CVEs |
| Faster pulls | <10MB Alpine vs 200MB Ubuntu |
| Simpler SBOM | Easier to diff & approve |
| Memoryefficient | Crucial for edge nodes running containerd |

---

## 17.2  Choosing a Base

| Option | Size | Notes |
|--------|------|-------|
| `alpine:3.20` | ~7MB | BusyBox; musl libc (watch glibc deps) |
| `ubuntu:24.04-minimal` | ~25MB | glibc; apt works |
| `busybox:uclibc` | ~2MB | Bare; add via `--install` |

Decision tree:

* Need glibc binary (e.g., Lynis)?  Ubuntu minimal  
* Pure Bash & coreutils?  Alpine or BusyBox.

---

## 17.3  Building an Alpine Bash Tool Image

`Dockerfile`:

```dockerfile
FROM alpine:3.20

RUN apk add --no-cache bash openssh-client parallel     && addgroup -S app && adduser -S app -G app

USER app
WORKDIR /app

COPY scripts/ /app/scripts/
ENTRYPOINT ["/bin/bash"]
```

### Hardening Flags

* **Rootless**: create nonroot `app` user.  
* **No package cache**: `--no-cache` prevents leftover index.  
* Copy only scripts; no build toolchain in final image.

---

## 17.4  Dropping Capabilities & Seccomp

Compose example:

```yaml
services:
  fwtool:
    image: isp/fwtool:1.0
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
```

* `cap_drop: ALL` removes NET_RAW, SYS_ADMIN, etc.  
* `no-new-privileges` stops `setuid` escalation.  
* `read_only` FS prevents tampering.

---

## 17.5  Trivy & Grype Image Scans in CI

GitLab job:

```yaml
scan_image:
  stage: security
  image: aquasec/trivy
  script:
    - trivy image --severity HIGH,CRITICAL --exit-code 1 isp/fwtool:$CI_COMMIT_SHA
```

Pipeline fails if severityHigh CVEs found.

---

## 17.6  Rootless Podman for Local Testing

```bash
podman run --rm -it --security-opt=no-new-privileges fwtool bash
```

Podman drops root inside user namespace; mimics prod Kubernetes PSP.

---

## 17.7  Multistage Build Example

```dockerfile
# builder stage
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o fwtool ./cmd

# final
FROM alpine:3.20
RUN apk add --no-cache ca-certificates
COPY --from=build /src/fwtool /usr/local/bin/fwtool
USER 1001
ENTRYPOINT ["fwtool"]
```

Deliver 12MB final image vs 1.2GB Go builder.

---

## 17.8  SBOM Generation

```bash
syft packages docker:isp/fwtool:latest -o spdx-json >sbom.json
```

Attach SBOM as GitLab artifact; feeds vulnerability scanners.

---

## 17.9  Container Runtime Limits

Kubernetes pod spec snippets:

```yaml
securityContext:
  runAsNonRoot: true
  seccompProfile: { type: RuntimeDefault }
resources:
  limits:
    memory: 256Mi
    cpu: "0.5"
```

Prevents noisy script from exhausting node.

---

## 17.10  Exercises

1. Convert existing `fwupdate` Bash script into a container image that passes
   Trivy with zero HIGH CVEs.  
2. Add GitHub Action that builds, signs (`cosign sign`), and uploads SBOM.  
3. Experiment with `dockle` lintingfail CI if `USER` still root.

---

> **End of container hardening chapter.**
