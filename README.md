

# PS4 14.00 BD-JB (Blu-ray Java Jailbreak)

A Blu-ray Disc (BD-J) jailbreak image for PlayStation 4 consoles running **System Software 14.00**. This project leverages the BD-JB vulnerability to achieve userland code execution and load a Homebrew Enabler (HEN) via a crafted Blu-ray disc.

> **Status:** Confirmed working by *dr.yenyen* after 14 attempts (per repository description).

---

## Table of Contents
1. [Repository Structure (`/src`)](#repository-structure-src)
2. [What It Does](#what-it-does)
3. [Important Notice & Safety](#important-notice--safety)
4. [How to Use (General BD-JB Workflow)](#how-to-use-general-bd-jb-workflow)
5. [Credits](#credits)

---

## Repository Structure (`/src`)

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

## What It Does

According to `src/NOTICE.txt`, the payload is a **“Bonk HEN 14.00 local HEN review candidate.”** The execution flow is:

1.  The PS4 mounts the Blu-ray disc and automatically starts the BD-J Xlet defined in `BDMV/`.
2.  The Xlet **validates** local classes, metadata, and the HEN environment.
3.  It **waits for a single “X” button press** on the controller.
4.  It calls the original `Boot.exec` with a **scrubbed, result-gated command sequence** to load the HEN.
5.  **No HTTP server** is required; the entire exploit runs from the disc.

The presence of `Hx1400.class` indicates a stage that directly interacts with the PS4 kernel (or attempts to) after the initial Java sandbox escape. On firmware 14.00, the kernel portion is considered the most fragile and experimental part of the chain.

---

## Important Notice & Safety

The following warnings are taken directly from `src/NOTICE.txt` and are **critical** for anyone attempting to use this disc:

> *“The Hx stage may write unknown kernel objects even on a failed pairing run. There is no automatic retry. After a failure, **cold-reboot the console** before any new attempt. A displayed Hx success report must be independently verified on the console.”*

**Key safety rules:**
- **Do not retry** the exploit immediately after a failure. Always perform a **full cold reboot** (power off completely, not just rest mode) before inserting the disc again.
- The HEN loading is **result-gated**, meaning the Xlet expects a specific success signal from the kernel stage. If the signal is missing, the process may leave the system in an unstable state.
- Because the kernel stage is experimental for firmware 14.00, **assume a non-trivial risk of kernel panic or system instability** on each attempt.
- The success message shown on screen by the Hx stage **must be verified independently** (e.g., by checking the HEN menu or payload loader) — do not trust the on-screen report alone.

---

## How to Use (General BD-JB Workflow)

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
    - The BD-J Xlet should start automatically from the disc’s “movie” menu. If it does not, navigate to the disc icon on the XMB and launch it manually.
    - Follow the on-screen prompt: **press the “X” button once**.
    - Observe the result. If the HEN does not load, **perform a cold reboot** and try again later. Do not spam attempts without rebooting.

> **Note:** The original repository includes a pre-built `hen.iso` in the root. You can use that directly if you do not need to modify the source. The `/src` folder is provided for inspection and rebuilding.

---

## Credits

- **Exploit authors / testers (per `NOTICE.txt`):** Kat (@avieah), Slopp (@slopp), and Pigeon (@daburningpigeon) on Discord.
- **Confirmation:** *dr.yenyen* (14 attempts).
- **Repository maintainer:** OptiTronOffical.
- **Leaked:** XeMoisties

---

[![Screenshot-2026-10-30-21-58-47.png](https://i.postimg.cc/HLWjZzNG/Screenshot-2026-10-30-21-58-47.png)](https://postimg.cc/WdK2tGjS)
<a href="https://postimages.org/" target="_blank"><img src="https://i.postimg.cc/3JyRzW9y/HV5e-Mtv-W0AAz2P7.png" alt="HV5e-Mtv-W0AAz2P7"></a><br><br>
<a href="https://postimages.org/" target="_blank"><img src="https://i.postimg.cc/ZK9RX0jd/image.webp" alt="image"></a><br><br>
