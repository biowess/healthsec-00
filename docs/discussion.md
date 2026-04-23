## Discussion

### Interpretation of the Lynis Score Stability

The unchanged Lynis Hardening Index of 70 should not be interpreted as evidence that hardening had no effect. The score is computed across a broad range of controls, many of which were intentionally deferred to later phases. The controls addressed in Phase 0. SSH hardening, firewall, AIDE, and basic auditd constitute a meaningful foundational layer but represent only a fraction of Lynis’s full test catalog. The more meaningful Lynis indicator for Phase 0 is the elimination of the sole pre-hardening warning. The increase in suggestion count from 27 to 33 is not a regression; the six new SSH-7408 suggestions appeared because Lynis now found a running SSH daemon to evaluate.

### Significance of the AIDE Results

In a clinical environment, filesystem integrity monitoring provides a mechanism to detect unauthorized modifications to system files. The AIDE post-hardening scan demonstrates the capability to distinguish expected configuration changes from unexpected ones. In the post-hardening scan, the majority of observed changes are directly attributable to the documented hardening steps. The RPM database and SSSD log entries (Section 3.3) represent a gap between the plan and the evidence accounting, but are consistent with expected system activity. In a production deployment, any AIDE finding not pre-registered against a known maintenance action would warrant investigation.

### The ehr-sim.service nobody Warning

The systemd warning about the nobody user running ehr-sim.service is a genuine concern. The nobody account is a system-wide shared identity; if the service were compromised, the attacker would operate under that identity across the system. In healthcare deployments, clinical services are typically expected to run under dedicated, purpose-limited accounts with access restricted to their own working directories. Future phases should replace the nobody account with a dedicated service account restricted to /srv/ehr-sim.

### The Firewall and SSH Exposure

The nftables ruleset (as preserved in the evidence archive) applies a default-drop input policy, which is the appropriate starting point for a server. The decision to allow TCP port 22 inbound is justified for administrative access and introduces an attack surface that did not exist before hardening, when SSH was inactive. SSH with public-key authentication, no root login, and a single named AllowUsers entry represents a reasonable baseline. The MaxAuthTries 3 setting limits automated brute-force attempts per connection. However, as noted in Section 2.3.2, the exact ruleset loaded into the kernel cannot be confirmed from the preserved evidence due to the path discrepancy between /etc/nftables.conf and /etc/sysconfig/nftables.conf.

The lynis-diff.txt also records that sshd.service and [user@1001.service](mailto:user@1001.service) received UNSAFE systemd hardening scores, reflecting the default sshd unit file’s absence of sandboxing directives. This is a documented gap for Phase 0.

### OpenSCAP Remaining Failures and the Auditd Gap

The dominant category of OpenSCAP failures after hardening was auditd rule coverage. The Standard System Security Profile requires a comprehensive set of audit rules tracking syscall-level events: modifications to permissions, file deletions, failed access attempts, kernel module loading, login/logout events, time modifications, and privileged command execution. Phase 0 added only two targeted file-watch rules. This gap is expected and acknowledged as work for subsequent phases.

In a HIPAA context, comprehensive audit coverage under 45 CFR §164.312(b) requires that activity in information systems containing PHI be recorded and examinable. A system with only two file-watch rules has not addressed this requirement. The auditd daemon is running and logging, but the scope of what it captures is severely limited.

### HIPAA Regulatory Context

The controls applied in Phase 0 relate to several HIPAA Security Rule technical safeguard categories. The deployment of AIDE addresses integrity controls referenced under 45 CFR §164.312(c)(1). SSH hardening with public-key authentication and AllowUsers restrictions relates to access control requirements under 45 CFR §164.312(a)(1). A nftables firewall with a default-drop input policy relates to transmission security under 45 CFR §164.312(e)(1). The two auditd file-watch rules represent a minimal start toward audit controls under 45 CFR §164.312(b), but do not constitute comprehensive coverage. These observations are framed as context for a simulated environment; the lab does not claim HIPAA compliance, which requires organizational, administrative, and physical safeguards well beyond this scope.

