
```
# Chapter 60 – pytest-Bash Integration for CI

You already unit-test Python with `pytest`, but Bash rarely gets the same rigor.  
The **pytest-bash** plugin (plus vanilla `pytest` fixtures) lets you treat Bash
functions as first-class test targets—complete with xUnit output, coverage
metrics, and parallel execution in GitLab/GitHub runners.

_Last updated: 2025-07-10_

---

## 60.1  Install Plugin

```bash
pip install pytest pytest-bash
```

(Optionally add **pytest-coverage** for branch metrics.)

---

## **60.2**  **Baseline Bash Script Under Test**

fw_utils.sh

```
#!/usr/bin/env bash
set -euo pipefail

calc_sha () {
  sha256sum "$1" | awk '{print $1}'
}

needs_upgrade () {
  local ver_cur=$1 ver_new=$2
  [[ "$ver_cur" != "$ver_new" ]]
}
```

Source it in tests.

---

## **60.3**  **Minimal pytest Test File**

test_fw_utils.py

```
import bash

def test_sha(tmp_path):
    file = tmp_path / "foo.bin"
    file.write_bytes(b"abc")
    result = bash.run("calc_sha %s" % file)
    assert result.stdout.strip() == "ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad"

def test_needs_upgrade_true(bash_session):
    out = bash_session.run("needs_upgrade 1.0 2.0")
    assert out.returncode == 0

def test_needs_upgrade_false(bash_session):
    out = bash_session.run("needs_upgrade 2.0 2.0", check=False)
    assert out.returncode == 1
```

Plugin auto-loads functions from the current directory’s ***.sh**.

---

## **60.4**  **Fixtures & Helpers**

```
import pytest, bash

@pytest.fixture
def fw_file(tmp_path):
    p = tmp_path / "fw.bin"
    p.write_bytes(b"1234")
    return p

def test_sha_fixture(fw_file):
    result = bash.run(f"calc_sha {fw_file}")
    assert len(result.stdout.strip()) == 64
```

---

## **60.5**  **Coverage for Bash**

```
pip install coverage-bash
export BASHCOVERAGE_FILE=.bash_coverage
pytest &&
coverage-bash --report
```

Shows which functions/branches ran.

---

## **60.6**  **GitLab CI Example**

.gitlab-ci.yml

```
stages: [lint, test]
lint:
  image: koalaman/shellcheck
  script: shellcheck *.sh

test:
  image: python:3.12
  script:
    - pip install pytest pytest-bash coverage-bash
    - pytest -q
    - coverage-bash --fail-under=80
```

Fails pipeline if Bash coverage < 80 %.

---

## **60.7**  **Parametrized Tests for Edge Cases**

```
import pytest, bash

@pytest.mark.parametrize("cur,new,expected", [
    ("1.0","2.0",0),
    ("2.0","2.0",1),
    ("2.0","1.0",1),
])
def test_upgrade_matrix(cur,new,expected):
    rc = bash.session.run(f"needs_upgrade {cur} {new}", check=False).returncode
    assert rc == expected
```

---

## **60.8**  **Mixing Python & Bash in Same Suite**

* Python unit-tests (**tests/**) check data transforms.
* Bash tests (via plugin) verify CLI wrappers.
* **pytest** merges results into single JUnit XML for Jenkins/GitLab.

---

## **60.9**  **Exercises**

1. Add a failing test for **calc_sha** when file is missing—ensure function exits with code 2 and proper stderr.
2. Integrate **pytest-bash** into Chapter 31’s Expect scripts by invoking Expect via Bash and asserting prompts appear.
3. Generate HTML coverage report (**coverage-bash --html**) and upload as CI artifact.

---

```
Feel free to embed this Markdown into your repo; let me know if you’d like further enhancements or a packaging retry when environment issues clear up.
```
