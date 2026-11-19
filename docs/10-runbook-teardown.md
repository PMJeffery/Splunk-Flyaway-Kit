# 10 - Runbook: Teardown

End-of-mission procedure: get the data out, hand off evidence, securely wipe, repack, and
write the post-mission report. Teardown is where data is most at risk of being lost or
leaked - work it as deliberately as deployment.

## Order of operations

1. Stop ingest.
2. Export / preserve required data.
3. Hand off (evidence / deliverables).
4. Secure wipe.
5. Repack and inventory.
6. Post-mission report.

Do not skip to wipe before export and a verified handoff. Confirm with the mission owner
that the handoff is complete and acknowledged before any destructive step.

## Phase 1 - Stop ingest

1. Disconnect the site VLAN first so no new org data enters while you finalize.
2. Stop site-facing inputs (HF/HEC/SC4S) and remote replication
   ([14](14-remote-replication-vpn.md)).
3. Let in-flight data drain to the indexers; confirm indexing latency returns to zero in
   the MC.

## Phase 2 - Export / preserve

Pick the method the mission requires:

- **Raw export (SPL):** export result sets or raw events for specific indexes/time ranges:
  ```spl
  index=mission_* earliest=0 latest=now | ...        # dump to CSV/JSON via UI export or
  ```
  or use `| outputcsv` / the REST export endpoint for scripted dumps.
- **Bucket-level preservation (full fidelity):** copy the index buckets
  (`$SPLUNK_DB/<index>/db|colddb`) for the mission indexes to the evidence drive. This
  preserves raw + tsidx so the data can be re-ingested later for full search fidelity.
- **Frozen archive:** roll relevant buckets to frozen and copy the frozen archive.
- **Remote mirror (if used):** confirm the remote DC/cloud copy is complete and verified
  ([14](14-remote-replication-vpn.md)) before wiping the local copy.

Checksum everything you export (`sha256sum`) and record the hashes in the report.

## Phase 3 - Handoff (evidence / deliverables)

For IR missions, chain of custody is mandatory (see
[11-security.md](11-security.md#data-handling-and-chain-of-custody)):

- Record each exported artifact: index, time range, sha256, size, who exported it, when,
  and the source kit host.
- Transfer on an **encrypted** evidence drive; record who received it, when, and their
  acknowledgement.
- Keep the chain-of-custody form with the artifact; a copy goes in the mission report.
- Do not wipe the kit until the recipient confirms the handoff is readable and complete.

## Phase 4 - Secure wipe

After a verified handoff, sanitize per the mission's data classification
([11](11-security.md#data-classification)).

- **Stop Splunk** on all hosts (`splunk stop`), then **stop SOAR**.
- **Crypto-erase / secure-erase the data NVMe** (indexer data volumes, SOAR
  `/opt/phantom`, SOAR PostgreSQL, SC4S spool, any export staging). If full-disk
  encryption was used ([11](11-security.md)), destroying the key (crypto-erase) plus an
  `nvme format`/`blkdiscard` is the fast, reliable path for NVMe; follow the mission's
  sanitization standard (e.g. NIST 800-88 purge) for the required method and verification.
- **Wipe network gear config** - reset the firewall and switch to a known clean baseline
  so no site topology, VPN keys, or rules leak to the next mission.
- **Rotate secrets** that were in use (cluster `pass4SymmKey`, HEC tokens, VPN keys,
  admin passwords) so a captured artifact can't unlock a future deployment
  ([11](11-security.md)).
- Record the wipe method and verification per device in the report.

> The OS disk layout keeps data on a separate volume
> ([04](04-os-baseline.md#disk-layout)) specifically so this wipe targets the data volume
> cleanly. Re-image OS volumes at base, not in the field, unless the classification
> requires in-field OS wipe too.

## Phase 5 - Repack and inventory

1. Power down in reverse order (hosts, then firewall, then switch, then UPS).
2. Inventory every item against the BOM ([02](02-hardware-bom.md)); note anything damaged,
   failed, or consumed (spares used).
3. Re-verify the **offline media** still passes `sha256sum -c` for the next mission, or
   flag it for refresh at base.
4. Repack to the foam layout; cables coiled and labeled.

## Phase 6 - Post-mission report

Capture while fresh:

- Tier deployed, actual deploy time vs target, headcount.
- Data sources onboarded and volumes seen (from the MC).
- Failures/degraded events and how they were handled ([09](09-runbook-operations.md)).
- Exports produced: artifacts, checksums, handoff acknowledgements.
- Wipe/sanitization record per device.
- Hardware issues, spares used, and resupply needed.
- Lessons learned / changes to feed back into these docs and the kit manifest.

A printable teardown checklist is in
[appendix-checklists.md](appendix-checklists.md#teardown).
