# NeoZygisk under late-load root: failure analysis and working sequence

> **Status:** solved on this device. The analysis is kept because the failure mode took ~8 scripted attempts to isolate, and every intermediate finding is a trap the next person will hit. A similar upstream report was closed as *not planned* (see [Cross-reference](#cross-reference-same-signature-different-stack)), so this is where the knowledge lives.
>
> **Applies to:** NeoZygisk v2.4 (`289-08080ef-release`) under KernelSU late-load root on the TCL C6K / T653T01. The mechanisms below don't look device-specific; the cross-reference shows the same signature on a completely different stack.

## Environment

| | |
| --- | --- |
| Device | TCL C6K (T653T01) Android TV, MediaTek MT5896 |
| ABI | 32-bit userspace / 64-bit kernel (`ro.zygote=zygote32`) |
| OS | Android 14, kernel `5.15.180-android14-11` |
| Security | SELinux enforcing, locked bootloader, green AVB |
| Root | KernelSU loaded **late**: post-boot and volatile (GhostLock chain → `ksud` late-load). No boot-time module stage and no init-triggered module scripts. |
| Manager | ReSukiSU (TCL build), KernelSU-family; since renamed BakaSU |
| Module | NeoZygisk v2.4 (`289-08080ef-release`) |

## TL;DR

Zygisk assumes it is set up **before** zygote starts. Under late-load root it is set up **after**, and three assumptions break at once:

1. **Nothing applies the module's `sepolicy.rule`.** On a normal install this happens during boot; late-load has no boot stage.
2. **Nothing starts the `zygiskd` + ptrace monitor pair.** On a normal install `post-fs-data.sh` is run during the boot-time module stage; under late-load it never runs unless you run it.
3. **Zygote is already running**, so the monitor needs a zygote *death event* to catch a fresh one.

All three are solvable, and the [working sequence](#the-working-sequence) below is reliable.

## The failure signature

`/data/adb/neozygisk/module.prop` during the failure:

```text
monitor:    ❌ stopped(zygote crashed)
zygote32:   ❓ unknown
daemon32:   😋 running
```

Injector log (`zygisk-ptrace32`, output detached to a file; excerpt):

```text
starting monitor mode...
error: failed to trace zygote, killing process 12385 to prevent system instability
preparing to trace PID: 12385
zygote restart requested...
```

This repeats for ~4 zygote generations, then the monitor parks in `epoll_wait`.

The monitor's safety logic is working as designed. When the ptrace attach fails, it SIGKILLs the half-traced zygote and retries on init's respawn, and after ~4 failures it parks. The framework recovers and the TV stays usable in this mode.

Earlier failure modes, before the fixes below, could hard-hang the framework. On this device that needs a power cycle, because network adb only comes up once the framework has finished booting.

## Root-cause chain

These are listed in the order they were discovered, not by importance. Each was an independent problem that had to be fixed before the next one became visible. **#1 is the actual cause of the injection failure**; the rest are launcher-side traps.

### 1. `sepolicy.rule` is never applied, and `apply` ≠ `patch`

`post-fs-data.sh` stages files and starts the monitor but never applies the module's `sepolicy.rule`. That is correct for boot-time installs, where the root implementation applies module sepolicy during boot. Under late-load nothing does it, the rules are absent at attach time, and the attach fails.

On this `ksud` build, `ksud sepolicy patch <file>` did **not** make the rules live, and `ksud sepolicy apply <file>` did. I haven't investigated the two subcommands' implementations, so treat this as observed behaviour on this build only.

The failure is not an ordinary permission denial. The type is **undefined** in the live policy:

```text
SELinux: Context u:object_r:zygisk_file:s0 is not valid (left unmapped)
```

This matters because `setenforce 0` cannot help: permissive mode skips denials, but it doesn't define missing types. See the cross-reference.

### 2. The monitor needs zygote death events, not service restarts

`setprop ctl.restart zygote` crashes the monitor every time (`stopped(zygote crashed)`). A plain `kill -9 $(pidof zygote)` is handled correctly, and the monitor injects init's respawn.

The working theory, from reading `loader/src/ptracer/main.cpp` and `ptracer.cpp`, is that a controlled `ctl.restart` teardown presents a different process state to the SEIZE/ATTACH state machine than a spontaneous death and respawn does. This is unconfirmed.

### 3. Concurrent monitors collide fatally

If a second monitor starts while one is parked or alive, the next injection attempt is guaranteed to fail, because two ptracers end up on one tracee. Always clear old instances first:

```sh
pkill -x zygisk-ptrace32; pkill -x zygiskd32
```

> **Footgun:** `pkill -f <name>` matches the *invoking shell's own command line* and kills your `su -c` mid-script. Under `set -e` this is a silent death. Use `pkill -x`.

### 4. The monitor must not hold a network pipe as stdout

If the monitor's stdout is an `adb shell` connection, it dies of SIGPIPE mid-injection when the framework restart drops network adb. Launch it detached:

```sh
nohup sh /data/adb/modules/zygisksu/post-fs-data.sh > /data/local/tmp/zygisk-init.log 2>&1 &
```

This also fixes launcher hangs. adb keeps a shell connection open until *every* process holding its stdio pipe exits, so any daemonised child of an `adb shell` blocks the caller indefinitely. The stock root chain hits the same class of bug at its handoff step and stalls for ~3 minutes until its `timeout 200` fires.

### 5. Emoji status markers break naive parsing

The status markers in `module.prop` (😋 / ❌ / ❓) are multi-byte UTF-8. Some `grep` pipelines classify the file as binary (`grep: (standard input): binary file matches`) and silently return nothing. Use `grep -a`.

Also, `not injected` contains `injected`, so exclude the negative before matching the positive.

### 6. `sys.boot_completed` never resets on a framework restart

A wait loop gated on `sys.boot_completed=1` passes instantly during the restart cycle. Use `init.svc.bootanim` going from `running` to `stopped` as the real framework-recovery signal.

## The working sequence

This is implemented in `tools/run-tcl-c855-v643-adb.sh` behind `GHOSTLOCK_ZYGISK=1`. It runs as one command per boot and has been verified end-to-end from a fresh boot.

```sh
# After the root chain completes and validates:

# 1. Make the module's SELinux rules live (apply, NOT patch)
ksud sepolicy apply /data/adb/modules/zygisksu/sepolicy.rule

# 2. Guarantee a single monitor
pkill -x zygisk-ptrace32; pkill -x zygiskd32

# 3. Start daemon + monitor, detached from the adb pipe
nohup sh /data/adb/modules/zygisksu/post-fs-data.sh > /data/local/tmp/zygisk-init.log 2>&1 &

# 4. Wait until module.prop shows the monitor tracing
until grep -a 'monitor:' /data/adb/neozygisk/module.prop | grep -q 'tracing'; do sleep 1; done

# 5. Death event (NOT ctl.restart)
kill -9 $(pidof zygote)

# 6. Wait for injection AND framework recovery (~60–90 s)
until grep -a 'zygote32:' /data/adb/neozygisk/module.prop | grep -v 'not injected' | grep -q 'injected' \
   && [ "$(getprop init.svc.bootanim)" = "stopped" ]; do sleep 2; done
```

To verify that injected code is actually running in spawned apps, not just flagged in `module.prop`:

```sh
adb shell monkey -p <any.app.package> 1
adb shell su -c "logcat -d | grep -a zygisk-core32"
# Lines like "zygisk-core32: [...] is on the denylist" mean Zygisk's core
# is executing inside the app process.
```

## Why the one early success worked

The single success before this analysis happened on the one boot where the sepolicy rules had been applied interactively with `apply` shortly before the monitor attached. Every scripted attempt used `patch`, which silently left the rules absent.

That is why the success looked unreproducible for ~8 attempts: the one differing variable was invisible.

## Cross-reference: same signature, different stack

A separate upstream issue closed as *not planned*, reports the identical failure triple under a very different setup:

- `monitor: stopped (zygote crashed)`
- `zygote64: unknown`
- the same `zygisk_file ... left unmapped` line

That stack was KernelSU-Next (SUSFS, inline hooks) with a **boot-time** install, on Android 16 / arm64 (Pixel 6). The reporter found that `setenforce 0` didn't help, which fits an undefined type: permissive mode can't grant anything for a type the loaded policy doesn't know.

Taken together with the late-load findings here, this **suggests** the gap isn't specific to the device, install mode or root implementation. NeoZygisk relies on something external to load its SELinux types and rules, has no self-application fallback, and fails the same way whenever that step doesn't happen. Under late-load, the cause is that there is no boot stage. On the KSU-Next stack the cause is unknown. Applying `ksud sepolicy apply` before the monitor starts would plausibly fix that case too, but that is untested on their hardware.

## Suggestions for NeoZygisk (not filed upstream)

Not filed: given that the similar report was closed as *not planned*, late-load support appears to be out of scope for the NeoZygisk project. Recorded here for anyone patching it locally.

- Detect late-load in `post-fs-data.sh` (KernelSU present, no boot-time module stage) and self-apply `sepolicy.rule` before starting the monitor.
- Make `ctl.restart`-style restarts non-fatal to the monitor, or document that injection must be triggered by a death event.
- Guard against concurrent monitor instances.
- Log the underlying errno at the `main.cpp:158` failure path. The real error currently goes to logcat via `PLOGE`, where launcher scripts reading stderr can't see it.
