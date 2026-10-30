# PS4 14.00 BD-JB — Comprehensive Repository Analysis

> **Document scope:** A complete technical analysis of `OptiTronOffical/PS4-14.00-BD-JB`, covering the disc file system, the signed JAR, the four helper classes, the 40-command exploit chain, the kernel stage, and the trust/safety model. Where a component's source is not published, the analysis states that it is inferred from the surrounding code, hash pins, and command semantics.

---

## 1. Executive Summary

`PS4-14.00-BD-JB` is a Blu-ray Disc image that delivers a **fully offline, signed, one-shot BD-J (Blu-ray Java) exploit chain** for PlayStation 4 firmware **14.00**. It does not use a network server, does not require a WebKit entry point, and does not rely on any user interaction beyond a single controller button press.

The disc’s Java application (`org.bdj.InitXlet`) performs a **multi-layered SHA-256 preflight** of every asset it will use — four compiled helper classes, a 562,240-byte HEN image, seven metadata binaries, and forty command templates — all of which live inside the signed JAR or in `/x/offline/` on the disc. Only after every hash matches does it allow the user to press X, at which point a **40-command, result-gated state machine** (`LocalResultLedger` + `OfflineRenderer`) drives a binary reflection interpreter (`Boot`) through a chain that:

1. Escapes the Java sandbox via `sun.misc.Unsafe` and `ClassLoader$NativeLibrary` (`AXZXkk`).
2. Corrupts a Java2D `int[]` length to `Integer.MAX_VALUE` via a `getRGB` type confusion (`EkIKZR`).
3. Scans the JVM heap for four random marker words and derives heap pointers (`J196rR`).
4. Uses the resulting native call bridge to perform a **144-socket spray + kqueue/AIO race** that yields **arbitrary kernel read/write** (`Hx1400` stages `A`–`F`).
5. Patches `sysent[11].sy_call` to a custom `kexec` handler and loads a **562,240-byte HEN** (`Hx1400` stages `G`–`H`).
6. Verifies the result against a fixed set of success strings, then quarantines every corrupted kernel structure so the console can be cleanly cold-rebooted.

The repository is a **rebuild of a previously leaked chain** (the README explicitly says “THIS IS STOLEN CODE”). The most significant technical components — the AIO/kqueue kernel race and the HEN loader — are the hardest part, and their failure modes are documented in the on-disc `NOTICE.txt`: **no automatic retry, one attempt per boot, cold reboot after any failure, and the HEN success message must be independently verified.**

---

## 2. Repository Structure

The repository root contains:

```
PS4-14.00-BD-JB/
├── README.md              # Overview, credits, warning
├── JAR-ANALYSIS.md        # Analysis of the .java sources inside the JAR
├── class-analysis.md      # Analysis of the four .class helpers in /x/offline/
├── hen.iso                # Pre-built Blu-ray ISO (root of repo)
└── src/                   # Blu-ray disc file system source
    ├── BDMV/
    │   ├── AUXDATA/
    │   ├── BACKUP/
    │   ├── BDJO/                # BD-J Object (entry point pointer)
    │   ├── CLIPINF/
    │   ├── JAR/
    │   │   └── 00000.jar        # Signed JAR: org/bdj/*.class + /bonk/*
    │   ├── PLAYLIST/
    │   ├── STREAM/
    │   ├── MovieObject.bdmv
    │   └── index.bdmv           # Top-level BD-J index
    ├── CERTIFICATE/
    │   ├── BACKUP/
    │   ├── app.discroot.crt
    │   ├── bu.discroot.crt
    │   └── id.bdmv
    ├── x/
    │   └── offline/
    │       ├── AXZXkk.class
    │       ├── EkIKZR.class
    │       ├── Hx1400.class
    │       └── J196rR.class
    └── NOTICE.txt               # Safety-critical warnings
```

### 2.1 The disc file system (`/src`)

`/src` is a **Blu-ray Disc File System (BDFS)** layout. It is not a source tree — it is the exact byte-for-byte structure that must appear on the disc, or on an ISO mounted as a disc. The three critical subtrees are:

| Subtree | Purpose |
|---|---|
| `BDMV/` | Blu-ray navigation. `index.bdmv` and `MovieObject.bdmv` tell the player which BD-J object to run automatically. `BDJO/` contains the BD-J object descriptor that names the Xlet class. `JAR/00000.jar` is the signed Java application. |
| `CERTIFICATE/` | Disc root certificates (`app.discroot.crt`, `bu.discroot.crt`) and `id.bdmv`. The PS4 uses these to decide whether the disc is a valid Blu-ray video and whether its Java is allowed to run. Without these, the BD-J runtime refuses to launch the Xlet. |
| `x/offline/` | Four compiled `.class` files, outside the JAR, referenced by the chain via `DVBClassLoader` fallback. |

The `x/offline/` directory is **not** a standard Blu-ray location. Placing classes there is the exploit’s only “extension” of the Blu-ray format — and it is why `InitXlet` has to locate itself inside the JAR and compute `discRoot` from the `jar:file:` URL.

### 2.2 The signed JAR (`BDMV/JAR/00000.jar`)

The JAR contains:

```
org/bdj/
  ├── InitXlet.class         # Xlet entry point
  ├── Boot.class             # Binary reflection interpreter
  ├── OfflineRenderer.class  # Template verifier + relocator
  ├── LocalResultLedger.class# Command state machine
  ├── Screen.class           # AWT text display
  └── Cfg.class              # Build/server constants

bonk/
  ├── hen.bin                # 562,240-byte HEN (SHA-256 29ec095f…)
  └── local/
      ├── rules.bin          # 205 relocation rules
      ├── shapes.bin         # 47 result-shape declarations
      ├── guards.bin         # 10 native deltas + 3 code-window hashes
      ├── schedule.bin       # 40-command schedule
      ├── template-hashes.bin# 40 SHA-256 pins
      ├── marker-sites.bin   # 4 marker-patch offsets
      ├── write-sites.bin    # 81 write-destination checks
      └── templates/
          ├── cmd-0000.bin … cmd-0039.bin
```

This is the entire **payload tree**. The four helper classes in `/x/offline/` are the only things outside the JAR.

### 2.3 The four helper classes (`/x/offline/`)

| File | SHA-256 |
|---|---|
| `AXZXkk.class` | `2362afc9efa07a688169b9121d9e5f8bdacbead58f87cafd23fdddffea58b429` |
| `EkIKZR.class` | `df30bfbd25c3cdbe67965d907d575efb562738cf29c258895a698ff2ef525137` |
| `J196rR.class` | `f59953027867d15d1434179991feeeb94bf274422f742e739067fd65400a264d` |
| `Hx1400.class` | `e597710078e9afab5b24f4c22ea31fe7e5a5643b602b06c2866273b61c9c57e9` |

Only `Hx1400` has published source (`Hx1400.java`). The other three are described in `class-analysis.md` and in this document from the evidence of `InitXlet`, `OfflineRenderer`, `Boot`, and `Hx1400`.

---

## 3. The Java Application Layer

This section analyzes the six `.java` files that make up the JAR’s `org/bdj/` package. They are, in execution order:

`InitXlet` → `OfflineRenderer` / `LocalResultLedger` / `Boot` → `Screen` / `Cfg`.

### 3.1 `InitXlet` — Entry point and preflight

`InitXlet` implements `javax.tv.xlet.Xlet`. The BD-J runtime calls `initXlet`, `startXlet`, `pauseXlet`, and `destroyXlet`. The class spawns a **single preflight thread** in `startXlet` and refuses to spawn more if one is already running.

#### Preflight steps

The preflight thread is the `run()` method. It executes in strict order:

1. **Locate itself.** `Class.forName("org.bdj.InitXlet").getResource("InitXlet.class")` must return a `jar:file:` URL containing `/BDMV/JAR/`. If not, preflight fails immediately.

2. **Compute the disc root.** `discRoot = url.substring(4, url.indexOf("/BDMV/JAR/"))`. Must start with `file:`.

3. **Hash the four helper classes.** For each of `J196rR`, `AXZXkk`, `EkIKZR`, `Hx1400`, call `checkHash(new URL(discRoot + "/x/offline/…"), sha256, 1, 100000)`. The bounds are 1–100,000 bytes and the hash is SHA-256.

4. **Verify `hen.bin`.** Locate `/bonk/hen.bin` via `getResource`. Require a `jar:file:` URL, exactly **562,240 bytes**, and SHA-256 `29ec095fbede651e29954de43411599093138791e62105aaee6fdced58edec06`.

5. **Verify the seven metadata binaries.** For each of `rules.bin`, `shapes.bin`, `guards.bin`, `schedule.bin`, `template-hashes.bin`, `marker-sites.bin`, `write-sites.bin`: read with `readBounded(url, 8192)`, compare length against the expected value in `META[][]`, and compare SHA-256 against the expected value.

6. **Parse metadata** into `Rule[]`, `Shape[]`, `Guards`, `ScheduleEntry[]`, `TemplatePin[]`, `MarkerPlan`, `WriteSite[]`.

7. **Cross-check schedule vs pins.** `pins.length == schedule.length == 40` and `pins[i].command == schedule[i].command` for all `i`.

8. **Verify the 40 templates.** For each `TemplatePin`, load `/bonk/local/templates/cmd-NNNN.bin` (zero-padded), require it to be a `jar:file:`, require exact length, and require SHA-256 to match.

9. **Print success.** “LOCAL PREFLIGHT PASS. Press X once to run.” and “One attempt only; restart after any failure.”

Any failure prints `PREFLIGHT FAIL: <class>: <message>` and leaves `ready = false`. The key listener then refuses to do anything.

#### The key handler

```java
if (keyCode == 10 || keyCode == 32 || keyCode == 88) {   // Enter, Space, X
    synchronized (this) {
        if (stopped || !ready || attempted) return;
        attempted = true;
    }
    new Thread(new Attempt(), "Bonk HEN one-shot local run").start();
}
```

The `attempted` flag guarantees **one attempt per boot**. This is a deliberate design decision that reflects the on-disc warning: after any kernel-stage failure, the only safe recovery is a cold reboot.

#### The `Attempt` driver

`Attempt.run()` is the actual exploit driver. It sets:

```java
Boot.EP  = InitXlet.this.discRoot;
Boot.SID = "offline";
Boot.PAR = InitXlet.class.getClassLoader();
Boot.SK  = null;
```

Then it generates four random marker words via `OfflineRenderer.freshMarkerWords()`, constructs a `LocalResultLedger`, and loops over the 40-command schedule. For each command it:

1. Gets the command ID from `ledger.currentCommand()`.
2. Loads the template with `loadTemplate(cmd, pins[i].length)`.
3. Renders it with `ledger.renderCurrent(tmpl)` — which re-hashes the template, runs all shape and marker checks, applies the 205 relocation rules, patches marker words (command 2), and verifies the 81 write-site destinations.
4. If command is 32, calls `patchLocalHenUrl(rendered, henUrl)` to replace the sentinel HEN URL.
5. If command is 44, polls up to 10 times over 45 seconds while `Hx1400` is still running.
6. Calls `Boot.exec(rendered)` and feeds the result into `ledger.acceptResult()` or `ledger.finishTerminal()`.
7. Prints a log line for commands 9, 27, 32, 44, 46.

At the end it calls `ledger.complete()` — if false, the run is rejected — and prints “Hx reports HEN start. Verify menu and kernel R/W separately.” on success.

On failure, `Attempt` prints a diagnostic, optionally calls `showEarlyStatus()` (commands 4/5/6) or `showHxDiagnostic()` (command 44), and prints “No retry in this boot. Cold reboot before another attempt.”

#### The HEN URL patch

`patchLocalHenUrl(byte[], String)` is the **only runtime mutation of a verified template**. It replaces the sentinel `file:/__bonk___offline_missing__/hen.bin` (45 bytes) with the actual JAR URL `jar:file:...!/bonk/hen.bin`. It does this by:

1. Scanning `data[13..]` for the sentinel bytes.
2. Confirming uniqueness.
3. Confirming the exact framing: `0x05` opcode, 8-byte `reg = 45`, 4-byte `len = 45`.
4. Copying the new URL in and updating the length field.
5. Rejecting any new length > 65,535.

The sentinel is never actually fetched; it must exist so the template hash matches, then it is patched away.

### 3.2 `Boot` — Binary reflection interpreter

`Boot` is the most substantial of the six classes. It is a **pure-Java binary command interpreter** whose opcodes execute reflection: `Class.forName`, `getDeclaredMethod`, `getDeclaredField`, `invoke`, `get`, `set`, `Array.newInstance`, `Array.get`, `Array.set`, integer arithmetic, and so on.

#### Registers

```java
static long[]   R = new long[128];    // scalar registers
static Object[] O = new Object[64];   // object registers
static Object[] M = new Object[64];   // method/field registers
static String[] S = new String[64];   // string registers
static byte[][] Z = new byte[32][];   // byte-array registers
```

Registers are indexed with `& 63` (or `& 127` for `R`). This is why templates can use register IDs that exceed the array bounds without crashing — they are masked.

#### Opcodes (30 in total)

The full opcode set, extracted from the `exec()` switch:

| Opcode | Operation |
|---|---|
| 1 | Halt with a return code |
| 2 | Print a scalar to the log |
| 3 | `Thread.sleep(ms)` |
| 4 | Busy-loop for N microseconds |
| 5 | Set `S[reg]` from UTF-8 bytes |
| 6 | Set `Z[reg]` from bytes |
| 7 | `O[reg] = Class.forName(S[name], true, PAR)` |
| 8 | `O[reg] = new L(bytes, name).loadClass(name)` with DVB fallback |
| 9 | Construct an object with `newInstance` |
| 10 | Resolve a `Method` |
| 11 | Resolve a `Field` |
| 12 | `R[reg] = (long) field.get(obj)` |
| 13 | `field.set(obj, arg)` |
| 14 | `R[reg] = (long) method.invoke(obj, args)` |
| 15 | `O[reg] = Array.newInstance(cls, n)` |
| 16 | `R[reg] = Array.get(O[arr], i)` |
| 17 | `Array.set(O[arr], i, arg)` |
| 18 | `R[reg] = Array.getLength(O[arr])` |
| 19 | `R[reg] = num(arg)` |
| 20 | Arithmetic (`+ - & \| ^ << >> * / == < >`) |
| 21 | Write a scalar to the output stream |
| 22 | Serialize `O[reg]` (byte[], int[], long[]) |
| 23 | Serialize `Z[reg]` |
| 24 | Conditional skip |
| 25 | `System.gc()` × 2 |
| 26 | `O[dst] = O[src]` |
| 27 | `O[reg] = arg` |
| 28 | `O[reg] = new Object[]{O[src]}` |
| 29 | Spawn a thread calling `M[reg].invoke(obj, args)` |
| 30 | `O[reg] = M[method]` |

#### The result frame

Every `exec()` returns:

```
long    status           // 0 = OK, -1 = exception, -2989 = unknown opcode
int     instruction      // opcode index reached
int     nScalars
long[]  scalars
int     nVectors
for each: int len; byte[] data
int     nNotes
for each: int len; byte[] note
```

This is exactly the shape `OfflineRenderer.frame()` parses and `verifyNewResult()` checks against `Shape` declarations.

#### The remote path

`Boot.io(String, byte[])` is the HTTP transport used by the **remote** (network) version of the exploit. It POSTs to `EP + path`, sets header `X-Ticket: c6.<seq>.<hash>`, XOR-encrypts the body, and reads/decrypts the response. It is **not called in offline mode** — `InitXlet.Attempt.run()` drives the chain from pre-verified templates.

The ticket hash and XOR keystream are derived from `CKB` (the 16-byte constant `942a76d6…`, hex of the same value) plus the session key `SK`. In offline mode `SK = null`, so the ticket becomes `mix(CKB, buf)` without a session key. This is **obfuscation, not cryptography** — it only prevents casual sniffing of the remote version.

#### The custom loader `L`

```java
static class L extends ClassLoader {
    L(byte[] bytes, String name) { super(Boot.PAR); ... }
    protected Class loadClass(String n, boolean resolve) {
        if (n.equals(this.n))
            return defineClass(n, this.c, 0, this.c.length);
        return super.loadClass(n, resolve);
    }
}
```

Used by opcode 8 to load helper classes from raw byte arrays. In offline mode the DVB fallback loads them from `file:<discRoot>/x/offline/`.

### 3.3 `OfflineRenderer` — Template verifier and relocator

`OfflineRenderer` is the security core of the chain. It is responsible for turning a **SHA-256-pinned template** into a runnable command byte stream by applying relocation rules and verifying every write destination.

#### The seven metadata binaries

All seven share the header format:

```
magic  4 bytes  (FMR1, FMS1, FMG1, FMT1, FMC1, FMM1, FMW1)
version 2 bytes (== 1)
payload ...
```

`metadata(byte[], String)` rejects any file whose magic or version does not match. `rules()`, `shapes()`, `guards()`, `templatePins()`, `schedule()`, `markerPlan()`, and `writeSites()` then parse the payload and **reject any trailing bytes** — so a file cannot be a valid prefix of a longer one.

| File | Struct | Count | Fields |
|---|---|---|---|
| `rules.bin` | `Rule` | 205 | `(command, instruction, offset, request, kind, slot, high, low, addend)` |
| `shapes.bin` | `Shape` | 1–47 | `(request, status, instruction, scalarCount, vectorLengths[])` |
| `guards.bin` | `Guards` | 10 + 3 | 10 native deltas, 3 × 32-byte code-window hashes |
| `schedule.bin` | `ScheduleEntry` | 40 | `(command, nextResultRequest)` |
| `template-hashes.bin` | `TemplatePin` | 40 | `(command, length, sha256[32])` |
| `marker-sites.bin` | `MarkerPlan` | 4 | 4 offsets |
| `write-sites.bin` | `WriteSite` | 81 | `(command, instruction, offset)` |

Every parser enforces strict structural invariants. `schedule()` requires the commands to be strictly increasing, the last entry to be `(46, 65535)`, and the final `nextResultRequest` to be 65535. `writeSites()` requires the sites to be ordered and unique, and each site to correspond to exactly one `Rule` (i.e., it must be a legitimate relocation).

#### `renderCandidate()` — The hot path

`renderCandidate(command, template, frames, rules, shapes, guards, pins, markerPlan, markerWords, writeSites)` is the function that produces the next command’s bytes. It performs these steps in order:

1. **Template identity.** Find the `TemplatePin` for `command`, check `template.length == pin.length`, and `hash(template) == pin.sha256`. This is the **second** hash check (the first was in preflight), and it runs on **every render**.

2. **Marker check** (`command >= 7`). Verify the prior marker scan (result 7) and marker echo (result 8) match `markerWords`. This is `verifyPriorMarkerResults()`.

3. **Shape check** (all commands). For every `Shape` whose `request <= command`, verify the stored frame matches the shape. This is `verifyShapes()`.

4. **Native layout check** (`command >= 12`). Verify the 10 native symbol deltas in `frames[12]` match `guards.nativeDeltas`. This ensures the JVM’s native layout has not changed.

5. **Alias check** (`command >= 17`). Verify scalars 3, 5, 7, 9, 11 in `frames[16]` are all equal to scalar 1. This is `verifyAliases()`.

6. **Code-window check** (`command >= 27`). Verify the three 16-byte code windows in `frames[27]` match `guards.windowHashes[0..2]`. This is `verifyCodeWindows()`.

7. **Relocate.** For each `Rule` matching `command`, verify the 8 bytes at `offset` are zero (rejects already-patched templates), compute the value, add the addend, and write big-endian.

8. **Marker patch** (`command == 2`). Write `markerWords[i]` at `markerPlan.offsets[i]`, after verifying the surrounding `(17, ?, 3, i, 0,0,0,0)` structure.

9. **Write-site check.** Every `WriteSite` in `command` must have a pointer-shaped value in `[0x100000000, 0x7FFFFFFFFFFFFFF7]` after relocation.

10. Return the patched byte array.

The zero-byte check in step 7 is what makes the chain **safe against template corruption**: no relocation can overwrite non-zero bytes, so a malicious or truncated template cannot smuggle in arbitrary pointers.

#### `verifyHenReport()` — The HEN success oracle

After command 44 runs, `verifyHenReport(frames[46])` checks the final frame against a fixed set of success strings. The summary (vector 0) must contain:

- `HEN14V`
- `APPLIED=1`
- `UID=0 uid2=0`
- `*** ROOT ***`

The log (vector 1) must contain:

- `K 6/6 sites now hold the PATCHED byte`
- `K sysent[11].sy_narg=2`
- `K *** 1400.bin APPLIED - syscall 11 IS NOW kexec ***`
- `G2 sysfilt_ops restore=true`
- `H4 fetched 562240/562240 B`
- `H5c launching`
- `H7 pthread_create rc=0 handle=`
- `H7 *** hen.bin RUNNING`

And it must **not** contain:

- `STOP`, `no pair`, `blind writes`, `APPLIED=0` (summary)
- `K NOT fully applied`, `safeDown threw` (log)

In addition, if both `K sysent[11].sy_call=` and `APPLIED=true` appear on the same line, and both `L set to ` and `ok=true` appear on the same line, the log is considered “kernel + ShellCore readback present.” Otherwise the report is rejected. `lineContainsBoth()` and `contains()` do byte-level substring search on the raw vectors, so they work even with non-ASCII content.

#### `isPendingHenResult()` — The retry oracle

While `Hx1400` is running, it produces a frame with:

```
status      = 0
instruction = 7
scalars     = []
vectors     = [ "not run" (7 bytes), LOGS() (≤128 KiB) ]
```

`isPendingHenResult()` recognizes this exact shape and returns `true`. `InitXlet.Attempt` uses it to poll up to 10 times over 45 seconds.

#### `freshMarkerWords()`

Generates four random `int`s via `SecureRandom`, each non-zero, not `-1`, not `-2`, and all distinct. `-1` and `-2` are reserved because `Hx1400` uses them as sentinel values in `showHxDiagnostic`. Command 2 writes these into a template, and command 7 scans for them — this is how the exploit confirms the JVM heap layout before touching anything critical.

#### `earlyIndexes()` — The pointer derivation

This is the single most important function in the chain. It takes the 1024-byte echo from command 8 and extracts four heap indices:

```java
long v13_14 = qword(vec, 13, 14);
long v11_12 = qword(vec, 11, 12);
long v29_28 = qword(vec, 29, 28);
long base   = v13_14;
long idxA   = (v11_12 - base - 24) / 4;
long idxB   = (v29_28 - base - 8)  / 4;
return new long[]{ idxA, idxA + 40, idxA + 41, idxB };
```

These four values become the **relocation sources** for `Rule.kind == 2` (early index kind). This is why `J196rR`’s marker scan is the critical gate: if it produces wrong indices, every pointer downstream is wrong.

### 3.4 `LocalResultLedger` — The state machine

`LocalResultLedger` enforces the strict command ordering. Its state is:

- `at` — index into the schedule
- `awaitingResult` — true between `renderCurrent()` and `acceptResult()`
- `failed` — true on any error
- `successWindowConfirmed` — set when result 27 arrives
- `frames[48]` — stored result frames
- `pendingInstructionCount` — the instruction count from the last rendered template

Its API is:

- `currentCommand()` — returns `schedule[at].command`
- `renderCurrent(template)` — calls `OfflineRenderer.renderCandidate()`, sets `awaitingResult = true`
- `acceptResult(frameBytes)` — parses, verifies, stores
- `finishTerminal(frameBytes)` — verifies the terminal frame is empty
- `complete()` — true iff all 40 commands processed without failure

The `successWindowConfirmed` flag means that **any command with ID ≥ 28 requires result 27 to have arrived** — this prevents the exploit from firing the kernel stage if the code-window signature check has not passed.

`acceptResult()` rejects:

- Duplicate result requests (`frames[var2] != null`)
- Wrong shape (`verifyNewResult` throws)
- Wrong marker echo (`verifyMarkerNewResult` throws)

The `frames` array has 48 slots, so result IDs must be `< 48`. The schedule’s last entry must be `(46, 65535)`.

### 3.5 `Screen` and `Cfg`

`Screen` is a `java.awt.Container` that maintains a scrollback of up to 46 status lines and paints them. It has `println`, `printStackTrace`, `clear`, and a re-entrant-safe `paint()`. It is the visible UI during preflight and execution.

`Cfg` holds the remote-version constants:

```java
SERVER = "tester.f5rvoice.com";
PORT   = 1992;
BUILD  = "6";
CID    = "c6";
CKEY   = "942a76d6946a34ce476bdabb09989f4c";
```

`CKEY` is the hex of `CKB`, a 16-byte constant hardcoded in both `Cfg` and `Boot`. It is used only by the remote version’s ticket and XOR obfuscation. In offline mode, `Boot.SK = null` so `CKEY` becomes a pure hash seed.

---

## 4. The Helper Classes

This section describes the four compiled `.class` files in `/x/offline/`. Only `Hx1400` has published source. The other three are reconstructed from the surrounding code, the `verifyEarlyStatus()` checks in `OfflineRenderer`, and the primitive usage in `Hx1400`.

### 4.1 `AXZXkk` — Native access bootstrap

**SHA-256:** `2362afc9efa07a688169b9121d9e5f8bdacbead58f87cafd23fdddffea58b429`

**Command:** 4

**Evidence:** `verifyEarlyStatus()` for command 4:

```java
if (frame.vectors.length != 2
    || frame.vectors[0].length != 4
    || readInt(frame.vectors[0], 0) != 0)
    throw new IOException("helper setup failed");
```

So command 4 returns `{status=0}`.

**Reconstruction:** Based on `Hx1400.java`, which uses the same technique, `AXZXkk` performs the `sun.misc.Unsafe` + `ClassLoader$NativeLibrary.findEntry` bootstrap:

```java
Field f = Class.forName("sun.misc.Unsafe").getDeclaredField("theUnsafe");
f.setAccessible(true);
Unsafe u = (Unsafe) f.get(null);

long base = u.getLong(new Object[]{DM}, 24L);   // DM = new int[]{1}
rAK = u.getLong(base + 8L);

Class<?> lib = Class.forName("java.lang.ClassLoader$NativeLibrary");
Method find = lib.getDeclaredMethod("find", String.class);
find.setAccessible(true);
Field handle = lib.getDeclaredField("handle");
handle.setAccessible(true);

Constructor<?> ctor = lib.getDeclaredConstructors()[0];
ctor.setAccessible(true);
Object inst = ctor.newInstance(Class.forName("Hx1400"), "x", true);
handle.setLong(inst, 8193L);
long sigsetjmp = (Long) find.invoke(inst, "sigsetjmp");
long setcontext = (Long) find.invoke(inst, "__Ux86_64_setcontext");
long getpid     = (Long) find.invoke(inst, "getpid");
// ... etc
```

`AXZXkk` is the class that wraps this into a `find(String)` / `call(long, long...)` API and exposes the native call trampoline. It is loaded first because everything else — the `getRGB` corruption, the marker scan, the `Hx1400` bridge — needs it.

### 4.2 `EkIKZR` — Image and `getRGB` array-length corruption

**SHA-256:** `df30bfbd25c3cdbe67965d907d575efb562738cf29c258895a698ff2ef525137`

**Commands:** 5 and 6

**Evidence:** `verifyEarlyStatus()`:

```java
if (command == 5 || command == 6) {
    int expected = (command == 5) ? 4 : Integer.MAX_VALUE;
    if (frame.vectors.length != 1
        || frame.vectors[0].length != 8
        || readInt(frame.vectors[0], 0) != 0
        || readInt(frame.vectors[0], 4) != expected)
        throw new IOException(command == 5
            ? "helper image setup failed"
            : "getRGB array-length corruption failed");
}
```

So command 5 returns `{status=0, length=4}` and command 6 returns `{status=0, length=0x7FFFFFFF}`.

**Reconstruction:** `EkIKZR` packages a Java2D `getRGB` type confusion that overwrites the `length` field of an `int[]` array header. After command 6, `EkIKZR` exposes a method (likely `int[] getOobArray()`) returning an `int[]` whose declared length is `Integer.MAX_VALUE` but whose backing memory is 16 bytes. Reading past index 3 reads adjacent JVM heap; writing past index 3 writes it.

This is the **userland OOB primitive**. It is used by `J196rR` to scan the heap.

### 4.3 `J196rR` — Marker scan and memory pivot

**SHA-256:** `f59953027867d15d1434179991feeeb94bf274422f742e739067fd65400a264d`

**Commands:** 7 and 8 (and indirectly 9–27 via `earlyIndexes()`)

**Evidence:** `verifyMarkerNewResult()`:

```java
if (request == 7) {
    if (frame.vectors.length != 1 || frame.vectors[0].length != 4)
        throw new IOException("marker scan failed");
    int idx = readInt(frame.vectors[0], 0);
    if (idx < 4 || idx > 16383)
        throw new IOException("marker scan outside valid search range");
}

if (request == 8) {
    if (frame.vectors.length != 1 || frame.vectors[0].length != 1024)
        throw new IOException("marker echo vector changed");
    for (int i = 0; i < 4; i++)
        if (readInt(frame.vectors[0], i * 4) != markerWords[i])
            throw new IOException("marker echo mismatch");
}
```

So command 7 returns the word-index of the first marker found (in `[4, 16383]`), and command 8 returns a 1024-byte echo whose first 4 `int`s are the marker words.

**Reconstruction:** `J196rR` scans the OOB `int[]` from `EkIKZR` for the four marker words. It then returns:

- Command 7: the word index of the first marker.
- Command 8: a 1024-byte snapshot starting at that offset, which contains the known pointers needed by `earlyIndexes()`.

`J196rR` also provides the native-call trampoline that `AXZXkk` set up, which commands 9–27 use to call `dlsym`, `getpid`, `socket`, `sendto`, `fstat`, `sysctl`, `mmap`, `kqueue`, `kevent`, etc.

### 4.4 `Hx1400` — Kernel exploit and HEN loader

**SHA-256:** `e597710078e9afab5b24f4c22ea31fe7e5a5643b602b06c2866273b61c9c57e9`

**Command:** 44

`Hx1400` has published source. Its interface to the offline loader is:

- It reads six static fields set by earlier commands: `P` (`long[31]`), `PG` (`long[]`), `PGL` (`int[]`), `PGN` (`int[]`), `PGW` (`int[]`), `PKB` (`String`), and `HENURL` (`String`).
- `tableFault()` validates the fields. If any is missing or the wrong length, `run()` returns `"NO TABLE: …"` immediately.
- `run()` calls `probe()` and stores the result in `result`.
- `probe()` is the full 15-stage kernel exploit (see §5).
- While running, it produces a pending frame with `instruction = 7` and `vector[0] = "not run"`.

---

## 5. The Kernel Exploit (Hx1400 `probe()`)

`Hx1400.probe()` is a 15-stage kernel exploit. It is documented in detail in the earlier analysis; this section summarizes each stage and its role.

### 5.1 Stage A — Pin a core and enumerate `/dev/dce`

The exploit pins itself to a core via `cpuset` (syscall 488) and opens `/dev/dce`. It fstats the fd to learn its inode, then iterates fds 3–37 to find every fd whose `st_ino` matches. These are the app’s own `/dev/dce` fds. It then plugs each matching fd with an inert device (`/dev/null`, `/dev/zero`, `/dev/urandom`, `/dev/random`) so the subsequent spray cannot reuse them.

### 5.2 Stage B — Spray 144 sockets

For `NSPRAY = 144`, it opens a UDP socket (syscall 97) and calls `setsockopt` (105) with a 248-byte payload that begins with the magic `MLO = 1128481603` and `MHI = 1414812756`. It records the highest fd reached.

### 5.3 Stage C — kqueue/AIO race

The race is the core of the exploit. It:

1. Creates a `kqueue` (syscall 362).
2. For each spray socket, re-packs the 248-byte `setsockopt` payload.
3. For each candidate index in `CANDS[30]`, arms the socket with `EV_ADD`, filter `-18` (`FILT`), and calls `kevent` (363).
4. Opens a new `/dev/dce` fd, which gives a pointer-shaped value `(fd & 0xFFFF) << 16 | 39002`.
5. Sweeps the other sockets to find one whose `setsockopt` value has been overwritten with this pointer — this is the **knote leak**.
6. If exactly one match is found, it re-arms with `EV_ADD`, filter `-18` but with `data = 2` — this arms the fake `sysfilt_ops` slot.
7. It re-runs for up to `ROUNDS = 6` rounds, capped at `MAXARM = 24` arms per round.

Each **miss** writes 8 bytes into an unknown kernel zone — this is the “blind write” risk documented in `NOTICE.txt`. The exploit **cannot repair** these writes; a cold reboot is required.

### 5.4 Stage D — mmap the fake page

Once a pair is held, it computes:

```
fake = (dce & 0xFFFF) << 16 | 38912
page = fake & ~0x3FFF
```

and maps `page` with `mmap(page, 16384, PROT_READ|WRITE|EXEC, MAP_FIXED|MAP_ANON, -1, 0)`. It then joins 96 victims, each of which is opened with `EV_ADD` filter `-18` at the fake page’s address.

### 5.5 Stage E — Build a fake kqueue tree

It constructs a three-node red-black tree inside the fake page:

- Root at `fake + 2304`
- Left child at `fake + 2688`
- Right child at `fake + 2816`

Each node has a `knote` structure with pointers to the others. It then arms the victim socket to point at this tree.

### 5.6 Stage F — Rotate in a fake `sysfilt_ops`

It rotates the tree so that `fake + 2304` is the slot, and writes a fake `sysfilt_ops` structure whose function pointers are the `R_*` register values from `P`:

| Field | Symbol | Purpose |
|---|---|---|
| `isfd` | (0) | 0 |
| `attach` | `R_RET0` | No-op |
| `touch` | `R_TOUCH` | The pivot |
| `f_event` | `R_CONT` | The continuation |
| `f_attach` | `R_UIDPAIR` | UID-zeroing |

The slot is **never restored to its original value** — it stays hooked so that the exploit can re-enter ring 0 as many times as needed.

### 5.7 Stage G — ROP chain

The fake `sysfilt_ops` `touch` pointer is `R_TOUCH`, which is invoked by `kevent(EV_ADD, -23, ident)` on a second kqueue. Each invocation executes a small ROP chain built on `gSTACK`:

1. `R_PRDI` / `gCTX` / `R_SAVE` — save context
2. `R_PRAX` / `MARKV` / `R_PRDI` / `gMARK` / `R_STRDIRAX` — write a sentinel so the exploit knows the chain ran
3. `R_PRAX` / `gCTX + 16` / `R_LDRAX` — load the first argument
4. `R_PRDX` / `-120` / `R_ADDRAX` — add an offset
5. `R_PRDI` / `gCTX + 8` / `R_STRDIRAX` — write the result
6. `R_PRAX` / `R_CONT` — continue
7. `R_PRDI` / `gCTX + 56` / `R_STRDIRAX` — return value
8. `R_PRDI` / `gCTX` / `R_REST` — restore context

The chain does **nothing** on its own — it is a general-purpose ROP trampoline that runs `kread`, `kwrite`, and the UID-pairing code.

### 5.8 Stage H — Patch `sysent[11]`

It reads `sysent[11].sy_call` (at `SE11 = P[0]`) and compares it to `KEXECG = P[1]`:

- If `sy_call == KEXECG`, the kernel is **already patched** — skip the patch.
- If `sy_call == sys_nosys`, the kernel is stock — apply the patch.

The patch writes `KEXECG` into `sysent[11].sy_call` and sets `sy_narg = 2`. It also spot-checks 6 of the 10 guard RVAs (`GRVA`) and verifies their bytes match the expected patched values (`GNEW`). If any guard is odd, it **refuses to fire** and reports “MIXED kernel” — cold reboot required.

### 5.9 Stage I — Map `jitshm` page and write the patch blob

It creates a `jitshm` fd via `syscall 533` and maps 16,384 bytes at `0x9400C000`. It writes the 632-byte blob from `PKB` (hex-decoded) into this page. It verifies the blob head is `0xB9 0x82` and the tail is `0xC3`.

### 5.10 Stage J — Fire `EV_ADD` once

It fires `EV_ADD -23` on the second kqueue. The `f_touch` pointer is temporarily swapped to the `jitshm` page. The kernel executes the blob as x86-64 shellcode in ring 0. This is the actual kexec patch. After it returns, `f_touch` is restored.

### 5.11 Stage K — Verify the patch

It reads back `sysent[11].sy_call`, `sysent[11].sy_narg`, and 6 of the 10 guard RVAs. It requires:

- `sy_call == KEXECG`
- `sy_narg == 2`
- All 6 guard RVAs match `GNEW`

If any check fails, it reports “NOT fully applied” and the chain stops.

### 5.12 Stage L — Console sink patch

It reads the console-sink byte at RVA `0x1A46E0` (27557776) and zeroes it. This is the “L set” line checked by `verifyHenReport()`. The full comment in `Hx1400.java` explains that this byte is read by one getter at RVA `0x1d6e00`, called from two sites — one skips a block, the other walks the console device list (the UART path).

### 5.13 Stage M — Proc escape

It walks the `kern_proc` list (RVA `0x1B2838`) to find the current process, then writes the `rootvnode` pointer at `pfd + 16`, `+24`, `+32` to escape the sandbox. It also writes `/data/hen/hen.ini` with the standard config:

```
[HEN]
config_version = 3
exploit_fixes = 1
mmap_patches = 1
block_updates = 1
disable_aslr = 1
nobd_patches = 0
target_id = 0
skip_patches = 0
upload_prx = 1
enable_plugins = 1
```

### 5.14 Stage N — Fetch and launch HEN

It maps 589,824 bytes at `0x950000000` (`HEN_MAP`) via `jitshm`, then fetches `hen.bin` from `HENURL` (the patched `jar:file:` URL). It verifies:

- Head byte is `0xE9` (JMP)
- Entry offset is `HEN_ENTRY`
- BSS is zero

It then patches the embedded config bytes (offsets `OFF_FIXES` through `CFG_PLUGINS`) to `1, 1, 1, 1, 0, 0, 1, 1`. It resolves `pthread_create` or `scePthreadCreate`, verifies its prologue is `0x55` or `0xF3`, and calls it with the HEN base as the entry point.

### 5.15 `safeDown()` — The safety mechanism

Before any armed action, `Hx1400` calls `safeDown()` on any exit. `safeDown()`:

1. Pins the fake page (`refcount = 0x40000000`).
2. Restores `sysfilt_ops` if possible.
3. Deletes the knote and closes the bridge kqueue.
4. **Quarantines** the 96 victims and 144 spray sockets instead of freeing them — the kernel reference count is pinned, so the kernel can never walk a dangling pointer.
5. Saves the sysfilt mask for later recovery.

The quarantine is the key safety property: it means that even a failed run does **not** free the fake page, so the kernel is not left with a dangling pointer. The cost is that the fake page and hundreds of fds stay open until cold reboot — which is exactly what `NOTICE.txt` warns about.

---

## 6. The 40-Command Exploit Chain

The `schedule.bin` file defines the 40 commands. Based on the `verifyEarlyStatus()` checks and the `Hx1400.java` interface, the commands are:

| # | Command | Class | Role |
|---|---|---|---|
| 0 | (intro) | — | Intro / version |
| 1 | (table setup) | — | Initialize patch tables `P`, `PG`, etc. |
| 2 | (marker setup) | — | Write the 4 random marker words |
| 3 | (JVM warmup) | — | Thread sleep + GC |
| 4 | **AXZXkk.init** | `AXZXkk` | Native bootstrap; returns `{status=0}` |
| 5 | **EkIKZR.initImage** | `EkIKZR` | Allocate 4-element `int[]`; returns `{status=0, len=4}` |
| 6 | **EkIKZR.corruptGetRGB** | `EkIKZR` | Corrupt `length` to `0x7FFFFFFF`; returns `{status=0, len=0x7FFFFFFF}` |
| 7 | **J196rR.scan** | `J196rR` | Find marker words; returns `{index}` in `[4, 16383]` |
| 8 | **J196rR.echo** | `J196rR` | 1024-byte heap snapshot |
| 9 | (early memory) | — | Uses the OOB array |
| 10 | (early memory) | — | Uses the OOB array |
| 11 | (early memory) | — | Uses the OOB array |
| 12 | (native symbols) | — | Verify 10 native deltas |
| 13–15 | (native symbols) | — | More native symbol resolution |
| 16 | (aliases) | — | Verify 5 shared-base aliases |
| 17–26 | (prepare kernel access) | — | Build the kernel bridge |
| 27 | (code windows) | — | Verify 3 × 16-byte code windows |
| 28–31 | (kernel stage prep) | — | Prepare `Hx1400` tables |
| 32 | **HEN URL patch** | — | Replace sentinel with actual URL |
| 33–42 | (kernel stage prep) | — | Prepare the kernel patch blob |
| 43 | (pre-Hx) | — | Print “Entering one-shot Hx/HEN stage” |
| 44 | **Hx1400.run** | `Hx1400` | The kernel exploit |
| 45 | (Hx diagnostic) | — | Print the Hx log on failure |
| 46 | **terminal** | — | Empty result; final success |

The exact mapping is inferred; the exact command IDs 0–43 are only partially visible in the code.

### 6.1 The `verifyEarlyStatus` checks in detail

For commands 4, 5, 6:

- **4**: `vector[0] = {0,0,0,0}` — status 0.
- **5**: `vector[0] = {0, 0, 0, 4}` — status 0, length 4.
- **6**: `vector[0] = {0, 0, 0, 7F FF FF FF}` — status 0, length `Integer.MAX_VALUE`.

For command 7: `vector[0]` must be a 4-byte index in `[4, 16383]`.

For command 8: `vector[0]` must be 1024 bytes, and its first 4 `int`s must be the marker words.

For command 12: `scalars[]` must have 13 entries; the first 10 must satisfy `scalars[i] - scalars[0] == guards.nativeDeltas[i]`.

For command 16: `scalars[]` must have at least 12 entries; scalars 3, 5, 7, 9, 11 must all equal scalar 1.

For command 27: `vectors[]` must have 3 entries, each 16 bytes, each hashing to `guards.windowHashes[i]`.

For command 46: `verifyHenReport()` must pass.

---

## 7. The Safety and Trust Model

### 7.1 Three-layer integrity

The offline loader has three layers of integrity checking:

1. **`InitXlet` preflight** (static): SHA-256 of every asset before any code runs.
2. **`OfflineRenderer.renderCandidate`** (per-command): re-hash each template, verify prior result shapes, verify relocation sources, verify write destinations.
3. **`LocalResultLedger`** (state machine): enforce command order, enforce result uniqueness, enforce terminal shape, enforce `verifyHenReport()` success.

The net effect: **an attacker who modifies any asset must also update the SHA-256 pin in `InitXlet.class` and the `TemplatePin` in `template-hashes.bin`, which are themselves inside the signed JAR.** This is the standard Blu-ray trust model, and it is what makes the loader safe to distribute.

### 7.2 One attempt per boot

The `attempted` flag in `InitXlet.keyPressed` guarantees that the exploit only runs once per boot. `NOTICE.txt` explains why:

> *“The Hx stage may write unknown kernel objects even on a failed pairing run. There is no automatic retry. After a failure, **cold-reboot the console** before any new attempt.”*

`Hx1400.safeDown()` quarantines the fake page and hundreds of fds, which means a failed run leaves the kernel in a state where a second attempt **could** walk a dangling pointer. The one-attempt policy is the safety mechanism.

### 7.3 Independent success verification

`verifyHenReport()` is a **string-based oracle**: it looks for specific success strings in the Hx log. The `NOTICE.txt` warns:

> *“A displayed Hx success report must be independently verified on the console.”*

This is why `InitXlet` prints “Verify menu and kernel R/W separately” at the end.

### 7.4 The `safeDown` quarantine

`safeDown()` is the most important safety function in the entire chain. It:

- Pins the fake page (`refcount = 0x40000000`).
- Restores `sysfilt_ops` if possible.
- Deletes the knote and closes the bridge kqueue.
- **Quarantines** the 96 victims and 144 spray sockets instead of freeing them.

The quarantine is the **only** reason a failed run does not panic. The cost is that the fake page and fds stay open until cold reboot.

---

## 8. Strengths and Weaknesses

### 8.1 Strengths

- **Fully offline.** No network, no server, no WebKit, no user data required.
- **Hash-verified at every layer.** Every template is re-hashed on every render. Every relocation source is checked. Every write destination is validated.
- **Deterministic failure.** `safeDown()` quarantines rather than frees, so the kernel does not panic even on a failed run.
- **Re-entrant kernel access.** The `sysfilt_ops` slot is never restored, so `kread`/`kwrite` can be called as many times as needed during the run.
- **Clear success oracle.** `verifyHenReport()` rejects both partial success and full failure.
- **One-attempt policy.** Prevents the user from re-running a chain that has left the kernel in an unknown state.

### 8.2 Weaknesses

- **Blind writes.** Any missed arm in stage C writes 8 bytes into an unknown kernel zone. The `safeDown()` quarantine prevents a panic, but the writes themselves cannot be undone.
- **Kernel stage is fragile.** The `Hx` stage is the most experimental part. The `NOTICE.txt` explicitly says so.
- **No KEX source.** The `Hx` stage assumes the user has a 14.00 kernel exploit. The AIO/kqueue race is the KEX, but it is only published here in Java form — there is no C port.
- **Stolen code.** The README says so.
- **No `.iso` rebuild instructions.** The repo ships a pre-built `hen.iso`, but the `src/` directory is not documented for rebuilding.
- **Opaque helper classes.** `AXZXkk`, `EkIKZR`, and `J196rR` are compiled with obfuscated names and have no published source. This makes independent audit difficult.

---

## 9. Comparison with Other PS4 Chains

| Feature | BD-JB (this repo) | WebKit (13.52) | Remote (this repo’s ancestor) |
|---|---|---|---|
| Entry point | Blu-ray disc | Browser | Network |
| Userland exploit | JVM reflection | JavaScriptCore type confusion | JVM reflection |
| Kernel exploit | AIO/kqueue race | (requires separate KEX) | AIO/kqueue race |
| HEN delivery | In-disc `hen.bin` | (requires separate HEN) | Over the network |
| Reboot required | Yes | Yes | Yes |
| Trust root | Signed Blu-ray JAR | Browser sandbox | Network server |
| Difficulty to build | High (needs BD-J runtime) | Medium (needs browser) | High (needs server) |
| Difficulty to use | Low (insert disc) | Medium (open URL) | Medium (run server) |

The BD-JB chain is unique in that **the entire chain fits on a disc**. The remote chain requires a running HTTP server; the WebKit chain requires a browser. The BD-JB chain is the only one that works from a cold boot with no network.

---

## 10. Conclusions

`PS4-14.00-BD-JB` is a **complete, self-contained, hash-verified, one-shot BD-J exploit chain** for PS4 firmware 14.00. It is a Java-to-kernel chain that uses a Java2D `getRGB` bug for userland OOB, a kqueue/AIO race for kernel R/W, and `pthread_create` to launch a 562,240-byte HEN.

The chain is notable for:

- **The interpreter design.** `Boot` is a pure-Java binary interpreter. This makes the chain platform-independent: the same interpreter runs on any BD-J stack, with only the command templates differing.
- **The hash verification.** Every asset is SHA-256 pinned. The verification runs twice — once at preflight, once per render. This is what makes the loader safe.
- **The quarantine safety.** `safeDown()` quarantines rather than frees, which is the only reason a failed run does not panic the kernel.
- **The one-attempt policy.** This is the only safe way to use a chain that leaves the kernel in an unknown state.

The chain is **not** a general-purpose kernel exploit. It is a specifically crafted chain for PS4 14.00, with hardcoded offsets, specific syscalls, and specific marker words. Its correctness depends on:

- The JVM’s heap layout (for `earlyIndexes`)
- The JVM’s native symbol layout (for `verifyNativeLayout`)
- The PS4 14.00 kernel’s `sysent` layout (for `SE11` and `KEXECG`)
- The PS4 14.00 kernel’s `kern_proc` list (for `rootvnode`)
- The PS4 14.00 kernel’s `console-sink` byte (for `L set to 0`)

If any of these change — in a future firmware, or a modified console — the chain will fail. The `safeDown()` quarantine ensures that a failure does not panic, but it does not guarantee success.

The repository is best understood as a **carefully engineered failure-safe exploit**, not a generic tool. Its authors clearly knew what they were doing: the three-layer trust model, the re-entrant kernel bridge, the deterministic failure handling, and the one-attempt policy are all evidence of significant engineering. The `NOTICE.txt` warnings are the honest disclosure of the chain’s limits.
