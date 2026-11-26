# Cyb3r_N1nj4 // Threat Research

Source for the threat-research blog published at
**https://mosamakhalid.github.io/threat-research/**

Technical write-ups on malware observed during threat hunting and analysis:
execution chains, recovered indicators, detection content, and MITRE ATT&CK mapping.

## Publication policy

Posts cover campaigns that are already public, or activity observed against publicly
reachable infrastructure. Findings tied to a specific organisation's internal
environment are not published here. Detection logic from such cases may be released
separately, stripped of identifying context.

Indicators are defanged. No runnable malware is hosted in this repository.

## Adding a post

Create one Markdown file in `_posts/` named `YYYY-MM-DD-slug.md`:

```markdown
---
title: "Post title"
date: 2026-11-26 10:00:00 +0500
family: FamilyName
summary: "One or two sentences shown on the index page."
tags: [Tag, Tag]
---

Body in Markdown.
```

The index page picks it up automatically on the next build. No other file needs editing.

Suggested section order, for consistency between posts:
delivery → stages → indicators of compromise → hunting → MITRE ATT&CK → disclosure.

## Detection content

YARA rules and IOC sets: https://github.com/MOsamaKhalid/Malware_Analysis
