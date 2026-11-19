# 06 - Offline Repository and Connected-Mode Policy

The kit must deploy fully air-gapped, which means every bit needed to build and run it is
carried on offline media. This document defines what that media contains, how it is built
and refreshed at base before a mission, how integrity is verified on site, and the policy
for when the kit is allowed online.

## Offline media layout

Carry the repository on **at least two independent drives** (primary + backup), stored in
separate cases. Encrypted at rest (see [11-security.md](11-security.md)).

```
flyaway-media/
  MANIFEST.yaml            # single source of truth: versions, checksums, build date
  CHECKSUMS.sha256         # sha256 of every artifact
  os/
    rocky-9-dvd.iso        # OS install media (and Ubuntu alt if used)
    packages/              # offline dnf repo mirror for OS deps
  splunk/
    splunk-enterprise-10.6-*.rpm
    splunkforwarder-10.x-*.rpm
    splunk-es-<ver>.spl
  soar/
    splunk-soar-onprem-8.7-*.tgz
  apps-tas/
    Splunk_TA_windows/ Splunk_TA_nix/ Splunk_TA_<vendor>/ ...
  sc4s/
    sc4s-<ver>.container   # SC4S image tarball for offline load
  images/                  # optional: docker/podman save tarballs (splunk/splunk, UF, mgmt tools)
    splunk-splunk-10.6.tar # see 07-containerization.md
  licenses/
    splunk-enterprise.lic
  configs/
    kit_base/              # base app: outputs, indexes, serverclasses templates
  vpn/
    headscale-<ver>/       # if Headscale carried (see doc 13)
  docs/                    # a copy of this repo's rendered docs + checklists
```

## MANIFEST.yaml

The manifest is the authoritative record of exactly what this kit build contains. Pin
every version; nothing is "latest".

```yaml
kit_build: 2026-11-18
tier_target: medium
software:
  splunk_enterprise: "10.6.x"      # exact build
  universal_forwarder: "10.x.x"
  enterprise_security: "<ver>"
  soar: "8.7.x"
  sc4s: "<ver>"
os:
  primary: "Rocky Linux 9.x"
  alt: "Ubuntu 24.04.x"
apps_tas:
  - {name: Splunk_TA_windows, version: "<ver>"}
  - {name: Splunk_TA_nix, version: "<ver>"}
license:
  file: licenses/splunk-enterprise.lic
  expires: "<date>"
checksums: CHECKSUMS.sha256
```

Keep `MANIFEST.yaml` in version control (this repo) so each mission's build is auditable,
and so the BOM/manifest could later be generated from it (noted in
[02-hardware-bom.md](02-hardware-bom.md)).

## Building / refreshing at base (connected)

Done at home base where internet is available, before deployment:

1. Download the pinned Splunk Enterprise, UF, ES, SOAR, and SC4S artifacts; download the
   OS ISO.
2. Mirror the OS package repo for offline `dnf` (so hardening/deps install with no
   internet). For RHEL-family: `reposync` the needed repos into `os/packages/`.
3. Collect the required TAs/apps from Splunkbase at pinned versions into `apps-tas/`.
4. Place the license in `licenses/`.
5. Generate checksums: `sha256sum` every artifact into `CHECKSUMS.sha256`.
6. Update `MANIFEST.yaml` with versions, dates, and license expiry.
7. Write to both media drives; verify each drive against `CHECKSUMS.sha256`.
8. Record the refresh in the pre-departure checklist
   ([appendix-checklists.md](appendix-checklists.md)).

Refresh cadence: before every mission, and whenever a security patch or license renewal
lands. License expiry is a go/no-go item - check it at pack time, not on site.

## Verifying on site (air-gapped)

Before installing anything, verify integrity so a corrupted or tampered drive is caught
early:

```bash
cd flyaway-media
sha256sum -c CHECKSUMS.sha256
```

All lines must report `OK`. If the primary drive fails, switch to the backup drive and
note the failure in the mission report. Set up the OS offline repo by pointing `dnf` at
`os/packages/` (a local `.repo` file), then proceed with
[04-os-baseline.md](04-os-baseline.md) and [05-splunk-deployment.md](05-splunk-deployment.md).

## Connected-mode policy

When a mission has internet, online access is **opt-in and controlled**, never the
default. The air-gapped path is always the primary path so behavior is identical whether
or not a link exists.

| Activity | Air-gapped | Connected (allowed if approved) |
|----------|-----------|----------------------------------|
| OS / Splunk install | Offline media only | Offline media only (do not pull "latest" mid-mission) |
| Security patches | Carried on media | May pull **pinned** patches through the firewall to a staging host, re-checksum, then deploy |
| App/TA updates | Carried on media | Pinned downloads only, via firewall, logged |
| License | File on media | File on media (online activation only if the mission explicitly allows) |
| Remote replication | n/a | Per [14-remote-replication-vpn.md](14-remote-replication-vpn.md) |
| Time sync | chrony free-run / GPS | chrony to approved upstream |

Rules:

- No kit host reaches the internet directly; any egress is through the firewall with an
  explicit allow rule and is logged.
- Nothing installed mid-mission bypasses the checksum/manifest discipline: download to a
  staging area, verify, update the manifest, then deploy.
- Meraki-managed gear (if carried) inherently needs the cloud dashboard - using it is a
  connected-mode decision, incompatible with air-gapped missions
  ([02-hardware-bom.md](02-hardware-bom.md), [03-network-design.md](03-network-design.md)).

## Integrity self-check

The one runnable check this doc leaves behind: on both media drives, `sha256sum -c
CHECKSUMS.sha256` must pass end-to-end, and the versions it covers must match
`MANIFEST.yaml`. If either drive or the version match fails, the media is not mission-ready.
