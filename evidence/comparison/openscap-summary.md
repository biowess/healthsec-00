# OpenSCAP Comparison

## Profile Used
`xccdf_org.ssgproject.content_profile_standard`

## Baseline
- Pass: **26**
- Fail: **48**
- Not Selected: **1028**

## Post-Hardening
- Pass: **28**
- Fail: **46**
- Not Selected: **1028**

## Change
- Passed Rules: **+2**
- Failed Rules: **-2**

## Analysis

The OpenSCAP scan shows a **small but measurable improvement** after hardening:

- Some security controls were successfully remediated
- The overall system remains far from compliance with the selected profile

### Key Insight

OpenSCAP is sensitive to **specific configuration rules**, unlike Lynis’ broader scoring model.

### Takeaway

- Hardening had **real impact**, even if limited
- Full compliance requires **many more system-level changes**
