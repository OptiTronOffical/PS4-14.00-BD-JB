# BD-JB Offline HEN Loader — Java Source README

This document explains the Java source files that make up the **offline Blu-ray Java (BD-J) HEN loader** for PS4 firmware 14.00. The code is a fully self-contained, hash-verified, one-shot exploit chain that runs entirely from a signed Blu-ray JAR — no network server is contacted during execution.

The source comprises six files:

| File | Role |
|---|---|
| `Cfg.java` | Build/server constants (mostly unused in offline mode) |
| `Screen.java` | On-disc AWT text display |
| `Boot.java` | Bytecode interpreter that executes the exploit command stream |
| `OfflineRenderer.java` | Template verifier + relocator (the security core) |
| `LocalResultLedger.java` | State machine enforcing command order and result shapes |
| `InitXlet.java` | Xlet entry point: preflight, template loading, one-shot driver |
| `Hx1400.java` | (separate file) Kernel exploit + HEN loader, invoked as command 44 |

The execution flow is: **`InitXlet` → verify everything → user presses X → `LocalResultLedger` walks the 40-command schedule → `OfflineRenderer` renders each command from a verified template → `Boot.exec()` interprets it → results fed back → command 44 triggers `Hx1400`.**

---

## 1. Architecture Overview

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

---

## 2. `Cfg.java` — Constants

```java
public static final String SERVER = "tester.f5rvoice.com";
public static final int    PORT   = 1992;
public static final String BUILD  = "6";
public static final String CID    = "c6";
public static final String CKEY   = "942a76d6946a34ce476bdabb09989f4c";
```

These constants were used by the **remote** version of the exploit (`Boot.run()` used to POST to `SERVER:PORT`). In the **offline** version, `Boot.EP` is set to the local disc root by `InitXlet.Attempt.run()`, and `Boot.SID = "offline"`. `CKEY` is still used as the seed for the ticket hash and XOR keystream — but in offline mode, `Boot.SK` is set to `null`, so the ticket becomes `mix(CKB, …)` without a session key.

`CKEY` is the hex form of `CKB` (see `Boot.hexb("942a76d6946a34ce476bdabb09989f4c")`), a 16-byte constant hardcoded into both `Boot` and `Cfg`. It is not a secret — it's an obfuscation constant.

---

## 3. `Screen.java` — Display

A `java.awt.Container` subclass that maintains a scrollback of up to 46 status lines and paints them in white-on-black. Used by `InitXlet` and `Boot` for user-visible progress.

Key methods:

- `Screen.println(String)` — append + repaint
- `Screen.println(String, repaint, overwriteLast)` — append with options
- `Screen.printStackTrace(Throwable)` — formats a stack trace into the scrollback
- `Screen.clear()` — clears the scrollback
- `paint(Graphics)` — draws the messages, re-entrant safe via `isPainting`/`isDirty` flags

In `Boot.run()`, it prints `.`, `+`, `-`, `!` as progress glyphs (used only in remote mode).

In `InitXlet`, the Xlet uses its own `Lines` inner class for the fancy orange title bar, but still calls `System.out.println("Bonk HEN local: " + msg)` for the log.

---

## 4. `Boot.java` — The Interpreter

`Boot` is a **binary command interpreter** written in pure Java. It is the component that actually performs reflection: loading classes, constructing objects, invoking methods, and reading/writing fields. It is also where the *result frame* format is defined.

### 4.1 Static state

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

Register indices are `<id> & 63` (or `& 127` for `R`). All opcodes address these register banks.

### 4.2 `exec(byte[])` — The interpreter loop

Reads a `short` count (`var38`), then loops reading one **opcode byte** followed by opcode-specific operands. Supported opcodes (case labels in `switch (var9)`):

| Opcode | Meaning |
|---|---|
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
int     instruction      (opcode index reached, or var4)
int     nScalars
long[]  scalars
int     nVectors
for each: int len; byte[] data
int     nNotes
for each: int len; byte[] note (UTF-8, e.g. exception message)
```

This frame is exactly what `OfflineRenderer.frame()` parses.

### 4.3 `io(String, byte[])` — HTTP transport (remote mode only)

POSTs to `EP + path`, sets header `X-Ticket: c6.<seq>.<hash>`, XOR-encrypts the body with a keystream derived from `CKB`/`SK`/random, then reads and decrypts the response. **In offline mode, this method is never called** — `InitXlet` sets `Boot.EP = discRoot`, but the `Attempt` loop uses pre-verified templates from the JAR, not `io()`.

### 4.4 `ticket()` and `xor()` — Crypto

`ticket(header, seq, len)`:

```
buf = header[8] || seq[4 BE] || SID[UTF-8] || len[4 BE]
ticket = "c6." + seq + "." + hex16( mix(CKB||SK, buf) )
```

`mix()` is a simple LCG-based hash: `x = x*KSB[1] + byte + KSB[2]` over the concatenated buffers. `xor()` re-seeds the same LCG with `SK` + `buf` and emits 8-bit keystream bytes at bit offset `KSB[3] & 63`.

Both are **obfuscation, not cryptography** — they were only used to prevent casual network sniffing in the remote version. In offline mode, they are unused.

### 4.5 `L` — The custom classloader

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

This is how the exploit loads helper classes (`J196rR`, `Hx1400`, etc.) from raw byte arrays that arrived over the wire. In offline mode, they arrive from `/x/offline/*.class` via the `DVBClassLoader` fallback path (`var121 != null` branch), because the JAR-signed copy is used instead. The `InitXlet.checkHash()` preflight verified those byte arrays first.

---

## 5. `OfflineRenderer.java` — The Security Core

This is the most important class to understand. It is responsible for **turning a verified template into a runnable command byte stream**.

### 5.1 The metadata binaries

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

### 5.2 `renderCandidate()` — The hot path

```java
static byte[] renderCandidate(int command, byte[] template,
                              Frame[] frames, Rule[] rules, Shape[] shapes,
                              Guards guards, TemplatePin[] pins,
                              MarkerPlan markerPlan, int[] markerWords,
                              WriteSite[] writeSites) throws Exception
```

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
   - Compute `value = value(rule, frames[request], frames)`:
     - `kind 0`: `frames[request].scalars[slot]`
     - `kind 1`: `qword(frames[request].vectors[slot], high, low)` (big-endian 64-bit from two 32-bit words)
     - `kind 2`: `earlyIndexes(frames[8].vectors[0])[slot]` (only valid for `request == 8`)
   - Add `rule.addend` (overflow-checked).
   - Write big-endian into `template[offset..offset+8]`.
4. `command == 2`: `patchMarkerWords()` — write `markerWords[i]` at `markerPlan.offsets[i]`, after verifying the surrounding `(17, ?, 3, i, 0,0,0,0)` structure.
5. `verifyWriteDestinations()` — every `WriteSite` in the command must have a pointer-shaped value in `[0x100000000, 0x7FFFFFFFFFFFFFF7]` after relocation.
6. Return the patched byte array.

The zero-byte and shape checks are what make this safe: a malicious or corrupted template **cannot** introduce arbitrary pointers into the kernel. Every pointer written to the kernel must come from a prior *verified* result frame.

### 5.3 `verifyHenReport()` — The HEN success oracle

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

Additionally, if both `K sysent[11].sy_call=` and `APPLIED=true` appear on the same line, and both `L set to ` and `ok=true` appear on the same line, the log is considered "kernel + ShellCore readback present" — otherwise the report is rejected.

`lineContainsBoth()` and `contains()` do byte-level substring search on the raw vectors, so they work even if the vectors contain non-ASCII.

### 5.4 `isPendingHenResult()` — The retry oracle

When `Hx1400` is still running (command 44 in progress), its result frame looks like:

```
status      = 0
instruction = 7
scalars     = []
vectors     = [ "not run" (7 bytes), <log> (≤128 KiB) ]
```

`InitXlet.Attempt` uses this to distinguish "Hx still working" from "Hx finished or crashed". If it never resolves within 45 s or 10 polls, the attempt is aborted.

### 5.5 `freshMarkerWords()`

Generates 4 cryptographically random `int`s, each non-zero, not `-1`, not `-2`, and all distinct. `-1` and `-2` are reserved because `Hx1400` uses them as special marker values in `showHxDiagnostic`. Command 2 writes these into a template, and command 7 scans for them in kernel memory (via `Hx1400`'s early memory helpers) — this is how the exploit confirms it has the correct memory layout before touching anything important.

---

## 6. `LocalResultLedger.java` — The State Machine

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

---

## 7. `InitXlet.java` — Entry Point + Preflight

This is the `javax.tv.xlet.Xlet` implementation — the class the BD-J runtime actually launches.

### 7.1 `startXlet()` — Preflight kickoff

```java
this.preflightStarted = true;
new Thread(this, "Bonk HEN local asset preflight").start();
```

Only one preflight thread is ever started. If `startXlet()` is called again (e.g. after `pauseXlet()`), the preflight is skipped.

### 7.2 `run()` — Preflight (executes in the thread)

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
   - Verify length against `META[i][2]` and SHA-256 against `META[i][1]`.
8. Parse metadata via `OfflineRenderer.rules/shapes/guards/schedule/templatePins/markerPlan/writeSites`.
9. Cross-check: `pins.length == schedule.length == 40`, and `pins[i].command == schedule[i].command` for every `i`.
10. For every `TemplatePin`:
    - Load `/bonk/local/templates/cmd-NNNN.bin` (zero-padded to 4 digits)
    - Must be `jar:file:` and ≤ `pin.length` bytes, and exactly `pin.length` bytes
    - Verify SHA-256 == `pin.sha256`
11. Print `LOCAL PREFLIGHT PASS. Press X once to run.` and `One attempt only; restart after any failure.`

Any failure prints `PREFLIGHT FAIL: <exception>` and leaves `ready = false`. The Xlet then refuses to run.

### 7.3 `keyPressed(KeyEvent)` — One-shot trigger

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

### 7.4 `Attempt.run()` — The actual exploit driver

```java
Boot.EP  = InitXlet.this.discRoot;   // file: URL of disc root
Boot.SID = "offline";
Boot.PAR = InitXlet.class.getClassLoader();
Boot.SK  = null;                     // no session key
```

Then:

```java
int[] markerWords = OfflineRenderer.freshMarkerWords();
LocalResultLedger ledger = new LocalResultLedger(
    schedule, rules, shapes, guards, pins, markerPlan, markerWords, writeSites);

for (int i = 0; i < schedule.length; i++) {
    if (stopped) throw new IOException("Xlet stopped");
    int cmd = ledger.currentCommand();
    byte[] tmpl = InitXlet.this.loadTemplate(cmd, pins[i].length);
    byte[] rendered = ledger.renderCurrent(tmpl);
    if (cmd == 32) rendered = patchLocalHenUrl(rendered, henUrl);   // see §7.5
    if (cmd == 28) line("Native code-window branch verified");
    if (cmd == 43) line("Entering one-shot Hx/HEN stage");
    if (cmd != 44) {
        byte[] result = Boot.exec(rendered);
        if (schedule[i].nextResultRequest == 65535) ledger.finishTerminal(result);
        else                                          ledger.acceptResult(result);
    } else {
        // Command 44 (Hx1400) may take up to 45 s; poll up to 10 times
        long deadline = System.currentTimeMillis() + 45000;
        for (int poll = 1; poll <= 10; poll++) {
            byte[] result = Boot.exec(rendered);
            if (OfflineRenderer.isPendingHenResult(result)) {
                line("Hx still running; read-only status " + poll);
                if (poll == 10 || System.currentTimeMillis() > deadline)
                    throw new IOException("Hx status remained pending");
                continue;
            }
            ledger.acceptResult(result);
            break;
        }
    }
    if (cmd in {9, 27, 32, 44, 46}) line("Command " + cmd + " result accepted");
}

if (!ledger.complete()) throw new IOException("local result ledger incomplete");
line("Hx reports HEN start. Verify menu and kernel R/W separately.");
```

Key points:

- **Command 44 is special-cased.** It may return the "pending" frame, in which case the driver polls again (up to 10 times or 45 s). `Hx1400` runs `hen.bin` on a background thread and returns its status while it runs.
- **`ledger.complete()`** is the final gate — all 40 commands must have been processed, and result 46 must have arrived with the expected `verifyHenReport()` shape.
- The final message is **deliberately cautious**: `Verify menu and kernel R/W separately` — the Hx log is not trusted as the final word.
- **On failure**, `Attempt` prints the class name and message, optionally calls `showEarlyStatus(cmd, result)` for commands 4/5/6, and optionally calls `showHxDiagnostic(result)` for command 44. It then prints `No retry in this boot. Cold reboot before another attempt.`

### 7.5 `patchLocalHenUrl()` — The one runtime patch

The original `Hx1400` command 32 template contains the sentinel URL:

```
file:/__bonk___offline_missing__/hen.bin
```

This is a **45-byte placeholder**. `patchLocalHenUrl` replaces it with the actual JAR URL (`jar:file:...!/bonk/hen.bin`) so `Hx1400`'s `HEN_URL` field points at the signed JAR resource, not a filesystem path.

It does this by:
1. Scanning `data[13..]` for the sentinel bytes.
2. Confirming the sentinel is unique.
3. Confirming it is preceded by the exact framing: `0x05` opcode, 8-byte `reg` = 45, 4-byte `len` = 45.
4. Copying the new URL in and updating the 4-byte length field.
5. Rejecting any new length > 65535.

The 45-byte length matches `Hx1400.HEN_URL = "file:/__bonk___offline_missing__/hen.bin"` exactly (it's the Java string literal). The sentinel is *never* actually fetched — it just has to be *there* so the template hash matches, then it's patched away.

This is the only runtime mutation of a verified template, and it is tightly constrained.

### 7.6 `showEarlyStatus()` and `showHxDiagnostic()` — Failure diagnostics

- `showEarlyStatus(cmd, frame)`: for commands 4, 5, 6 — reads the helper status ints from `vector[0]` and prints them.
- `showHxDiagnostic(frame)`: for command 44 — expects 2 vectors (`summary`, `log`), both within bounds. Prints the summary (truncated to 96 chars) and extracts up to 3 interesting log lines (matching `STOP`, `no pair`, `K NOT fully applied`, `G2 sysfilt_ops`, `safeDown threw`, `H7 pthread_create`, `H7 ***`, `L set `).

Both are wrapped in `try/catch` so a diagnostic failure never masks the real error.

---

## 8. `Hx1400.java` — The Kernel Stage (cross-reference)

`Hx1400` is loaded and invoked by command 44. It is documented in the earlier README and is out of scope for this document, but its relationship to the offline driver is:

- It reads `P`, `PG`, `PGL`, `PGN`, `PGW`, `PKB` static fields which are set by earlier commands (the **patch tables** and **heap marker blob**).
- It assumes `Boot.EP` and `Boot.SID` are already set — this is why `Attempt.run()` sets them before the loop.
- It reads `HEN_URL` (patched at command 32) and fetches `hen.bin` from the JAR.
- It reports its progress as a `Frame` with `instruction = 7` and `vector[0] = "not run"` until it succeeds or fails — this is what `isPendingHenResult()` detects.

The driver treats command 44's result frame as terminal (`nextResultRequest == 65535` is false for command 44 — it has a normal result request). `verifyHenReport()` is called when that result arrives.

---

## 9. Execution Flow (End to End)

```
User inserts disc
   │
   ▼
BD-J runtime launches org.bdj.InitXlet
   │
   ▼
startXlet() → new Thread(run())     [preflight]
   │
   ├─ Verify own class is in a disc JAR
   ├─ Verify 4 helper classes in /x/offline/
   ├─ Verify /bonk/hen.bin (562240 B, SHA-256)
   ├─ Verify 7 metadata files
   ├─ Parse metadata → rules/shapes/guards/schedule/pins/markerPlan/writeSites
   ├─ Cross-check schedule vs pins
   ├─ Verify 40 command templates against their pins
   └─ Print "LOCAL PREFLIGHT PASS. Press X once to run."
   │
   ▼
User presses X → new Thread(Attempt)   [one-shot]
   │
   ├─ Set Boot.EP/SID/PAR/SK
   ├─ Generate 4 random marker words
   ├─ Create LocalResultLedger
   │
   ├─ for i in 0..39:
   │    cmd = schedule[i].command
   │    tmpl = loadTemplate(cmd, pins[i].length)
   │    rendered = ledger.renderCurrent(tmpl)
   │         ├─ verify template hash
   │         ├─ verify prior result shapes
   │         ├─ verify marker / native / alias / code-window checks (per cmd)
   │         ├─ apply 205 relocation rules
   │         ├─ patch marker words (cmd 2)
   │         └─ verify 81 write-destinations
   │    if cmd == 32: patch HEN URL to JAR resource
   │    if cmd == 44: poll up to 10× / 45s
   │    result = Boot.exec(rendered)
   │    ledger.acceptResult(result)  or  ledger.finishTerminal(result)
   │
   ├─ if !ledger.complete(): "local result ledger incomplete"
   └─ Print "Hx reports HEN start. Verify menu and kernel R/W separately."
   │
   ▼
Done. Cold reboot required before another attempt.
```

---

## 10. Security Model

The offline loader has **three layers** of integrity checking:

### Layer 1: `InitXlet` preflight (static)
Every asset is SHA-256 pinned and length-bounded **before** any code runs. If any hash fails, the loader never starts.

### Layer 2: `OfflineRenderer.renderCandidate` (per-command)
Every template is re-hashed against its `TemplatePin` **on each render**. Prior results must match their declared `Shape`. Relocation sources must be from earlier frames. Marker words must be unique and well-formed. Every write destination must be a plausible user/kernel pointer.

### Layer 3: `LocalResultLedger` (state machine)
Commands must be executed in the exact schedule order. Result-request IDs must be unique and in range. The final result must be `{status=0, scalarCount=0, vectorCount=0}`. `verifyHenReport()` requires all the specific success strings and rejects all the failure strings.

The net effect: **an attacker who can modify any asset must also update the SHA-256 pin in `InitXlet.class` and the `TemplatePin` in `template-hashes.bin`, which are themselves inside the signed JAR.** This is the standard Blu-ray trust model, and it is what makes the offline loader safe to distribute — you cannot silently weaponize a template.

The `Hx1400` kernel stage is the only component that operates on live kernel memory. It is invoked exactly once, its result is verified against a fixed set of success strings, and any failure routes through `safeDown()` which quarantines (never frees) the corrupted structures so the kernel can be cleanly cold-rebooted.

---

## 11. Quick Reference: Notable Constants

| Constant | Value | Where | Meaning |
|---|---|---|---|
| `HEN_SIZE` | 562240 | `InitXlet`, `Hx1400` | Exact size of `hen.bin` |
| `HEN_SENTINEL` | `file:/__bonk___offline_missing__/hen.bin` | `InitXlet`, `Hx1400` | 45-byte placeholder patched at runtime |
| `HEN_MAP` | 39996882944 (`0x950000000`) | `Hx1400` | Preferred mmap address for HEN |
| `HEN_SPAN` | 589824 | `Hx1400` | HEN mapping size |
| `HEN_ENTRY` | `0x?` | `Hx1400` | Entry offset within `hen.bin` |
| `SE11` | `P[0]` | `Hx1400` | `sysent[11]` kernel address |
| `KEXECG` | `P[1]` | `Hx1400` | Patch target for syscall 11 |
| `MARKV` | 5570198814713989681 | `Hx1400` | Sentinel used to prove ROP chain ran |
| `NSPRAY` | 144 | `Hx1400` | Socket spray count |
| `CANDS` | 30 ints | `Hx1400` | kqueue victim slot candidates |
| `ROUNDS` | 6 | `Hx1400` | Pack-and-sweep rounds |
| `MAXARM` | 24 | `Hx1400` | Max blind EV_ADD arms per round |
| `schedule[39]` | `(46, 65535)` | `OfflineRenderer` | Final terminal command |
| `templateCount` | 40 | `OfflineRenderer` | Number of command templates |
| `ruleCount` | 205 | `OfflineRenderer` | Number of relocation rules |
| `writeSiteCount` | 81 | `OfflineRenderer` | Number of write-destination checks |
| `markerSiteCount` | 4 | `OfflineRenderer` | Number of marker words |
| `guardNativeDeltas` | 10 | `OfflineRenderer` | Number of native symbol deltas |
| `guardWindows` | 3 | `OfflineRenderer` | Number of code windows |

---

## 12. Glossary

| Term | Meaning |
|---|---|
| **Xlet** | A Java application for BD-J, analogous to an applet |
| **BD-J** | Blu-ray Disc Java — the runtime that executes JAR content on a Blu-ray player (including the PS4) |
| **Frame** | `Boot.exec()`'s result tuple: `(status, instruction, scalars[], vectors[], notes[])` |
| **Shape** | The expected structure of a frame (status, instruction, scalar count, vector lengths) |
| **Template** | A pre-verified command byte stream in `/bonk/local/templates/` |
| **Relocation** | Patching an 8-byte value from a previous frame into a template at a fixed offset |
| **Pin** | A SHA-256 hash + length that identifies a template |
| **Marker words** | 4 random 32-bit values written by command 2 and searched for by command 7, proving memory layout |
| **Write site** | A location in a command 1..28 template that must contain a pointer-shaped value after relocation |
| **Code window** | A 16-byte snapshot of native code that must match a pinned hash (proves the exploit reached the right code) |
| **HEN** | Homebrew ENabler — the kernel payload loaded by `Hx1400` |
| **KEX** | Kernel exploit — the vulnerability that provides kernel read/write (in this chain, the AIO/kqueue race) |

---

*This README is generated from reading the six `.java` files (`Cfg`, `Screen`, `Boot`, `OfflineRenderer`, `LocalResultLedger`, `InitXlet`) plus `Hx1400.java`. It describes the offline mode of operation; the remote mode (`Boot.run()` + `io()`) is present in the code but is not exercised by the offline Xlet.*
