## Methodology

### Target System Description

The target system was a QEMU/KVM virtual machine running Fedora Linux 43 (LXQt), kernel version 6.17.1-300.fc43.x86_64, as confirmed by `fedora-release.txt` and `uname.txt`. The system hostname was unset at the time of the lab and was reported as `localhost-live` by `hostnamectl`.

Hardware was identified as a *Standard PC (Q35 + ICH9, 2009)* virtual model with QEMU vendor firmware.

A synthetic EHR service (`ehr-sim.service`) was deployed as a systemd unit. The service ran as user `nobody` and group `nobody`, was bound exclusively to `127.0.0.1:8080`, and was marked as enabled. Its unit file defines a simple HTTP server with `WorkingDirectory=/srv/ehr-sim` and a `Restart=on-failure` policy.

The service remained active and running throughout the lab, as confirmed in `ehr-sim-status.txt` and `service-status-before.txt`.

---

### Baseline Assessment

#### Lynis Audit

Lynis version 3.1.6 (as stated in the scan header) was used to produce a comprehensive baseline audit stored in `lynis-baseline.txt`.

Lynis performs individual security tests organized by category, including:

* Boot configuration
* Authentication
* Networking
* Logging
* Filesystem integrity

Each run concludes with:

* A Hardening Index score (0–100)
* A list of warnings
* A list of suggestions

At baseline, Lynis reported:

* Hardening Index: 70
* Warnings: 1
* Suggestions: 27

The single warning indicated the absence of an AIDE database (control FINT-4316), meaning file integrity monitoring had not yet been configured.

---

#### OpenSCAP

OpenSCAP was used to evaluate the system against the *Standard System Security Profile for Fedora* (profile ID: `xccdf_org.ssgproject.content_profile_standard`), as identified in `openscap-info.txt`.

The datastream file used was `ssg-fedora-ds.xml` from the `scap-security-guide` package.

This profile evaluates rules across multiple areas, including:

* Cryptographic policy
* Authentication
* `auditd` configuration
* Package integrity
* SSH configuration

Baseline evaluation output was captured in `openscap-terminal.txt`.

Results:

* Total rules tested: 75
* Passed: 26
* Failed: 48
* Not applicable: 1

Full results were preserved in:

* `openscap-results-before.xml`
* `openscap-report.html`

---

#### Service and Network Enumeration

The active network surface was documented using `ss`, producing:

* `listening-ports-before.txt`
* `listening-sockets-before.txt`

These two files are byte-for-byte identical, indicating the same command was executed twice under different filenames (see Section 5.1.3).

Key observations:

* The EHR service was bound only to `127.0.0.1:8080`
* DNS-related services (`systemd-resolved`) listened on `127.0.0.53` and `127.0.0.54`
* LLMNR (port 5355, TCP) was listening on `0.0.0.0` and `[::]`
* SSH (port 22) was not present in any TCP LISTEN entry

Additionally, `service-status-before.txt` confirms that `sshd.service` was:

* Inactive (dead)
* Disabled

**Note:**
The `ss` output files include a “Process” column header but contain no process name data in the rows. Therefore, owning processes cannot be identified from these files alone.

---

### Hardening Actions

#### SSH Hardening

The OpenSSH server was installed, enabled, and configured using a drop-in configuration file located at:

```
/etc/ssh/sshd_config.d/10-phase0-hardening.conf
```

This file (included in the evidence archive) applies the following controls:

* `PermitRootLogin no`
* `PasswordAuthentication no`
* `PubkeyAuthentication yes`
* `MaxAuthTries 3`
* `AllowUsers medadmin`
* `X11Forwarding no`
* `ClientAliveInterval 300`
* `ClientAliveCountMax 2`

A dedicated user account `medadmin` (UID 1001) was created, and an ED25519 SSH key pair was generated.

The post-hardening service log (`service-status.txt`) records two connection events:

* At 20:39:55:

  ```
  Connection closed by authenticating user medadmin 127.0.0.1 port 47968 [preauth]
  ```

  This indicates termination during the pre-authentication phase. The log does not specify the authentication method and does not confirm that a password was attempted or rejected.

* At 20:40:56:

  ```
  Accepted publickey for medadmin from 127.0.0.1 port 52192 ssh2: ED25519 SHA256:gfcD098ny5h/cB/57PmxsLCAe6jCvgR7w9O/133Jcls
  ```

  This confirms successful public-key authentication.

The evidence demonstrates that key-based login succeeded but does not directly confirm that password authentication was attempted and blocked.

---

#### Firewall Configuration

`nftables` was configured with a minimal default-drop policy.

The intended ruleset, stored in `configs/nftables.conf`, defines:

* One `inet` filter table
* Three chains: `input`, `forward`, and `output`

Key rules:

* `input` chain:

  * Accept loopback traffic (`iif lo`)
  * Accept established and related connections
  * Accept ICMP and ICMPv6
  * Accept TCP port 22 (SSH)
  * Drop all other inbound traffic

* `forward` chain:

  * Drop all forwarded traffic

* `output` chain:

  * Accept all outbound traffic

**Path discrepancy:**

Evidence shows a mismatch:

* AIDE reports `/etc/nftables.conf` as newly added
* `nftables.service` uses:

  ```
  ExecStart=/sbin/nft -f /etc/sysconfig/nftables.conf
  ```

This indicates the service read from `/etc/sysconfig/nftables.conf`, not `/etc/nftables.conf`.

The contents of `/etc/sysconfig/nftables.conf` are not included in the evidence archive, and no `nft list ruleset` output was captured.

Therefore:

* The service exited successfully (`status=0/SUCCESS`)
* The exact ruleset loaded into the kernel cannot be independently verified

The file in `configs/nftables.conf` matches the intended ruleset but cannot be confirmed as the one actually applied.

This limitation is discussed in Section 5.1.4.

---

#### AIDE Initialization

AIDE version 0.19.2 was initialized to create a cryptographic filesystem baseline.

The initial scan (`aide-check-before-hardening.txt`):

* Catalogued 156,607 filesystem entries
* Found no differences between the database and the live system

This confirms a clean baseline state.

The following hash algorithms are documented:

* SHA256
* SHA512
* STRIBOG256
* STRIBOG512
* SHA512/256
* SHA3-256
* SHA3-512

These provide a tamper-evident reference for future comparisons.

---

#### Auditd Rules

Three audit rules were added (`audit-rules.txt`):

1. Monitor SSH configuration:

   ```
   -w /etc/ssh/sshd_config.d -p wa -k sshd_hardening
   ```

2. Monitor firewall configuration:

   ```
   -w /etc/nftables.conf -p wa -k firewall_changes
   ```

3. Reduce log noise:

   ```
   -a never,task
   ```

The first two rules create a forensic trail for write and attribute-change events.
Comprehensive syscall-level audit coverage was not implemented in Phase 0.

---

### Post-Hardening Assessment

After applying hardening measures, all evaluation tools were re-run.

Outputs were stored as follows:

* Lynis: `lynis-posthardening.txt`
* OpenSCAP:

  * `openscap-results-after.xml`
  * `openscap-report-after.html`
  * `openscap-terminal.txt` (posthardening/)
* AIDE: `aide-check-after-hardening.txt`
* Network state: `listening-ports-after.txt`

A unified diff between baseline and post-hardening Lynis runs was computed and stored in:

* `lynis-diff.txt`
