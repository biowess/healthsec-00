# Troubleshooting Log: Phase 0 Hardening Lab

This document captures the issues encountered while building the Phase 0 cybersecurity lab and the fixes that resolved them. It is written for clarity and reproducibility, while preserving the actual troubleshooting path.

## Scope

The lab was built on a local Fedora VM and included:

* SSH hardening
* nftables firewall configuration
* auditd rules
* Lynis, OpenSCAP, and AIDE re-checks
* a small synthetic local service for lab context

All actions were performed in a local, synthetic environment only.

---

## Issue 1: SSH key file was missing

### What happened

While setting up SSH key-based access for `medadmin`, the public key file was not found:

```bash
min_lab.pub: No such file or directory
```

This happened because the expected public key path did not exist yet, or the filename used in the command did not match the key that had actually been created.

### Why it mattered

Without the correct public key, `authorized_keys` could not be populated, so SSH login for the new account would fail.

### Fix

I generated and used the correct key pair, then copied the public key into the target user's SSH directory:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/medadmin_lab -C "medadmin-lab"

sudo mkdir -p /home/medadmin/.ssh
sudo chmod 700 /home/medadmin/.ssh
sudo sh -c 'cat ~/.ssh/medadmin_lab.pub >> /home/medadmin/.ssh/authorized_keys'
sudo chown -R medadmin:medadmin /home/medadmin/.ssh
sudo chmod 600 /home/medadmin/.ssh/authorized_keys
```

### Verification

I confirmed that the SSH directory and key file existed with the right ownership and permissions, and then retried the SSH login.

---

## Issue 2: Permission denied when reading or copying root-owned files

### What happened

When copying hardened config files and saving evidence, permission errors appeared:

```bash
cp: cannot stat '/etc/ssh/sshd_config.d/10-phase0-hardening.conf': Permission denied
cp: cannot create regular file '/srv/healthsec-phase0/configs/nftables.conf': Permission denied
-bash: /srv/healthsec-phase0/posthardening/service-status.txt: Permission denied
-bash: /srv/healthsec-phase0/posthardening/listening-ports-after.txt: Permission denied
```

Later, the same issue appeared again when copying the SSH config drop-in without elevated privileges.

### Why it mattered

Some source files lived under `/etc`, which requires root access to read. Some destination paths under `/srv/healthsec-phase0` were also protected or had ownership mismatches.

### Fix

I used `sudo` for protected reads and writes, then adjusted ownership of the working directory so the normal user could save evidence cleanly.

```bash
sudo mkdir -p /srv/healthsec-phase0/{configs,posthardening}
sudo chown -R medadmin:medadmin /srv/healthsec-phase0

sudo cp /etc/ssh/sshd_config.d/10-phase0-hardening.conf /srv/healthsec-phase0/configs/
sudo cp /etc/nftables.conf /srv/healthsec-phase0/configs/

systemctl status sshd nftables auditd ehr-sim --no-pager | sudo tee /srv/healthsec-phase0/posthardening/service-status.txt > /dev/null
ss -tulpn | sudo tee /srv/healthsec-phase0/posthardening/listening-ports-after.txt > /dev/null
```

When direct redirection still caused problems, I used `sudo tee` or wrote into a directory owned by the lab user.

### Verification

The files were copied successfully, and the post-hardening evidence files were created without permission errors.

---

## Issue 3: SSH server was not listening on port 22

### What happened

An SSH test failed with connection refused:

```bash
ssh: connect to host 127.0.0.1 port 22: Connection refused
```

### Why it mattered

This meant the SSH daemon was not running or not listening yet, so key-based login could not succeed.

### Fix

I installed and enabled the OpenSSH server service:

```bash
sudo dnf install -y openssh-server
sudo systemctl enable --now sshd
systemctl status sshd
ss -tulpn | grep :22
```

### Verification

After enabling `sshd`, port 22 was listening and the SSH login test worked.

The first successful connection prompted the expected host key confirmation message:

```bash
The authenticity of host '127.0.0.1 (127.0.0.1)' can't be established.
```

That was normal for a first-time local SSH connection.

---

## Issue 4: OpenSCAP datastream path was not set correctly

### What happened

OpenSCAP failed with:

```bash
OpenSCAP Error: Unable to open file: ''
```

When I tried to inspect the datastream with:

```bash
oscap info $DS
```

I got:

```bash
ERROR:    SCAP file needs to be specified!
```

### Why it mattered

The `$DS` variable was empty or not set to the correct Fedora SCAP datastream path, so OpenSCAP had nothing to scan.

### Fix

I set the datastream and profile explicitly:

```bash
DS="/usr/share/xml/scap/ssg/content/ssg-fedora-ds.xml"
PROFILE="xccdf_org.ssgproject.content_profile_standard"
```

Then I verified both values:

```bash
echo "$DS"
echo "$PROFILE"
```

After that, I reran OpenSCAP using the explicit values:

```bash
sudo oscap xccdf eval \
  --profile "$PROFILE" \
  --results /srv/healthsec-phase0/posthardening/openscap-results.xml \
  --report /srv/healthsec-phase0/posthardening/openscap-report.html \
  "$DS" | tee /srv/healthsec-phase0/posthardening/openscap-terminal.txt
```

### Verification

The scan completed successfully after the datastream path and profile were set correctly.

---

## Issue 5: Evidence files could not be written from the current shell context

### What happened

When saving post-hardening outputs, some commands failed because the shell was writing into protected paths without the right permissions.

### Why it mattered

Evidence collection needs to be repeatable and clean. If the output files cannot be created, the lab loses traceability.

### Fix

I made sure the evidence directory existed and was writable for the lab user before repeating the commands:

```bash
sudo mkdir -p /srv/healthsec-phase0/{configs,posthardening}
sudo chown -R medadmin:medadmin /srv/healthsec-phase0
```

After that, the same commands succeeded.

---

## Issue 6: First SSH login required host key confirmation

### What happened

The first SSH login to `127.0.0.1` showed the host authenticity prompt.

### Why it mattered

This is not an error, but it can look like one the first time you see it.

### Fix

I confirmed the prompt once and continued. That allowed SSH to store the local host fingerprint for future logins.

### Verification

Subsequent SSH connections no longer showed the same first-time trust prompt.

---

## Summary of the real troubleshooting pattern

The main issues were not about the hardening logic itself. They were about:

* missing or misnamed key files
* service not yet enabled
* root-owned config files
* output files being written without sufficient privileges
* an unset or incorrect OpenSCAP datastream path

The fixes were straightforward once each failure was isolated:

* create or locate the correct SSH key pair
* enable and verify `sshd`
* use `sudo` for root-owned files
* use `sudo tee` when writing protected outputs
* define `DS` and `PROFILE` explicitly before scanning

---

## Practical lessons learned

1. Check file paths before assuming the tool is broken.
2. Verify service status before testing connectivity.
3. Use explicit variables for OpenSCAP inputs.
4. Separate normal-user work from privileged operations.
5. Save evidence as you go, not only at the end.

---

## Clean commands that were ultimately used

```bash
sudo dnf install -y openssh-server
sudo systemctl enable --now sshd
systemctl status sshd
ss -tulpn | grep :22

sudo mkdir -p /home/medadmin/.ssh
sudo chmod 700 /home/medadmin/.ssh
sudo cat ~/.ssh/medadmin_lab.pub | sudo tee -a /home/medadmin/.ssh/authorized_keys > /dev/null
sudo chown -R medadmin:medadmin /home/medadmin/.ssh
sudo chmod 600 /home/medadmin/.ssh/authorized_keys

sudo mkdir -p /srv/healthsec-phase0/{configs,posthardening}
sudo chown -R medadmin:medadmin /srv/healthsec-phase0
sudo cp /etc/ssh/sshd_config.d/10-phase0-hardening.conf /srv/healthsec-phase0/configs/
sudo cp /etc/nftables.conf /srv/healthsec-phase0/configs/

DS="/usr/share/xml/scap/ssg/content/ssg-fedora-ds.xml"
PROFILE="xccdf_org.ssgproject.content_profile_standard"
```

---

## Final note

The troubleshooting path is worth including in the repository because it shows real lab work, not just polished end results. It also makes the project more believable and easier for someone else to reproduce.

