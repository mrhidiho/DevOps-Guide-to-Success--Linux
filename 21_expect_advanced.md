
# Chapter21  Advanced Expect Scripting Patterns

Expect lets you drive interactive CLIs programmaticallya lifesaver on legacy
routers and switches. This chapter dives beyond basics: conditional prompts,
pagination handling, robust timeouts, and reusable Expect libraries.

---

## 21.1  Recap: Simple Expect Flow

```expect
spawn ssh admin@$ip
expect "Password:" { send "$env(PASS)\r" }
expect ">"         { send "show version\r" }
expect ">"         { send "exit\r" }
```

---

## 21.2  Handling Conditional Prompts

Some devices ask for **y/n** confirmation only when config is dirty.

```expect
expect {
  "Overwrite existing file?" {
      send "y\r"
      exp_continue
  }
  ">" {
      # main prompt reached
  }
}
```

`exp_continue` tells Expect to keep processing the original `expect` list.

---

## 21.3  Dealing with Pagination (More)

Three strategies:

### 1) Disable pager command

```expect
send "terminal length 0\r"
```

### 2) Autospace on More

```expect
expect {
  -re "--More--|<space>" {
      send " "
      exp_continue
  }
  ">" { }
}
```

### 3) Set Expect autoresponse globally

```expect
set timeout 2
match_max 100000
```

---

## 21.4  Timeout & Retry Logic

Function `run_cmd`:

```expect
proc run_cmd {cmd retry} {
  set tries 0
  while {$tries <= $retry} {
    send "$cmd\r"
    expect {
      -re "\r?(.+?)\r?\n" {
        return $expect_out(1,string)
      }
      timeout {
        incr tries
        if {$tries > $retry} { error "timeout on $cmd" }
      }
    }
  }
}
```

Use:

```expect
set output [run_cmd "show env" 2]
```

---

## 21.5  Modular Expect Library

`lib/cisco.exp`:

```expect
namespace eval cisco {
  proc connect {ip user pass} {
    spawn ssh $user@$ip
    expect "Password:" { send "$pass\r" }
    expect ">" { send "terminal length 0\r" }
    expect ">" { }
  }
  proc save_config {} {
    send "copy run start\r"
    expect {
      "Destination filename" { send "\r"; exp_continue }
      ">" { }
    }
  }
  proc close {} { send "exit\r"; expect eof }
}
```

Reuse:

```expect
source lib/cisco.exp
cisco::connect $ip $user $pass
cisco::save_config
cisco::close
```

---

## 21.6  Logging & Transcript Files

```expect
log_file -a "transcripts/$ip-$(date +%F).txt"
```

`-a` appends; every send/expect captured for audit.

---

## 21.7  Parallel Execution

Run Expect script in parallel using GNU parallel:

```bash
parallel -j20 ./flash_expect.exp ::: $(<old_switches.txt)
```

Ensure script exits cleanly (`expect eof`) so parallel doesnt hang.

---

## 21.8  Embedding Expect in Bash (HereDocs)

```bash
expect <<'EOS'
spawn telnet $IP
set timeout 5
expect "Username:" { send "admin" }
...
EOS
```

Allows storing all logic inside single rollout script.

---

## 21.9  Exercises

1. Write an Expect procedure that detects when a device drops into ROMMON
   versus normal mode and branches accordingly.  
2. Modify `run_cmd` to accept a regexp for success and another for error,
   returning different exit codes.  
3. Build a generic `expectlib` that supports Cisco, Juniper, and MikroTik
   with a switchcase on prompt regexes.

---

> Next optional expansion: **Bandwidth & Latency Testing in Bash**.
