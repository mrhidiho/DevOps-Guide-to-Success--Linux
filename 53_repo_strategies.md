
```
# Chapter 53 – Multi-Repo Orchestration & Monorepo Strategies

Managing thousands of device-specific configs plus firmware artifacts quickly
turns messy if each vendor lives in its own Git island.  This chapter compares
**multi-repo** (Google `repo`, Git submodules) versus **monorepo** (single
giant repository) approaches, then shows Bash tooling to keep them in sync and
enable atomic rollbacks.

_Last updated: 2025-07-10_

---

## 53.1  Trade-Off Matrix

| Model        | Pros                                   | Cons                                |
|--------------|----------------------------------------|-------------------------------------|
| **Monorepo** | Atomic commits across all vendors; one CI pipeline; easy code-owners | Huge clone size; permissions harder; long CI runtimes |
| **Submodules** | Independent history per vendor; smaller clones | Submodule drift; harder rollbacks; nested PR flow |
| **Google `repo`** | Declarative manifest; pin SHAs; built-in parallel sync | Learning curve; extra tooling in CI |

---

## 53.2  Creating a `repo` Manifest

`default.xml`

```xml
<manifest>
  <remote  name="origin" fetch="https://git.internal/" />
  <default remote="origin" revision="main" />
  <project name="router-firmware" path="router/fw" revision="fw-20250710" />
  <project name="switch-configs"  path="switch/cfg" />
  <project name="monitoring-rules" />
</manifest>
```

Bootstrap:

```
repo init -u https://git.internal/manifests -m default.xml
repo sync -j8
```

---

## **53.3**  **Atomic Tag Rollouts (Monorepo)**

1. Merge firmware + config + CI pipeline change into **main**.
2. Create release tag:

```
git tag -a rollout-2025-07-10 -m "FW 2025 GA"
git push origin rollout-2025-07-10
```

3. GitLab deploy job triggers **only** on tags **rollout-***.

**Rollback: **git revert -m1 rollout-2025-07-10**.**

---

## **53.4**  **Submodule Version Pinning**

```
git submodule add https://git.internal/router-fw router/fw
git commit -m "Add router-fw submodule"
```

Update:

```
cd router/fw && git fetch --tags && git checkout fw-20250710
cd ../.. && git add router/fw && git commit -m "Bump fw"
```

CI ensures submodules are initialized:

```
before_script:
  - git submodule sync --recursive
  - git submodule update --init --recursive
```

---

## **53.5**  **Bash Guard Against Detached Submodules**

check_submodules.sh

```
git submodule foreach '
  branch=$(git symbolic-ref -q --short HEAD || echo detached)
  if [[ $branch == detached ]]; then
     echo "$name is detached" >&2; exit 1
  fi
'
```

Fails pipeline if any submodule is on a detached SHA unexpectedly.

---

## **53.6**  **Repo “partial clone” for CI Speed**

```
repo sync --depth=1 -c -j16
```

**-c** = current branch only; **--depth=1** = shallow.

---

## **53.7**  **Permissions Model**

* **Monorepo**: use CODEOWNERS sections (**/router/** owned by NetEng; **/docs/** by PM).
* **Multi-repo**: repo manifest pins SHAs, but GitLab group permissions isolate vendor code.

---

## **53.8**  **Exercises**

1. Write a Bash script that parses **default.xml** and outputs a CSV of project → SHA so you can diff manifests across branches.
2. Add a Git pre-receive hook that rejects pushes to **main** if any submodule reference moves **backward** (prevents accidental downgrade).
3. Prototype migrating your current submodule layout into a monorepo using **git filter-repo** and compare CI runtimes.

---

```
If you prefer a ZIP archive once the runtime issues are resolved, just let me know and I’ll package it.
```
