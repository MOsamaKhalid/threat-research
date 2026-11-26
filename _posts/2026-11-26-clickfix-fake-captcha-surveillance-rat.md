---
title: "ClickFix on a Compromised Restaurant Site: Polymorphic Loader to Syscall-Evading Surveillance RAT"
date: 2026-11-26 10:00:00 +0500
family: ClickFix
summary: "A compromised hospitality website served a fake Cloudflare human-verification page that talked victims into running malware themselves. Three stages, no exploit, and a RAT that screenshots, steals the clipboard and calls Windows by syscall to stay out of EDR's way."
tags: [ClickFix, Social Engineering, Process Injection, RAT, Threat Hunting]
description: "Technical breakdown of a ClickFix campaign delivering a three-stage surveillance RAT, with IOCs and ATT&CK mapping."
---

I went to a restaurant website to place an order. I got a malware campaign instead.

The site — Ranchers Cafe, a Pakistani restaurant — was serving a fake Cloudflare
"Verify you are human" page. Not a real challenge: a lure that instructs the visitor
to press `Win + R`, paste a command and hit Enter. This is **ClickFix**, and the thing
worth sitting with is that there is no exploit anywhere in it. No zero-day, no drive-by,
no vulnerable plugin in the delivery path. The victim is the exploit. Every technical
control in the chain is bypassed because a human was persuaded to run the command on
their own machine.

What follows is what I found when I pulled the chain apart.

## Delivery

A visitor loads the compromised site. Instead of the menu, they get a Cloudflare-styled
interstitial claiming it needs to confirm they are human. The instruction is to open the
Run dialog, paste a supplied command, and press Enter.

Dressing it as Cloudflare is doing a lot of work here. People have been trained by years
of real challenges to accept an interruption from Cloudflare as routine, and the branding
makes the unusual request — *paste this into Run* — feel like part of the process.

## Stage 1 — polymorphic PowerShell loader

The pasted command retrieves a PowerShell loader. It is generated per request:
variable names are randomised and the payload URLs are produced dynamically, so no
two victims receive a byte-identical script.

The practical consequence is that **hash-based detection on stage 1 is worthless**.
Any IOC list that leads with the loader's hash is already stale. Detection has to key
on behaviour: the `Win + R` → `powershell` ancestry, the download-and-execute shape,
and the network destination.

## Stage 2 — shellcode injector

The loader pulls a second stage of roughly **50 KB**. This component:

- enumerates running processes to pick a host;
- inspects privilege tokens, so it knows what it can inject into;
- injects the final payload into a legitimate process.

Observed injection targets were `svchost.exe` and `explorer.exe` — both unremarkable
in a process listing, both routinely holding network connections, and `explorer.exe`
in particular being something an analyst is unlikely to kill on a hunch.

## Stage 3 — surveillance RAT

The final payload is around **255 KB** and is built for watching rather than destroying.
Confirmed capabilities:

- **Screen capture** via GDI `BitBlt` — full-screen grabs.
- **Clipboard theft.** On a normal workstation that is passwords pulled from a manager,
  cryptocurrency addresses, and one-time codes pasted during sign-in.
- **System and user fingerprinting.**
- **WMI-based reconnaissance and persistence.**
- **EDR evasion through direct Windows syscalls**, sidestepping the userland API hooks
  most endpoint products rely on.

Command and control runs to a deliberate typosquat, `clouddelivry[.]com`, sitting behind
DDoS-Guard — which complicates takedown and obscures the hosting behind it.

The clipboard capability deserves more alarm than it usually gets. Advice to use a
password manager and paste credentials is good advice, right up to the point where
something is reading the clipboard; then the manager becomes a convenient queue of
every secret the user touches.

## Indicators of compromise

Hashes are given exactly as recovered. Where an algorithm is not listed, that value
was not obtained during analysis — it is not withheld, and nothing here is reconstructed.

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

No persistence mechanism, registry key, scheduled task or dropped file path is listed
here, because none was confirmed during this analysis. WMI-based persistence was
observed as a capability of the payload, but the specific artefact was not recovered.

## Hunting

Behavioural leads, in rough order of signal quality:

1. **`explorer.exe` spawning `powershell.exe`** with an encoded or download-shaped
   command line. The Run dialog is a child of `explorer.exe`; this ancestry is the
   single strongest ClickFix tell, and it is rare in normal use.
2. **RunMRU registry activity** — what the user typed into the Run dialog is recorded
   under `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU`. On a
   suspected host this is often the fastest confirmation available.
3. **`svchost.exe` or `explorer.exe` making outbound connections to newly registered
   or low-reputation domains**, particularly typosquats of CDN and delivery brands.
4. **Screen-capture and clipboard API use from an injected thread** in a process with
   no business touching either.

## MITRE ATT&CK

| ATT&CK Tactics | ATT&CK Techniques |
|---|---|
| Initial Access | Drive-by Compromise (T1189) |
| Execution | User Execution (T1204); Command and Scripting Interpreter: PowerShell (T1059.001); Windows Management Instrumentation (T1047) |
| Persistence | Event Triggered Execution: WMI Event Subscription (T1546.003) |
| Defense Evasion | Process Injection (T1055); Native API (T1106); Obfuscated Files or Information (T1027) |
| Discovery | Process Discovery (T1057); System Information Discovery (T1082); System Owner/User Discovery (T1033) |
| Collection | Screen Capture (T1113); Clipboard Data (T1115) |
| Command and Control | Application Layer Protocol (T1071) |

Confidence note: Initial Access, Execution, Defense Evasion, Discovery, Collection and
C2 are **high confidence** — each maps to behaviour directly observed. Persistence is
**moderate**: WMI persistence was present as a capability, but the subscription artefact
itself was not recovered.

## If you visited the site

If a website asks you to press `Win + R` and paste anything, that is the attack. Close
the page, clear your clipboard, and do not run it. If you already did, treat the machine
as compromised: every credential typed or pasted on it since, and anything visible on
screen, should be assumed captured and rotated.

## Disclosure

This was reported for awareness and responsible disclosure. The site appeared to be
compromised and used as a delivery point — the operators of the restaurant are a victim
here, not a participant. Indicators are published so defenders can hunt the same activity.
