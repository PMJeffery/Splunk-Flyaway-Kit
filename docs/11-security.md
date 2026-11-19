# 11 - Security and Data Handling

The kit holds and transports potentially sensitive data on portable hardware into
uncontrolled environments. This document covers host and Splunk hardening, TLS, secrets
and VPN key handling, physical security, data classification, and chain of custody.

## Threat model (brief)

- **Loss or theft** of a laptop/desktop/drive in transit or on site -> full-disk
  encryption and secure wipe are mandatory.
- **Hostile / untrusted site network** -> the kit integrates through a firewall with
  default-deny; it does not trust the org LAN.
- **Data leakage between missions** -> teardown wipe and secret rotation prevent one
  mission's data or keys from surviving into the next.
- **Tampered media/supply** -> checksum/manifest discipline
  ([06](06-offline-repo.md#integrity-self-check)).

## Host hardening

Builds on [04-os-baseline.md](04-os-baseline.md):

- **Full-disk encryption** (LUKS) on every host, especially data volumes. Keys are not
  stored on the same device; use a passphrase or an externally-carried key. Crypto-erase
  at teardown destroys the key ([10](10-runbook-teardown.md#phase-4--secure-wipe)).
- **SELinux enforcing**, **firewalld default-deny** per host, minimal package set.
- Per-operator OS accounts; no shared logins; `sudo` logged and forwarded to Splunk
  (`_audit`/`index=os`).
- Disable USB mass storage auto-mount except the designated media/evidence drives.
- BIOS/UEFI password and Secure Boot where supported.

## Splunk hardening

- **Change all default credentials** on first boot; no `changeme`/`admin:changeme`
  survives Phase 3 of deploy.
- **TLS everywhere** - Splunk Web (8000), splunkd/REST (8089), forwarder-to-indexer
  (9997), HEC (8088), and inter-node management. Carry a kit CA on the offline media and
  issue per-host certs; distribute the CA so clients validate. Avoid shipping the default
  self-signed certs into production use.
- **Role-based access** on the ES SH: analysts get analyst roles, not admin. Limit who
  can touch the MC/CM/LM to admins ([Splunk guidance](https://help.splunk.com/en/splunk-enterprise/administer/monitor/10.6/configure-the-monitoring-console/which-instance-should-host-the-console)).
- **pass4SymmKey** secrets (indexer cluster, SHC) are strong and unique per kit build,
  stored only in `server.conf` (which lives on encrypted disk) and in the sealed secrets
  record - never in this repo.
- **HEC tokens** are generated at deploy, scoped to the minimum index/sourcetype, and
  rotated at teardown. No tokens committed anywhere in plaintext.
- Forward Splunk's own `_internal`/`_audit` so the kit audits itself.

## TLS and secrets inventory

Track every secret the kit uses so rotation at teardown is complete:

| Secret | Where | Rotate at teardown? |
|--------|-------|---------------------|
| Full-disk encryption keys | Externally carried / passphrase | Yes (crypto-erase) |
| Splunk admin passwords | Each instance | Yes |
| Indexer cluster `pass4SymmKey` | CM + peers `server.conf` | Yes |
| SHC `pass4SymmKey` | ES SHC members | Yes |
| HEC tokens | HF/indexer inputs | Yes |
| Kit CA + host private keys | mgmt / each host | Yes (reissue per build) |
| VPN keys (IPsec PSK/certs, WireGuard keys) | Firewall / Headscale | Yes |
| SOAR admin + DB creds | SOAR host | Yes |
| Firewall/switch admin creds | Network gear | Yes |

Keep a sealed, encrypted **secrets record** per mission (not in this repo). It is the
authoritative list the teardown rotation step works from.

## VPN key handling

For remote replication ([14](14-remote-replication-vpn.md)):

- **IPsec:** prefer certificate auth over a pre-shared key; if PSK, make it long and
  unique per mission. Store on the firewall only.
- **Headscale/WireGuard:** each node has its own keypair; the Headscale control server
  holds node keys and ACLs. Pre-authvkeys are short-lived and single-use. Revoke nodes in
  Headscale at teardown and rotate the control server's keys.
- Treat the remote peer config as sensitive: it reveals the remote DC/cloud topology.
  Wipe it from the firewall at teardown.

## Physical security

- Cases locked (and tamper-evident sealed for IR/evidence missions).
- Hosts and drives inventoried by serial/asset tag ([02](02-hardware-bom.md)); the
  inventory is checked at pack and at repack.
- Evidence drives handled under chain of custody (below), stored separately from the
  compute cases.
- Never leave an unlocked, logged-in analyst session unattended on site.

## Data classification

Classify mission data on arrival; it drives retention, export, and wipe method:

| Class | Examples | Handling |
|-------|----------|----------|
| Public / demo | Training data, synthetic | Standard wipe at teardown |
| Internal | Operational logs, non-PII telemetry | Encrypted at rest + standard secure wipe |
| Sensitive / PII | User data, customer logs | Encrypted; restricted access; purge-level wipe; documented export only |
| Evidence | IR artifacts for legal/forensic use | Chain of custody; full-fidelity preservation; purge-level wipe after verified handoff |

## Data handling and chain of custody

For incident-response missions, treat captured data as potential evidence from the moment
of collection:

- Record collection: source, index, time range, who collected, when, and the kit host.
- Preserve integrity: checksum (sha256) artifacts at export; prefer bucket-level or frozen
  preservation for full fidelity ([10](10-runbook-teardown.md#phase-2--export--preserve)).
- Document every transfer/handoff with recipient, time, and acknowledgement on the
  chain-of-custody form.
- Minimize copies; track every copy; destroy working copies at teardown after the
  authoritative handoff is verified.
- Keep the chain-of-custody form with the artifact and in the post-mission report.

> This is operational guidance, not legal advice. For evidence that may be
> used in legal proceedings, follow the organization's and jurisdiction's forensic
> standards; they override anything here.

## Connected-mode caution

Going online expands the attack surface. Every egress is through the firewall with an
explicit, logged allow rule; nothing bypasses the checksum/manifest discipline
([06](06-offline-repo.md#connected-mode-policy)). Meraki-managed gear's cloud dependency
is itself a connected-mode decision and must not be used on air-gapped or sensitive
missions.
