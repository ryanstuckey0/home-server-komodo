# home-server-komodo

## Helpful Docs

- [Home Assistant in Docker Compose](https://www.home-assistant.io/installation/alternative/#docker-compose)
- [Dashboard Icons](https://dashboardicons.com)

## Hardware

- Raspberry Pi 5 (RP5)
  - M.2 Hat containing 1TB SSD
- TerraMaster F4-425 Plus (NAS)
  - 2X Seagate IronWolf 8TB NAS HDDs (`pool01`)
  - M.2 NVMe SSD (`pool02`) — app working data
- Ubiquiti UniFi Dream Router 7
  - Ubiquiti Flex Mini 2.5G 5-Port Switch

## Software

### Operating Systems

- TrueNAS Scale
- Raspberry Pi OS

### Services

| App            | Location | DNS                    | Notes                                                                                                                                                                                                                                             |
| -------------- | -------- | ---------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Komodo         | RP5      | km.beansrn.com         | Komodo Core — manages both hosts                                                                                                                                                                                                                     |
| Komodo         | NAS      | ---                    | Periphery only, managed by Komodo on the Raspberry Pi                                                                                                                                                                                                |
| Caddy          | RP5      | ---                    | Reverse proxy for RP5 services. Wildcard `*.beansrn.com` + `*.rp5.beansrn.com` certs via Cloudflare DNS-01                                                                                                                                            |
| Caddy          | NAS      | ---                    | Reverse proxy for NAS services. Wildcard `*.beansrn.com` + `*.nas.beansrn.com` certs via Cloudflare DNS-01. Runs on the NAS so its 2X 2.5GbE ports aren't bottlenecked by proxying through RP5. Network-critical apps (e.g. AdGuard) stay on RP5      |
| AdGuard Home   | RP5      | ag.beansrn.com         | DNS + ad blocking, runs in host network mode                                                                                                                                                                                                        |
| Home Assistant | RP5      | ha.beansrn.com         |                                                                                                                                                                                                                                                     |
| Vaultwarden    | RP5      | vw.beansrn.com         | Runs on RP5; database backed up regularly to NAS                                                                                                                                                                                                     |
| Homepage       | RP5      | home.beansrn.com       | Dashboard                                                                                                                                                                                                                                            |
| Syncthing      | RP5      | st.rp5.beansrn.com     | Always-on hub node for file sync (Satisfactory blueprints)                                                                                                                                                                                           |
| Cup            | RP5      | cup.beansrn.com        | Docker image update checker (server); aggregates the NAS agent for a combined view                                                                                                                                                                   |
| TrueNAS Scale  | NAS      | nas.beansrn.com        |                                                                                                                                                                                                                                                     |
| Jellyfin       | NAS      | jf.beansrn.com         |                                                                                                                                                                                                                                                     |
| Glances        | NAS      | glances.nas.beansrn.com | System monitoring                                                                                                                                                                                                                                    |
| Cup            | NAS      | cup.nas.beansrn.com    | Cup agent; reports NAS container image updates to the RP5 instance                                                                                                                                                                                   |
| Syncthing      | NAS      | st.nas.beansrn.com     | Redundant/backup sync node (only up when NAS is powered on)                                                                                                                                                                                          |

### Reverse Proxy

Both hosts use [Caddy](https://caddyserver.com/) as their reverse proxy. Each host runs its own Caddy instance serving a
wildcard `*.beansrn.com` certificate obtained via the Cloudflare DNS-01 challenge
(requires `CF_DNS_API_TOKEN`).

- RP5 routes: `stacks/rp5/infra/caddy/Caddyfile`
- NAS routes: `stacks/truenas/infra/caddy/Caddyfile`

To add a new service, add a host matcher block inside the `*.beansrn.com` block of the
relevant Caddyfile. Apps that need the client's real IP/protocol (auth-sensitive apps
like AdGuard) also need `header_up X-Forwarded-Proto https` and
`header_up X-Real-IP {remote_host}` in their `reverse_proxy` block.

### Komodo

- Komodo Periphery on NAS
```bash
curl -sSL https://raw.githubusercontent.com/moghtech/komodo/main/scripts/setup-periphery.py > setup-periphery.py
sudo python3 setup-periphery.py \
  --core-address="http://192.168.1.2:9120" \
  --connect-as="$(hostname)" \
  --onboarding-key="ONBOARDING KEY HERE"
```
```bash
curl -sSL https://raw.githubusercontent.com/moghtech/komodo/main/scripts/setup-periphery.py \
  | python3 - --user \
  --core-address="http://192.168.1.2:9120" \
  --connect-as="$(hostname)" \
  --onboarding-key="ONBOARDING KEY HERE"
```

### Storage (TrueNAS)

Two ZFS pools on the NAS:

- **`pool01`** — 2X 8TB IronWolf HDD. Bulk data (Jellyfin media library, per-user data) and, going forward, the destination for app backups.
- **`pool02`** — NVMe SSD. App working data (config + cache), so services run off the faster disk.

App data is namespaced under `pool02/apps`, split into **`appdata/`** (persistent config — the
backup source of record) and **`temp/`** (disposable scratch — never backed up). ZFS tuning is
applied only on the specific dataset that needs it; parents stay at defaults so new siblings don't
inherit special settings.

```
/mnt/pool02/apps/
  appdata/                     persistent config  (back this up)
    jellyfin/config            recordsize=16K
    caddy/                     (plain dir, inherits)
    syncthing/                 (plain dir, inherits)
  temp/                        disposable scratch (never backed up)
    jellyfin/cache/            (plain dir, inherits)
    jellyfin/transcodes/       recordsize=1M, compression=off, sync=disabled
```

Baseline properties are set once on `pool02/apps` (`atime=off`, `compression=lz4`, `sync=standard`,
default 128K recordsize) and inherited down. Only two datasets override the baseline:

| Dataset                                  | recordsize | compression | sync     | Why                                              |
| ---------------------------------------- | ---------- | ----------- | -------- | ------------------------------------------------ |
| `pool02/apps`                            | 128K       | lz4         | standard | Baseline inherited by everything below           |
| `pool02/apps/appdata/jellyfin/config`    | **16K**    | lz4         | standard | Jellyfin SQLite DBs — small random writes        |
| `pool02/apps/temp/jellyfin/transcodes`   | **1M**     | **off**     | **disabled** | Large, transient, already-compressed segments |

`caddy`, `syncthing`, and `temp/jellyfin/cache` are plain directories inheriting their parent
(128K/lz4). `atime=off` applies everywhere via the baseline.

**Datasets vs directories:** a dataset is created only where it needs distinct tuning (or is a
snapshot/backup boundary). Structural parents (`apps`, `appdata`, `temp`, `jellyfin`) are datasets
so ZFS can nest the tuned leaves, and so `pool02/apps/appdata` can be snapshotted/replicated as a
single backup root.

#### Per-user data (`pool01`)

User data is organized by user and kept separate from the SSH login home:

```
/mnt/pool01/
  home/<user>/       SSH login home (.ssh, dotfiles) — do NOT mix bulk data here
  users/<user>/      per-user data root (dataset)
    games/saves/     Syncthing-synced game saves (e.g. Satisfactory blueprints)
    games/roms/      console ROM backups
    docs/            Nextcloud data (planned)
  media/             Jellyfin library
```

- `pool01/users` and `pool01/users/<user>` are datasets (the per-user snapshot / backup / quota
  boundary); `games`, `saves`, `roms`, `docs` are plain dirs inheriting them.
- `saves/satisfactory/blueprints` is owned by the `syncthing` service account (uid 3003). Its
  wrapper dir is given group-traverse for that account (`chgrp 3003 satisfactory && chmod 710`) so
  the single Syncthing daemon can reach it without opening it to every user.

#### Backups (planned)

App config on the `pool02` SSD is to be backed up to the `pool01` HDD (and offsite) via
restic/backrest, using a pre-backup hook that stops each app so its database is copied cold /
quiesced. `pool02/apps/temp` is intentionally excluded (regenerable scratch).
