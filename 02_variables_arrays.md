
# Chapter2  Variables, Arrays & Parameter Expansion

> If strict mode keeps your script alive, **parameter expansion** makes it
> *smart*.  NetworkOps maxim

This chapter dives deep into Bash variablesfrom simple scalars to associative
mapsshowing how to leverage them for device inventories, firmware matrices,
and state tracking across thousands of routers and modems.

---

## 2.1  Scalar Variables

### Declaring & assigning

```bash
fw="FW2025"                 # always quote!
device_count=0              # integers need no `$(( ))` here
```

*Never* add spaces around `=``fw = prod` spawns `fw` external command.

### Readonly & integer attributes

```bash
readonly MODE=prod          # accidental overwrite kills scripts
declare -i retries=0        # arithmetic autocasts
```

Why care? A readonly `SITE_ID` prevents rogue subscripts overriding site
target and bricking another POP.

---

## 2.2  Parameter Expansion Patterns

| Syntax | Purpose | Example | Result |
|--------|---------|---------|--------|
| `${var:-default}` | Fallback if unset/empty | `${fw:-FW2023}` | `FW2025` |
| `${var:?ERR}` | Abort if unset | `${PORT:?port missing}` | exits1 |
| `${var^^}` / `${var,,}` | Case modify | `${role^^}` | `EDGE` |
| `${file##*/}` | Strip longest prefix | `/tmp/fw.bin`  `fw.bin` |
| `${file%.*}` | Strip shortest suffix | `fw.bin`  `fw` |

### Realworld: build OTA URL

```bash
fw_file="/tmp/fw_${fw,,}.bin"
url="https://dl.isp.com/$SITE_ID/${fw_file##*/}"
```

*Lowercases* firmware name and strips path for clean URL.

---

## 2.3  Arithmetic Expansion & Counters

```bash
((success++))
(( percent = success * 100 / total ))
printf 'Progress: %d%%\n' $percent
```

Counters run inshellno external `expr`  10 faster inside 5knode loops.

---

## 2.4  Indexed Arrays

```bash
edges=(edge{1..4})
echo "${edges[0]}"      # edge1
edges+=("edge5")        # append
echo "${#edges[@]} nodes total"
```

### Iterating safely

```bash
for ip in "${edges[@]}"; do
  ssh "$ip" 'uptime'
done
```

Using `${edges[@]}` preserves spaces in hostnames like `"qa switch"`.

### Reading an array from file

```bash
mapfile -t cmts <cmts_ips.txt      # bash 4+
```

**mapfile** (aka `readarray`) avoids whileloops and is Cspeed.

---

## 2.5  Associative Arrays (Bash4+)

```bash
declare -A state
state[$ip]=success
state[$ip,sha]=$sha              # compound key hack

for key in "${!state[@]}"; do
  printf '%s => %s\n' "$key" "${state[$key]}"
done
```

#### Usecase: success/fail histogram

```bash
declare -A tally=( [ok]=0 [fail]=0 )
for rc in "${state[@]}"; do ((tally[$rc]++)); done
printf 'OK %d  FAIL %d\n' ${tally[ok]} ${tally[fail]}
```

---

## 2.6  Passing Variables & Arrays to Functions

```bash
flash_batch(){ local -n list=$1
  for ip in "${list[@]}"; do flash_fw "$ip" & done; wait
}

flash_batch edges       # pass by reference (bash 4.3+)
```

`local -n list=$1` creates **nameref**the function works on callers array
without copy.

---

## 2.7  Indirect Expansion & Dynamic Variable Names

```bash
site=den; var="fw_$site"
declare "$var=FW2025"
echo "${!var}"          # indirect: prints FW2025
```

Great for persite overrides without big case statements.

---

## 2.8  Quoting & WordSplitting Pitfalls

Wrong:

```bash
for f in $(ls *.cfg); do ...   # breaks on spaces/newlines
```

Right:

```bash
for f in ./*.cfg; do
  [[ -e $f ]] || break
  cp "$f" backup/
done
```

Using `$(ls)` is *UUOC* (useless use of`ls`) **and** breaks strictmode
scripts on >2GB glob lists.

---

## 2.9  RealWorld Example: IP  Firmware Lookup

```bash
declare -A target_fw
while IFS=, read -r ip role site fw; do
  [[ $ip == ip ]] && continue
  target_fw[$ip]=$fw
done < devices.csv

for ip in "${!target_fw[@]}"; do
  want=${target_fw[$ip]}
  have=$(ssh "$ip" 'cat /etc/version')
  [[ $want == $have ]] || flash_fw "$ip" "firmware/$want.bin"
done
```

*Associative array* enables O(1) lookup instead of O(N) nested greps.

---

## 2.10 Exercises

1. Write a function that takes an associative array **by reference** and
   returns the key with the highest failure count.  
2. Using parameter expansion only (no `cut`), extract the VLAN id from
   `gig0/0/1.4099`  `4099`.  
3. Convert a CSV inventory into three persite arrays (`den_nodes[]`, `kc[]`
   ) using `mapfile` and brace expansion.

---

> Next: **Chapter3  Functions, Libraries & Loop Patterns**.
