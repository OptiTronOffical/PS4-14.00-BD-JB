# BD-JB Offline HEN Loader — Helper Class README

This document describes the four **compiled helper classes** that live in `/x/offline/` on the Blu-ray disc and are loaded by the command chain at runtime:

| Class | File | SHA-256 |
|---|---|---|
| `AXZXkk` | `/x/offline/AXZXkk.class` | `2362afc9efa07a688169b9121d9e5f8bdacbead58f87cafd23fdddffea58b429` |
| `EkIKZR` | `/x/offline/EkIKZR.class` | `df30bfbd25c3cdbe67965d907d575efb562738cf29c258895a698ff2ef525137` |
| `J196rR` | `/x/offline/J196rR.class` | `f59953027867d15d1434179991feeeb94bf274422f742e739067fd65400a264d` |
| `Hx1400` | `/x/offline/Hx1400.class` | `e597710078e9afab5b24f4c22ea31fe7e5a5643b602b06c2866273b61c9c57e9` |

Unlike the `.java` files described in the previous README, these are **not open source in this repository** — only `Hx1400.java` was decompiled and made available. For the other three, this document describes what they do, as reconstructed from:

1. The **hash pins** in `InitXlet.java` (lines `CLASSES[][]`)
2. The **preflight** and **command schedule** in `InitXlet.java`
3. The **`verifyEarlyStatus()`** checks in `OfflineRenderer.java`
4. The **`Hx1400.java` source**, which references some of their symbols
5. The **`Boot.exec()` opcode semantics** that the templates use to load them

The names are deliberately opaque (`AXZXkk`, `EkIKZR`, `J196rR`) to make static analysis harder. `Hx1400` is the only one whose name is descriptive (Hx = kernel exploit stage, 1400 = firmware).

---

## 1. The Two-Phase Verification Model

Every helper class is verified **twice** before it is ever loaded:

### Phase 1 — Preflight hash check (`InitXlet.run()`)

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

- Bounds: **1 to 100000 bytes** per class
- Hash: **SHA-256**
- If any check fails → `PREFLIGHT FAIL` → Xlet refuses to run

This is the **static** trust anchor. The class bytes on disc must match the hashes compiled into `InitXlet.class` inside the signed JAR.

### Phase 2 — Runtime load (`Boot.exec()` opcodes 7 or 8)

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

In offline mode, `EP = discRoot` and `SID = "offline"`, so the DVB fallback resolves to `file:<discRoot>/x/offline/`. The custom loader `Boot.L` tries `defineClass` first; if that throws (e.g., because the class was already loaded by a different loader), the DVB fallback loads it from the same directory.

The `note` field (`d1:<exception>`) records any fallback that occurred, visible in the result frame's notes.

---

## 2. `AXZXkk` — Native Access Bootstrap

**SHA-256:** `2362afc9efa07a688169b9121d9e5f8bdacbead58f87cafd23fdddffea58b429`

**Used by:** command 4 ("helper setup")

### Purpose

`AXZXkk` is the entry point for **crossing from the Java sandbox into native code**. It is the class that the command templates refer to as "helper" in the sense of "the helper that gets us a native handle". The `verifyEarlyStatus()` check for command 4 is:

```java
if (command == 4) {
    if (frame.vectors.length != 2
        || frame.vectors[0].length != 4
        || readInt(frame.vectors[0], 0) != 0)
        throw new IOException("helper setup failed");
}
```

So command 4 returns `vector[0] = {0, 0, 0, 0}` — a single 32-bit status of 0. The second vector is not checked for command 4, but it exists.

### What it does (reconstructed)

Based on the `Hx1400.java` code, which uses the same native-access technique, `AXZXkk` performs the **`sun.misc.Unsafe` + `ClassLoader$NativeLibrary.findEntry` bootstrap**:

```java
// From Hx1400.java, describing the same technique AXZXkk uses
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

### Why it is a separate class

Because the preflight hash check in `InitXlet` requires it to be. The command templates reference `AXZXkk` by name when they need native access — if `AXZXkk` were inlined into `Hx1400`, the templates would need to hash-check the full kernel exploit on every command. Keeping it separate lets the early commands (4–43) get native access without loading `Hx1400`.

---

## 3. `EkIKZR` — Image / `getRGB` Array-Length Corruption

**SHA-256:** `df30bfbd25c3cdbe67965d907d575efb562738cf29c258895a698ff2ef525137`

**Used by:** commands 5 ("helper image setup") and 6 ("getRGB array-length corruption")

### Purpose

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

This is the **userland OOB primitive**. After command 6 succeeds, `EkIKZR` exposes a method (likely `int[] getOobArray()` or similar) that returns a `int[]` whose `.length` is `Integer.MAX_VALUE` but whose backing memory is only 16 bytes. Reading past index 3 reads adjacent JVM heap; writing past index 3 writes it.

### The `getRGB` bug

The specific bug is a well-known issue in older Java2D implementations:

```java
BufferedImage img = new BufferedImage(1, 1, BufferedImage.TYPE_INT_ARGB);
int[] pixels = ((DataBufferInt) img.getRaster().getDataBuffer()).getData();
// pixels.length == 1
img.getRGB(0, 0, 1, 1, pixels, 0, 1);   // normally writes 1 int
```

A type-confusion variant lets you pass a `long`-typed object where the JVM expects an `int[]` array header, and the `getRGB` implementation writes a `length` field derived from `width * height` without bounds-checking the destination array — effectively writing `0x7FFFFFFF` into the `int[]` length field of the caller's array.

`EkIKZR` packages this up as a method that returns the corrupted array, and the command templates use it via opcode 14 (method invocation) to seed the OOB access used by `J196rR`.

---

## 4. `J196rR` — Marker Scan and Memory Pivot

**SHA-256:** `f59953027867d15d1434179991feeeb94bf274422f742e739067fd65400a264d`

**Used by:** commands 7 (marker scan), 8 (marker echo), and indirectly by commands 9–27

### Purpose

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

- **Command 7** returns `{index}` where `index` is the word-offset of the first marker found. It must be in `[4, 16383]` — i.e., the marker must be found within 64 KiB of the array start but not at the very start (which would mean the OOB is not really working).
- **Command 8** returns a **1024-byte** buffer containing 256 `int` values starting at the marker offset. The first 4 `int`s must match `markerWords` exactly. The rest is whatever is in memory at that offset — this is the **early heap snapshot** used by `earlyIndexes()` in `OfflineRenderer` to derive pointer offsets.

### What `earlyIndexes()` does with command 8

```java
private static long[] earlyIndexes(byte[] vec) throws IOException {
    if (vec.length != 1024) throw new IOException("early vector changed");
    long v13_14 = qword(vec, 13, 14);              // 64-bit value from ints 13 and 14
    long v11_12 = qword(vec, 11, 12);
    long v29_28 = qword(vec, 29, 28);
    long base   = v13_14;
    long idxA   = (v11_12 - base - 24) / 4;
    long idxB   = (v29_28 - base - 8)  / 4;
    // must be word-aligned and in [0, 134217728)
    return new long[]{ idxA, idxA + 40, idxA + 41, idxB };
}
```

So the 1024-byte snapshot at the marker offset contains, at word offsets 11–14 and 28–29, **pointer-shaped values whose differences reveal the layout of the JVM heap**. `J196rR` deliberately snapshots this region because it contains known pointers (likely `Unsafe` field offsets, `int[]` array headers, or `String` interned pointers) that let `OfflineRenderer` compute:

- `idxA` — a 32-bit word index into a known structure
- `idxA + 40` and `idxA + 41` — sibling fields
- `idxB` — a second index into a different structure

These four values become the **relocation sources** for commands 9–27 via `Rule.kind == 2`:

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

### The `nativeCall` bridge

`J196rR` also provides the native call trampoline that `AXZXkk` set up. The `nativeCall(long, int[])` native method is declared in `Hx1400.java` but the JNI registration (via `NativeLibrary` handle 8193) is what `J196rR` initialises. This is the mechanism by which commands 9–27 can call `dlsym`, `getpid`, `socket`, `sendto`, `fstat`, `sysctl`, `mmap`, `kqueue`, `kevent`, etc. — all of which are needed before `Hx1400` can do the AIO/kqueue race.

---

## 5. `Hx1400` — Kernel Exploit + HEN Loader

**SHA-256:** `e597710078e9afab5b24f4c22ea31fe7e5a5643b602b06c2866273b61c9c57e9`

**Used by:** command 44 (the one-shot kernel stage)

Full source is in `Hx1400.java` (decompiled). This document only summarises its interface to the offline loader.

### Required static fields

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
```

If `tableFault()` returns non-null, `Hx1400.run()` returns `"NO TABLE: ..."` immediately — the kernel exploit refuses to run without a complete table. This is why commands 1–43 must fully populate `P`, `PG`, `PGL`, `PGN`, `PGW`, `PKB` before command 44.

### `run()` → `probe()`

`Hx1400.run()` calls `this.probe()` and stores the result in `Hx1400.result` (the static volatile `String`). The `probe()` method is the full 15-stage kernel exploit described in the earlier README. It is called via opcode 29 (thread spawn) from the command 44 template, so it runs on a daemon thread and reports its status through `Hx1400.result` and the log buffer.

### The "pending" frame

While `probe()` is running, the command 44 template polls `Hx1400.result`. If it is still `"not run"`, `Hx1400.LOGS()` returns the accumulated log, and the frame is:

```
status      = 0
instruction = 7
scalars     = []
vectors     = [ "not run" (7 bytes), LOGS() (≤131072 bytes) ]
```

`OfflineRenderer.isPendingHenResult()` recognises this exact shape. `InitXlet.Attempt` uses it to decide whether to poll again (up to 10 times / 45 s) or to accept the result.

### The success frame

On success, `probe()` returns a `String` of the form:

```
HEN14V pid=<n> KBASE=<hex> round=<n> arms=<n> pair=<a>/<b> KVA=<hex> victim=<n>
       bridge=<n> walked=<n> APPLIED=<0|1> se11=<hex> UID=<n> uid2=<n>  *** ROOT ***
```

And the log buffer contains the `K`, `G`, `H`, `L`, `M` progress lines checked by `OfflineRenderer.verifyHenReport()`.

### Why it is a separate class

Same reason as the other three: the preflight hash pins it independently. If the kernel stage were inlined into a template, the template's 562 KB HEN image would need to be re-hashed on every command. As a separate class, `Hx1400` is loaded once (opcode 8 in the command 44 template) and its 562 KB `hen.bin` is fetched once from `/bonk/hen.bin`.

---

## 6. Class Loading Order (Reconstructed)

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

---

## 7. What is *not* in the class files

The helper classes do **not** contain:

- **The 40 command templates** — those are in `/bonk/local/templates/cmd-NNNN.bin`
- **The HEN image** — that is in `/bonk/hen.bin` (562240 bytes)
- **The metadata binaries** (`rules.bin`, `shapes.bin`, etc.) — those are in `/bonk/local/`
- **The kernel patch blob** — that is hex-encoded in `PKB`, set by an earlier command
- **The 205 relocation rules** — those are in `rules.bin`, not in the class files
- **The 81 write-site guards** — those are in `write-sites.bin`

The class files contain **only** the Java logic (native bridge, image corruption, memory scan, kernel exploit) and the constants needed to make that logic run. All *data* — including all kernel addresses — is in the metadata binaries and the templates, all of which are separately SHA-256 pinned.

---

## 8. Trust Chain Summary

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

The class files are the **bottom of the static trust chain** — they are pinned once at preflight and never re-verified (because they are never re-read). The templates are **re-verified on every render**, which is why a modified template cannot be smuggled in even if the attacker somehow gets past preflight.

If any helper class is modified, the preflight fails and the Xlet refuses to start. If the modified class somehow loads (e.g., the preflight is bypassed by patching `InitXlet.class` itself), the JAR signature check will fail and the BD-J runtime will refuse to execute the Xlet.

---

## 9. Glossary

| Term | Meaning |
|---|---|
| **Helper class** | One of the four `.class` files in `/x/offline/` |
| **Native bridge** | The `ClassLoader$NativeLibrary` + `Unsafe` + `nativeCall` mechanism that lets Java code call arbitrary native functions |
| **OOB array** | An `int[]` whose `length` field has been corrupted to `Integer.MAX_VALUE`, giving out-of-bounds heap access |
| **Marker words** | 4 random 32-bit values written by command 2 and located by `J196rR.scan()` |
| **Early snapshot** | The 1024-byte region returned by command 8, used to derive heap offsets |
| **`earlyIndexes()`** | The function that extracts 4 pointer-relative indices from the early snapshot |
| **`nativeCall(long, int[])`** | The JNI method declared in `Hx1400` that invokes a native function pointer with up to 7 args |
| **`PKB`** | Hex-encoded 632-byte kernel patch blob applied to `sysent[11]` |
| **`GRVA` / `GNOW` / `GNEW`** | Guard RVA array, expected stock bytes, expected patched bytes |
| **`tableFault()`** | `Hx1400`'s validator that refuses to run without a complete patch table |

---

*This README is reconstructed from `InitXlet.java`, `OfflineRenderer.java`, `Boot.java`, and `Hx1400.java`. Only `Hx1400` has published source; the behavior of `AXZXkk`, `EkIKZR`, and `J196rR` is inferred from their hash pins, their command-chain role, the `verifyEarlyStatus()` checks, and the primitives that `Hx1400` uses. Where the inference is not certain, the text says so.*
