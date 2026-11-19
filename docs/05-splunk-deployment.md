# 05 - Splunk Deployment

Per-tier install order and role configuration for Splunk Enterprise 10.6, Enterprise
Security, and SOAR 8.7. Config snippets are copy-paste starting points - set them in app
context (e.g. `$SPLUNK_HOME/etc/apps/kit_base/local/`) where practical, not in
`system/local`, so they travel with the kit's base app.

All hostnames/IPs/ports refer to [03-network-design.md](03-network-design.md). Licensing,
clustering secrets, and TLS are covered in [11-security.md](11-security.md).

This document covers the bare-metal install. Any role except SOAR can instead run as a
Docker/Podman container for faster deployment and host consolidation - see
[07-containerization.md](07-containerization.md). The two styles mix freely within one kit.

## Install order (all tiers)

Order matters so each role can register with its manager as it comes up:

1. **License manager (LM)** - so every other instance can point its license at it.
2. **Cluster manager (CM)** - Small/Medium/Large, before indexers.
3. **Indexers** - join the cluster (S/M/L) or come up standalone (S).
4. **Deployment server (DS)** - before forwarders/HF so they pull apps.
5. **Search heads** (ES, ad-hoc) / **ES SH cluster + deployer** (L).
6. **Monitoring console (MC)** - last, so it discovers a complete topology.
7. **Ingest**: HF, HEC, SC4S.
8. **SOAR**, then the optional **AI server**.

## Licensing

Point every instance at the LM. On the LM, install the license; on peers:

```ini
# $SPLUNK_HOME/etc/system/local/server.conf  (on each non-LM instance)
[license]
manager_uri = https://mgmt01.kit.local:8089
```

Carry the license file on the offline media ([06-offline-repo.md](06-offline-repo.md)).
For disconnected missions confirm the license terms allow offline operation; the
no-enforcement grace behavior should be understood before you rely on it.

## Management roles and colocation

Follow the colocation rules in
[01-architecture.md](01-architecture.md#management-role-colocation). The critical one:
**the cluster manager must not share a host with the deployment server, an indexer, or a
search head.**

- **Small (no cluster):** `mgmt01` = LM + DS + MC + DNS/DHCP/NTP. MC-on-DS is supported
  because deployment clients are far under 50.
- **Medium/Large (clustered):** `mgmt01` = CM + LM + MC; `mgmt02` = DS (+ ES SHC deployer
  on Large) + DNS/DHCP/NTP. This keeps CM away from DS.

## Indexers

### Standalone (Small)

Enable receiving and define indexes. No cluster config.

```ini
# inputs.conf - receive from forwarders
[splunktcp://9997]
```

```ini
# indexes.conf - mission indexes (example)
[mission_wineventlog]
homePath   = $SPLUNK_DB/mission_wineventlog/db
coldPath   = $SPLUNK_DB/mission_wineventlog/colddb
thawedPath = $SPLUNK_DB/mission_wineventlog/thaweddb
maxTotalDataSizeMB = 512000
frozenTimePeriodInSecs = 2592000   # 30 days; tune per retention
```

### Clustered (Medium/Large)

On the **CM** (`mgmt01`):

```ini
# server.conf
[clustering]
mode = manager
replication_factor = 2
search_factor = 2
pass4SymmKey = <cluster-secret>     # same on CM and all peers; see 11-security.md
cluster_label = flyaway

[license]
manager_uri = https://mgmt01.kit.local:8089   # LM is local on mgmt01
```

On each **peer/indexer**:

```ini
# server.conf
[replication_port://9887]

[clustering]
mode = peer
manager_uri = https://mgmt01.kit.local:8089
pass4SymmKey = <cluster-secret>
cluster_label = flyaway
```

```ini
# inputs.conf
[splunktcp://9997]
```

Push indexes to peers from the CM as the cluster bundle
(`$SPLUNK_HOME/etc/manager-apps/`) rather than editing peers directly:

```ini
# manager-apps/kit_indexes/local/indexes.conf  (replicated to all peers)
[mission_wineventlog]
homePath   = $SPLUNK_DB/mission_wineventlog/db
coldPath   = $SPLUNK_DB/mission_wineventlog/colddb
thawedPath = $SPLUNK_DB/mission_wineventlog/thaweddb
repFactor  = auto
frozenTimePeriodInSecs = 2592000
```

Apply with `splunk apply cluster-bundle` on the CM. Confirm the cluster is complete
(`splunk show cluster-status`) before ingesting.

## Search heads

### ES search head (all tiers)

Dedicated host. Add the indexers as search peers (standalone) or configure as a cluster
search head (clustered). Clustered:

```ini
# server.conf on the ES SH
[clustering]
mode = searchhead
manager_uri = https://mgmt01.kit.local:8089
pass4SymmKey = <cluster-secret>
```

Install Enterprise Security (see [ES integration](#enterprise-security-integration)).

### ES search head cluster (Large)

An odd number of members (3) plus a deployer (on `mgmt02`). Initialize each member and the
captain:

```ini
# server.conf on each SHC member
[shclustering]
conf_deploy_fetch_url = https://mgmt02.kit.local:8089
pass4SymmKey = <shc-secret>
shcluster_label = flyaway_shc
replication_factor = 2
```

Then bootstrap the captain (`splunk bootstrap shcluster-captain ...`). Deploy ES and apps
from the deployer with `splunk apply shcluster-bundle`. ES on an SHC has extra steps -
follow the ES install guide for SHC and install ES on the deployer first.

### Ad-hoc search head (Medium, optional)

Same cluster-search-head config as ES SH, without ES. For hunting/dashboards so analysts
don't add load to the ES SH.

## Deployment server (forwarder management)

The DS pushes TAs/apps to UFs and HFs via serverclasses. Define classes by data source so
the Flyaway Kit app concept ([15-flyaway-kit-app-concept.md](15-flyaway-kit-app-concept.md))
can map a checked source to a serverclass.

```ini
# serverclass.conf on the DS (mgmt01 in S, mgmt02 in M/L)
[serverClass:all_forwarders]
whitelist.0 = *

[serverClass:all_forwarders:app:Splunk_TA_windows]
restartSplunkd = true

[serverClass:syslog_sources:app:Splunk_TA_nix]
whitelist.0 = *linux*
```

Clients point at the DS:

```ini
# deploymentclient.conf on each UF/HF
[deployment-client]
[target-broker:deploymentServer]
targetUri = mgmt02.kit.local:8089   # mgmt01 in Small
```

## Ingest tier

### Forwarder output (UF/HF -> indexers)

Standalone indexers - list them directly; clustered - also list all peers (indexer
discovery via the CM is the cleaner option for clusters):

```ini
# outputs.conf on UF/HF
[tcpout]
defaultGroup = kit_idx

[tcpout:kit_idx]
server = idx01.kit.local:9997, idx02.kit.local:9997, idx03.kit.local:9997
# clustered alternative - indexer discovery:
# [indexer_discovery:kit]
# manager_uri = https://mgmt01.kit.local:8089
```

### HEC (on hf01/indexers)

```ini
# inputs.conf
[http]
disabled = 0
port = 8088
enableSSL = 1

[http://mission_app]
token = <generated-token>
index = mission_app
sourcetype = mission:app
```

Manage tokens from the mgmt/token admin; carry no plaintext tokens in the repo.

### SC4S (syslog)

SC4S is Splunk's supported containerized syslog collector. On `sc4s01` (or `hf01` in
Small) run the SC4S container (Podman/Docker), listening on 514/UDP-TCP and 6514/TLS, and
forwarding to the indexers via the same `kit_idx` output group over HEC or splunktcp.
Point network/security devices at `sc4s01`'s site-VLAN address
([13-site-integration.md](13-site-integration.md)).

## Monitoring console

Bring the MC up last on `mgmt01`, set it to distributed mode, and add all instances as
search peers so the whole kit is visible in one place. Respect the colocation rules: in a
clustered kit the MC lives with the CM/LM on `mgmt01`, never on an indexer.

## Enterprise Security integration

- Install ES on the dedicated ES SH (or on the deployer first for an SHC).
- Confirm the ES version is compatible with Splunk Enterprise 10.6 before the mission -
  pin both in the manifest.
- Map mission indexes into the ES/CIM data models so correlation searches and dashboards
  populate. The Flyaway Kit app concept tracks CIM coverage per onboarded source.
- Accelerate the data models you actually use; on a kit, disable the ones you don't to
  save indexer CPU.

## SOAR deployment

- Install on a RHEL-family host only ([04-os-baseline.md](04-os-baseline.md#soar-os-constraints-verified-against-splunk-docs)).
  Not on Ubuntu, not in a container.
- Unprivileged install as `phantom` into `/opt/phantom`, umask 0022.
- Connect SOAR to Splunk: configure the Splunk asset in SOAR pointing at the ES SH
  (REST/8089) and configure ES/SOAR to send notables to SOAR for playbook automation.
- Keep playbooks and custom apps on the offline media; load them during deployment.

## AI server (optional)

See [01-architecture.md](01-architecture.md#ai-server-optional) for the concept. `ai01` is
a simple PC/laptop running an AI harness connected to the Splunk MCP server.

1. **Stand up the Splunk MCP server** on `ai01` (or on the search head) and point it at the
   ES SH / indexers over REST (8089). Scope its Splunk credentials to a read-oriented role.
2. **Install the chosen harness** on `ai01`:
   - *Local harness (e.g. Pi)* - carry its installer and local model on the offline media
     ([06](06-offline-repo.md)); works air-gapped.
   - *Subscription harness (Claude / Codex)* - install the vendor client and configure its
     API key; requires firewall egress to the vendor (connected mode only).
   - *Team's private AI service* - install the team's harness (e.g. a Pi Agent) and point
     it at their private endpoint, reached per [14](14-remote-replication-vpn.md).
3. **Register the MCP server with the harness** so the agent can list indexes, run
   searches, and explain/optimize SPL against kit data.
4. **Integrate** for analysts: use the harness directly, or invoke it from SOAR actions.
5. Confirm the data-handling posture is allowed for the mission before sending any mission
   data to a non-local harness ([11](11-security.md#data-classification)).

## Validation

After install, before declaring the kit operational, run the checks in
[08-runbook-deploy.md](08-runbook-deploy.md#phase-6--validation) and
[appendix-checklists.md](appendix-checklists.md). Minimum: cluster complete, ES SH sees
all peers, a test event via each ingest path is searchable, MC shows all instances green.
