
# Chapter10  Testing, Linting & CI/CD Integration

A script that isnt tested will break at 3AM. This chapter shows how to use
ShellCheck, Bats, and CI pipelines (GitLab/GitHub) to keep your Bash tooling
productionready.

---

## 10.1  ShellCheck Static Analysis

### Installation

```bash
sudo apt-get install shellcheck   # Debian/Ubuntu
brew install shellcheck           # macOS
```

### Error Classes

| Code | Meaning | Example |
|------|---------|---------|
| **SC2086** | Unquoted var could expand to multiple words | `rm $file` |
| **SC2046** | Quote for wordsplitting in $(  ) | `echo $(ls *.cfg)` |
| **SC2164** | Missing `cd` error check | `cd /tmp` |
| **SC2034** | Unused variable | Typos in var names |

CI job:

```yaml
shellcheck:
  image: koalaman/shellcheck:latest
  script:
    - shellcheck -x **/*.sh
```

`-x` follows `source` directives.

---

## 10.2  Bats  Bash Automated Testing System

### Install

```bash
git clone https://github.com/bats-core/bats-core /opt/bats
/opt/bats/install.sh /usr/local
```

### Test Example

`test/flash.bats`

```bash
setup() {
  load 'test_helper/bats-support/load'
}

@test "flash exits 0 on success" {
  run bash -c 'source lib/fw_ops.sh; flash_fw 127.0.0.1 fw.bin'
  [ "$status" -eq 0 ]
}
```

Mock external commands:

```bash
scp() { return 0; }
ssh() { return 0; }
```

Run all tests:

```bash
bats test/*.bats
```

---

## 10.3  GitLab CI Pipeline (Example)

`.gitlab-ci.yml`

```yaml
stages:
  - lint
  - test
  - package

variables:
  SHELLCHECK_OPTS: "-x -e SC1091"

lint:
  stage: lint
  image: koalaman/shellcheck:latest
  script:
    - shellcheck $SHELLCHECK_OPTS **/*.sh

unit_tests:
  stage: test
  image: alpine
  script:
    - apk add --no-cache bash bats
    - bats test

package:
  stage: package
  image: alpine
  script:
    - tar czf scripts.tar.gz *.sh lib/
  artifacts:
    paths: [scripts.tar.gz]
```

Pipeline aborts if **any** ShellCheck error or Bats failure.

---

## 10.4  GitHub Actions Workflow

```yaml
name: Bash CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      - uses: ludeeus/action-shellcheck@v2
      - name: Run Bats
        run: |
          sudo apt-get install -y bats
          bats test
```

---

## 10.5  EndtoEnd Test with Docker

`Dockerfile.test`

```dockerfile
FROM alpine
RUN apk add --no-cache bash coreutils openssh
COPY . /app
WORKDIR /app
CMD bats e2e/*.bats
```

CI stage:

```yaml
docker_test:
  stage: test
  image: docker:24
  services: [docker:dind]
  script:
    - docker build -f Dockerfile.test -t bash-e2e .
    - docker run --rm bash-e2e
```

---

## 10.6  Canary Deployment Validator

Bats test hits a **canary modem** before greenlighting production batch:

```bash
@test "canary modem returns FW2025 after flash" {
  run ssh canary 'cat /etc/version'
  [ "$output" = "FW2025" ]
}
```

CI pipeline gate:

```yaml
canary_check:
  stage: test
  when: on_success
```

---

## 10.7  Exercises

1. Add ShellCheck to precommit hook so commits fail fast.  
2. Write a Bats test that mocks `ssh` failure and asserts `flash_fw` returns 1.  
3. Create GitHub Action that lints **and** builds a Docker image artifact only
   if all tests pass.

---

> Next: **Chapter11  Reference Tables & CheatSheets**.
