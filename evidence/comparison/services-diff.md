# Service State Comparison

## Before Hardening
- sshd: inactive or not installed
- nftables: inactive
- auditd: inactive
- ehr-sim: active

## After Hardening
- sshd: active
- nftables: active
- auditd: active
- ehr-sim: active

## Analysis

Security-critical services were explicitly enabled during the hardening phase.

### Key Insight

The system transitioned from a **minimal default state** to a **managed security posture**.

### Takeaway

- Hardening included **service activation**, not just configuration edits
- Improves system visibility, logging, and access control
