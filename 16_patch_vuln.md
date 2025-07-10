
# Chapter16  Patch & Vulnerability Scanning Automation

Staying ahead of CVEs is critical for ISP infrastructure. This chapter
demonstrates a **Bashdriven pipeline** that:

1. Runs lightweight scanners (Lynis, OpenSCAP, Trivy) on firmware images or
   live nodes.  
2. Compares current package/kernel versions to the CVE feed.  
3. Generates diff reports and Slack/Jira tickets.  
4. Automates kernel/firmware patch rollout windows.

---

## 16.1  Choosing the Right Scanner

| Scanner | Target | Pros | Cons |
|---------|--------|------|------|
| **Lynis** | Live Linux host | Fast (<2min), no agent | Limited container insight |
| **OpenSCAP** | Image or host | DISA STIG profiles, XML output | Heavier runtime |
| **Trivy** | Container image / filesystem | CVE DB autoupdate, SBOM support | Requires internet for DB |

Most router firmware is monolithic squashfsextract and feed to Trivy.

---

## 16.2  Installing Lynis on JumpBox

```bash
apt-get install -y lynis
lynis audit system --quick --report-file /var/log/lynis_$(date +%F).dat
```

Parse result:

```bash
grep '^warning\|^suggestion' /var/log/lynis_*.dat | column -t
```

---

## 16.3  OpenSCAP Compliance Check

```bash
oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_standard      --results results.xml      --report report.html      /usr/share/openscap/scap-yaml/ssg-ubuntu2204-ds.xml
```

Automate in Bash loop across fleet:

```bash
for host in $(<jump_hosts.txt); do
  ssh $host "oscap xccdf eval --profile standard --results - oval.xml"       >reports/$host.xml
done
```

---

## 16.4  Trivy Image Scan for Firmware

1. **Extract** squashfs:

```bash
unsquashfs -d fw_root firmware.bin
```

2. **Scan**:

```bash
trivy fs --severity HIGH,CRITICAL fw_root --format json >trivy.json
```

3. **Diff** against previous scan:

```bash
jq -S . trivy_prev.json >a; jq -S . trivy.json >b
diff -u a b >cve.diff || true
```

Post diff to Slack if nonempty.

---

## 16.5  Kernel Patch Window Automation

Weekly cron on jumpbox:

```bash
if [[ $(uname -r) != $(ssh $host 'uname -r') ]]; then
   ssh $host 'apt-get update && apt-get -y dist-upgrade && reboot'
fi
```

Batch via GNU parallel Saturday 02:00 for POP den hosts.

---

## 16.6  CVE Feed Comparison (Ubuntu Example)

```bash
apt-get update
grep -A1 "^Package: linux-image" /var/lib/apt/lists/*security* |   awk '/linux-image/ {print $2}' >security_kernels.txt
```

Compare to running kernel.

---

## 16.7  CI Integration

GitLab job scanning container before push:

```yaml
trivy_scan:
  image: aquasec/trivy:latest
  script:
    - trivy fs --exit-code 1 --severity HIGH,CRITICAL .
```

Pipeline fails if new severe CVEs found.

---

## 16.8  Jira Ticket AutoCreate

```bash
if [[ -s cve.diff ]]; then
  curl -u $JIRA_USER:$JIRA_TOKEN -X POST     --data @issue.json     -H "Content-Type: application/json"     https://jira/api/2/issue
fi
```

`issue.json` prepared by jq from scan output.

---

## 16.9  Exercises

1. Write a Bash wrapper that runs Trivy on every `.bin` in `firmware/`
   directory and stores results under `sbom/`.  
2. Schedule Lynis weekly and send only new warnings since last run via
   diff.  
3. Build a script that reads `devices.csv`, SSHes RHEL devices, runs
   `yum updateinfo list security all`, and exports a CSV of unapplied CVEs.

---
