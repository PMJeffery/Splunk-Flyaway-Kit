# 12 - Mission Profiles

The kit is general-purpose. A mission profile is a preset - which tier, which apps/TAs,
which indexes, and which operational emphasis - so the team loads the right content for
the job instead of improvising on site. Profiles are additive on top of the kit base app.

| Profile | Default tier | Emphasis |
|---------|-------------|----------|
| [Incident response / DFIR](#incident-response--dfir) | Medium | Fast collection, evidence integrity, ES correlation, SOAR |
| [Tactical / disconnected ops](#tactical--disconnected-operations) | Small-Medium | Air-gap, resilience, low power/weight |
| [Demo / training / workshop](#demo--training--workshop) | Small | Self-contained, repeatable, disposable data |

## Incident response / DFIR

**Goal:** stand up collection and analysis at a compromised or at-risk site quickly, with
evidence handled defensibly.

- **Tier:** Medium (cluster survives one indexer loss; ES + SOAR). Large if volume/
  retention demand it.
- **Indexes:** `mission_wineventlog`, `mission_sysmon`, `mission_edr`, `mission_firewall`,
  `mission_proxy`, `mission_dns`, `mission_auth`, plus a catch-all `mission_misc`.
- **Apps / TAs:** Enterprise Security; Splunk_TA_windows, Splunk_TA_microsoft_sysmon,
  Splunk_TA_nix, the relevant EDR and firewall TAs; SOAR with IR playbooks.
- **Ingestion:** UFs to live endpoints via the DS; HF for API sources (EDR/cloud); SC4S
  for network/security device syslog; one-shot file upload / `oneshot` for collected
  artifacts (triage images, exported logs).
- **Emphasis:** chain of custody from first collection
  ([11](11-security.md#data-handling-and-chain-of-custody)); full-fidelity preservation at
  export ([10](10-runbook-teardown.md#phase-2--export--preserve)); ES notables -> SOAR for
  repeatable response; tight RBAC so analysts can't alter evidence indexes.
- **Teardown:** purge-level wipe after verified evidence handoff; rotate all secrets.

## Tactical / disconnected operations

**Goal:** monitoring where connectivity is poor or absent (field, disaster response,
exercise), with resilience and minimal logistics.

- **Tier:** Small or Medium depending on volume. Favor Medium's cluster if a node loss is
  likely and data must survive.
- **Indexes:** mission-specific - sensor/telemetry, comms logs, local infra, `mission_misc`.
- **Apps / TAs:** only what the mission needs - keep the footprint small to save indexer
  CPU and power. Dashboards for the specific watch; alerting tuned for an operator, not a
  SOC.
- **Ingestion:** mostly UF and SC4S from local systems; HEC for any local apps. Assume no
  internet - everything comes off the offline media
  ([06](06-offline-repo.md)).
- **Emphasis:** air-gapped operation is the default; chrony/GPS time discipline for long
  deployments ([03](03-network-design.md#kit-services-dns-dhcp-ntp)); power/weight economy
  ([02](02-hardware-bom.md)); degraded-mode drills so one person can keep it running
  ([09](09-runbook-operations.md#degraded-mode-operation)).
- **Optional:** remote replication to a home DC/cloud **only if** an intermittent link
  exists, using low-bandwidth scheduled export ([14](14-remote-replication-vpn.md)).

## Demo / training / workshop

**Goal:** a self-contained teaching or proof-of-value environment that sets up fast and
resets cleanly.

- **Tier:** Small (single analyst view; cluster not required).
- **Indexes:** `demo_*` with pre-canned or synthetic data; short retention.
- **Apps / TAs:** ES and a couple of representative TAs to show CIM/correlation; sample
  dashboards; optionally SOAR to demo playbooks.
- **Ingestion:** mostly pre-loaded sample data plus a live HEC/UF demo source.
- **Emphasis:** repeatability (same build every time from the manifest), a fast reset
  (re-load sample data, no evidence handling), and disposable data (standard wipe).
- **Connectivity:** either mode; demos are a fine place to exercise connected-mode
  features (Meraki dashboard, remote replication) without evidence risk.

## Choosing and loading a profile

1. Pick the profile and confirm the tier against expected daily volume and retention
   ([01 sizing](01-architecture.md#sizing-math)).
2. Ensure the profile's apps/TAs and sample data are on the offline media and in the
   manifest ([06](06-offline-repo.md)).
3. Create the profile's indexes (cluster bundle on M/L) and DS serverclasses mapping each
   data source to its TA ([05](05-splunk-deployment.md)).
4. Use the Flyaway Kit app concept ([15](15-flyaway-kit-app-concept.md)) to check off the
   site's actual sources and confirm each required TA/UF is configured and reporting.

> Profiles are presets, not hard limits. The general-purpose kit can mix elements (e.g. an
> IR mission that also needs tactical air-gap discipline).
