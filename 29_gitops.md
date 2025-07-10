
# Chapter29  GitOps & ChangeFreeze Workflows

Production changes should flow through a versioncontrolled pipelinenever a
oneoff SSH command. GitOps brings declarative configs, merge approvals, and
automatic rollouts; changefreeze guards prevent deployments during blackout
windows (e.g., Superbowl broadcast).

---

## 29.1  Branch & Tag Strategy

| Branch | Purpose |
|--------|---------|
| `main` | Always deployable |
| `dev` | Integration; unit tests pass |
| `hotfix/*` | Emergency patches |
| Tag `fwYYYYMMDD` | Immutable firmware release |

Deploy job triggers only on tags matching `fw-*`.

---

## 29.2  GitLab CI Rollout Template

`.gitlab-ci.yml`:

```yaml
stages: [lint, test, deploy]

deploy_firmware:
  stage: deploy
  rules:
    - if: '$CI_COMMIT_TAG =~ /^fw-/'
      when: on_success
  image: alpine
  script:
    - apk add --no-cache openssh parallel
    - parallel -j50 scp fw.bin admin@{}:/tmp/fw.bin ::: $(<edges.txt)
    - ./apply.sh
  environment:
    name: production
    url: https://grafana/firmware
```

Rollback job on manual trigger:

```yaml
rollback:
  stage: deploy
  when: manual
  script: ./rollback.sh
```

---

## 29.3  Protected Branch & Tag Settings

* Protect `main` and `fw-*` tagsonly CI can create.  
* Require **2 Code Owners** approval on `devices.csv` changes.  
* Enable *Signed Commits* (GPG) for audit trail.

---

## 29.4  PreCommit Hooks

`.git/hooks/pre-commit`:

```bash
#!/usr/bin/env bash
shellcheck -x **/*.sh || exit 1
python lint_csv.py devices.csv || exit 1
```

Ensures linter passes before MR.

---

## 29.5  Automatic Release Notes

```bash
git tag -a fw-20250710 -m "FW2025 GA"
git log --pretty=format:'* %s' $(git describe --tags --abbrev=0)..HEAD >release.md
```

Uploads markdown to Confluence via REST.

---

## 29.6  ChangeFreeze Guard

`freeze_guard.sh`:

```bash
freeze=$(curl -s http://calendar/api/freeze | jq -r .on)
if [[ $freeze == "true" ]]; then
  echo "Change freeze active"; exit 2
fi
```

Call at top of deploy job; pipeline fails fast.

Fallback: YAML rules:

```yaml
rules:
  - if: '$CHANGE_FREEZE == "true"'
    when: never
```

`CHANGE_FREEZE` variable injected by schedule.

---

## 29.7  GitOps PullBased Model

Kubernetes Flux example:

* Repo contains `helmrelease.yaml` referencing firmware image.  
* Flux watches branch; applies new image tag (~1min).  
* Rollback: `git revert` commit  Flux reconciles.

---

## 29.8  Exercises

1. Write a Bash script that compares `devices.csv` from `main` vs MR branch
   and fails CI if >10% of devices changed without Jira link.  
2. Implement a Git prereceive hook on server that blocks pushes during
   freeze window.  
3. Create GitLab `jobs:<name>.only:variables` rule that allows hotfix tag
   deployment even during freeze.

---

