# 02 - Hardware Bill of Materials

Per-tier bill of materials for compute, storage, networking, power, and transport. All
model numbers are **references** - verify the current SKU, availability, and spec at time
of purchase. Quantities assume one spare is carried for critical single-point roles where
noted.

> Cisco Catalyst and Cisco Meraki are the reference networking
> brands and models here - but the kit is **vendor-agnostic**: every model below is "or similar," and
> an equivalent from another vendor that matches the port/speed/management posture is fine.
> Meraki (MX firewalls, cloud-managed switches) requires internet access to
> the Meraki dashboard and so is **connected-mode only** - see
> [03-network-design.md](03-network-design.md) and
> [14-remote-replication-vpn.md](14-remote-replication-vpn.md). For air-gapped missions,
> use locally-managed Catalyst switches and Cisco Secure Firewall.

## Compute reference specs

Match hosts to roles. These are minimums; more RAM/NVMe is always welcome on Search Heads and Indexers.

**Recommendation (Nov, 2026):**

Purchase the highest core/thread count CPUs possible per instance.  More CPU cores/threads reduce the need for clustering or reduce the number of cluster nodes/members.

For laptop instances:

- AMD Ryzen 8000/9000-series
- Intel Core-series

For desktop instances:

- AMD Ryzen 7000/9000-series or Ryzen Threadripper
- Intel Core Ultra-series or Xeon 600-seriesHigh-end Desktop (HEDT) CPUs are highly recommended or should be used for Medium to High volume deployment scenarios.

| Class | Role(s) | CPU | RAM | Storage | Notes |
| --- | --- | --- | --- | --- | --- |
| **Indexer desktop** | Indexer | 12+ physical cores | 32-64 GB | 2x NVMe (OS + data), size per [01 sizing](01-architecture.md#sizing-math) internal server-class SATA/SAS HDD for additional storage.  USB HDD/SSD/NVMe storage optional. | SFF/mini-tower; fast sustained write is the priority |
| **ES/search laptop/desktop** | ES SH, ad-hoc SH, Splunk MCP | 8+ cores | 64 GB | 1-2 TB NVMe | High RAM for data models / ES |
| **Management laptop** | CM/LM/DS/MC/DNS/DHCP/NTP | 8 cores | 32 GB | 1 TB NVMe | Battery = built-in UPS |
| **Ingest host** | HF, HEC, SC4S | 8 cores | 32 GB | 512 GB-1 TB NVMe | SC4S needs a container runtime |
| **SOAR desktop** | SOAR | 8+ cores | 64 GB | 1 TB+ NVMe | RHEL-family only; PostgreSQL 15 local; not containerized |
| **AI server** (optional) | AI harness + Splunk MCP | 4+ cores | 16 GB (more for a local model) | 256 GB-1 TB | Simple PC/laptop; a local-model harness may need more RAM/VRAM - see [01 AI server](01-architecture.md#ai-server-optional) |

Note:

Rugged/field notes: prefer business-class laptops with large batteries;
prefer desktops with tool-less drive bays and standard ATX/SFX power for field-replaceable
PSUs. Carry at least one cold spare indexer desktop in Medium/Large.

Backup power supply recommended for desktops.

Host-count note: running Splunk roles as containers
([07-containerization.md](07-containerization.md)) lets several roles share one physical
host, which can reduce the machine count below (especially the management laptops). The
per-role capacity (CPU/RAM/NVMe) is unchanged - containers consolidate hardware, they do
not shrink what each role needs.

## STORAGE

**Recommendation:**

###

- Use solid-state (SSD/NVMe) storage for the fastest, most reliable performance
- Hard drives (HDD) can be used for larger or longer deployments for slower, bulk storage
- Direct Attached Storage (DAS) with SAS, Thunderbolt, USB, etc. connections for bulk storage for Indexers
- Network Attached Storage (NAS) with minimum 1GbE (2.5GbE or greater recommended) network interfaces for bulk storage for Indexers

Splunk provides[ index tier storage](https://help.splunk.com/en/splunk-enterprise/administer/manage-indexers-and-indexer-clusters/10.6/manage-index-storage/configure-index-storage#d56c616c_422a_4bcc_9f6d_356f45faa863--en) per indexes based on timestamps of data, Hot, Warm, Cold buckets.

**<u>WARNING</u>:** Avoid Frozen bucket tier during active investigations, data is archived and not searchable!

Link: [Estimate indexer storage requirements](https://help.splunk.com/en/splunk-enterprise/get-started/deployment-capacity-manual/10.6/hardware-capacity-planning/estimate-your-storage-requirements)

Recommend to store at least the last 24 hours of data on SSD/NVMe.

## Networking

Bandwidth policy: **1 GbE minimum on every host, 10 GbE recommended** for indexers,
search heads, and the switch uplink/backbone. Indexer replication (port 9887) and
distributed search are the traffic that benefits most from 10 GbE.

### Switches (reference)

All models below are references - **or similar** equivalents are acceptable.

| Tier | Ports | Speed | Catalyst (or similar, local-managed) | Meraki (or similar, cloud-managed; connected only) |
|------|-------|-------|--------------------------------------|----------------------------------------------------|
| Small | 8-12 | 1 GbE min, mGig/10G preferred | Catalyst 1300 compact (e.g. C1300-12XT, 12x 10G copper) | MS130 compact (mGig/10G) |
| Medium | 16-24 | 10G | Catalyst 1300 24-port 10G (e.g. C1300-24XT) or Catalyst 9200CX | MS150 series |
| Large | 24-48 | 10G, 10/25G uplinks | Catalyst 9300X | MS390 / MS450-class |

- **Or similar.** These models are references; the kit is vendor-agnostic. Any managed
  switch meeting the port count, speed, and management posture for the tier is acceptable
  (for example, Aruba/HPE, Juniper, Netgear/MikroTik business lines).
- Catalyst 1300 / 9200 / 9300 (or similar locally-managed switches) work air-gapped via
  web UI / CLI.
- Meraki switches (or any cloud-managed switch) need the cloud dashboard - list them only
  for connected missions.
- Carry a spare small unmanaged 5-8 port 1G switch as a field backup.

### Firewall (reference)

Optional in Small, recommended in Medium/Large.

| Option | Model reference | Management | Air-gap? | Site-to-site VPN |
|--------|-----------------|------------|----------|------------------|
| **Cisco Secure Firewall (default)** | 1200 series, e.g. 1210, or similar | Local (FDM) or optional cloud (CDO) | Yes (FDM local) | IPsec |
| Cisco Meraki MX, or similar | MX67 / MX75 / MX85 | Cloud dashboard (required) | No | Auto VPN + IPsec |

**Or similar.** A locally-managed firewall from another vendor (for example,
pfSense/OPNsense, Fortinet, Palo Alto) is acceptable as long as it supports local
management for air-gap and standards-based site-to-site IPsec. The deciding factor is
local vs. cloud management, not the brand.

Secure Firewall 1200 with local FDM is the default because it works disconnected. Use
Meraki MX only when the mission is connected and the remote end is also Meraki (Auto VPN
simplifies site-to-site).

### Cabling and transit networking

- Cat6a patch cables (10G-rated), one per host plus 25% spares, pre-labeled to the IP
  plan hostnames.
- SFP+/SFP28 transceivers and DAC/fiber as required by the chosen switch uplinks.
- A small labeled cable pouch; color-code by VLAN (see [03](03-network-design.md)).
- Optional: cellular/5G modem or BGAN/satellite terminal for connected mode at remote
  sites (feeds the firewall WAN).

## Power

- **UPS:** at least one UPS sized to the desktop + switch + firewall load for clean
shutdown (laptops self-protect on battery). ~1000-1500 VA covers Small/Medium desktop
counts; size Large to measured load.
- **PDU:** a rugged surge-protected PDU with enough outlets for all desktops, switch,
firewall, UPS, and chargers.
- **Input:** carry IEC cables plus region adapters; note voltage range (auto-ranging
PSUs preferred for 100-240 V missions).
- A generator or large-capacity portable power station is a mission-dependent add-on for
truly off-grid tactical use.

### Power budget (planning estimate)

Steady-state draw; measure actual on the bench before the mission.

| Item | Watts each | Small qty | Medium qty | Large qty |
|------|-----------|-----------|------------|-----------|
| Indexer desktop | ~150-250 | 1-2 | 3 | 4-6 |
| Laptop (SH/mgmt/ingest) | ~60-90 | 3 | 5 | 7 |
| SOAR desktop | ~150-250 | 0-1 | 1 | 1 |
| AI server (optional) | ~60-90 (PC/laptop); more if running a local GPU model | 0-1 | 0-1 | 0-1 |
| Switch | ~30-150 | 1 | 1 | 1 |
| Firewall | ~30-60 | 0-1 | 1 | 1 |
| **Approx total** | | **~0.6-1.0 kW** | **~1.5-2.2 kW** | **~2.5-4+ kW** |

The AI server is optional and, as a plain PC/laptop running a harness, draws about as much
as the other laptops. If a *local* GPU model is chosen instead, re-cost its power when the
spec is set ([01 AI server](01-architecture.md#ai-server-optional)).

## Transport / cases

- Rugged wheeled cases with custom foam (Pelican-class) sized per tier; keep total
  per-case weight within what one person can lift (~20-23 kg) and within airline/freight
  limits.
- Separate the "compute" case(s) from the "network + power" case for faster setup.
- Include: labeled cables, spare drives, the offline media
  ([06-offline-repo.md](06-offline-repo.md)) on at least two separate drives, printed
  runbooks and checklists ([appendix-checklists.md](appendix-checklists.md)), and a small
  tool/label kit.

### Weight budget (planning estimate)

| Tier | Compute | Network + power | Approx total |
|------|---------|-----------------|--------------|
| Small | ~15-20 kg | ~10-15 kg | ~25-35 kg (2 cases) |
| Medium | ~30-40 kg | ~15-20 kg | ~45-60 kg (3-4 cases) |
| Large | ~55-75 kg | ~20-30 kg | ~75-105 kg (5+ cases) |

## Per-tier BOM summary

### Small
- 1-2 indexer desktops, 1 management laptop, 1 ES/search laptop, 1 ingest host
- Optional 1 SOAR desktop
- 1 compact switch (Catalyst 1300 compact or similar), optional firewall (Secure Firewall 1210 or similar)
- 1 UPS, 1 PDU, cabling, 2 cases

### Medium
- 3 indexer desktops (+1 cold spare recommended), 2 management laptops, 1 ES laptop,
  optional 1 ad-hoc SH laptop, 1 HF host, 1 SC4S host, 1 SOAR desktop, 1 AI desktop
- 1 24-port 10G switch (Catalyst 1300-24XT / 9200CX or similar)
- 1 Secure Firewall 1210 (or similar)
- 1-2 UPS, 1-2 PDU, cabling, 3-4 cases

### Large
- 4-6 indexer desktops (+1-2 cold spares), 2 management hosts, 3 ES SH cluster laptops,
  2 HF hosts, 1 SC4S host, 1 SOAR desktop, 1 AI desktop
- 1 24-48 port 10G switch (Catalyst 9300X or similar)
- 1 Secure Firewall 1210 or similar (or larger if WAN throughput demands)
- 2+ UPS, 2 PDU, cabling, 5+ cases

> Automation hook (out of scope here): a packing/manifest generator could emit this BOM
> and the serial/asset list from the kit manifest. Tracked in
> [06-offline-repo.md](06-offline-repo.md) manifest.
