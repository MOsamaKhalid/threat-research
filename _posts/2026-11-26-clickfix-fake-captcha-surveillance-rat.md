---
title: "ClickFix on a Compromised Restaurant Site: Polymorphic Loader to Syscall-Evading Surveillance RAT"
date: 2026-11-26 10:00:00 +0500
family: ClickFix
summary: "A hospitality website served a fake Cloudflare check that talked visitors into running the malware themselves. Three stages, no exploit anywhere in the chain, and a RAT that screenshots, reads the clipboard and calls Windows by syscall to stay under EDR."
tags: [ClickFix, Social Engineering, Process Injection, RAT, Threat Hunting]
description: "Technical breakdown of a ClickFix campaign delivering a three-stage surveillance RAT, with IOCs, hunting leads and ATT&CK mapping."
---

<div class="stats">
  <div><b>3</b><span>stages in the chain</span></div>
  <div><b>0</b><span>exploits used</span></div>
  <div><b>~255 KB</b><span>final payload</span></div>
  <div><b>2</b><span>injection targets</span></div>
</div>

## Executive summary

I opened a restaurant website to place an order. It was serving a ClickFix campaign.

**There is no exploit anywhere in this chain.** No zero-day, no drive-by, no vulnerable
plugin in the delivery path. The site displays a fake Cloudflare "verify you are human"
page that instructs the visitor to press `Win + R`, paste a command and press Enter.
Every technical control is bypassed because a person was persuaded to run the command
on their own machine.

**Stage 1 cannot be caught by hash.** The PowerShell loader is generated per request
with randomised variable names and dynamically produced payload URLs. Any IOC list
that leads with the loader's hash is stale before it is published.

**The final payload is built to watch, not to wreck.** Full-screen capture via GDI
`BitBlt`, clipboard theft, system and user fingerprinting, WMI reconnaissance, and
EDR evasion through direct syscalls that bypass userland API hooks.

**It hides in processes nobody kills.** The injector inspects privilege tokens, picks
a host, and injects into `svchost.exe` or `explorer.exe` — both unremarkable in a
process list and both routinely holding network connections.

C2 runs to a typosquat, `clouddelivry[.]com`, behind DDoS-Guard.

## Contents

1. [The lure](#01--the-lure)
2. [The chain](#02--the-chain)
3. [What the payload does](#03--what-the-payload-does)
4. [Hunting](#04--hunting)
5. [MITRE ATT&CK](#05--mitre-attck)
6. [Indicators of compromise](#06--indicators-of-compromise)
7. [Scope and limits](#07--scope-and-limits)
8. [If you visited the site](#08--if-you-visited-the-site)

## 01 — The lure

A visitor loads the compromised site and, instead of the menu, gets a Cloudflare-styled
interstitial claiming it needs to confirm they are human. The instruction is to open the
Run dialog, paste a supplied command, and press Enter.

The Cloudflare branding is doing the heavy lifting. Years of genuine challenges have
trained people to accept an interruption from Cloudflare as routine, and that
familiarity makes the one genuinely strange part of the request — *paste this into
Run* — feel like a step in the process rather than the attack itself.

The restaurant is a victim here, not a participant. The site was compromised and used
as a delivery point.

## 02 — The chain

```
victim pastes into Win+R
 └─ powershell (polymorphic loader, generated per request)
     └─ stage 2: shellcode injector (~50 KB)
         ├─ enumerate processes
         ├─ inspect privilege tokens
         └─ inject into svchost.exe / explorer.exe
             └─ stage 3: surveillance RAT (~255 KB)
                 ├─ GDI BitBlt screen capture
                 ├─ clipboard theft
                 ├─ WMI recon and persistence
                 ├─ direct syscalls (EDR evasion)
                 └─ C2 → clouddelivry[.]com (DDoS-Guard)
```

| Stage | What it is | Size | How it hides |
|---|---|---|---|
| 1 | PowerShell loader | varies | Polymorphic: randomised variable names, generated payload URLs — no stable hash |
| 2 | Shellcode injector | ~50 KB | Token-aware host selection; payload never lands as a file on disk |
| 3 | Surveillance RAT | ~255 KB | Runs inside a trusted process; direct syscalls bypass userland hooks |

## 03 — What the payload does

Confirmed capabilities of the final stage:

- **Screen capture** via GDI `BitBlt` — full-screen grabs.
- **Clipboard theft.** On a normal workstation that means passwords pulled from a
  manager, cryptocurrency addresses, and one-time codes pasted during sign-in.
- **System and user fingerprinting.**
- **WMI-based reconnaissance and persistence.**
- **Direct Windows syscalls**, sidestepping the userland API hooks most endpoint
  products depend on.

The clipboard capability deserves more alarm than it usually gets. "Use a password
manager and paste your credentials" is good advice right up until something is reading
the clipboard — at which point the manager becomes a tidy queue of every secret the
user touches.

## 04 — Hunting

Behavioural leads, strongest first:

1. **`explorer.exe` spawning `powershell.exe`** with an encoded or download-shaped
   command line. The Run dialog is a child of `explorer.exe`; this ancestry is the
   single best ClickFix tell and is rare in normal use.
2. **RunMRU registry activity.** What the user typed into the Run dialog is recorded
   under `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU`. On a
   suspected host this is usually the fastest confirmation available.
3. **`svchost.exe` or `explorer.exe` connecting to newly registered or low-reputation
   domains**, especially typosquats of CDN and delivery brands.
4. **Screen capture or clipboard APIs called from an injected thread** in a process
   with no business touching either.

Leads 1 and 2 catch the technique regardless of payload, which matters because stage 1
changes on every request.

## 05 — MITRE ATT&CK

| ATT&CK Tactics | ATT&CK Techniques |
|---|---|
| Initial Access | Drive-by Compromise (T1189) |
| Execution | User Execution (T1204); Command and Scripting Interpreter: PowerShell (T1059.001); Windows Management Instrumentation (T1047) |
| Persistence | Event Triggered Execution: WMI Event Subscription (T1546.003) |
| Defense Evasion | Process Injection (T1055); Native API (T1106); Obfuscated Files or Information (T1027) |
| Discovery | Process Discovery (T1057); System Information Discovery (T1082); System Owner/User Discovery (T1033) |
| Collection | Screen Capture (T1113); Clipboard Data (T1115) |
| Command and Control | Application Layer Protocol (T1071) |

**Confidence.** Initial Access, Execution, Defense Evasion, Discovery, Collection and
C2 are high confidence — each maps to behaviour directly observed. Persistence is
moderate: WMI persistence was present as a capability of the payload, but the
subscription artefact itself was not recovered.

## 06 — Indicators of compromise

Hashes appear exactly as recovered. Where an algorithm is absent, that value was not
obtained — it is not withheld, and nothing here is reconstructed to fill a column.

### Files

| Artefact | Size | MD5 | SHA1 / SHA256 |
|---|---|---|---|
| Stage 2 — shellcode injector | ~50 KB | `77FEB600B378EF4AE5EDA534C3BE2ABC` | not recovered |
| Stage 3 — surveillance RAT | ~255 KB | `41dfb8773ffabe06d5241182e6fdfa05` | not recovered |
| Stage 1 — PowerShell loader | varies | polymorphic — no stable hash | n/a |

### Network

| Type | Indicator | Note |
|---|---|---|
| Domain | `clouddelivry[.]com` | C2, typosquat, behind DDoS-Guard |

### Host and behavioural

| Category | Indicator |
|---|---|
| Campaign beacon ID | `d7e967ff695909bf3a7c93578f49b57ac` |
| Injection targets | `svchost.exe`, `explorer.exe` |
| Execution ancestry | `explorer.exe` → Run dialog → `powershell.exe` |
| Behaviour | Full-screen `BitBlt` capture; clipboard reads; WMI recon; direct syscalls |

## 07 — Scope and limits

What this analysis does **not** establish, stated plainly:

- **No SHA1 or SHA256** for either recovered stage — only MD5 was captured.
- **No persistence artefact.** WMI persistence is a capability of the payload; the
  specific subscription, registry key or scheduled task was not recovered, so none is
  listed as an indicator.
- **No C2 protocol analysis.** Traffic was observed reaching the typosquat domain;
  the protocol was not decrypted and no tasking was recovered.
- **No YARA rule yet.** Stage 1 is polymorphic and therefore a poor signature target;
  stages 2 and 3 warrant rules, which are not written at the time of publication.
- **Single observation.** One compromised site, one chain. Whether the same operator
  runs other sites is unknown.

## 08 — If you visited the site

If a website asks you to press `Win + R` and paste anything, that *is* the attack.
Close the page, clear your clipboard, and do not run it.

If you already ran it, treat the machine as compromised. Every credential typed or
pasted on it since, and anything that was visible on screen, should be assumed
captured and rotated.

---

Reported for awareness and responsible disclosure. Indicators are published so other
defenders can hunt the same activity.
