
# Chapter88  Live Kernel Debug with KPatch+GDB skas Mode

Rebooting a lab router just to add a printk breaks SLAs. **KPatch** lets you
hotpatch kernel functions on the fly, while **GDB skas** (samekernel
address space) attaches to those patched symbols for singlestep debugging
without dropping traffic.

_Last updated: 2025-07-10_

---

## 88.1  Prerequisites

| Component | Version/Config |
|-----------|----------------|
| Kernel    | 6.1with `CONFIG_LIVEPATCH=y` |
| KPatch    | 1.2+ (`kpatch-build`, `kpatch load`) |
| GDB       | 13+ (`add-symbol-file`) |
| Debug symbols | `kernel-debuginfo` RPM/DEB |

---

## 88.2  Build a Minimal Hot Patch

Create `fix_bug.c`:

```c
#include <linux/livepatch.h>

static int lp_fix(int x)
{
    if (x < 0)         /* new guard */
        return -EINVAL;
    return x * 2;
}

static struct klp_func funcs[] = {
    { .old_name = "buggy_func", .new_func = lp_fix },
    { }
};

static struct klp_object objs[] = {
    { .name = NULL, .funcs = funcs },  /* vmlinux */
    { }
};

struct klp_patch patch = { .mod = THIS_MODULE, .objs = objs };

static int __init patch_init(void) { return klp_enable_patch(&patch); }
static void __exit patch_exit(void) { klp_disable_patch(&patch); }
module_init(patch_init)
module_exit(patch_exit)
MODULE_LICENSE("GPL");
```

Build:

```bash
kpatch-build -s /usr/src/kernels/$(uname -r) fix_bug.c -o kpatch_bug.ko
```

Load:

```bash
kpatch load kpatch_bug.ko
kpatch list        # verify "enabled"
```

---

## 88.3  Attach GDB in skas Mode

```bash
gdb vmlinux /proc/kcore
(gdb) add-symbol-file kpatch_bug.ko 0xffffffffc12f4000
(gdb) b lp_fix
(gdb) c
```

`/proc/modules` shows load address; or `dmesg | grep lp_fix`.

---

## 88.4  CI/CD Flow

```yaml
stages: [build, test, deploy]
build:
  script:
    - kpatch-build -s /kernel fix_bug.c -o kpatch_bug.ko
  artifacts: {paths: ["kpatch_bug.ko"]}

test:
  script:
    - kpatch load kpatch_bug.ko
    - ./selftest.sh
    - kpatch unload kpatch_bug.ko

deploy:
  when: manual
  script: kpatch load kpatch_bug.ko
```

Rollback: `kpatch unload kpatch_bug`.

---

## 88.5  Prometheus Metric

```bash
count=$(kpatch list | grep -c enabled)
echo "livepatch_enabled_total {} $count" \
 > /var/lib/node_exporter/text/kpatch.prom
```

Alert if enabled patches=0.

---

## 88.6  RealWorld Example

Patched `mlx5e_rx_drop` race in production; attached GDB skas, stepped ISR
while 40Gb traffic flowed; pps drop <0.1%.

---

## 88.7  Exercises

1. Write a hot patch replacing unsafe `memcpy` with `memmove` in
   `drivers/net/foo.c`, measure PPS.  
2. Use `perf probe -x kpatch_bug.ko lp_fix` to add uprobes after patch load.  
3. Automate GDB `add-symbol-file` via `.gdbinit` hook whenever new patch
   appears in `/sys/kernel/livepatch`.

---
