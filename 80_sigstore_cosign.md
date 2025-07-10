
# Chapter80  Sigstore & cosign: Script & Binary SupplyChain Attestation

With hundreds of firmware tools, a single unsigned `.pex` can introduce
malware. **Sigstore** provides free, keyless signing (Fulcio certificates +
Rekor transparency log). `cosign` verifies signatures at deploy time and in
Kubernetes admission webhooksno private CA needed.

_Last updated: 2025-07-10_

---

## 80.1  Core Components

| Tool | Role |
|------|------|
| **cosign** | Sign / verify container images, files |
| **Fulcio** | Issues shortlived X.509 cert bound to OIDC identity |
| **Rekor** | Public appendonly log (transparency) |
| **PolicyController** | Gatekeeper/OPA enforcing sigstore policies in K8s |

---

## 80.2  Install cosign CLI

```bash
curl -L https://github.com/sigstore/cosign/releases/download/v2.2.4/cosign-linux-amd64 \
  -o /usr/local/bin/cosign && chmod +x /usr/local/bin/cosign
```

Login (GitHub OIDC):

```bash
COSIGN_EXPERIMENTAL=1 cosign login ghcr.io
```

---

## 80.3  Keyless Sign Firmware Binary

```bash
COSIGN_EXPERIMENTAL=1 \
  cosign sign-blob --yes --oidc-issuer=https://token.actions.githubusercontent.com \
  --output-signature fw.bin.sig --output-certificate fw.bin.pem fw.bin
```

Signature + cert uploaded to Rekor; log index printed.

Verify later:

```bash
cosign verify-blob --certificate fw.bin.pem --signature fw.bin.sig fw.bin
```

---

## 80.4  Sign & Push Container to GHCR

```bash
docker build -t ghcr.io/org/fwcli:1.5 .
docker push ghcr.io/org/fwcli:1.5
COSIGN_EXPERIMENTAL=1 cosign sign --yes ghcr.io/org/fwcli:1.5
```

`ghcr.io/org/fwcli:1.5` now has an OCI signature artifact.

---

## 80.5  GitHub Actions Workflow

`.github/workflows/sign.yml`

```yaml
permissions:
  id-token: write
  contents: read
jobs:
  sign:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: sigstore/cosign-installer@v3
      - run: |
          docker build -t ghcr.io/${{ github.repository_owner }}/fwcli:${{ github.sha }} .
          docker push ghcr.io/${{ github.repository_owner }}/fwcli:${{ github.sha }}
          COSIGN_EXPERIMENTAL=1 cosign sign --yes \
            ghcr.io/${{ github.repository_owner }}/fwcli:${{ github.sha }}
```

No longlived keys; GitHub OIDC token bound in cert.

---

## 80.6  Kubernetes Admission Policy

Install PolicyController:

```bash
kubectl apply -f https://storage.googleapis.com/cosign-e2e/policy-controller.yaml
```

Create policy:

```yaml
apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
  name: fwcli-signed
spec:
  images:
  - glob: "ghcr.io/org/fwcli*"
  authorities:
  - keyless:
      url: https://fulcio.sigstore.dev
      identities:
      - issuer: "https://token.actions.githubusercontent.com"
        subject: "repo:org/fwcli:*"
```

Unsigned image deploy attempt  403 denied.

---

## 80.7  Attesting SBOM & Vulnerability Scan

Generate SBOM:

```bash
syft packages image:ghcr.io/org/fwcli:1.5 -o spdx-json > sbom.json
cosign attest --predicate sbom.json --type spdx \
  ghcr.io/org/fwcli:1.5
```

Verify:

```bash
cosign verify-attestation --type spdx ghcr.io/org/fwcli:1.5
```

---

## 80.8  Prometheus Exporter for Verification Latency

Textfile:

```bash
lat=$(cosign verify-blob --certificate fw.pem --signature fw.sig fw.bin \
      2>&1 | grep 'Time verified' | awk '{print $3}')
echo "sigstore_verify_ms $lat" > /var/lib/node_exporter/text/sigstore.prom
```

Alert if `>500`ms (Rekor latency).

---

## 80.9  RealWorld Win

Supplychain audit (202506): All 312 repos passed conformance; unauthorized
binary found in stagingblocked by ClusterImagePolicy before rollout.

---

## 80.10  Exercises

1. Create a GitLab pipeline using IDtokens (`oidc` job JWT) to sign `fw.bin`.  
2. Configure Harbor registry with Notary v2 plugin and push cosign signature.  
3. Write a Bash script that bulk verifies all images in `kubectl get pods -o json`.

---
