# HealthSec Phase 0

A beginner-friendly cybersecurity hardening lab on a local Fedora VM.

## Overview

This repository documents a Phase 0 defensive hardening lab performed in a **local, synthetic environment only**. The lab starts from a clean Fedora VM baseline, captures evidence, applies a small set of hardening changes, and then re-runs the same checks to compare results.

The goal is to show a reproducible workflow, not to claim perfect security.

## Objectives

- Capture a clean baseline of the VM
- Run host-audit and compliance tools
- Harden SSH, firewall, and audit logging
- Re-check the system after changes
- Preserve evidence in a clean, readable format

## Environment

- OS: Fedora Linux 43 (LXQt)
- Virtualization: KVM/QEMU
- Hostname: `localhost-live`
- Kernel: `6.17.1-300.fc43.x86_64`
- User-facing lab service: synthetic EHR support service on `127.0.0.1:8080`

## Tools used

- Lynis
- OpenSCAP (`oscap`)
- SCAP Security Guide
- AIDE
- nftables
- OpenSSH server (`sshd`)
- auditd / augenrules
- systemd / journalctl
- jq, tree, curl, vim

## Methodology

1. Record system facts and running services
2. Run baseline Lynis, OpenSCAP, and AIDE checks
3. Add audit rules for SSH and firewall config files
4. Harden SSH access
5. Apply a minimal nftables ruleset
6. Save the final config files and service state
7. Re-run the same checks after hardening
8. Compare before/after outputs

## Key results

### Lynis

- Baseline hardening index: **70**
- Post-hardening hardening index: **70**
- Tests performed: **251 → 255**

Lynis still reported SSH-related suggestions after hardening, so the score did not move even though the configuration changed.

### OpenSCAP

Using the Fedora SCAP datastream and the `standard` profile:

- Pass: **26 → 28**
- Fail: **48 → 46**
- Not selected: **1028** both runs

This shows a small improvement, but the system is still not fully compliant.

### AIDE

- Baseline check: **no differences**
- After hardening: **14 added**, **17 changed**, **0 removed**

Most of the AIDE changes are expected because the lab intentionally modified SSH, audit, nftables, and related system files.

## Key insights

- SSH hardening mattered more for traceability than for score movement.
- Lynis can keep suggesting extra SSH tweaks even after a valid hardening pass.
- OpenSCAP is useful for before/after comparison, but the chosen profile matters.
- AIDE is a good way to prove that config files changed in a controlled way.
- The repository is strongest when it shows the full workflow, including mistakes and fixes.

## Limitations

- nftables was applied locally and should be treated as a minimal lab firewall, not a production policy.
- OpenSCAP still had failures after hardening.
- Lynis still showed SSH suggestions after the changes.
- This lab only covers a small slice of host hardening.
- No real patient data, no real clinical network, and no external targets were used.

## Repository structure

```text
healthsec-phase0/
├── README.md
├── docs/
│   ├── methodology.md
│   ├── results.md
│   ├── discussion.md
│   └── reproduction-guide.md
├── evidence/
│   ├── baseline/
│   ├── posthardening/
│   └── comparison/
├── configs/
│   ├── ehr-sim.service
│   ├── 10-phase0-hardening.conf
│   └── nftables.conf
├── notes/
│   └── troubleshooting-log.md
└── assets/
    ├── diagrams/
    └── charts/
```

## What goes where

- `docs/`: polished write-up and reproduction instructions
- `evidence/`: terminal outputs, reports, and comparison files
- `configs/`: final config files that were actually applied
- `notes/`: cleaned troubleshooting notes, not the messy working draft
- `scripts/`: only small helper scripts if they really help reproducibility
- `assets/`: diagrams and simple charts

## Naming conventions

Use simple, stable names:

- `baseline-*` for before-state evidence
- `posthardening-*` for after-state evidence
- `*-report.html` for generated reports
- `*-results.xml` for machine-readable scan output
- `*-status.txt` for service snapshots
- `*-terminal.txt` for command output captured from the shell

Prefer lowercase, hyphen-separated filenames.

## Notes policy

I did **not** publish the raw messy notes as the main story of the repository. I kept them only as an internal reference.

Published a cleaned `notes/troubleshooting-log.md` and `docs/reproduction-guide.md` instead.

## How to reproduce

See `docs/reproduction-guide.md`.

## Disclaimer

This project was run in a **local lab on synthetic data only**. It does not target real systems, real patient records, or any external environment.

