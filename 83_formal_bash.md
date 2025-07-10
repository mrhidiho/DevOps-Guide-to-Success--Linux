
# Chapter83  Formal Verification of Bash Logic with SMT Solvers

Shell scripts run firmware on **5000 routers**one offbyone bug can brick a
POP.  By translating critical Bash to an intermediate language (Boogie) and
feeding it to an SMT solver like **Z3**, we can _prove_ properties: loops
terminate, variables stay within bounds, files never exceed quota.

_Last updated: 2025-07-10_

---

## 83.1  Toolchain Overview

| Stage | Tool | Purpose |
|-------|------|---------|
| Bash  AST | `bash-parser` (TreeSitter) | Produce JSON AST |
| AST  Boogie | `bash2boogie` (Python) | Generate Boogie procedures |
| Boogie  SMT | **Boogie** CLI | Inserts verification conditions |
| SMT  Proof | **Z3** solver | Proves / finds counterexample |

Install:

```bash
pip install tree_sitter boogie-linter bash2boogie z3-solver
```

Boogie binary:

```bash
wget https://github.com/boogie-org/boogie/releases/download/v3.0.3/Boogie.zip
unzip Boogie.zip -d /opt/boogie
export PATH=$PATH:/opt/boogie
```

---

## 83.2  Example Bash Function

`fw_utils.sh`

```bash
copy_chunks () {
  local infile=$1 outfile=$2 chunksize=$3
  while read -r chunk; do
      echo "$chunk" >> "$outfile"   # append
      (( bytes += ${#chunk} ))
      if (( bytes > 1073741824 )); then  # >1 GiB
         echo "quota exceeded" >&2; return 1
      fi
  done < <(split -b "$chunksize" "$infile" -)
}
```

Properties to prove:

1. `bytes`  1GiB on all paths.  
2. Function terminates.  
3. `outfile` grows monotonically.

---

## 83.3  Generate Boogie

```bash
bash2boogie fw_utils.sh --func copy_chunks -o copy_chunks.bpl
```

Excerpt:

```boogie
procedure copy_chunks(infile:Ref, outfile:Ref, chunksize:int)
  modifies FS;
  ensures FS[outfile].size <= 1073741824;
{
  var bytes:int := 0;
  while (hasNextChunk(infile)) 
    invariant 0 <= bytes <= 1073741824;
  {
     var chunk:Ref := nextChunk(infile,chunksize);
     FS[outfile] := FS[outfile] + chunk;
     bytes := bytes + len(chunk);
     if (bytes > 1073741824) { return 1; }
  }
  return 0;
}
```

---

## 83.4  Run Boogie + Z3

```bash
boogie -z3exe z3 -z3opt /proverOpt:O copy_chunks.bpl
```

Output:

```
copy_chunks ... verified
```

Change guard to `>=` and Boogie finds counterexample:

```
assertion violation ...
```

---

## 83.5  Automate in CI

GitLab job:

```yaml
verify:
  image: python:3.12
  script:
    - pip install bash2boogie z3-solver boogie-linter
    - bash2boogie fw_utils.sh -o fw_utils.bpl
    - boogie fw_utils.bpl
```

Pipeline fails on proof error.

---

## 83.6  Modeling File Quota in Boogie

`FS` map models filecontent. Add ghost variable:

```boogie
var quota:int;
axiom quota == 1073741824;
```

Assert `len(FS[outfile]) <= quota`.

---

## 83.7  LargeScale Strategy

* Gate only _safetycritical_ functions (flashing, deletion).  
* Keep UI or log loops outside verification scope.  
* Use `expect` stubs for CLI interaction, model pre/poststate.

---

## 83.8  Prometheus Export of Proof Runtime

```bash
t=$(boogie fw.bpl 2>&1 | grep 'verified' -B1 | head -1 | awk '{print $3}')
echo "boogie_proof_ms $t" > /var/lib/node_exporter/text/boogie.prom
```

Alert if proof >10s (complexity blowup).

---

## 83.9  RealWorld Find

Reviewer missed `bytes += $chunksize` outside loop; proof failed, counterexample
showed quota overshoot at 1.2GiB. Bug fixed before production.

---

## 83.10  Exercises

1. Add invariant ensuring `chunksize > 0`; watch Boogie prove loop progress.  
2. Translate Chapter24 kerneltuning script and ensure sysctl values stay 
   max.  
3. Extend `bash2boogie` to model associative arrays via `map<Ref,int>`.

---
