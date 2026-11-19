# 09 - Runbook: Operations

Keeping the kit healthy during a mission: what to watch, routine checks, common failures,
and how the kit behaves (and what you do) when a node is lost. Field conditions - power,
heat, bumps - make hardware failure the expected case, not the exception.

## Daily operational checks

Run each operational period (start of shift):

1. **Monitoring console** - open the MC on `mgmt01`; confirm all instances green, no
   skipped searches, no indexing latency alerts, no missing cluster peers.
2. **Cluster status** (M/L): `splunk show cluster-status` on the CM - all peers up, RF and
   SF met, no buckets fixing for long.
3. **Ingest flowing** - per source:
   ```spl
   index=* earliest=-15m | stats count latest(_time) as last_seen by index, sourcetype
   ```
   Any expected source with `count=0` or a stale `last_seen` is investigated.
4. **Disk headroom** - on every indexer, data volume < 80% used.
5. **Time** - `chronyc tracking` within tolerance everywhere.
6. **Power** - UPS load and runtime nominal; no host on battery unexpectedly.
7. **Firewall** - review deny log for unexpected site traffic.

## Key health searches

```spl
# Indexing latency (event time vs index time)
index=_internal source=*metrics.log group=per_index_thruput
| eval latency = _indextime - _time | stats avg(latency) max(latency) by series

# Forwarder connectivity
| rest /services/deployment/server/clients | table hostname, utsname, lastPhoneHomeTime

# Skipped/queued searches (search head pressure)
index=_internal sourcetype=scheduler status=skipped
| stats count by savedsearch_name
```

## Common failures

| Symptom | Likely cause | Action |
|---------|-------------|--------|
| Source shows `count=0` | Site FW rule missing, wrong index/sourcetype, UF not phoning home | Check [13](13-site-integration.md) FW rules; check DS serverclass; check `outputs.conf` |
| Indexing latency rising | Indexer CPU/disk saturated, or a single hot source | Spread load; check [sizing](01-architecture.md#sizing-math); throttle noisy source |
| Cluster "buckets fixing" | A peer restarted/returned | Normal short-term; if persistent, check peer disk/health |
| MC shows peer missing | Host down, network, or splunkd stopped | Ping/SSH; `systemctl status Splunkd`; check switch port |
| HEC 401/403 | Bad/disabled token | Reissue token; confirm `enableSSL` and port 8088 open |
| SC4S not ingesting | Container down, device not pointed at it, FW | `podman ps`; confirm device syslog target; check 514/6514 |
| Time skew | chrony lost source / drift | `chronyc sources`; re-point to kit NTP/GPS |
| Disk nearly full | Retention too long for volume | Lower `frozenTimePeriodInSecs` or roll to frozen/export; see teardown |
| License warning | Volume over daily quota / license near expiry | Check LM; reduce ingest or acknowledge; expiry is a pack-time check |

## Degraded-mode operation

### Losing an indexer

- **Clustered (M/L, RF=2/SF=2):** the cluster tolerates **one** indexer loss with no data
  loss and full searchability. The CM re-replicates buckets to restore RF/SF using the
  survivors' headroom. Actions: confirm in `cluster-status` that fix-up is progressing;
  watch disk on survivors (re-replication consumes space); bring up the cold-spare
  indexer and join it to the cluster to restore full capacity. Losing a **second** indexer
  before fix-up completes risks data loss - prioritize the spare.
- **Standalone (Small):** that indexer's data is **unavailable** until restored; new data
  routes to the surviving indexer if outputs list both. There is no automatic recovery -
  this is the accepted trade for the Small tier's size. If survivability matters, deploy
  Medium.

### Losing a search head

- **ES SHC (Large):** 3 members tolerate one loss; the captain re-elects automatically.
  Analysts keep working via another member. Rejoin or replace the member.
- **Single ES SH (S/M):** searching/ES is down until restored, but **ingestion and
  indexing continue** - no data is lost, only analyst access. Restore from the host's
  config (kept on offline media) or rebuild the SH and re-add peers.

### Losing the management node

- **LM down:** peers enter the license grace period and keep running; restore the LM
  before the grace window closes.
- **CM down (M/L):** the cluster keeps indexing and searching existing buckets, but
  bucket fix-up and new-peer joins pause until the CM returns. Restore promptly; do not
  make cluster changes while it is down.
- **DS down:** forwarders keep running with their current apps; only new app pushes pause.
- Because management roles are colocated on `mgmt` laptops (which have battery), a power
  blip rarely takes them down - but keep their configs on offline media for fast rebuild.

### Power loss

Laptops ride through on battery. Desktops (indexers, SOAR, AI) depend on the UPS for a
clean shutdown - if the UPS signals low battery, shut down indexers gracefully
(`splunk stop`) to avoid bucket corruption. On power return, hosts auto-start Splunk
(systemd boot-start) and the cluster self-heals; verify via Phase 6 validation
([08](08-runbook-deploy.md#phase-6--validation)).

## Routine maintenance

- Watch indexer disk daily; adjust retention (`frozenTimePeriodInSecs`) before volumes
  fill rather than after.
- Keep an eye on skipped searches on the ES SH; disable unused accelerated data models.
- Back up critical config (CM bundle, ES config, SOAR playbooks) to the offline media
  periodically so a host rebuild is fast.
- If connected, apply only pinned, checksummed patches per the connected-mode policy
  ([06](06-offline-repo.md#connected-mode-policy)).
