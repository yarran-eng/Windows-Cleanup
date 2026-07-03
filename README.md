# Windows Cleanup — Safe C Drive Space Release & Deep App Uninstall

**A general-purpose module for industrial-grade, zero-residual Windows system slimming and trace-wiping uninstallation.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://github.com/yarran-eng/Windows-Cleanup/releases)

---

## Overview

Designed specifically for Windows systems, this module provides two core operational modes:

| Mode | Trigger | Goal |
|------|---------|------|
| **C Drive Cleanup** | "Clean C drive / free up space / disk full" | Safely release disk space following 9 cache/temp file categories |
| **Deep App Uninstall** | "Completely uninstall / remove all traces of an application" | Zero residue across 8 dimensions |

Bound strictly by the core principle **"Better to leave ten cluttered directories than mistakenly delete a single user file,"** it utilizes ironclad safety constraints and dynamic scanning mechanisms.

---

## Key Features

### 1. ⚖️ Reverse-Onus Principle (Protected by Default)
The agent carries no rigid pre-defined cleanup path list. **All directories are protected by default.** The agent must prove — through file-content sampling — that a directory is safe to delete. If in doubt, it is skipped. No exceptions.

### 2. 🥷 Hardcore Zero-Residual Mechanism
Throughout the entire scanning and cleanup process, the creation of **any** temporary files on disk is strictly prohibited:
- ❌ No `.ps1` scripts
- ❌ No `.bat` / `.cmd` batch files  
- ❌ No `.txt` / `.log` temporary logs
- ❌ No `.csv` / `.json` data exports

All complex operations are executed via in-memory inline command chains, with post-cleanup residual verification.

### 3. 🔍 Smart Disk Scanning & 9-Category Deep Classification
The agent adopts a pure information-gathering strategy — expanding and sorting subdirectories layer-by-layer by size. It accurately identifies and matches **9 specific cache types**:

| # | Category | Safety |
|---|----------|--------|
| 1 | Temporary Files | High |
| 2 | Package Manager Download Cache | High |
| 3 | Application Updater Residuals | Medium-High |
| 4 | Crash Dump Files | High |
| 5 | Runtime & Environment Cache | Medium-High |
| 5.1 | Browser AI/ML Model Cache | Medium-High |
| 6 | GPU Shader/Render Cache | High |
| 7 | Browser Cache | Medium |
| 8 | General Application Cache | Low-Medium |
| 9 | IDE & Development Tool Cache | Medium |

### 4. 🛡️ Robust Vendor & User Data Protection
Structural path-based protection rules enforce an absolute prohibition on touching:
- **Microsoft AppData** (Local & Roaming)
- **Google Chrome** user profiles
- **Communication/social apps** (chat histories, contacts)
- **Virtual machines & emulators**
- **Cloud sync clients**
- **IDE** user configurations, settings, and extensions
- **Browser** login data, bookmarks, cookies, history

### 5. 🛑 Triple-Confirmation Workflow
All deletions must pass through three strict gates:

1. **Scan & Structured Candidate List Generation** — discover, classify, and present findings
2. **AI Self-Audit** — cross-check against 10 core indicators (protection rules, active processes, content verification, size anomalies, recoverability, etc.)
3. **Final User Confirmation** — nothing is deleted without explicit consent

### 6. 🧹 8-Dimensional Deep Uninstallation & Final Verification
In uninstall mode, the agent wipes application traces across **8 distinct dimensions**:

| Dimension | Scope |
|-----------|-------|
| File System | Installation dirs, AppData, hidden configs, Documents, shortcuts |
| Registry | HKCU/HKLM Uninstall keys, ProfileList entries |
| AppX Packages | Current-user, all-users, provisioned packages |
| Prefetch | Windows prefetch files |
| Firewall Rules | Inbound & outbound rules |
| Windows Services | Registered services |
| Scheduled Tasks | Background tasks |
| Sandbox Accounts | Isolated user accounts & profiles |

Completion is verified by an automated terminal validation script, with a formal deliverable report.

---

## Installation

Copy `SKILL.md` to your AI agent's skills or prompts directory:

```bash
# Example: manual installation
cp SKILL.md ~/.agents/skills/windows-cleanup/
```

---

## Usage

Load the skill into your AI agent session and describe your need:

- *"Clean up my C drive"*
- *"Completely uninstall VS Code"*
- *"Free up disk space on my computer"*

The agent will scan, classify, present candidates, self-audit, and wait for your confirmation before executing any deletions.

---

## Safety Philosophy

> **"Better to leave ten cluttered directories than mistakenly delete a single user file."**

- Everything is protected until proven safe
- Directory names alone are never sufficient — content must be sampled
- Any uncertainty defaults to "do not delete"
- No script files ever touch your disk
- No registry modifications outside of explicit uninstall scope

---

## License

MIT © [yarran-eng](https://github.com/yarran-eng)
