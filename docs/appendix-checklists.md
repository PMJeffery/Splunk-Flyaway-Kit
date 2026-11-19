# Appendix - Checklists

Printable, mission-agnostic checklists. Print and carry; initial each line. They
summarize the runbooks - the detail lives in [08](08-runbook-deploy.md),
[09](09-runbook-operations.md), and [10](10-runbook-teardown.md).

## Pre-departure (at base)

- [ ] Tier selected for expected daily volume and retention
- [ ] All hosts imaged to the OS baseline ([04](04-os-baseline.md))
- [ ] OS baseline self-check passed on every host (below)
- [ ] Offline media refreshed; `MANIFEST.yaml` updated with pinned versions
- [ ] `sha256sum -c CHECKSUMS.sha256` passes on BOTH media drives
- [ ] License present and NOT near expiry
- [ ] Required apps/TAs and mission profile content on media
- [ ] Secrets record prepared (sealed/encrypted)
- [ ] Hardware inventory by serial/asset tag complete; spares packed
- [ ] Switch/firewall configs staged (Catalyst/Secure FW for air-gap; Meraki only if connected)
- [ ] UPS charged; PDU, region power adapters, cables (10G-rated) labeled and packed
- [ ] Printed runbooks + these checklists in the kit

## OS baseline self-check (per host)

- [ ] Hostname + static IP match the IP plan ([03](03-network-design.md))
- [ ] THP disabled (`cat /sys/kernel/mm/transparent_hugepage/enabled` -> `[never]`)
- [ ] ulimits for `splunk`: `nofile>=64000`, `nproc>=16000`
- [ ] chrony synced (`chronyc tracking`)
- [ ] `splunk` user + `/opt/splunk` volume present (`phantom` + `/opt/phantom` on SOAR)
- [ ] firewalld active, default-deny, host's ports open
- [ ] SELinux enforcing (`getenforce`)
- [ ] Resolvable by name from `mgmt`

## Arrival / deployment

Phases per [08](08-runbook-deploy.md).

- [ ] Phase 0: switch + firewall up; all hosts cabled, powered, reachable
- [ ] Phase 1: DNS, DHCP (reserved range), NTP verified from a client
- [ ] Phase 2: offline media verified on site; license not expired
- [ ] Phase 3: LM -> CM -> indexers -> DS -> SHs -> MC up in order
- [ ] Phase 3: cluster complete (`splunk show cluster-status`), RF/SF met (M/L)
- [ ] Phase 3: ES SH lists all indexers as search peers
- [ ] Phase 4: HF/HEC/SC4S up; outputs reach indexers
- [ ] Phase 5: SOAR connected to ES SH; optional AI server harness + MCP up (data posture checked)
- [ ] Phase 6 VALIDATION GATE - all must pass:
  - [ ] `| rest /services/server/info` lists every instance at the right version
  - [ ] Synthetic event searchable via forwarder path
  - [ ] Synthetic event searchable via HEC
  - [ ] Synthetic event searchable via SC4S syslog
  - [ ] MC all green; no missing peers / replication alerts
  - [ ] Time within tolerance on all hosts
  - [ ] ES loads; targeted data models populate
- [ ] Phase 7: site VLAN connected; onboarding started ([13](13-site-integration.md))
- [ ] Remote replication up and verified if used ([14](14-remote-replication-vpn.md))

## Daily operations (per shift)

Per [09](09-runbook-operations.md).

- [ ] MC all green; no skipped searches; no indexing latency alerts
- [ ] Cluster status: peers up, RF/SF met (M/L)
- [ ] Each expected source reporting within last 15 min
- [ ] Every indexer data volume < 80% full
- [ ] Time in sync everywhere
- [ ] UPS load/runtime nominal; no host unexpectedly on battery
- [ ] Firewall deny log reviewed for unexpected site traffic

## Teardown

Per [10](10-runbook-teardown.md). Do NOT wipe before export + verified handoff.

- [ ] Site VLAN disconnected; ingest stopped; in-flight data drained
- [ ] Required data exported; artifacts checksummed (sha256)
- [ ] Remote mirror confirmed complete (if used)
- [ ] Evidence handed off; chain-of-custody form signed/acknowledged
- [ ] Handoff confirmed readable/complete by recipient
- [ ] Secure wipe of all data volumes (method per classification)
- [ ] Firewall + switch configs reset to clean baseline
- [ ] All secrets rotated (cluster/SHC keys, HEC tokens, VPN keys, admin creds, FDE keys)
- [ ] Wipe method + verification recorded per device
- [ ] Hardware inventoried against BOM; damage/spares noted
- [ ] Offline media re-verified or flagged for refresh
- [ ] Repacked to foam; cables coiled/labeled
- [ ] Post-mission report written ([10](10-runbook-teardown.md#phase-6--post-mission-report))
