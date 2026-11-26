---
layout: default
title: About
description: Who writes this and what gets published here.
---

# About

I write as **CyberNinja (Cyb3r_N1nj4)**. By day I work malware analysis and
detection engineering: reverse engineering samples, tracing execution chains,
recovering indicators and turning them into YARA and Sigma rules.

This site is where the public half of that work goes.

## What gets published

Write-ups here cover one of two things:

- Campaigns that are **already public**, where more technical detail helps defenders.
- Activity observed against **publicly reachable infrastructure** — a compromised
  website serving a lure, an exposed C2, a campaign hitting many targets at once.

## What does not

Anything tied to a specific organisation's internal environment stays unpublished.
If a sample was detected inside an estate and the findings would expose that estate,
its tooling or its gaps, it does not appear here — regardless of how interesting the
malware is. Detection logic derived from such cases may still be released on its own,
stripped of any context that identifies where it came from.

## Format

Each post follows the same shape: what the sample is, how the chain executes,
what was recovered, indicators in a single block you can copy, detection content,
and a MITRE ATT&CK mapping. Indicators are defanged. Nothing published here is
runnable malware.

## Detection content

YARA rules and IOC sets live in the
[Malware_Analysis repository](https://github.com/MOsamaKhalid/Malware_Analysis).

## Contact

Corrections and takedown requests from affected site owners are welcome — open an
issue on the repository.
