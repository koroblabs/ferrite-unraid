# Ferrite Unraid Templates

Unraid Community Applications template repository for **Ferrite** — a
self-hosted DNS filter and router for the home network. Ferrite blocks ads,
trackers and malware for every device, applies per-device profiles (blocked
services such as TikTok or Roblox, schedules, SafeSearch, default-deny), and
sends chosen domains, services or devices through **WireGuard, SOCKS5 (Tor) or
a DPI-evasion path**. It resolves through encrypted upstreams (DoH/DoT/DoQ),
shows what each device is doing, and notifies you when a tunnel goes down or a
new device joins.

One Rust binary. WireGuard runs kernel-backed for line-rate throughput (the
template grants `NET_ADMIN`, and Unraid 6.9+ already ships the module) and
falls back to an in-process userspace tunnel — no root, no TUN device —
wherever that is unavailable.

The template pulls the container image from the GitHub Container Registry:

```text
ghcr.io/koroblabs/ferrite-server:latest
```

## Install via Community Applications

Once the template is approved in Community Applications:

1. Open the **Apps** tab in Unraid and search for **Ferrite**.
2. Click **Install**. The template pre-fills ports, the appdata path, and the
   advanced variables described below.
3. Pick a network. A dedicated IP on a custom Docker network / `br0` is
   recommended: port 80 on the Unraid host is usually taken by the Unraid web
   UI, and the selective-routing (tunnels) feature needs Ferrite reachable on
   real ports 80/443 (see [Networking notes](#networking-notes)). Plain DNS
   blocking works fine in bridge mode with a remapped Web UI port (e.g. `8088`).
4. Click **Apply**. On first start the container downloads the Ferrite server
   binary and web UI from GitHub releases (verified by SHA-256) into appdata,
   then starts serving DNS on port 53 and the web UI on the mapped port.
5. Open the **WebUI** link from the Docker tab to reach the dashboard.

### Manual install (before CA approval)

Copy `ferrite/ferrite.xml` to a user-template file on the Unraid flash drive:

```text
/boot/config/plugins/dockerMan/templates-user/my-Ferrite.xml
```

Then open **Docker → Add Container** and select the **Ferrite** user template.

## Configuration reference

| Setting | Container target | Default | Notes |
| --- | --- | --- | --- |
| Appdata | `/var/lib/ferrite` | `/mnt/user/appdata/ferrite` | All persistent state (config, DB, caches, snapshots, web assets). |
| DNS TCP | `53/tcp` | `53` | DNS listener (TCP). |
| DNS UDP | `53/udp` | `53` | DNS listener (UDP). |
| Web UI | `80/tcp` | `80` | API + web UI. Also intercepts HTTP for routed domains (demultiplexed by `Host`). Remap to a free host port if 80 is taken. |
| HTTPS Routing | `443/tcp` | `443` | HTTPS (SNI) intercept for the selective-routing/tunnels feature. Only bound while routing is enabled; leave unmapped if you don't use tunnels. Must stay `443:443` for routing to work — use a dedicated IP if the host's 443 is taken. |
| Server Release Version | `FERRITE_SERVER_VERSION` | `latest` | `latest` auto-updates on restart; pin (e.g. `0.1.1`) to freeze. |
| Web Release Version | `FERRITE_WEB_VERSION` | _(blank)_ | Follows the server version when blank; pin separately if needed. |
| Panel IP | `FERRITE_PANEL_IP` | _(blank)_ | LAN IP for the built-in `fe.te` A record. Set to the Unraid host IP in bridge mode. |
| Panel Domain | `FERRITE_PANEL_DOMAIN` | _(blank)_ | Override the built-in panel shortcut domain (default `fe.te`). |
| Panel URL | `FERRITE_PANEL_URL` | _(blank)_ | Startup-log shortcut URL. Set to `http://fe.te:<port>` when Web UI is not on host port 80. |
| GitHub Release Token | `FERRITE_RELEASE_TOKEN` | _(blank)_ | Optional. Only needed if you hit GitHub's anonymous API rate limit (60/hour per IP) or pull from a private mirror. Masked. |

### Health status

The container ships a Docker `HEALTHCHECK` that probes the public
`GET /api/auth` endpoint. Unraid surfaces this as the container health indicator
on the Docker tab — green once the API is serving, with a generous start period
so the first-start release download does not flap the status.

## Persistence

The template mounts one appdata directory:

```text
/mnt/user/appdata/ferrite -> /var/lib/ferrite
```

Ferrite stores all mutable state below that appdata path:

- Config: `/var/lib/ferrite/.config/ferrite/config.toml`
- SQLite database: `/var/lib/ferrite/ferrite.db`
- Blocklist cache and snapshots: `/var/lib/ferrite/.local/share/ferrite`
- Web UI assets: `/var/lib/ferrite/web`

Updating the Docker image should not remove settings, database history, caches,
or web assets.

The Community Applications icon is exported from the Ferrite web UI mark as
`icons/ferrite.png` so the Unraid listing matches the dashboard branding.

## Networking notes

Ferrite listens on container port 80 for the API/web UI. A custom Docker network
or fixed IP is recommended on Unraid because the Unraid web UI often already
uses host port 80.

When using bridge networking, set the advanced **Panel IP** variable to the
Unraid host/LAN IP if you want Ferrite's built-in `fe.te` DNS shortcut to
resolve outside the container. If the Web UI host port is not 80, open
`http://fe.te:<web-ui-host-port>` or set the advanced **Panel URL** variable to
the same URL so startup logs show the reachable address.

### Selective routing / tunnels

For domains matched by a routing rule, Ferrite answers DNS with **its own IP**;
the client then connects to Ferrite on ports **80/443**, where Ferrite peeks the
SNI/Host (without terminating TLS) and splices the connection through the chosen
egress (WireGuard, SOCKS5/Tor, DPI evasion, or direct). This redirect model
means clients must be able to reach the container on the **real** ports 80 and
443 at the advertised IP:

- **Dedicated IP (`br0` / custom network): recommended.** The container owns
  its ports, nothing conflicts with the Unraid web UI, and routing works out of
  the box.
- **Bridge mode:** works for DNS blocking, but routed domains would point
  clients at remapped ports — which they never use. If you must stay on bridge,
  keep `80:80` and `443:443` mappings intact and set `[proxy] advertise_ipv4`
  in `config.toml` to the Unraid host IP; otherwise leave the tunnels feature
  off.

Routing rules, egresses, and listener settings hot-reload from the web UI — no
container restart needed.

### Client hostnames

Ferrite identifies LAN devices by MAC where possible (ARP/NDP, mDNS, EUI-64),
so device names and per-device history survive IP changes. In bridge networking
the container cannot see the LAN neighbor table, so hostnames fall back to
reverse DNS: point Ferrite's reverse-DNS zone at the router. For a
`192.168.1.0/24` LAN with router `192.168.1.1`, patch settings with:

```bash
curl -sS -X PATCH http://<unraid-ip>:<web-ui-port>/api/settings \
  -H 'Content-Type: application/json' \
  -d '{"zones":[{"name":"1.168.192.in-addr.arpa","upstream":"192.168.1.1:53"}]}'
```

A dedicated `br0` IP also gives Ferrite direct LAN visibility for device
identification.

## Screenshots

Listing screenshots are captured from the running web UI and live in the
[web UI repo](https://github.com/koroblabs/ferrite-web/tree/main/screenshots),
referenced from the Community Applications listing:

| File | View |
| --- | --- |
| `dashboard.png` | Dashboard with query/blocked totals and timeseries. |
| `queries.png` | Live query log with device/status filters. |
| `tunnels.png` | Tunnels — egresses, routing rules, health. |
