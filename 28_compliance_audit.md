
# Chapter28  Compliance & CIS/STIG Audit Automation

Passing yearly audits doesnt have to be a scramble. Bash can automate CISCat,
OpenSCAP, and custom regex tests, producing diffable reports and Jira tickets.

---

## 28.1  Running CISCat Lite

```bash
java -jar CISCAT-Lite.jar -b benchmark/Ubuntu20STIG -o reports/$(hostname).xml
```

Parse score:

```bash
score=$(xmllint --xpath 'string(//overall-score/text())' reports/*.xml)
echo "CIS_score{host=\"$(hostname)\"} $score" >/var/lib/node_exporter/text/cis.prom
```

---

## 28.2  OpenSCAP HTML Reports

```bash
oscap xccdf eval --profile stig --report osc.html ssg-ubuntu-ds.xml
```

Attach HTML to GitLab artifacts.

---

## 28.3  Custom Grep Tests

`custom_audit.sh`:

```bash
check(){
  if ! grep -q "$2" "$1"; then echo "FAIL $1 $2"; fi
}
check /etc/ssh/sshd_config "^Protocol 2"
check /etc/ssh/sshd_config "^PermitRootLogin no"
```

Log fails to `custom_audit.log`.

---

## 28.4  Jira Integration

```bash
while read -r line; do
  summary=$(echo $line | cut -d' ' -f2-)
  curl -u $JIRA -X POST --data "{"summary":"$summary","project":...}"        -H "Content-Type: application/json" https://jira/api/2/issue
done <custom_audit.log
```

---

## 28.5  CI Job Combining All

```yaml
audit:
  stage: security
  image: registry/gitlab.com/isp/audit:latest
  script:
    - ./cis_scan.sh
    - ./osc_scan.sh
    - ./custom_audit.sh
```

Pipeline fails if any tool returns score <90.

---

## 28.6  Exercises

1. Write Bash to diff new CIS score vs previous run and open PagerDuty if drop
   >5%.  
2. Extend `custom_audit.sh` to ensure `/boot` is separate partition.  
3. Automate update of STIG benchmark XML weekly via Git cron.

---
