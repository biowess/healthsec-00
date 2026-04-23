# Network Exposure Comparison

## Before Hardening
- Limited number of listening services
- Local service on `127.0.0.1:8080`

## After Hardening
- SSH (port 22) explicitly enabled
- Local service still bound to `127.0.0.1:8080`
- No unintended external-facing services introduced

## Analysis

- Firewall rules were applied using nftables
- Loopback traffic allowed
- Inbound connections restricted except for SSH

### Key Insight

The system maintains a **minimal attack surface** while allowing controlled access.

### Takeaway

- No accidental exposure introduced
- Network behavior remains predictable and restricted
