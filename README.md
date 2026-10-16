# NAS foundation stack

A single Docker Compose project for a small home server:

- Pi-hole v6 for network DNS filtering and its embedded dnsmasq-compatible DNS service
- Jellyfin for media playback
- Caddy for memorable local HTTPS addresses
- Portainer CE for container management

The base deployment works without a GPU. Optional Compose overlays enable Intel
Quick Sync/VA-API or NVIDIA NVENC/NVDEC for Jellyfin.

## Before deploying

You need:

- A Linux host with Docker Engine and Docker Compose v2
- A static LAN address for the host
- TCP and UDP port 53 available on the host
- TCP ports 80 and 443 available on the host
- An existing media directory readable by the chosen `PUID` and `PGID`

Do not deploy this over an existing Pi-hole, Jellyfin, Caddy, or Portainer
installation without first backing up its persistent data. This project uses
bind-mounted data below `./data` so its state can be backed up directly.

## Configure

```sh
cp .env.example .env
```

Edit `.env` and set at least:

- `SERVER_IP` to the server's static LAN address
- `MEDIA_PATH` to the media directory on the host
- `PIHOLE_PASSWORD` to a unique password
- `PUID` and `PGID` to the account that can read the media directory

Create the persistent directories and give Jellyfin ownership of its writable
directories. Substitute the values selected for `PUID` and `PGID` if they are
not `1000`:

```sh
mkdir -p data/{caddy/config,caddy/data,jellyfin/cache,jellyfin/config,pihole,portainer}
sudo chown -R 1000:1000 data/jellyfin
```

Generate a password without placing it in shell history:

```sh
openssl rand -base64 32
```

The `.env` file is ignored by Git. Do not commit it.

## Start the stack

CPU-only Jellyfin:

```sh
docker compose config --quiet
docker compose up -d
```

Intel GPU:

```sh
docker compose -f compose.yaml -f compose.intel.yaml config --quiet
docker compose -f compose.yaml -f compose.intel.yaml up -d
```

NVIDIA GPU:

```sh
docker compose -f compose.yaml -f compose.nvidia.yaml config --quiet
docker compose -f compose.yaml -f compose.nvidia.yaml up -d
```

The NVIDIA option requires a working NVIDIA driver and NVIDIA Container
Toolkit on the host. Confirm `nvidia-smi` works before starting the stack.

## Put Pi-hole into service

The Compose project does not change your router automatically. First verify the
container directly:

```sh
docker compose ps
dig @SERVER_IP pi.hole
dig @SERVER_IP watch.home.arpa
```

Replace `SERVER_IP` with the address from `.env`. After those tests succeed,
configure the router's LAN DHCP/DNS setting to advertise that address as its
DNS server.

Do not point the Docker host itself exclusively at the new Pi-hole until the
container has proven that it starts reliably after a reboot. Otherwise a
container outage can prevent the host from resolving image registries.

Pi-hole v6 embeds the DNS service previously configured through standalone
dnsmasq. Local service records are supplied through
`FTLCONF_misc_dnsmasq_lines`. Extra dnsmasq-compatible files can be added under
`dnsmasq.d/`.

DHCP service is deliberately omitted. Running DHCP from a bridged container
requires additional network and capability decisions and can conflict with the
router's existing DHCP server.

## Open the services

After Pi-hole is the LAN DNS server:

- Jellyfin: `https://watch.home.arpa`
- Pi-hole: `https://dns.home.arpa/admin/`
- Portainer: `https://portainer.home.arpa`

Caddy uses its internal certificate authority for these private names. Import
this certificate into client trust stores to remove browser warnings:

```text
data/caddy/data/caddy/pki/authorities/local/root.crt
```

Treat the root certificate as public data, but never distribute the private key
stored beside it.

## Jellyfin hardware acceleration

The overlay only makes the GPU available to the container. In Jellyfin, open
Dashboard, Playback, Transcoding and select the matching acceleration method:

- Intel: Quick Sync or VA-API
- NVIDIA: NVENC

Enable only codecs supported by the installed GPU. Test with a forced
transcode and check Jellyfin's FFmpeg log before depending on it.

## Operations

Update and recreate the containers:

```sh
docker compose pull
docker compose up -d
```

For a GPU deployment, include its overlay in every Compose command.

View status and recent logs:

```sh
docker compose ps
docker compose logs --tail=100
```

Back up these directories while the relevant container is stopped:

```text
data/pihole
data/jellyfin/config
data/portainer
data/caddy
```

Portainer is part of the project it manages. Keep command-line Docker access
available: deleting or breaking the entire project also removes the Portainer
UI until the project is started again from the command line.

## Security notes

- Keep these administration services restricted to the trusted LAN.
- Do not forward Pi-hole, Portainer, or Caddy administration ports from the internet.
- Protect `.env` and the complete `data/` directory as sensitive backup material.
- The Docker socket gives Portainer administrative control of the host.
- Pin image tags or digests before using the project in a change-controlled environment.

## Port conflicts

If Pi-hole cannot bind port 53, identify the current listener before changing
anything:

```sh
sudo ss -lntup '( sport = :53 )'
```

On systems using `systemd-resolved`, follow the operating system's documented
procedure for freeing port 53. Do not disable the resolver remotely unless an
alternate working resolver is already configured.

