## Results

### Lynis Hardening Index

The Lynis Hardening Index was 70 in both the pre-hardening and post-hardening scans. This numerical stability requires careful interpretation. The score did not change because Lynis evaluates a broad range of controls, many of which were not addressed in Phase 0, such as password aging policy, core dump configuration, login banners, USB storage restrictions, and `sysctl` kernel parameter tuning.

The post-hardening scan also introduced six new SSH-specific suggestions (`SSH-7408`), covering `AllowTcpForwarding`, `LogLevel`, `MaxSessions`, `Port`, `TCPKeepAlive`, and `AllowAgentForwarding`. These suggestions were absent from the baseline because SSH was not running at that time. They account for the increase in suggestion count from 27 to 33.

The meaningful Lynis change was the elimination of the sole warning: the missing AIDE database (`FINT-4316`) was resolved, and the post-hardening scan reported “Great, no warnings.”

The `lynis-diff.txt` file documents the precise differences between the two scans, including the AIDE database changing from `NOT FOUND` to `FOUND`, the SSH daemon changing from `NOT FOUND` to `FOUND`, running services increasing from 35 to 38, enabled services increasing from 50 to 52, and SELinux unconfined processes increasing from 50 to 61. The number of tests performed increased from 251 to 255, because the SSH module now evaluated a running daemon.

The `lynis-diff.txt` file also records that `sshd.service` received a systemd service hardening score of 9.6, rated UNSAFE, and `user@1001.service` received 9.4. These scores reflect the default Fedora `sshd` unit file’s lack of systemd sandboxing directives and are a known gap for Phase 0.

| Metric           |    Before Hardening |               After Hardening |
| ---------------- | ------------------: | ----------------------------: |
| Hardening Index  |            70 / 100 |                      70 / 100 |
| Warnings         | 1 (AIDE DB missing) |      0 (“Great, no warnings”) |
| Suggestions      |                  27 | 33 (includes 6 new SSH items) |
| Tests Performed  |                 251 |                           255 |
| AIDE Database    |           NOT FOUND |                         FOUND |
| SSH Daemon       |           NOT FOUND |                         FOUND |
| Running Services |                  35 |                            38 |

*Table 1. Lynis scan comparison: key metrics before and after hardening.*

### OpenSCAP Results

The OpenSCAP evaluation against the Standard System Security Profile for Fedora tested 75 rules in both scans, with 1 not-applicable rule in each. Before hardening, 26 rules passed and 48 failed. After hardening, 28 rules passed and 46 failed, for a net change of +2 passing rules.

A rule-by-rule comparison of `openscap-terminal.txt` from the baseline and post-hardening runs confirms exactly two status changes: `aide_build_database` (Build and Test AIDE Database) moved from fail to pass, and `sshd_disable_root_login` (Disable SSH Root Login) moved from fail to pass. No rules regressed from pass to fail.

Several important findings remained failures after hardening. The dominant category was `auditd` rule coverage: the Phase 0 ruleset added only two targeted file-watch rules, leaving required rules for syscall-level events such as permissions, file deletions, MAC modifications, privilege use, time modification, and kernel module loading unaddressed. These are expected gaps for Phase 0.

In addition, the rules for Disable SSH Access via Empty Passwords and Direct root Logins Not Allowed continued to fail. The SSH empty-password rule likely reflects how OpenSCAP evaluates the PAM stack, which was not modified in this phase. The direct root login rule relates to `/etc/securetty` configuration, which was also not modified.

| Outcome        | Before Hardening | After Hardening | Change |
| -------------- | ---------------: | --------------: | -----: |
| Pass           |               26 |              28 |     +2 |
| Fail           |               48 |              46 |     −2 |
| Not Applicable |                1 |               1 |      0 |
| Total Rules    |               75 |              75 |      — |

*Table 2. OpenSCAP rule results summary before and after hardening.*

| Rule ID (short)           | Description                  | Before | After |
| ------------------------- | ---------------------------- | ------ | ----- |
| `aide_build_database`     | Build and Test AIDE Database | FAIL   | PASS  |
| `sshd_disable_root_login` | Disable SSH Root Login       | FAIL   | PASS  |

*Table 3. OpenSCAP rules that changed status between scans (both confirmed by file comparison).*

### AIDE Filesystem Integrity

The pre-hardening AIDE check (`aide-check-before-hardening.txt`) reported no differences across 156,607 monitored entries, establishing a verified clean baseline. The post-hardening AIDE check (`aide-check-after-hardening.txt`) reported 14 added entries and 17 changed entries across 156,621 total monitored entries.

Added entries (14):

* `/etc/audit/audit.rules.prev` and `/etc/audit/rules.d/phase0.rules` — consistent with auditd rule additions.
* `/etc/nftables.conf` — consistent with firewall ruleset file creation per the lab plan.
* `/etc/ssh/ssh_host_ecdsa_key`, `ssh_host_ecdsa_key.pub`, `ssh_host_ed25519_key`, `ssh_host_ed25519_key.pub`, `ssh_host_rsa_key`, `ssh_host_rsa_key.pub` — SSH host keys generated when `sshd` was initialized.
* `/etc/ssh/sshd_config.d/10-phase0-hardening.conf` — SSH hardening drop-in.
* `/etc/systemd/system/multi-user.target.wants/nftables.service` and `/etc/systemd/system/multi-user.target.wants/sshd.service` — service symlinks created when `nftables` and `sshd` were enabled.
* `/var/log/journal/acfe3227c9384038aa3578b3c9c530d2/user-1001.journal` — journal file created for the `medadmin` (`uid 1001`) session.
* `/var/log/sssd/sssd_kcm.log` — SSSD Kerberos credential manager log file. This entry was not anticipated in the lab plan; it is consistent with SSSD activity triggered by the SSH/PAM session but was not explicitly produced by the documented hardening steps.

Changed entries (17):

* `/etc/audit/audit.rules` — content changed due to new audit rules.
* `/etc/audit/rules.d` — directory metadata updated.
* `/etc/group`, `/etc/group-`, `/etc/gshadow`, `/etc/gshadow-`, `/etc/passwd`, `/etc/passwd-`, `/etc/shadow` — modified to add the `medadmin` user account.
* `/etc/shadow-` — prior backup of `/etc/shadow`.
* `/etc/ssh/sshd_config.d` — directory size changed to reflect the new drop-in file.
* `/etc/subgid` and `/etc/subuid` — inode number changes, consistent with user account creation.
* `/usr/lib/sysimage/rpm/rpmdb.sqlite-shm` and `/usr/lib/sysimage/rpm/rpmdb.sqlite-wal` — RPM database shared memory and WAL files. These entries were not individually anticipated in the lab plan. They are consistent with RPM database activity during package installation (`openssh-server`), but were not listed in the draft’s original accounting.
* `/var/log/lastlog` — size grew from 0 to 292,584 bytes following the first SSH login.
* `/var/log/lynis-report.dat` — extended file attribute modification, consistent with a completed Lynis scan.

The SSSD log and RPM database entries, while not anticipated in the original hardening plan, are consistent with expected side effects of package installation and SSH authentication activity. No entries in the AIDE output are unexplained in the sense of being inconsistent with the documented lab actions. However, the earlier draft’s statement that “no unexpected modifications were observed” is an overstatement of what the evidence strictly proves; the more defensible claim is that all observed changes are consistent with the documented actions and their known side effects, with the caveats noted above.

### Network Surface

The pre-hardening network state (`listening-ports-before.txt`) shows no SSH listener; port 22 is absent from all TCP LISTEN entries. The post-hardening state (`listening-ports-after.txt`) shows two new TCP LISTEN entries: `0.0.0.0:22` and `[::]:22`, confirming that `sshd` was successfully started. The EHR service (`127.0.0.1:8080`) remained unchanged in both files, confirming that it stayed bound exclusively to the loopback interface throughout the lab.

| Port / Service                   | Before Hardening       | After Hardening        | Notes                          |
| -------------------------------- | ---------------------- | ---------------------- | ------------------------------ |
| `127.0.0.1:8080` (`ehr-sim`)     | LISTEN (loopback only) | LISTEN (loopback only) | No change — correctly isolated |
| `0.0.0.0:22` / `[::]:22` (`SSH`) | Not present            | LISTEN                 | Added by hardening             |
| DNS / LLMNR (53, 5355)           | Present                | Present                | Unchanged system services      |

*Table 4. Listening TCP sockets before and after hardening.*

### Service Status

The post-hardening service status (`service-status.txt`) confirmed that `sshd.service` was active (running) and enabled, `nftables.service` was active (exited, status=0/SUCCESS), `auditd.service` remained active (running) and enabled, and `ehr-sim.service` remained active (running) and enabled.

The `service-status.txt` output includes a systemd warning — “Special user nobody configured, this is not safe!” — for `ehr-sim.service`, produced at both `20:33:20` and `20:40:38`. This is a systemd advisory indicating that running a service as the `nobody` account is not recommended practice for dedicated services.

