# Lynis Comparison

## Baseline
- Hardening Index: **70**
- Tests Performed: **251**

## Post-Hardening
- Hardening Index: **70**
- Tests Performed: **255**

## Change
- Score: **No change**
- Tests: **+4 additional checks executed**

## Analysis

Despite applying SSH hardening, firewall rules, and audit logging:

- The Lynis hardening index **did not increase**
- Additional tests were executed in the post-hardening scan

### Key Insight

Lynis evaluates a **broad set of controls**, many of which were not addressed in this phase.  
As a result, targeted improvements (e.g., SSH hardening) did not significantly affect the overall score.

### Takeaway

Lynis score stability does **not mean no improvement** — it highlights that:

- Hardening was **partial and scoped**
- More categories must be addressed for score movement
