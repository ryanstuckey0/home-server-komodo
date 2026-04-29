# home-server-komodo

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

| App            | Location | DNS                | Notes                                                                                                                                                                                                                                                    |
| -------------- | -------- | ------------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Komodo         | RP5      | km.beansrn.com     |                                                                                                                                                                                                                                                          |
| Komodo         | NAS      | ---                | Periphery only, managed by Komodo on Raspberry Pi                                                                                                                                                                                                        |
| Traefik        | NAS      | tf.nas.beansrn.com | Some apps are installed in NAS, some in Raspberry Pi. Network critical apps, like AdGuard, are installed in Raspberry Pi. As the NAS has 2X 5GbE Ethernet ports, I didn't want traffic speeds to be limited by going through just Traefik running on RP5 |
| Traefik        | RP5      | tf.rp5.beansrn.com |                                                                                                                                                                                                                                                          |
| TrueNAS Scale  | NAS      | nas.beansrn.com    |                                                                                                                                                                                                                                                          |
| Jellyfin       | NAS      | jf.beansrn.com     |                                                                                                                                                                                                                                                          |
| NextCloud      | NAS      | nc.beansrn.com     |                                                                                                                                                                                                                                                          |
| Home Assistant | RP5      | ha.beansrn.com     |                                                                                                                                                                                                                                                          |
| VaultWarden    | RP5      | vw.beasnrn.com     | VaultWarden will run on RP5, but backup its database regularly to NAS                                                                                                                                                                                    |
|                |          |                    |                                                                                                                                                                                                                                                          |

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