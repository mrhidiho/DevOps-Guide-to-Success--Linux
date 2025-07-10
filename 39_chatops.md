
# Chapter39  ChatOps & RealTime Ops Collaboration

Fusing automation with Slack/Teams chat speeds decisionmaking and leaves an
audit trail of every command. Bash + webhooks turns your scripts into ChatOps
bots.

---

## 39.1  Slack Slash Command Trigger

1. Create Slash Command `/rollout`  HTTPS endpoint.  
2. Use **ngrok** + `bash -c` handler:

```bash
#!/usr/bin/env bash
read payload
branch=$(jq -r .text <<<"$payload")
git checkout $branch && ./deploy.sh >out.txt 2>&1
curl -XPOST -H 'Content-type: application/json'    -d "{"text":"$(cat out.txt | tail -n20)"}" $SLACK_RESPONSE_URL
```

---

## 39.2  Interactive Approvals

Script waits for emoji reaction:

```bash
ts=$(date +%s)
msg=$(jq -n --arg t "Deploy to PROD?" '{text:$t}')
resp=$(curl -s -XPOST -H 'Content-type: application/json'        -d "$msg" $SLACK_WEBHOOK)
ts=$(echo $resp | jq -r .ts)

while true; do
  if curl -s "https://slack/api/reactions.get?channel=$CHAN&timestamp=$ts" |
     jq -e '.message.reactions[]?|select(.name=="thumbsup")' >/dev/null; then
     ./deploy.sh && break
  fi
  sleep 5
done
```

---

## 39.3  Alert Buttons

Alert JSON:

```json
{
  "text": "*Firmware failed on 3 nodes*",
  "attachments":[{
    "text":"Rollback?",
    "fallback":"buttons",
    "callback_id":"fw_fail_123",
    "actions":[
      {"name":"yes","text":"Rollback","type":"button","value":"yes"}
    ]
  }]
}
```

Webhook handler triggers `rollback.sh`.

---

## 39.4  Teams Webhook

```bash
json=$(jq -n --arg t "$msg" '{text:$t}')
curl -H 'Content-Type: application/json'      -d "$json" $TEAMS_WEBHOOK
```

---

## 39.5  Audit Log Via Bot

Each script posts to `#ops-log` channel:

```bash
curl -s -XPOST -H 'Content-type: application/json'   -d "{"text":"$USER ran $0 $* on $(hostname)"}" $LOG_WEBHOOK
```

---

## 39.6  Exercises

1. Build a Slack bot that shows rollout progress bar by editing message .  
2. Add `/freeze on|off` command that toggles `CHANGE_FREEZE` variable in Git.  
3. Write a Teams adaptive card that lists failed nodes with reboot button.

---

