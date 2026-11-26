---
layout: default
title: About
description: Who writes this, how the analysis is done, and what gets published.
---

# About

I am a senior threat analyst. I take malware apart for a living — most of it aimed
at government and critical-sector targets — and turn what I find into detections
that other defenders can run the same day.

This site is the public half of that work. I publish as **Cyb3r_N1nj4**.

## How the work actually happens

Samples land in an isolated Windows lab I built and automate myself. A case runs the
same way every time, and in this order:

- **Triage first.** Hashes, packer and entropy checks, PE structure, strings and
  capability extraction. Plenty of samples are answered here without ever being run.
- **Static before dynamic.** I read the code before I execute it. Unpacking,
  deobfuscation, decompilation — IDA, x64dbg, WinDbg where it earns its place.
- **Detonation under instrumentation**, only once I know roughly what the thing does.
  Process, registry, filesystem and network capture throughout.
- **Memory last and hardest.** Injection and hollowing detection, carving shellcode
  back out of a live process when the payload never touched disk.
- **Detections out the other end.** YARA and Sigma written from artefacts I recovered,
  not from someone else's blog post.

Conclusions carry a confidence rating. If I could not prove something, the write-up
says so rather than rounding it up into a finding.

## What you will find here

Full chains, not highlights: how the lure works, what each stage does, what was
recovered, the indicators in a block you can copy, and an honest MITRE ATT&CK mapping.
Where a hash or artefact was not recovered, that gap is stated — I do not reconstruct
indicators to make a table look complete.

## What you will not

Nothing tied to a specific organisation's internal environment. If a sample was caught
inside an estate and publishing would expose that estate, its tooling or its blind
spots, it does not go up — however good the malware is. Detection logic from those
cases may still be released on its own, with everything identifying stripped out.

That line is not negotiable, and it is why some of the more interesting things I see
will never appear on this site.

## Background

Peer-reviewed research on machine-learning detection of fileless malware from memory,
published in *Sensors* (MDPI) as first author and cited 66 times. MS in Computer
Network and Security. CEH certified. Public analyses of live campaigns including
MMC side-loading backdoors, CHM-delivered reverse shells, njRAT, XWorm and ClickFix
intrusions.

## Detection content

YARA rules and IOC sets live in the
[Malware_Analysis repository](https://github.com/MOsamaKhalid/Malware_Analysis).

## Contact

Corrections, better explanations, and takedown requests from owners of compromised
sites are all welcome — open an issue on
[the repository](https://github.com/MOsamaKhalid/threat-research/issues).
