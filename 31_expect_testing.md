
# Chapter31  UnitTesting Expect Scripts with **pexpect** & **batsexpect**

Interactive automation can silently break when a vendor changes a prompt or
adds a confirmation question.  Treat Expect scripts like any other code:
unittest them in CI using **pexpect** (Python) and **batsexpect** (Bash).

---

## 31.1  Testing Matrix

| Layer | Tool | Goal |
|-------|------|------|
| Unit | **pexpect** | Mock CLI prompts, validate send/expect flow |
| Integration | **batsexpect** | Run Expect against a fake Telnet/SSH server |
| EndtoEnd | Real device in lab | Verify firmware flash full path |

---

## 31.2  pexpect Mock CLI

`mock_switch.py`:

```python
#!/usr/bin/env python
import pexpect, sys
PROMPT = 'SW>'
child = pexpect.spawn('cat')  # simple echo server
child.sendline('Booting...')
while True:
    cmd = child.readline().decode().strip()
    if cmd == 'show ver':
        child.sendline('FW 1.2.3')
    elif cmd == 'exit':
        sys.exit(0)
    child.sendline(PROMPT)
```

Run in background: `python mock_switch.py | socat - TCP-LISTEN:2323,reuseaddr &`

---

## 31.3  pexpect Unit Test

`test_flash.py`:

```python
import pexpect, unittest, os
class FlashTest(unittest.TestCase):
    def test_show_version(self):
        child = pexpect.spawn('telnet localhost 2323')
        child.expect('SW>')
        child.sendline('show ver')
        child.expect('FW 1.2.3')
        self.assertIn('FW 1.2.3', child.before.decode())
if __name__ == '__main__':
    unittest.main()
```

Run in CI:

```yaml
unittest:
  image: python:3.12
  script:
    - pip install pexpect
    - python mock_switch.py &   # background
    - python -m unittest test_flash.py
```

---

## 31.4  batsexpect Integration

Install:

```bash
git clone https://github.com/ztombol/bats-expect /opt/bats-expect
export BATS_LIBPATH=/opt/bats-expect
```

Test:

```bash
@test "Expect prompts" {
  expect <<'EOF'
  spawn bash -c "cat <<PROMPT; sleep 0.5; echo done; PROMPT"
  expect "done"
EOF
}
```

---

## 31.5  Mock Telnet Server with `socat`

```bash
socat -v tcp-l:2323,reuseaddr,fork exec:'bash -c "echo SW>'"
```

bats test uses `nc` to send commands and compares transcript.

---

## 31.6  Golden Transcript Diff

Record baseline:

```bash
expect -d flash.exp 2>baseline.log
```

CI diff:

```bash
expect -d flash.exp 2>run.log || true
diff -u baseline.log run.log
```

Fails if prompt flow changed.

---

## 31.7  Exercises

1. Write a pexpect script that asserts your Expect library handles password
   change prompt (`"New password:"`).  
2. Use batsexpect to feed 5 simultaneous sessions via `flock` on a pseudotty.  
3. Integrate unit & integration tests into GitHub Actions matrix (python3.9,
   python3.12).

---

