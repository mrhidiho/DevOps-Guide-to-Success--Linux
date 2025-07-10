
# Chapter19  Incident PostMortem & RCA Automation

After-action reviews (AAR) are critical in carrier environments. Bash can
autocollect logs, metrics, and timelines immediately after an incident,
reducing MTTR and improving institutional learning.

---

## 19.1  PostMortem Markdown Template

`templates/postmortem.md`:

```markdown
# Incident {{INCIDENT_ID}}  {{TITLE}}

|   | Detail |
|---|--------|
| Start Time | {{START}} |
| End Time   | {{END}} |
| Duration   | {{DURATION}} |
| Severity   | {{SEV}} |
| Affected Services | {{SERVICES}} |

## Timeline
| Time | Event |
|------|-------|
{{TIMELINE}}

## Root Cause
{{RCA}}

## Remediation
- [ ] Item 1
- [ ] Item 2

## Action Items
| Owner | Task | Due Date |
|-------|------|----------|
|       |      |          |
```

Variable placeholders substituted by Bash script.

---

## 19.2  Gathering Evidence Script

`collect_evidence.sh`:

```bash
#!/usr/bin/env bash
id=$(date +%Y%m%d-%H%M)
dest="incidents/$id"
mkdir -p "$dest"/{logs,metrics}

journalctl -b -1 >"$dest/logs/syslog_prev.txt"
cp /var/log/fw_rollout.log "$dest/logs/" || true

# Prometheus snapshot (requires snapshot API proxy)
curl -s http://prom/api/v1/admin/tsdb/snapshot >"$dest/metrics/snap.json"
```

---

## 19.3  Building the Timeline

```bash
timeline=$(journalctl -S "$START" -U "$END" -o short-iso |
           grep -E 'fwupdate|ssh' | sed 's/^/| /')

sed "s/{{TIMELINE}}/$timeline/" templates/postmortem.md >$dest/PM.md
```

`START`/`END` captured by monitoring alert webhooks.

---

## 19.4  Automatic Grafana Graph Exports

```bash
panel_id=17
png="$dest/metrics/latency.png"
curl -s -H "Authorization: Bearer $GRAF_TOKEN"   "https://grafana/api/dashboards/uid/lat?panelId=$panel_id&from=$FROM&to=$TO&png"   -o "$png"
```

Attach PNG to postmortem via Git commit.

---

## 19.5  Slack/Jira Integration

```bash
jq -n --arg text "Incident $id resolved. Draft PM ready."   '{text:$text}' | curl -XPOST -H 'Content-type: application/json'      -d @- $SLACK_WEBHOOK
```

---

## 19.6  CI Pipeline to Publish PM

```yaml
publish_pm:
  stage: report
  script:
    - git add incidents/$id
    - git commit -m "Postmortem $id"
    - git push
```

Optionally autocreate Jira ticket linking commit.

---

## 19.7  Action Item Extraction

Simple parser extracts `- [ ]` lines:

```bash
grep -n '\- \[ \]' $dest/PM.md >$dest/action_items.txt
```

Feeds into Jira REST loop.

---

## 19.8  Exercises

1. Extend `collect_evidence.sh` to snapshot `iptables-save` and
   `ip -s link` statistics.  
2. Add a section in template that embeds Prometheus alert history via the
   Alertmanager API (`amtool` JSON).  
3. Implement a Bash script that sends the PM Markdown to Confluence using
   its RESTAPI.

---

> Next optional chapter: **Serial Console Automation with minicom & Expect**.
