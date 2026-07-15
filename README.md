# NetFRAME Home Lab, Security Assessment (redacted public edition)

A defensive, non-destructive security assessment of my own home lab (**NetFRAME**), performed
under my own authorization. This public edition has all internal IP addresses, hostnames, MAC
addresses, topology specifics, and secrets **generalized or removed**; hosts appear as generic
roles (`NODE-A`, `STORAGE-HOST`, `WORKSTATION`). It is shared as a demonstration of
security-assessment methodology and reporting, not as an operational document.

## What this demonstrates

- **Scope and rules of engagement** written before any testing (read-only by default, a 95%
  safety bar, non-destructive, one change at a time, explicit approval for anything windowed).
- **Evidence-driven findings** ranked by severity with a confidence rating, each mapped to the
  finding it closes and given a concrete remediation, complexity, and priority.
- **Attack-chain analysis**: the assessment is framed around how individual medium findings
  *combine* into a realistic path from foothold to impact, not just a flat vulnerability list.
- **Honest coverage gaps**: the report states what was *not* tested (for example, detection
  efficacy was reasoned from configuration rather than proven by triggering alerts).
- **Confirmed positive controls**: what is already done right (default-deny inter-VLAN
  segmentation, out-of-band BMC isolation, key-only SSH, no inbound exposure, a fleet SIEM).
- **A prioritized remediation roadmap** (break-the-chain items first, then blast-radius
  reduction, then hardening).

## The environment (generalized)

A 7-node Proxmox VE cluster with segmented VLANs behind a single firewall, a Kubernetes cluster,
shared ZFS storage with off-box backups, out-of-band management on a dedicated VLAN, and a
Prometheus/Grafana/Loki plus SIEM observability stack. Enough context is preserved to make the
findings legible; enough is removed that nothing here aids an attacker.

## The report

**[NetFRAME-Security-Assessment.pdf](NetFRAME-Security-Assessment.pdf)**

Sections: engagement and authorization, scope and methodology, attack chains, findings register
(summary plus per-finding detail for the high-severity items), positive controls confirmed, and a
tiered remediation roadmap.

## Companion work (also public)

The same environment is documented across a small portfolio: an infrastructure homelab repository
([Home-Lab](https://github.com/machismo0311/Home-Lab)) with topology references, runbooks, and
RCAs, and a read-only cluster health monitor
([netframe-monitor](https://github.com/machismo0311/netframe-monitor)). The full unredacted
security edition, a red-team attack-path assessment, and an SRE reliability assessment are kept in
private repositories.

## Note

This is my own infrastructure, assessed under my own authorization, and published in redacted form
deliberately. Demonstrating what to *withhold* is part of the exercise.
