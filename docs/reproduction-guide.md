# Reproduction Guide

## 1. Prepare the machine

Use a fresh Fedora VM. Log in as a normal user first.

```bash
whoami
pwd
hostnamectl
```

## 2. Install tools

```bash
sudo dnf makecache
sudo dnf install -y   lynis   openscap-scanner   scap-security-guide   aide   nftables   openssh-server   audit   jq   tree   curl   vim
```

## 3. Create working folders

```bash
sudo mkdir -p /srv/healthsec-phase0/{baseline,posthardening,configs,evidence,notes}
sudo chown -R "$USER":"$USER" /srv/healthsec-phase0
```

## 4. Create the synthetic lab service

Create a small local-only service that binds to `127.0.0.1:8080`.

```bash
sudo tee /etc/systemd/system/ehr-sim.service >/dev/null <<'EOF'
[Unit]
Description=Synthetic EHR Support Service
After=network.target

[Service]
Type=simple
WorkingDirectory=/srv/ehr-sim
ExecStart=/usr/bin/python3 -m http.server 8080 --bind 127.0.0.1
Restart=on-failure
User=nobody
Group=nobody

[Install]
WantedBy=multi-user.target
EOF
```

Then start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ehr-sim
systemctl status ehr-sim --no-pager
ss -tulpn | grep 8080
curl -s http://127.0.0.1:8080/
```

Save the unit file and status snapshot.

## 5. Capture baseline evidence

```bash
cat /etc/fedora-release | tee /srv/healthsec-phase0/baseline/fedora-release.txt
uname -a | tee /srv/healthsec-phase0/baseline/uname.txt
hostnamectl | tee /srv/healthsec-phase0/baseline/hostnamectl.txt
systemctl status sshd nftables auditd ehr-sim --no-pager | tee /srv/healthsec-phase0/baseline/service-status-before.txt
ss -tulpn | tee /srv/healthsec-phase0/baseline/listening-sockets-before.txt
sudo lynis audit system | tee /srv/healthsec-phase0/baseline/lynis-baseline.txt
```

## 6. Run OpenSCAP before hardening

Use the Fedora datastream and the `standard` profile.

```bash
DS="/usr/share/xml/scap/ssg/content/ssg-fedora-ds.xml"
PROFILE="xccdf_org.ssgproject.content_profile_standard"

oscap info "$DS" | tee /srv/healthsec-phase0/baseline/openscap-info.txt

sudo oscap xccdf eval   --profile "$PROFILE"   --results /srv/healthsec-phase0/baseline/openscap-results-before.xml   --report /srv/healthsec-phase0/baseline/openscap-report.html   "$DS" | tee /srv/healthsec-phase0/baseline/openscap-terminal.txt
```

## 7. Initialize AIDE

```bash
sudo aide --init
sudo cp /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
sudo aide --check | tee /srv/healthsec-phase0/baseline/aide-check-before-hardening.txt
```

## 8. Add audit rules

```bash
sudo tee /etc/audit/rules.d/phase0.rules >/dev/null <<'EOF'
-w /etc/ssh/sshd_config.d/ -p wa -k sshd_hardening
-w /etc/nftables.conf -p wa -k firewall_changes
EOF

sudo augenrules --load
sudo systemctl restart auditd
sudo auditctl -l | tee /srv/healthsec-phase0/posthardening/audit-rules.txt
```

## 9. Harden SSH

Create a new admin user, install an SSH key, and add a drop-in config.

```bash
sudo useradd -m -G wheel medadmin
sudo passwd medadmin
ssh-keygen -t ed25519 -f ~/.ssh/medadmin_lab -C "medadmin-lab"

sudo install -d -m 700 /home/medadmin/.ssh
sudo sh -c 'cat ~/.ssh/medadmin_lab.pub >> /home/medadmin/.ssh/authorized_keys'
sudo chown -R medadmin:medadmin /home/medadmin/.ssh
sudo chmod 700 /home/medadmin/.ssh
sudo chmod 600 /home/medadmin/.ssh/authorized_keys
```

SSH drop-in:

```bash
sudo mkdir -p /etc/ssh/sshd_config.d
sudo tee /etc/ssh/sshd_config.d/10-phase0-hardening.conf >/dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers medadmin
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
EOF
```

Validate and restart:

```bash
sudo sshd -t
sudo systemctl restart sshd
sudo sshd -T | egrep 'permitrootlogin|passwordauthentication|pubkeyauthentication|maxauthtries|allowusers|x11forwarding|clientaliveinterval|clientalivecountmax'
ssh -i ~/.ssh/medadmin_lab medadmin@127.0.0.1
```

## 10. Apply nftables

```bash
sudo tee /etc/nftables.conf >/dev/null <<'EOF'
flush ruleset

table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;

    iif "lo" accept
    ct state established,related accept

    ip protocol icmp accept
    ip6 nexthdr ipv6-icmp accept

    tcp dport 22 accept

    counter drop
  }

  chain forward {
    type filter hook forward priority 0; policy drop;
  }

  chain output {
    type filter hook output priority 0; policy accept;
  }
}
EOF

sudo nft -f /etc/nftables.conf
sudo systemctl enable --now nftables
sudo nft list ruleset | tee /srv/healthsec-phase0/posthardening/nftables-ruleset.txt
```

## 11. Save final evidence

```bash
sudo cp /etc/ssh/sshd_config.d/10-phase0-hardening.conf /srv/healthsec-phase0/configs/
sudo cp /etc/nftables.conf /srv/healthsec-phase0/configs/
systemctl status sshd nftables auditd ehr-sim --no-pager | tee /srv/healthsec-phase0/posthardening/service-status.txt
ss -tulpn | tee /srv/healthsec-phase0/posthardening/listening-ports-after.txt
sudo lynis audit system | tee /srv/healthsec-phase0/posthardening/lynis-posthardening.txt
sudo oscap xccdf eval   --profile "$PROFILE"   --results /srv/healthsec-phase0/posthardening/openscap-results-after.xml   --report /srv/healthsec-phase0/posthardening/openscap-report-after.html   "$DS" | tee /srv/healthsec-phase0/posthardening/openscap-terminal.txt
sudo aide --check | tee /srv/healthsec-phase0/posthardening/aide-check-after-hardening.txt
journalctl -u sshd --no-pager | tail -n 50
journalctl -u nftables --no-pager | tail -n 50
journalctl -u auditd --no-pager | tail -n 50
```

## 12. Common mistakes and fixes

- `ssh: connect to host 127.0.0.1 port 22: Connection refused`  
  Fix: make sure `openssh-server` is installed, `sshd` is enabled, and port 22 is listening.

- `Permission denied` when copying config or writing evidence files  
  Fix: use `sudo cp` for protected files, and use `sudo tee` or `sudo sh -c` when writing to root-owned paths.

- `OpenSCAP Error: Unable to open file: ''`  
  Fix: set `DS` explicitly to `/usr/share/xml/scap/ssg/content/ssg-fedora-ds.xml` and set `PROFILE` before running the scan.

- `oscap info $DS` says the SCAP file needs to be specified  
  Fix: make sure the variable actually contains the datastream path by running `echo "$DS"`.

## 13. What to expect

- Lynis may still show SSH suggestions
- OpenSCAP will likely still have failures
- AIDE should show the config changes you made
- The final repo should present both the success and the limits clearly
