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

The summary below is drawn from that report so the substance is readable without opening the PDF.

---

## Assessment at a glance

| Field | Value |
|---|---|
| Engagement | Authorized internal security assessment of an owner-operated home laboratory |
| Authorization | Full written authorization by the environment owner; strictly defensive; non-destructive |
| Scope | Owner-controlled hosts, network, virtualization, Kubernetes, storage, services, repositories |
| Method | Passive and authenticated inspection, low-intensity enumeration, read-only auditors; no exploitation |
| Classification | Public: identifiers generalized, all secrets and addresses redacted |
| Date | 2026-07-13 |

**Headline.** No inbound Internet exposure and no critical, Internet-exploitable exposure was
identified. The risk is internal: four HIGH-severity findings that chain together.

## Findings register

| ID | Sev | Conf | Finding |
|---|---|---|---|
| F-001 | HIGH | Confirmed | Untrusted IoT/printer/mobile devices share Layer 2 with the management VLAN |
| F-003 | HIGH | Confirmed | Admin workstation exposes cleartext FTP, end-of-life SMB, and NFS on that segment |
| F-006 | HIGH | Confirmed | Unencrypted SSH keys granting cluster-wide root and commit signing |
| K8S-01 | HIGH | Confirmed | Private container registry is anonymous (no authentication) |
| VIRT-01 | MED | Confirmed | Hypervisor datacenter/node firewall fully disabled |
| VIRT-02 | MED | Confirmed | Hypervisor management API bound to all interfaces |
| VIRT-03 | MED | Confirmed | Sole cluster admin is a shared superuser with no 2FA |
| NET-01 | MED | Probable | OOB/BMC VLAN may initiate into management and server VLANs (asymmetric isolation) |
| NET-06 | MED | Confirmed | Config-backup credential can export all firewall secrets, including wildcard TLS key |
| HOST-01 | MED | Confirmed | Password-based root SSH enabled on four hosts |
| F-007 | MED | Confirmed | NFS exports use `no_root_squash` with UID-based auth |
| K8S-02 | MED | Confirmed | No Kubernetes network policies (flat pod network) |
| K8S-03 | MED | Confirmed | No Pod Security Admission on workload namespaces |
| K8S-04 | MED | Low | Kubernetes secrets likely not encrypted at rest |
| AI-01 | MED | Confirmed | LLM inference/router endpoint unauthenticated on the LAN |
| F-004 | MED | Suspected | Unidentified host exposing X11 and FTPS |
| HOST-02 | LOW | Confirmed | No brute-force/lockout protection (fail2ban inactive) anywhere |
| HOST-03 | LOW | Confirmed | Pending security updates across hosts |
| HOST-04 | LOW | Probable | `NOPASSWD` sudo entries to review |
| VIRT-04..06 | LOW | Confirmed | No fail2ban; container nesting features; corosync authkey world-readable |
| NET-02/04/05 | LOW | Confirmed | GUI on all interfaces; non-enforcing egress allowlist; appliances on mgmt VLAN |
| K8S-05/06 | LOW | Confirmed | A dashboard over plaintext HTTP; default SA token automount |
| F-008 | LOW | Confirmed | Storage encryption at rest disabled |
| NET-03 | INFO | Confirmed | Default SNMP community present in config (daemon not running) |
| VIRT-07/08, AI-02 | INFO | Confirmed | Non-expiring read-only token; leftover install media; a dashboard anon view |

### HIGH-severity detail

**F-001, untrusted devices on the management VLAN.** The firewall's ARP table and DHCP leases
show several non-infrastructure devices holding management-VLAN addresses: multiple cloud smart
plugs, a voice assistant, a network printer, and a personal mobile device. Because same-VLAN
traffic is not filtered by the firewall (which governs inter-VLAN only), these devices have
direct Layer-2 reach to the hypervisor management plane, the core switch, and the cluster
heartbeat. Root cause is Layer-2 placement, which bypasses an otherwise-correct Layer-3
segmentation design.

**F-003, legacy exposed services on the admin workstation.** The administrator workstation
(`WORKSTATION`), multi-homed on the management VLAN, runs cleartext FTP, an end-of-life SMB
implementation, NFS, and a web administration console, all reachable on the segment that F-001
shows also carries untrusted devices.

**F-006, unencrypted cluster-root SSH keys.** The workstation holds SSH private keys with no
passphrase; one authenticates as root across the whole cluster and also serves as the
commit-signing key. File permissions are correct, but there is no second factor on the key
material itself.

**K8S-01, anonymous private container registry.** The internal registry presents TLS but no
authentication challenge; unauthenticated clients can enumerate and pull images. Because cluster
nodes trust this registry, overwriting a tag yields cluster-wide code execution. (Read confirmed;
write inferred, as writing was out of scope.)

## Attack chains

Findings are most meaningful in combination. Four chains dominate the risk profile.

1. **IoT to cluster root (primary chain, configuration only, no exploit required).** An untrusted
   IoT device or the network printer sits on the management VLAN (F-001). From there it reaches
   the administrator workstation's cleartext FTP, end-of-life SMB, and NFS services (F-003). Local
   read access on that workstation yields an unencrypted SSH private key granting root across the
   entire cluster, which also signs commits (F-006). A twenty-dollar smart plug is thus three
   configuration steps from full cluster compromise.
2. **Service guest to hypervisor.** The hypervisor host firewall is fully disabled (VIRT-01) and
   the management API listens on all interfaces (VIRT-02), so a compromised service container has
   an unfiltered path to the node management port and SSH, where several hosts still permit
   password-based root login (HOST-01) with no brute-force protection (HOST-02).
3. **Supply chain.** The private container registry requires no authentication (K8S-01). Because
   the cluster trusts it, an attacker who can reach it on the LAN can overwrite an image tag and
   achieve code execution across the cluster.
4. **Workstation to edge secrets.** The unencrypted workstation key (F-006) plus the firewall
   config-backup credential stored there (NET-06) expose the wildcard TLS private key and full
   firewall configuration.

## Positive controls confirmed

- No inbound Internet exposure (no port-forwards; edge via an authenticated tunnel)
- Enforced inter-VLAN Layer-3 segmentation
- Authenticated and encrypted cluster heartbeat
- All containers unprivileged with no host-path mounts
- Clean Kubernetes role bindings and least-privilege hypervisor API token
- Grade-A TLS on all inspected services
- Good secret-file permissions and no hardcoded secrets in scripts or shell history
- A working SIEM and a mature alerting pipeline (including backup and quorum dead-man switches)

## What was NOT tested

Stating coverage gaps honestly is part of the exercise.

| Area | Why not covered |
|---|---|
| Core switch | Switch CLI authentication could not be reliably automated and the owner was remote. Layer-2 protections (DHCP snooping, dynamic ARP inspection, BPDU/root guard, storm control, 802.1X) and firmware status remain outstanding. |
| Kubernetes node-level CIS benchmark | Control-plane nodes were not directly reachable by shell, so node-file checks (kube-bench, secrets-encryption confirmation, kubelet/etcd flags) were not performed. |
| Firmware and wireless testing | Out of authorized scope. |
| Detection efficacy | Monitoring components are present and healthy, but attack traffic was deliberately not generated, so rule efficacy against the specific chains was not exercised. |
| CDN access-policy detail, per-application security headers | Require external-platform access not available in this pass. |

## Prioritized remediation roadmap

1. **Now (breaks the primary chain):** move IoT/printer/mobile devices off the management VLAN
   (F-001); add passphrases to the SSH keys (F-006); disable cleartext FTP and legacy SMB on the
   workstation (F-003); add authentication to the container registry (K8S-01).
2. **Near term (shrink internal blast radius):** enable the hypervisor firewall with a default-deny
   input policy and bind the management API to the management interface (VIRT-01/02); disable
   password-based root SSH and deploy brute-force protection (HOST-01/02); add a dedicated admin
   account with 2FA (VIRT-03); constrain the OOB-to-management path (NET-01).
3. **Hardening:** Kubernetes default-deny network policy and Pod Security Admission (K8S-02/03);
   confirm and enable Kubernetes secrets encryption (K8S-04); authenticate the LLM endpoint
   (AI-01); tighten NFS squashing (F-007); apply pending updates and review sudo scope
   (HOST-03/04).
4. **Follow-up:** complete the switch assessment when access is available; enforce the egress
   allowlist; rotate the config-backup credential and treat it as Tier-0 (NET-06); consider
   storage-at-rest encryption.

## Rules of engagement and methodology

Performed under explicit written authorization from the environment owner, against only
owner-controlled assets, strictly defensive and non-destructive:

- No exploitation, credential attacks, configuration changes, service restarts, or availability
  testing.
- Where a weakness would normally require exploitation to confirm, it was validated analytically
  from configuration and version evidence instead.
- Third-party platforms (the code-hosting provider, the CDN, the mobile carrier, and the external
  AI API) were treated as out of scope, except for review of the owner's own configuration.
- Evidence was recorded with commands and redacted output; secret values were never displayed.
  Raw evidence is retained by the assessor with restrictive permissions and is not committed to
  any repository.

Coverage spanned the network and segmentation, the firewall edge, host and OS hardening, the
virtualization and cluster identity layer, the Kubernetes/container platform, storage and backup,
identity and secrets, TLS, the application and AI layers, and monitoring. Tooling included nmap
(connect scans, service and cipher enumeration), gitleaks, trivy and kube-bench, authenticated
configuration inspection via SSH and the hypervisor/Kubernetes APIs, and read-only firewall
configuration review. Severity uses a qualitative scale informed by likelihood and
technical/operational impact; confidence is stated per finding.

## Companion work (also public)

The same environment is documented across a small portfolio:

- [Home-Lab](https://github.com/machismo0311/Home-Lab), the infrastructure itself, with topology
  references, runbooks, ADRs, and RCAs.
- [netframe-reliability-assessment-public](https://github.com/machismo0311/netframe-reliability-assessment-public),
  an SRE reliability assessment with a 12-mode FMEA, MTTR/MTBF reasoning, and DR planning.
- [netframe-ops-docs-public](https://github.com/machismo0311/netframe-ops-docs-public), a
  bare-metal build guide and disaster recovery plan.
- [netframe-monitor](https://github.com/machismo0311/netframe-monitor), a read-only cluster health
  monitor.

The full unredacted security edition and a red-team attack-path assessment are kept in private
repositories.

## Note

This is my own infrastructure, assessed under my own authorization, and published in redacted form
deliberately. Demonstrating what to *withhold* is part of the exercise.
