# 04 - OS Baseline

The Linux build standard every kit host is imaged to before Splunk is installed. A single
consistent baseline makes the deployment runbook deterministic and the configs portable.

## OS choice

| Role | Primary OS | Alternative | Why |
|------|-----------|-------------|-----|
| Splunk Enterprise (indexer, SH, mgmt, HF) | Rocky Linux 9 / RHEL 9 | Ubuntu 24.04 LTS | Both are supported by Splunk Enterprise 10.6 |
| SOAR | Rocky Linux 9.7+ / RHEL 9.7+ (or Oracle Linux 9) | - | **SOAR does not support Ubuntu**; see below |
| SC4S ingest host | Rocky/RHEL 9 or Ubuntu 24.04 | - | Needs a container runtime (Podman/Docker) |
| Containerized Splunk roles (optional) | Rocky/RHEL 9 or Ubuntu 24.04 | - | Host runs Podman/Docker; see [07-containerization.md](07-containerization.md). SOAR is the exception - it cannot be containerized |

Standardizing on **Rocky Linux 9 / RHEL 9 across the whole kit** is the recommended path:
it is the one OS that satisfies every role (including SOAR), which keeps the offline repo,
the hardening, and the runbooks single-track. Ubuntu is listed only as a fallback for
non-SOAR hosts if RHEL-family media is unavailable.

### SOAR OS constraints (verified against Splunk docs)

Splunk SOAR (On-premises) 8.7 supported operating systems:

- Red Hat Enterprise Linux 8 (8.10+) or 9 (9.7+)
- Oracle Linux 8 (8.10+) or 9 (9.7+)
- Amazon Linux 2023 (2023.10.x+)

Hard constraints:

- **Not Ubuntu / Debian.** Do not plan SOAR on Ubuntu.
- **Cannot be installed inside a Docker or Podman container.** SOAR runs on the host.
- Bundles **PostgreSQL 15**, Elasticsearch, and (for clustered SOAR) **GlusterFS** file
  shares. The installer expects **umask 0022**.
- Installs as the unprivileged `phantom` user into `/opt/phantom` (`<$PHANTOM_HOME>`).
- Uses Splunk Universal Forwarder 10.4.1 internally.
- Can run FIPS-compliant only if the OS kernel is in FIPS mode and it is a fresh
  unprivileged install.
- If `fapolicyd` is enabled on RHEL, stop it for install, then allow-list SOAR and
  restart it.

## Disk layout

NVMe, GPT, UEFI. Separate OS from Splunk data so a full data volume never takes down the
OS, and so teardown wipe targets the data volume directly.

| Mount | Size | Notes |
|-------|------|-------|
| `/boot/efi` | 1 GB | ESP |
| `/boot` | 1 GB | |
| `/` | 50-80 GB | OS |
| `/var` | 20 GB | separate so logs can't fill `/` |
| `/opt/splunk` | 50-100 GB | Splunk install + `var/log/splunk` (indexers: hot/warm goes on the data volume below) |
| `/opt/splunk/var/lib/splunk` (indexers) | remainder of data NVMe | hot/warm index storage; size per [01 sizing](01-architecture.md#sizing-math) |
| swap | per RAM / disable | Splunk prefers little/no swapping |

SOAR hosts: give `/opt/phantom` the large volume; keep `/home` and `/tmp` writable as the
docs require.

## OS tuning for Splunk

Apply on every Splunk host. These are the long-standing Splunk prerequisites.

- **Transparent Huge Pages (THP): disabled.** THP hurts Splunk performance. Disable via
  kernel cmdline or a `tuned`/systemd unit and verify at boot.
- **ulimits** for the `splunk` user (`/etc/security/limits.d/`):
  - `nofile` (open files) >= 64000
  - `nproc` (user processes) >= 16000
  - `fsize` (max file size) unlimited
  - `data`/`memory` unlimited
- **Time sync:** chrony enabled, pointed at the kit NTP source
  ([03-network-design.md](03-network-design.md)). No host free-runs independently.
- **Disable unnecessary services;** keep the box minimal.
- **I/O scheduler:** `none`/`mq-deadline` for NVMe.

## Accounts and service management

- Dedicated unprivileged **`splunk`** OS user and group; Splunk installed to
  `/opt/splunk` owned by `splunk`.
- Splunk runs as a **systemd** service (`systemctl enable --now Splunkd`) using
  `enable boot-start -user splunk -systemd-managed 1`, so hosts come back cleanly after
  a power event - important in the field.
- SOAR runs as the **`phantom`** user per its installer; managed by its own units.
- No shared logins; per-operator accounts; sudo logged (see [11-security.md](11-security.md)).

## Firewall and SELinux

- **firewalld** enabled on every host, default-deny, opening only that host's required
  ports from the matrix in [03-network-design.md](03-network-design.md). Example for an
  indexer: allow 8089, 9997, 9887 from the data VLAN only.
- **SELinux: enforcing.** Splunk Enterprise runs under enforcing SELinux. If a specific
  input or path needs a policy exception, document the exact `semanage`/`setsebool` rule
  in the host's build notes rather than disabling SELinux. (SOAR documents its own
  SELinux/`fapolicyd` handling above.)

## Build output

Each imaged host should end at a known state before Splunk install:

1. OS installed to the disk layout above, UEFI, hostname + static IP per the plan.
2. THP off, ulimits set, chrony synced, I/O scheduler set.
3. `splunk` (and `phantom` on SOAR) user created; `/opt/splunk` volume mounted.
4. firewalld + SELinux enforcing with the host's port set open.
5. Registered in kit DNS; reachable by name from `mgmt`.

> Automation hook (out of scope): this baseline is the natural target for a kickstart /
> autoinstall file plus an Ansible hardening role. The doc is written so each step maps
> 1:1 to a future task.

A quick post-build self-check is in
[appendix-checklists.md](appendix-checklists.md#os-baseline-self-check-per-host).
