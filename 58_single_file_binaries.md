
```
# Chapter 58 – PyInstaller & PEX: Shipping Single-File Binaries

Distributing Python + Bash hybrids across jump-boxes can be painful when
libraries or virtualenvs mismatch.  **PyInstaller** and **PEX** bundle your
tool into a single self-contained file—SCP it beside the firmware and run.

_Last updated: 2025-07-10_

---

## 58.1  PyInstaller One-Liner

```bash
pip install pyinstaller
pyinstaller --onefile fwcli.py
```

Output: **dist/fwcli** (≃12 MB) with embedded interpreter + libs.

Run on target:

```
scp dist/fwcli edge-srv:/usr/local/bin/
ssh edge-srv fwcli stage fw.bin
```

---

## **58.2**  **Reducing Size**

```
pyinstaller --onefile --strip --clean fwcli.py \
            --exclude-module tests \
            --add-data "templates/:templates"
```

**--strip** removes symbols; **--clean** deletes build tree.

---

## **58.3**  **PEX for Hermetic ZIP-App**

```
pip install pex
pex fabric invoke netmiko -c fwcli -o fwcli.pex
chmod +x fwcli.pex
```

*Runs on any host* with compatible glibc & Python 3.8+.

**Upgrade path: drop newer **fwcli.pex**, symlink **/usr/local/bin/fwcli → fwcli.pex**.**

---

## **58.4**  **Bash Self-Updater Wrapper**

```
#!/usr/bin/env bash
BIN=/opt/fwcli
curl -s -o $BIN.new https://artifacts/fwcli.pex
if sha256sum -c sha256.txt; then
  mv $BIN.new $BIN && chmod +x $BIN
fi
```

Cron weekly refresh.

---

## **58.5**  **Signing for Secure-Boot Chains**

```
openssl dgst -sha256 -sign capriv.pem -out fwcli.sig fwcli
scp fwcli fwcli.sig router:/flash/tools/
```

Router verifies via OpenSSL API before exec.

---

## **58.6**  **GitLab CI Packaging Job**

```
package:
  image: python:3.12
  script:
    - pip install pex
    - pex -r requirements.txt -c fwcli -o fwcli_${CI_COMMIT_TAG}.pex
  artifacts:
    paths: [fwcli_${CI_COMMIT_TAG}.pex]
    expire_in: 30d
```

---

## **58.7**  **Exercises**

1. Build a PyInstaller binary for your **rtbh.py** FlowSpec tool; test on Alpine and Ubuntu containers.
2. Add version flag (**--version**) that reads the embedded Git tag at build time (**--version-file**).
3. Measure startup time (**time ./fwcli --help**) of PyInstaller vs PEX; tune **PEX_ROOT** to cache unzip step.

---

```
---

Copy the above into your repository; if you still need a ZIP once the sandbox is functional again, just let me know! |oai:code-citation|
```
