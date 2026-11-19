# 08 - Runbook: Rapid Deployment

The on-site procedure to go from cases-closed to operational. Assumes hosts were imaged to
the [OS baseline](04-os-baseline.md) at base and the offline media is verified. Work the
steps in order; the order mirrors the Splunk install dependency chain.

## Time target

| Tier | Target (trained 2-person team, pre-imaged hosts) |
|------|--------------------------------------------------|
| Small | ~30-45 min |
| Medium | ~60-90 min |
| Large | ~2-3 hours |

If hosts are **not** pre-imaged, add the OS build time from
[04-os-baseline.md](04-os-baseline.md). Pre-imaging at base is strongly recommended.

## Phase 0 - Physical and power (all tiers)

1. Unpack network + power case first. Place the switch and firewall; connect the PDU to
   the UPS, UPS to mains.
2. Power on the **switch**, then the **firewall**. Confirm switch link lights.
3. Unpack compute. Rack/place indexers (desktops) and lay out laptops.
4. Cable every host to the switch per the VLAN color code
   ([03-network-design.md](03-network-design.md)); label both ends.
5. Power on hosts. Confirm each gets its static IP and is reachable.

## Phase 1 - Network services

1. Bring up `mgmt` services: DNS, DHCP (reserved range only), and NTP (chrony).
2. Verify from another host: `nslookup idx01.kit.local`, `chronyc sources`, time within
   tolerance on all hosts.
3. Verify firewall default-deny is active and the mgmt VLAN can administer switch/fw.

> Do not connect the site VLAN yet. Stand up the kit internally first, then integrate with
> the org network per [13-site-integration.md](13-site-integration.md).

## Phase 2 - Verify offline media

On `mgmt`: `sha256sum -c CHECKSUMS.sha256` passes; versions match `MANIFEST.yaml`
([06-offline-repo.md](06-offline-repo.md)). Confirm license not expired.

## Phase 3 - Splunk core (follow install order)

Per [05-splunk-deployment.md](05-splunk-deployment.md#install-order-all-tiers):

1. **LM** up; license installed; note the `manager_uri`.
2. **CM** up (M/L); cluster secret set; `replication_factor`/`search_factor` set.
3. **Indexers** up; standalone receiving (S) or joined to cluster (M/L). For clusters:
   `splunk apply cluster-bundle` on CM, then `splunk show cluster-status` = complete.
4. **DS** up; serverclasses defined.
5. **Search heads**: ES SH (all), ad-hoc SH (M opt), ES SHC + deployer (L).
6. **MC** up; distributed mode; all instances added as peers; dashboards green.

Checkpoint: every instance points at the LM; cluster is complete; ES SH lists all peers.

## Phase 4 - Ingest

1. Bring up HF/HEC and SC4S ([05](05-splunk-deployment.md#ingest-tier)).
2. Confirm forwarder outputs reach indexers (9997) and HEC answers on 8088.
3. Do **not** open to the site yet - test with a local synthetic event first (Phase 6).

## Phase 5 - SOAR and AI

1. SOAR on its RHEL-family host; connect its Splunk asset to the ES SH; wire ES notables
   to SOAR if the mission uses automation.
2. AI server (optional): deploy the harness + Splunk MCP server per
   ([05](05-splunk-deployment.md#ai-server-optional)); confirm the data posture is allowed
   for a non-local harness ([01](01-architecture.md#ai-server-optional)).

## Phase 6 - Validation

Run before declaring operational. Minimum gate:

```spl
| rest /services/server/info | table splunk_server, version
```

- **Cluster health** (M/L): `splunk show cluster-status` on CM = all peers up, RF/SF met.
- **Peers visible**: on ES SH, Settings > Distributed search shows all indexers up.
- **Ingest per path** - send one synthetic event through each path and confirm it is
  searchable:

```spl
index=* earliest=-5m | stats count by index, sourcetype, host
```

  - Forwarder path: a UF/HF test event appears.
  - HEC: `curl -k https://hf01.kit.local:8088/services/collector -H "Authorization: Splunk <token>" -d '{"event":"kit-hec-test"}'` then search for it.
  - SC4S: send a test syslog line to `sc4s01` and confirm.
- **MC**: all instances green, no missing-peer or replication alerts.
- **Time**: `chronyc tracking` within tolerance on all hosts.
- **ES**: loads; data models targeted at mission indexes populate.

Record pass/fail for each on the arrival checklist
([appendix-checklists.md](appendix-checklists.md#arrival--deployment)).

## Phase 7 - Site integration

Only after Phase 6 passes, connect the site VLAN and begin onboarding the org's data
sources following [13-site-integration.md](13-site-integration.md). Optionally bring up
remote replication ([14-remote-replication-vpn.md](14-remote-replication-vpn.md)).

## If something fails

Go to [09-runbook-operations.md](09-runbook-operations.md) for the common-failure table
and degraded-mode operation. Do not proceed past a failed validation gate; a kit that
silently drops data is worse than a kit that is late.
