# home-server-komodo

## Helpful Docs

- [Home Assistant in Docker Compose](https://www.home-assistant.io/installation/alternative/#docker-compose)

## Hardware

- Raspberry Pi 5 (RP5)
  - M.2 Hat containing 1TB SSD
- TerraMaster F4-425 Plus (NAS)
  - 2X Seagate IronWolf 8TB NAS HDDs
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
| Caddy          | RP5      | ---                    | Reverse proxy for RP5 services. Wildcard `*.beansrn.com` cert via Cloudflare DNS-01                                                                                                                                                                  |
| Caddy          | NAS      | ---                    | Reverse proxy for NAS services. Runs on the NAS so its 2X 2.5GbE ports aren't bottlenecked by proxying through RP5. Network-critical apps (e.g. AdGuard) stay on RP5                                                                                 |
| AdGuard Home   | RP5      | ag.beansrn.com         | DNS + ad blocking, runs in host network mode                                                                                                                                                                                                        |
| Home Assistant | RP5      | ha.beansrn.com         |                                                                                                                                                                                                                                                     |
| Vaultwarden    | RP5      | vw.beansrn.com         | Runs on RP5; database backed up regularly to NAS                                                                                                                                                                                                     |
| Homepage       | RP5      | home.beansrn.com       | Dashboard                                                                                                                                                                                                                                            |
| Syncthing      | RP5      | st.beansrn.com         | Always-on hub node for file sync (Satisfactory blueprints)                                                                                                                                                                                           |
| TrueNAS Scale  | NAS      | nas.beansrn.com        |                                                                                                                                                                                                                                                     |
| Jellyfin       | NAS      | jf.beansrn.com         |                                                                                                                                                                                                                                                     |
| Glances        | NAS      | glances-nas.beansrn.com | System monitoring                                                                                                                                                                                                                                    |
| Syncthing      | NAS      | st-nas.beansrn.com     | Redundant/backup sync node (only up when NAS is powered on)                                                                                                                                                                                          |

### Reverse Proxy

Both hosts use [Caddy](https://caddyserver.com/) as their reverse proxy (replacing the
earlier Traefik setup, whose config is retained under `stacks/*/infra/traefik/` for
reference but is no longer deployed). Each host runs its own Caddy instance serving a
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
  --onboarding-key="O_O2R1dwB3TaCH0L3u3Ko97hYx0Nni_O"
```
```bash
curl -sSL https://raw.githubusercontent.com/moghtech/komodo/main/scripts/setup-periphery.py \
  | python3 - --user \
  --core-address="http://192.168.1.2:9120" \
  --connect-as="$(hostname)" \
  --onboarding-key="O_O2R1dwB3TaCH0L3u3Ko97hYx0Nni_O"
```