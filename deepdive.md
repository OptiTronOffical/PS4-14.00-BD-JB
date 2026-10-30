Here is a combined README that merges the repository's three Markdown files—`README.md`, `JAR-ANALYSIS.md`, and `class-analysis.md`—into a single, cohesive document.

---

# PS4 14.00 BD-JB (Blu-ray Java Jailbreak) — Combined README

> **NOTE:** The repository contains the following notice: *"THIS IS STOLEN CODE, BUT TBH YOU TRY CHARGE WHAT DO U EXPECT, Play silly games win silly prizes."*

A Blu-ray Disc (BD-J) jailbreak image for PlayStation 4 consoles running **System Software 14.00**. This project leverages the BD-JB vulnerability to achieve userland code execution and load a Homebrew Enabler (HEN) via a crafted Blu-ray disc.

> **Status:** Confirmed working by *dr.yenyen* after 14 attempts (per repository description).

---

## Table of Contents

1. [Overview: What It Does](#1-overview-what-it-does)
2. [Repository Structure (`/src`)](#2-repository-structure-src)
3. [Important Notice & Safety](#3-important-notice--safety)
4. [How to Use (General BD-JB Workflow)](#4-how-to-use-general-bd-jb-workflow)
5. [Java Source Analysis (JAR Internals)](#5-java-source-analysis-jar-internals)
6. [Helper Class Analysis (`/x/offline/`)](#6-helper-class-analysis-xoffline)
7. [Security Model & Trust Chain](#7-security-model--trust-chain)
8. [Credits](#8-credits)

---

## 1. Overview: What It Does

According to `src/NOTICE.txt`, the payload is a **"Bonk HEN 14.00 local HEN review candidate."** The execution flow is:

1.  The PS4 mounts the Blu-ray disc and automatically starts the BD-J Xlet defined in `BDMV/`.
2.  The Xlet **validates** local classes, metadata, and the HEN environment.
3.  It **waits for a single "X" button press** on the controller.
4.  It calls the original `Boot.exec` with a **scrubbed, result-gated command sequence** to load the HEN.
5.  **No HTTP server** is required; the entire exploit runs from the disc.

The presence of `Hx1400.class` indicates a stage that directly interacts with the PS4 kernel (or attempts to) after the initial Java sandbox escape. On firmware 14.00, the kernel portion is considered the most fragile and experimental part of the chain.

---

## 2. Repository Structure (`/src`)

The `/src` directory mirrors the standard **Blu-ray Disc file system (BDFS)**. All contents must be burned to a Blu-ray disc or mounted as an ISO for the PS4 to recognize and execute the exploit.

```
/src
├── BDMV/
│   ├── AUXDATA/
│   ├── BACKUP/
│   ├── BDJO/                # BD-J Object files (execution entry points)
│   ├── CLIPINF/
│   ├── JAR/
│   │   └── 00000.jar        # Contains the Java Xlet (main logic of the exploit)
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
│       ├── Hx1400.class      # Likely the HEN stage / kernel interaction
│       └── J196rR.class
└── NOTICE.txt                # Critical usage & safety information
```

| Path | Purpose |
|------|---------|
| `BDMV/JAR/00000.jar` | The primary Java application executed by the PS4’s BD-J stack. This is where the exploit chain begins. |
| `BDMV/BDJO/` & `BDMV/index.bdmv` | Blu-ray navigation files that tell the player which BD-J object (and thus which Xlet) to launch automatically. |
| `CERTIFICATE/` | Disc root certificates and metadata required for the PS4 to treat the disc as a valid Blu-ray video. |
| `x/offline/*.class` | Additional compiled Java classes referenced by the Xlet. `Hx1400.class` suggests a stage specifically tailored for firmware 14.00 (Hx = likely the HEN/kernel payload stage). |
| `NOTICE.txt` | **Read this first.** Contains the exact behavior of the Xlet and mandatory safety precautions. |



---

## 3. Important Notice & Safety

The following warnings are taken directly from `src/NOTICE.txt` and are **critical** for anyone attempting to use this disc:

> *"The Hx stage may write unknown kernel objects even on a failed pairing run. There is no automatic retry. After a failure, **cold-reboot the console** before any new attempt. A displayed Hx success report must be independently verified on the console."*

**Key safety rules:**

- **Do not retry** the exploit immediately after a failure. Always perform a **full cold reboot** (power off completely, not just rest mode) before inserting the disc again.
- The HEN loading is **result-gated**, meaning the Xlet expects a specific success signal from the kernel stage. If the signal is missing, the process may leave the system in an unstable state.
- Because the kernel stage is experimental for firmware 14.00, **assume a non-trivial risk of kernel panic or system instability** on each attempt.
- The success message shown on screen by the Hx stage **must be verified independently** (e.g., by checking the HEN menu or payload loader) — do not trust the on-screen report alone.



---

## 4. How to Use (General BD-JB Workflow)

This repository provides the **file system source** for the exploit disc. To deploy it:

1.  **Obtain the image:** Clone or download this repository and navigate to the `/src` directory.
2.  **Build a Blu-ray image (ISO):**
    - Use Blu-ray authoring software (e.g., ImgBurn, or command-line tools like `genisoimage` with UDF support) to compile the **contents of `/src`** into a valid Blu-ray ISO.
    - Ensure the UDF file system is used, as the PS4’s BD-J environment expects a standard Blu-ray layout.
    - Alternatively, if you have a **jailbroken PS4** with a disc-burning utility (e.g., `FPKG` tools), you can write the extracted `/src` folder directly to a **BD-RE** disc as a UDF volume.
3.  **Burn to disc:**
    - Burn the ISO to a **BD-RE** (rewritable) or **BD-R** disc. BD-RE is recommended for testing because it can be erased and rewritten after a failed attempt.
    - Finalize the disc. The PS4 will not read an unfinalized disc.
4.  **Run on the console:**
    - Insert the disc into a PS4 running **firmware 14.00**.
    - The BD-J Xlet should start automatically from the disc’s "movie" menu. If it does not, navigate to the disc icon on the XMB and launch it manually.
    - Follow the on-screen prompt: **press the "X" button once**.
    - Observe the result. If the HEN does not load, **perform a cold reboot** and try again later. Do not spam attempts without rebooting.

> **Note:** The original repository includes a pre-built `hen.iso` in the root. You can use that directly if you do not need to modify the source. The `/src` folder is provided for inspection and rebuilding.

---

## 5. Java Source Analysis (JAR Internals)

This section explains the Java source files that make up the **offline Blu-ray Java (BD-J) HEN loader** for PS4 firmware 14.00. The code is a fully self-contained, hash-verified, one-shot exploit chain that runs entirely from a signed Blu-ray JAR — no network server is contacted during execution.

The source comprises six files:

| File | Role |
|------|------|
| `Cfg.java` | Build/server constants (mostly unused in offline mode) |
| `Screen.java` | On-disc AWT text display |
| `Boot.java` | Bytecode interpreter that executes the exploit command stream |
| `OfflineRenderer.java` | Template verifier + relocator (the security core) |
| `LocalResultLedger.java` | State machine enforcing command order and result shapes |
| `InitXlet.java` | Xlet entry point: preflight, template loading, one-shot driver |
| `Hx1400.java` | (separate file) Kernel exploit + HEN loader, invoked as command 44 |

The execution flow is: **`InitXlet` → verify everything → user presses X → `LocalResultLedger` walks the 40-command schedule → `OfflineRenderer` renders each command from a verified template → `Boot.exec()` interprets it → results fed back → command 44 triggers `Hx1400`.**

### 5.1 Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  Blu-ray JAR (signed)                                            │
│  /BDMV/JAR/00000.jar                                             │
│   ├─ org/bdj/InitXlet.class      ← Xlet entry point              │
│   ├─ org/bdj/Boot.class          ← interpreter                   │
│   ├─ org/bdj/OfflineRenderer.class                               │
│   ├─ org/bdj/LocalResultLedger.class                             │
│   ├─ org/bdj/Screen.class                                        │
│   ├─ bonk/hen.bin                ← HEN image (562240 bytes)      │
│   ├─ bonk/local/rules.bin        ← relocation rules (205)         │
│   ├─ bonk/local/shapes.bin       ← expected result shapes        │
│   ├─ bonk/local/guards.bin       ← native layout deltas         │
│   ├─ bonk/local/schedule.bin     ← 40-command order              │
│   ├─ bonk/local/template-hashes.bin ← 40 SHA-256 pins            │
│   ├─ bonk/local/marker-sites.bin ← 4 marker patch offsets        │
│   ├─ bonk/local/write-sites.bin  ← 81 destination checks         │
│   └─ bonk/local/templates/cmd-NNNN.bin ← 40 command templates    │
│                                                                  │
│  /x/offline/                                                     │
│   ├─ J196rR.class   ← helper (marker scan, etc.)                 │
│   ├─ AXZXkk.class   ← helper (early memory helpers)              │
│   ├─ EkIKZR.class   ← helper                                     │
│   └─ Hx1400.class   ← kernel exploit + HEN loader                │
└──────────────────────────────────────────────────────────────────┘
```

The JAR is the only trust root. Every asset is SHA-256 pinned in `InitXlet` before use. The Xlet refuses to run if any hash, length, or magic byte does not match.

### 5.2 `Cfg.java` — Constants

```java
public static final String SERVER = "tester.f5rvoice.com";
public static final int    PORT   = 1992;
public static final String BUILD  = "6";
public static final String CID    = "c6";
public static final String CKEY   = "942a76d6946a34ce476bdabb09989f4c";
```

These constants were used by the **remote** version of the exploit (`Boot.run()` used to POST to `SERVER:PORT`). In the **offline** version, `Boot.EP` is set to the local disc root by `InitXlet.Attempt.run()`, and `Boot.SID = "offline"`. `CKEY` is still used as the seed for the ticket hash and XOR keystream — but in offline mode, `Boot.SK` is set to `null`, so the ticket becomes `mix(CKB, …)` without a session key. `CKEY` is the hex form of `CKB` (see `Boot.hexb("942a76d6946a34ce476bdabb09989f4c")`), a 16-byte constant hardcoded into both `Boot` and `Cfg`. It is not a secret — it's an obfuscation constant.

### 5.3 `Screen.java` — Display

A `java.awt.Container` subclass that maintains a scrollback of up to 46 status lines and paints them in white-on-black. Used by `InitXlet` and `Boot` for user-visible progress.

Key methods:
- `Screen.println(String)` — append + repaint
- `Screen.println(String, repaint, overwriteLast)` — append with options
- `Screen.printStackTrace(Throwable)` — formats a stack trace into the scrollback
- `Screen.clear()` — clears the scrollback
- `paint(Graphics)` — draws the messages, re-entrant safe via `isPainting`/`isDirty` flags

In `Boot.run()`, it prints `.`, `+`, `-`, `!` as progress glyphs (used only in remote mode). In `InitXlet`, the Xlet uses its own `Lines` inner class for the fancy orange title bar, but still calls `System.out.println("Bonk HEN local: " + msg)` for the log.

### 5.4 `Boot.java` — The Interpreter

`Boot` is a **binary command interpreter** written in pure Java. It is the component that actually performs reflection: loading classes, constructing objects, invoking methods, and reading/writing fields. It is also where the *result frame* format is defined.

#### Static state

```java
static String    EP;          // base URL (disc root in offline mode)
static String    SID;         // session ID ("offline")
static byte[]    SK;          // session key (null in offline mode)
static long[]    R = new long[128];      // scalar registers
static Object[]  O = new Object[64];     // object registers
static Object[]  M = new Object[64];     // method/field registers
static String[]  S = new String[64];     // string registers
static byte[][]  Z = new byte[32][];     // byte-array registers
static ClassLoader PAR;      // parent classloader
static byte[]    CKB, KSB;   // ticket hash key, XOR keystream
```

Register indices are `& 63` (or `& 127` for `R`). All opcodes address these register banks.

#### `exec(byte[])` — The interpreter loop

Reads a `short` count, then loops reading one **opcode byte** followed by opcode-specific operands. Supported opcodes include:

| Opcode | Meaning |
|--------|---------|
| 1 | `halt` with return code |
| 2 | print scalar |
| 3 | `Thread.sleep(ms)` |
| 4 | burn `ms` microseconds (busy loop) |
| 5 | `S[reg] = new String(bytes, UTF-8)` |
| 6 | `Z[reg] = bytes` |
| 7 | `O[reg] = Class.forName(S[name])` |
| 8 | `O[reg] = new L(bytes, name).loadClass(name)` (custom loader, with DVB fallback) |
| 9 | `O[reg] = ctor(...).newInstance(args)` |
| 10 | `M[reg] = cls.getDeclaredMethod(S[name], paramTypes)` |
| 11 | `M[reg] = cls.getDeclaredField(S[name])` |
| 12 | `R[reg] = (long) field.get(obj)` |
| 13 | `field.set(obj, arg)` |
| 14 | `R[reg] = (long) method.invoke(obj, args)` |
| 15 | `O[reg] = Array.newInstance(cls, n)` |
| 16 | `R[reg] = Array.get(O[arr], i)` |
| 17 | `Array.set(O[arr], i, arg)` |
| 18 | `R[reg] = Array.getLength(O[arr])` |
| 19 | `R[reg] = (long) arg` (number) |
| 20 | `R[reg] = arithmetic(op, a, b)` (add/sub/and/or/xor/shl/shr/mul/div/cmp) |
| 21 | write scalar to output |
| 22 | serialize `O[reg]` (byte[], int[], long[]) to output |
| 23 | serialize `Z[reg]` to output |
| 24 | conditional skip (used by jump tables) |
| 25 | `System.gc()` × 2 |
| 26 | `O[dst] = O[src]` |
| 27 | `O[reg] = arg` |
| 28 | `O[reg] = new Object[]{O[src]}` (single-element wrapper) |
| 29 | spawn thread calling `M[reg].invoke(obj, args)` (used to run HEN on its own thread) |
| 30 | `O[reg] = M[method]` (method-as-object) |



Operands are decoded via `arg(DataInputStream)` (returns `Long`, `Integer`, `Boolean`, `String`, `byte[]`, etc.) and `cls(long)` (maps negative constants like `-1` → `long.class`, `-10` → `Object.class`).

**Result frame format** (returned by `exec`):

```
long    status           (0 = OK, -1 = exception, -2989 = unknown opcode)
int     instruction      (opcode index reached)
int     nScalars
long[]  scalars
int     nVectors
for each: int len; byte[] data
int     nNotes
for each: int len; byte[] note (UTF-8, e.g. exception message)
```

This frame is exactly what `OfflineRenderer.frame()` parses.

#### `io(String, byte[])` — HTTP transport (remote mode only)

POSTs to `EP + path`, sets header `X-Ticket: c6.<seq>.<hash>`, XOR-encrypts the body with a keystream derived from `CKB`/`SK`/random, then reads and decrypts the response. **In offline mode, this method is never called** — `InitXlet` sets `Boot.EP = discRoot`, but the `Attempt` loop uses pre-verified templates from the JAR, not `io()`.

#### `ticket()` and `xor()` — Crypto

`ticket(header, seq, len)`:
```
buf = header[8] || seq[4 BE] || SID[UTF-8] || len[4 BE]
ticket = "c6." + seq + "." + hex16( mix(CKB||SK, buf) )
```

`mix()` is a simple LCG-based hash: `x = x*KSB[1] + byte + KSB[2]` over the concatenated buffers. `xor()` re-seeds the same LCG with `SK` + `buf` and emits 8-bit keystream bytes at bit offset `KSB[3] & 63`. Both are **obfuscation, not cryptography** — they were only used to prevent casual network sniffing in the remote version. In offline mode, they are unused.

#### `L` — The custom classloader

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

This is how the exploit loads helper classes (`J196rR`, `Hx1400`, etc.) from raw byte arrays that arrived over the wire. In offline mode, they arrive from `/x/offline/*.class` via the `DVBClassLoader` fallback path, because the JAR-signed copy is used instead. The `InitXlet.checkHash()` preflight verified those byte arrays first.

### 5.5 `OfflineRenderer.java` — The Security Core

This is the most important class to understand. It is responsible for **turning a verified template into a runnable command byte stream**.

#### The metadata binaries

Each `/bonk/local/*.bin` file has the same header:

```
magic   4 bytes  (e.g. "FMR1")
version 2 bytes  (= 1)
payload ...
```

The magic strings are `FMR1` (rules), `FMS1` (shapes), `FMG1` (guards), `FMT1` (template pins), `FMC1` (schedule), `FMM1` (marker sites), `FMW1` (write sites). Counts are hardcoded (`rule count changed: 205`, `shape count changed: 1..47`, `guard count: 10`, `template count: 40`, `schedule count: 40`, `marker site count: 4`, `write site count: 81`).

| File | Parsed into | Purpose |
|---|---|---|
| `rules.bin` | `Rule[205]` | `(command, instruction, offset, request, kind, slot, high, low, addend)` — places a relocated 8-byte value at `offset` in the command template, sourced from a previous result frame |
| `shapes.bin` | `Shape[N]` | `(request, status, instruction, scalarCount, vectorLengths[])` — the expected shape of each result frame |
| `guards.bin` | `Guards` | `(nativeDeltas[10], windowHashes[3][32])` — expected relative offsets between native symbols, and SHA-256 of three 16-byte code windows |
| `schedule.bin` | `ScheduleEntry[40]` | `(command, nextResultRequest)` — strictly increasing command IDs, last one is `(46, 65535)` |
| `template-hashes.bin` | `TemplatePin[40]` | `(command, length, sha256)` — pins each `cmd-NNNN.bin` |
| `marker-sites.bin` | `MarkerPlan` | `offsets[4]` — where to patch the 4 random marker words in command 2 |
| `write-sites.bin` | `WriteSite[81]` | `(command, instruction, offset)` — every location that command 1..28 will write a pointer to; must land in `[0x100000000, 0x8000000000000000)` |

#### `renderCandidate()` — The hot path

Steps:
1. Find the `TemplatePin` for `command`, verify `template.length == pin.length` and `hash(template) == pin.sha256`.
2. **Pre-flight checks based on command number:**
   - `command >= 7`: verify prior marker-scan (result 7) and marker-echo (result 8) match `markerWords`.
   - All commands: `verifyShapes()` — every prior result frame must match its declared `Shape`.
   - `command >= 12`: `verifyNativeLayout(frames[12], guards)` — verify the 10 native symbol deltas.
   - `command >= 17`: `verifyAliases(frames[16])` — verify that scalars 3,5,7,9,11 share the same base as scalar 1.
   - `command >= 27`: `verifyCodeWindows(frames[27], guards)` — verify the three 16-byte code windows against their SHA-256.
3. **Relocate:** for each `Rule` matching `command`:
   - Check `offset` is in `[2, len-8]` and `request < command` and `request < frames.length`.
   - Verify the 8 bytes at `offset` are zero in the template (rejects already-patched templates).
   - Compute `value` based on `kind` (0 = scalar, 1 = vector qword, 2 = early index).
   - Add `rule.addend` (overflow-checked).
   - Write big-endian into `template[offset..offset+8]`.
4. `command == 2`: `patchMarkerWords()` — write `markerWords[i]` at `markerPlan.offsets[i]`, after verifying the surrounding `(17, ?, 3, i, 0,0,0,0)` structure.
5. `verifyWriteDestinations()` — every `WriteSite` in the command must have a pointer-shaped value in `[0x100000000, 0x7FFFFFFFFFFFFFF7]` after relocation.
6. Return the patched byte array.

The zero-byte and shape checks are what make this safe: a malicious or corrupted template **cannot** introduce arbitrary pointers into the kernel. Every pointer written to the kernel must come from a prior *verified* result frame.

#### `verifyHenReport()` — The HEN success oracle

This is what makes the one-shot HEN launch trustworthy. After command 44 runs, `verifyHenReport(frames[46])` requires:

**Summary (vector 0)** must contain:
- `HEN14V`
- `APPLIED=1`
- `UID=0 uid2=0`
- `*** ROOT ***`

**Log (vector 1)** must contain:
- `K 6/6 sites now hold the PATCHED byte`
- `K sysent[11].sy_narg=2`
- `K *** 1400.bin APPLIED - syscall 11 IS NOW kexec ***`
- `G2 sysfilt_ops restore=true`
- `H4 fetched 562240/562240 B`
- `H5c launching`
- `H7 pthread_create rc=0 handle=`
- `H7 *** hen.bin RUNNING`

And it must **not** contain:
- `STOP`, `no pair`, `blind writes`, `APPLIED=0` (in summary)
- `K NOT fully applied`, `safeDown threw` (in log)

Additionally, if both `K sysent[11].sy_call=` and `APPLIED=true` appear on the same line, and both `L set to ` and `ok=true` appear on the same line, the log is considered "kernel + ShellCore readback present" — otherwise the report is rejected. `lineContainsBoth()` and `contains()` do byte-level substring search on the raw vectors, so they work even if the vectors contain non-ASCII.

#### `isPendingHenResult()` — The retry oracle

When `Hx1400` is still running (command 44 in progress), its result frame looks like:

```
status      = 0
instruction = 7
scalars     = []
vectors     = [ "not run" (7 bytes), <log> (≤128 KiB) ]
```

`InitXlet.Attempt` uses this to distinguish "Hx still working" from "Hx finished or crashed". If it never resolves within 45 s or 10 polls, the attempt is aborted.

#### `freshMarkerWords()`

Generates 4 cryptographically random `int`s, each non-zero, not `-1`, not `-2`, and all distinct. `-1` and `-2` are reserved because `Hx1400` uses them as special marker values in `showHxDiagnostic`. Command 2 writes these into a template, and command 7 scans for them in kernel memory (via `Hx1400`'s early memory helpers) — this is how the exploit confirms it has the correct memory layout before touching anything important.

### 5.6 `LocalResultLedger.java` — The State Machine

The 40 commands are **strictly ordered**. `LocalResultLedger` enforces this:

- `currentCommand()` — returns `schedule[at].command`, or throws if failed/out-of-bounds
- `renderCurrent(template)` — calls `OfflineRenderer.renderCandidate()`, stores `pendingInstructionCount = (byte0<<8)|byte1` from the rendered template, sets `awaitingResult = true`
- `acceptResult(frameBytes)` — parses with `frame()`, calls `verifyNewResult()` and `verifyMarkerNewResult()`, stores in `frames[request]`, advances `at`, clears `awaitingResult`
- `finishTerminal(frameBytes)` — verifies the final result is exactly `{status=0, instruction=pendingInstructionCount, scalars=[], vectors=[]}`
- `complete()` — all 40 commands processed without failure

The `successWindowConfirmed` flag is set when result 27 (the code-window check) arrives. Any command with ID `>= 28` requires it to be true — this prevents the exploit from firing the kernel stage if the code-window signature check has not passed.

`acceptResult()` rejects:
- Duplicate result requests (`frames[var2] != null`)
- Wrong shape (`verifyNewResult` throws)
- Wrong marker echo (`verifyMarkerNewResult` throws)

The `frames` array has 48 slots, so only result-request IDs `< 48` are legal. The schedule's `nextResultRequest` values are checked in `schedule()` — the last command must be `(46, 65535)`.

### 5.7 `InitXlet.java` — Entry Point + Preflight

This is the `javax.tv.xlet.Xlet` implementation — the class the BD-J runtime actually launches.

#### `startXlet()` — Preflight kickoff

```java
this.preflightStarted = true;
new Thread(this, "Bonk HEN local asset preflight").start();
```

Only one preflight thread is ever started. If `startXlet()` is called again (e.g. after `pauseXlet()`), the preflight is skipped.

#### `run()` — Preflight (executes in the thread)

In order:
1. Print credits.
2. Locate own class via `Class.forName("org.bdj.InitXlet").getResource("InitXlet.class")`.
3. Verify it is a `jar:file:.../BDMV/JAR/...` URL. If not, `PREFLIGHT FAIL`.
4. Extract `discRoot = url.substring(4, indexOf("/BDMV/JAR/"))`. Must start with `file:`.
5. For each of the 4 helper classes (`J196rR`, `AXZXkk`, `EkIKZR`, `Hx1400`) in `/x/offline/`:
   - `checkHash(url, sha256, 1, 100000)` — the file must be 1..100000 bytes and match its SHA-256.
6. Locate `/bonk/hen.bin` via `getResource`. Must be a `jar:file:` URL. Verify exactly 562240 bytes and SHA-256 `29ec095f…`.
7. For each of the 7 `/bonk/local/*.bin` metadata files:
   - `readBounded(url, 8192)` — must be ≤8192 bytes
   - Verify length and SHA-256.
8. Parse metadata via `OfflineRenderer.rules/shapes/guards/schedule/templatePins/markerPlan/writeSites`.
9. Cross-check: `pins.length == schedule.length == 40`, and `pins[i].command == schedule[i].command` for every `i`.
10. For every `TemplatePin`:
    - Load `/bonk/local/templates/cmd-NNNN.bin` (zero-padded to 4 digits)
    - Must be `jar:file:` and ≤ `pin.length` bytes, and exactly `pin.length` bytes
    - Verify SHA-256 == `pin.sha256`
11. Print `LOCAL PREFLIGHT PASS. Press X once to run.` and `One attempt only; restart after any failure.`

Any failure prints `PREFLIGHT FAIL: <exception>` and leaves `ready = false`. The Xlet then refuses to run.

#### `keyPressed(KeyEvent)` — One-shot trigger

```java
if (keyCode == 10 || keyCode == 32 || keyCode == 88) {   // Enter, Space, X
    synchronized (this) {
        if (stopped || !ready || attempted) return;
        attempted = true;
    }
    new Thread(new Attempt(), "Bonk HEN one-shot local run").start();
}
```

The `attempted` flag guarantees **one attempt per boot**. `InitXlet` explicitly requires a cold reboot after any failure.

#### `Attempt.run()` — The actual exploit driver

```java
Boot.EP  = InitXlet.this.discRoot;   // file: URL of disc root
Boot.SID = "offline";
Boot.PAR = InitXlet.class.getClassLoader();
Boot.SK  = null;                     // no session key
```

Then it runs the 40-command loop, special-casing command 44 (the kernel stage) to poll up to 10 times over 45 seconds while `Hx1400` is still running. It verifies the final result with `ledger.complete()` and `verifyHenReport()`, and prints `"Hx reports HEN start. Verify menu and kernel R/W separately."` on success.

On failure, `Attempt` prints the class name and message, optionally calls `showEarlyStatus(cmd, result)` for commands 4/5/6, and optionally calls `showHxDiagnostic(result)` for command 44. It then prints `No retry in this boot. Cold reboot before another attempt.`

#### `patchLocalHenUrl()` — The one runtime patch

The original `Hx1400` command 32 template contains the sentinel URL:

```
file:/__bonk___offline_missing__/hen.bin
```

This is a **45-byte placeholder**. `patchLocalHenUrl` replaces it with the actual JAR URL (`jar:file:...!/bonk/hen.bin`) so `Hx1400`'s `HEN_URL` field points at the signed JAR resource, not a filesystem path. It does this by scanning for the sentinel bytes, confirming it is unique, confirming it is preceded by the exact framing (`0x05` opcode, 8-byte `reg` = 45, 4-byte `len` = 45), copying the new URL in and updating the 4-byte length field, and rejecting any new length > 65535. This is the only runtime mutation of a verified template, and it is tightly constrained.

#### `showEarlyStatus()` and `showHxDiagnostic()` — Failure diagnostics

- `showEarlyStatus(cmd, frame)`: for commands 4, 5, 6 — reads the helper status ints from `vector[0]` and prints them.
- `showHxDiagnostic(frame)`: for command 44 — expects 2 vectors (`summary`, `log`), both within bounds. Prints the summary (truncated to 96 chars) and extracts up to 3 interesting log lines (matching `STOP`, `no pair`, `K NOT fully applied`, `G2 sysfilt_ops`, `safeDown threw`, `H7 pthread_create`, `H7 ***`, `L set `).

Both are wrapped in `try/catch` so a diagnostic failure never masks the real error.

---

## 6. Helper Class Analysis (`/x/offline/`)

This section describes the four **compiled helper classes** that live in `/x/offline/` on the Blu-ray disc and are loaded by the command chain at runtime:

| Class | File | SHA-256 |
|---|---|---|
| `AXZXkk` | `/x/offline/AXZXkk.class` | `2362afc9efa07a688169b9121d9e5f8bdacbead58f87cafd23fdddffea58b429` |
| `EkIKZR` | `/x/offline/EkIKZR.class` | `df30bfbd25c3cdbe67965d907d575efb562738cf29c258895a698ff2ef525137` |
| `J196rR` | `/x/offline/J196rR.class` | `f59953027867d15d1434179991feeeb94bf274422f742e739067fd65400a264d` |
| `Hx1400` | `/x/offline/Hx1400.class` | `e597710078e9afab5b24f4c22ea31fe7e5a5643b602b06c2866273b61c9c57e9` |

Unlike the `.java` files described in the previous section, these are **not open source in this repository** — only `Hx1400.java` was decompiled and made available. For the other three, this document describes what they do, as reconstructed from the hash pins in `InitXlet.java`, the preflight and command schedule, the `verifyEarlyStatus()` checks in `OfflineRenderer.java`, the `Hx1400.java` source, and the `Boot.exec()` opcode semantics.

### 6.1 The Two-Phase Verification Model

Every helper class is verified **twice** before it is ever loaded.

#### Phase 1 — Preflight hash check (`InitXlet.run()`)

```java
private static final String[][] CLASSES = new String[][]{
    {"J196rR", "f59953027867d15d1434179991feeeb94bf274422f742e739067fd65400a264d"},
    {"AXZXkk", "2362afc9efa07a688169b9121d9e5f8bdacbead58f87cafd23fdddffea58b429"},
    {"EkIKZR", "df30bfbd25c3cdbe67965d907d575efb562738cf29c258895a698ff2ef525137"},
    {"Hx1400", "e597710078e9afab5b24f4c22ea31fe7e5a5643b602b06c2866273b61c9c57e9"}
};

for (int i = 0; i < CLASSES.length; i++) {
    URL url = new URL(discRoot + "/x/offline/" + CLASSES[i][0] + ".class");
    checkHash(url, CLASSES[i][1], 1, 100000);
}
```

Bounds: **1 to 100000 bytes** per class. Hash: **SHA-256**. If any check fails → `PREFLIGHT FAIL` → Xlet refuses to run. This is the **static** trust anchor: the class bytes on disc must match the hashes compiled into `InitXlet.class` inside the signed JAR.

#### Phase 2 — Runtime load (`Boot.exec()` opcodes 7 or 8)

The preflight only *hashes* the files — it does not load them. The actual loading happens when the command chain reaches the relevant command. The templates use one of two opcodes:

**Opcode 7 — `Class.forName`** (used when the class is on the JAR's classpath)
```java
case 7:
    O[reg] = Class.forName(S[nameReg], true, PAR);
```

**Opcode 8 — custom byte-array loader with DVB fallback** (used when the class is in `/x/offline/`)
```java
case 8:
    try {
        O[reg] = new L(Z[bytes], S[name]).loadClass(S[name], true);
    } catch (Throwable t) {
        if (flag == 0) throw t;
        Class<?> dvb = Class.forName("org.dvb.lang.DVBClassLoader");
        Class<?> urlArr = Class.forName("[Ljava.net.URL;");
        Method newInst = dvb.getMethod("newInstance", urlArr, ClassLoader.class);
        Object arr = Array.newInstance(URL.class, 1);
        Array.set(arr, 0, new URL(EP + "/x/" + SID + "/"));
        Object dvbLoader = newInst.invoke(null, arr, PAR);
        Method lc = ClassLoader.class.getMethod("loadClass", String.class);
        O[reg] = (Class<?>) lc.invoke(dvbLoader, S[name]);
    }
```

In offline mode, `EP = discRoot` and `SID = "offline"`, so the DVB fallback resolves to `file:<discRoot>/x/offline/`. The custom loader `Boot.L` tries `defineClass` first; if that throws, the DVB fallback loads it from the same directory. The `note` field (`d1:<exception>`) records any fallback that occurred, visible in the result frame's notes.

### 6.2 `AXZXkk` — Native Access Bootstrap

**SHA-256:** `2362afc9efa07a688169b9121d9e5f8bdacbead58f87cafd23fdddffea58b429`

**Used by:** command 4 ("helper setup")

#### Purpose

`AXZXkk` is the entry point for **crossing from the Java sandbox into native code**. The `verifyEarlyStatus()` check for command 4 is:

```java
if (command == 4) {
    if (frame.vectors.length != 2
        || frame.vectors[0].length != 4
        || readInt(frame.vectors[0], 0) != 0)
        throw new IOException("helper setup failed");
}
```

So command 4 returns `vector[0] = {0, 0, 0, 0}` — a single 32-bit status of 0. The second vector is not checked for command 4, but it exists.

#### What it does (reconstructed)

Based on the `Hx1400.java` code, which uses the same native-access technique, `AXZXkk` performs the **`sun.misc.Unsafe` + `ClassLoader$NativeLibrary.findEntry` bootstrap**:

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
handle.setLong(inst, 8193L);             // 8193 = libjvm or libjava handle index
long sigsetjmp = (Long) find.invoke(inst, "sigsetjmp");
long setcontext = (Long) find.invoke(inst, "__Ux86_64_setcontext");
long getpid     = (Long) find.invoke(inst, "getpid");
// ... etc
```

`AXZXkk` almost certainly wraps this pattern into a class with methods like:
- `init()` — installs `Unsafe`, opens a `NativeLibrary` handle, resolves a base set of symbols
- `find(String)` — resolves a symbol by name (returns `long`)
- `call(long fn, long... args)` — a native-call trampoline that uses `nativeCall(long, int[])` to invoke an arbitrary function pointer with up to 7 arguments
- `getUnsafe()` — returns the `Unsafe` instance

The `nativeCall` native method is declared in `Hx1400.java` (`private native long nativeCall(long, int[])`) — but `AXZXkk` is the class that sets up the native bridge that makes it work. `Hx1400` extends the same pattern; they likely share the same `NativeLibrary` handle index.

#### Why it is a separate class

Because the preflight hash check in `InitXlet` requires it to be. The command templates reference `AXZXkk` by name when they need native access — if `AXZXkk` were inlined into `Hx1400`, the templates would need to hash-check the full kernel exploit on every command. Keeping it separate lets the early commands (4–43) get native access without loading `Hx1400`.

### 6.3 `EkIKZR` — Image / `getRGB` Array-Length Corruption

**SHA-256:** `df30bfbd25c3cdbe67965d907d575efb562738cf29c258895a698ff2ef525137`

**Used by:** commands 5 ("helper image setup") and 6 ("getRGB array-length corruption")

#### Purpose

`EkIKZR` provides two things:

1. A **pre-built image** (`DataBufferInt` / `int[]`) that the exploit can use as a fake object backing store
2. The **`getRGB()` array-length corruption** — a classic Java2D bug where a mutable `int[]` backing an `Image` can have its length field overwritten via a type confusion, giving the caller out-of-bounds read/write into the JVM heap

The `verifyEarlyStatus()` checks are:

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

- **Command 5** returns `{status=0, length=4}` — the image was allocated with a 4-element backing array.
- **Command 6** returns `{status=0, length=0x7FFFFFFF}` — the array's `length` field has been corrupted to `Integer.MAX_VALUE`.

This is the **userland OOB primitive**. After command 6 succeeds, `EkIKZR` exposes a method (likely `int[] getOobArray()` or similar) that returns an `int[]` whose `.length` is `Integer.MAX_VALUE` but whose backing memory is only 16 bytes. Reading past index 3 reads adjacent JVM heap; writing past index 3 writes it.

#### The `getRGB` bug

The specific bug is a well-known issue in older Java2D implementations where a type-confusion variant lets you pass a `long`-typed object where the JVM expects an `int[]` array header, and the `getRGB` implementation writes a `length` field derived from `width * height` without bounds-checking the destination array — effectively writing `0x7FFFFFFF` into the `int[]` length field of the caller's array. `EkIKZR` packages this up as a method that returns the corrupted array, and the command templates use it via opcode 14 (method invocation) to seed the OOB access used by `J196rR`.

### 6.4 `J196rR` — Marker Scan and Memory Pivot

**SHA-256:** `f59953027867d15d1434179991feeeb94bf274422f742e739067fd65400a264d`

**Used by:** commands 7 (marker scan), 8 (marker echo), and indirectly by commands 9–27

#### Purpose

`J196rR` is the **memory scanner and pointer-leak pivot**. It takes the OOB `int[]` from `EkIKZR` and uses it to:

1. Scan the JVM heap for the 4 random marker words written by command 2
2. Read back the marker words to confirm the scan found the right region
3. Compute the **base address of the JVM heap** from the offset of the found markers
4. Expose a `readLong(long addr)` / `writeLong(long addr, long val)` primitive that operates on raw JVM addresses

The `verifyMarkerNewResult()` checks are:

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

- **Command 7** returns `{index}` where `index` is the word-offset of the first marker found. It must be in `[4, 16383]`.
- **Command 8** returns a **1024-byte** buffer containing 256 `int` values starting at the marker offset. The first 4 `int`s must match `markerWords` exactly. The rest is whatever is in memory at that offset — this is the **early heap snapshot** used by `earlyIndexes()` in `OfflineRenderer` to derive pointer offsets.

#### What `earlyIndexes()` does with command 8

```java
private static long[] earlyIndexes(byte[] vec) throws IOException {
    if (vec.length != 1024) throw new IOException("early vector changed");
    long v13_14 = qword(vec, 13, 14);
    long v11_12 = qword(vec, 11, 12);
    long v29_28 = qword(vec, 29, 28);
    long base   = v13_14;
    long idxA   = (v11_12 - base - 24) / 4;
    long idxB   = (v29_28 - base - 8)  / 4;
    // must be word-aligned and in [0, 134217728)
    return new long[]{ idxA, idxA + 40, idxA + 41, idxB };
}
```

So the 1024-byte snapshot at the marker offset contains, at word offsets 11–14 and 28–29, **pointer-shaped values whose differences reveal the layout of the JVM heap**. `J196rR` deliberately snapshots this region because it contains known pointers (likely `Unsafe` field offsets, `int[]` array headers, or `String` interned pointers) that let `OfflineRenderer` compute `idxA`, `idxA + 40`, `idxA + 41`, and `idxB`. These four values become the **relocation sources** for commands 9–27 via `Rule.kind == 2`:

```java
} else if (rule.kind == 2) {
    if (rule.request != 8 || frames[8] == null
        || frames[8].vectors.length == 0 || rule.slot > 3)
        throw new IOException("early index source missing");
    long[] idx = earlyIndexes(frames[8].vectors[0]);
    value = idx[rule.slot];
}
```

So **every later pointer** in the chain is derived from the 4 values in `J196rR`'s command 8 echo. This is why the marker scan is such a critical gate: if it produces wrong indices, everything downstream is wrong.

#### The `nativeCall` bridge

`J196rR` also provides the native call trampoline that `AXZXkk` set up. The `nativeCall(long, int[])` native method is declared in `Hx1400.java` but the JNI registration (via `NativeLibrary` handle 8193) is what `J196rR` initialises. This is the mechanism by which commands 9–27 can call `dlsym`, `getpid`, `socket`, `sendto`, `fstat`, `sysctl`, `mmap`, `kqueue`, `kevent`, etc. — all of which are needed before `Hx1400` can do the AIO/kqueue race.

### 6.5 `Hx1400` — Kernel Exploit + HEN Loader

**SHA-256:** `e597710078e9afab5b24f4c22ea31fe7e5a5643b602b06c2866273b61c9c57e9`

**Used by:** command 44 (the one-shot kernel stage)

Full source is in `Hx1400.java` (decompiled). This document only summarises its interface to the offline loader.

#### Required static fields

`Hx1400` reads these static fields, which are set by earlier commands via `Boot.exec()` opcodes 27/13:

| Field | Type | Set by | Meaning |
|---|---|---|---|
| `P` | `long[31]` | template (cmd 2?) | patch table — 31 relocatable kernel addresses |
| `PG` | `long[]` | template | `GRVA` — guard RVAs |
| `PGL` | `int[]` | template | guard lengths |
| `PGN` | `int[]` | template | guard "stock" byte values |
| `PGW` | `int[]` | template | guard "patched" byte values |
| `PKB` | `String` | template | hex-encoded 632-byte kernel patch blob |
| `HENURL` | `String` | `InitXlet.patchLocalHenUrl()` | actual `jar:file:` URL of `hen.bin` |

`Hx1400.tableFault()` is the validator:

```java
if (P != null && P.length == 31) {
    if (PG != null && PGL != null && PGN != null && PGW != null) {
        if (PG.length == PGL.length && PG.length == PGN.length && PG.length == PGW.length) {
            return PKB != null && PKB.length() >= 64 && (PKB.length() & 1) == 0
                ? null : "the payload is missing";
        } else return "the patch tables disagree in length";
    } else return "a patch table is missing";
} else return "P missing or wrong length";
}
```

If `tableFault()` returns non-null, `Hx1400.run()` returns `"NO TABLE: ..."` immediately — the kernel exploit refuses to run without a complete table. This is why commands 1–43 must fully populate `P`, `PG`, `PGL`, `PGN`, `PGW`, `PKB` before command 44.

#### `run()` → `probe()`

`Hx1400.run()` calls `this.probe()` and stores the result in `Hx1400.result` (the static volatile `String`). The `probe()` method is the full 15-stage kernel exploit. It is called via opcode 29 (thread spawn) from the command 44 template, so it runs on a daemon thread and reports its status through `Hx1400.result` and the log buffer.

#### The "pending" frame

While `probe()` is running, the command 44 template polls `Hx1400.result`. If it is still `"not run"`, `Hx1400.LOGS()` returns the accumulated log, and the frame is:

```
status      = 0
instruction = 7
scalars     = []
vectors     = [ "not run" (7 bytes), LOGS() (≤131072 bytes) ]
```

`OfflineRenderer.isPendingHenResult()` recognises this exact shape. `InitXlet.Attempt` uses it to decide whether to poll again (up to 10 times / 45 s) or to accept the result.

#### The success frame

On success, `probe()` returns a `String` of the form:

```
HEN14V pid=<n> KBASE=<hex> round=<n> arms=<n> pair=<a>/<b> KVA=<hex> victim=<n>
       bridge=<n> walked=<n> APPLIED=<0|1> se11=<hex> UID=<n> uid2=<n>  *** ROOT ***
```

And the log buffer contains the `K`, `G`, `H`, `L`, `M` progress lines checked by `OfflineRenderer.verifyHenReport()`.

#### Why it is a separate class

Same reason as the other three: the preflight hash pins it independently. If the kernel stage were inlined into a template, the template's 562 KB HEN image would need to be re-hashed on every command. As a separate class, `Hx1400` is loaded once (opcode 8 in the command 44 template) and its 562 KB `hen.bin` is fetched once from `/bonk/hen.bin`.

### 6.6 Class Loading Order (Reconstructed)

Based on `verifyEarlyStatus()` and the command schedule, the load order is:

| Command | Class | Method | Returns |
|---|---|---|---|
| 4 | `AXZXkk` | `init()` | `{status=0}` |
| 5 | `EkIKZR` | `initImage()` | `{status=0, len=4}` |
| 6 | `EkIKZR` | `corruptGetRGB()` | `{status=0, len=0x7FFFFFFF}` |
| 7 | `J196rR` | `scan(markerWords)` | `{index}` in `[4, 16383]` |
| 8 | `J196rR` | `echo()` | 1024-byte snapshot |
| 9–27 | (templates) | (using `AXZXkk`/`J196rR` primitives) | various |
| 28 | (template) | code-window branch | (native) |
| 29–43 | (templates) | (using `AXZXkk`/`J196rR`) | various |
| 44 | `Hx1400` | `run()` | pending / success / failure |
| 45 (implicit) | `InitXlet.showHxDiagnostic()` | (on failure) | — |
| 46 | `InitXlet` ledger | terminal frame | `{status=0, inst=N, [], []}` |

`AXZXkk` is loaded first because `EkIKZR`'s `getRGB` corruption needs native access. `EkIKZR` is loaded second because `J196rR`'s scan needs the OOB array. `J196rR` is loaded third because commands 9–27 need its native call trampoline. `Hx1400` is loaded last (command 44) because it needs everything else already in place.

### 6.7 What is *not* in the class files

The helper classes do **not** contain:
- **The 40 command templates** — those are in `/bonk/local/templates/cmd-NNNN.bin`
- **The HEN image** — that is in `/bonk/hen.bin` (562240 bytes)
- **The metadata binaries** (`rules.bin`, `shapes.bin`, etc.) — those are in `/bonk/local/`
- **The kernel patch blob** — that is hex-encoded in `PKB`, set by an earlier command
- **The 205 relocation rules** — those are in `rules.bin`, not in the class files
- **The 81 write-site guards** — those are in `write-sites.bin`

The class files contain **only** the Java logic (native bridge, image corruption, memory scan, kernel exploit) and the constants needed to make that logic run. All *data* — including all kernel addresses — is in the metadata binaries and the templates, all of which are separately SHA-256 pinned.

---

## 7. Security Model & Trust Chain

The offline loader has **three layers** of integrity checking:

### Layer 1: `InitXlet` preflight (static)
Every asset is SHA-256 pinned and length-bounded **before** any code runs. If any hash fails, the loader never starts.

### Layer 2: `OfflineRenderer.renderCandidate` (per-command)
Every template is re-hashed against its `TemplatePin` **on each render**. Prior results must match their declared `Shape`. Relocation sources must be from earlier frames. Marker words must be unique and well-formed. Every write destination must be a plausible user/kernel pointer.

### Layer 3: `LocalResultLedger` (state machine)
Commands must be executed in the exact schedule order. Result-request IDs must be unique and in range. The final result must be `{status=0, scalarCount=0, vectorCount=0}`. `verifyHenReport()` requires all the specific success strings and rejects all the failure strings.

The net effect: **an attacker who can modify any asset must also update the SHA-256 pin in `InitXlet.class` and the `TemplatePin` in `template-hashes.bin`, which are themselves inside the signed JAR.** This is the standard Blu-ray trust model, and it is what makes the offline loader safe to distribute — you cannot silently weaponize a template. The `Hx1400` kernel stage is the only component that operates on live kernel memory. It is invoked exactly once, its result is verified against a fixed set of success strings, and any failure routes through `safeDown()` which quarantines (never frees) the corrupted structures so the kernel can be cleanly cold-rebooted.

### Trust Chain Summary

```
InitXlet.class (in signed JAR)
   │
   ├─ SHA-256 pins ──► /x/offline/AXZXkk.class
   ├─ SHA-256 pins ──► /x/offline/EkIKZR.class
   ├─ SHA-256 pins ──► /x/offline/J196rR.class
   ├─ SHA-256 pins ──► /x/offline/Hx1400.class
   ├─ SHA-256 pins ──► /bonk/hen.bin          (562240 B)
   └─ SHA-256 pins ──► /bonk/local/*.bin      (7 metadata files)
                       /bonk/local/templates/cmd-NNNN.bin  (40 files)
                        │
                        └─ TemplatePin.sha256 pins each template again
                           at every renderCandidate() call
```

The class files are the **bottom of the static trust chain** — they are pinned once at preflight and never re-verified (because they are never re-read). The templates are **re-verified on every render**, which is why a modified template cannot be smuggled in even if the attacker somehow gets past preflight. If any helper class is modified, the preflight fails and the Xlet refuses to start. If the modified class somehow loads (e.g., the preflight is bypassed by patching `InitXlet.class` itself), the JAR signature check will fail and the BD-J runtime will refuse to execute the Xlet.

---

## 8. Credits

- **Exploit authors / testers (per `NOTICE.txt`):** @ASaudidos on X
- **Additional credits:** Kat (@avieah), Slopp (@slopp), and Pigeon (@daburningpigeon) on Discord.
- **Confirmation:** *dr.yenyen* (14 attempts).
- **Repository maintainer:** OptiTronOffical.
- **Leaked:** XeMoisties

