# NAS foundation stack

Run four common home-server services from one Docker Compose project:

- Pi-hole v6 filters DNS and handles local records through its embedded dnsmasq-compatible service.
- Jellyfin serves media from a read-only host directory.
- Caddy gives each service a local HTTPS address.
- Portainer CE provides a browser-based view of Docker.

Jellyfin runs on the CPU by default. Optional Compose overlays enable Intel
Quick Sync or VA-API, and NVIDIA NVENC/NVDEC.

## Requirements

The host needs:

- A Linux host with Docker Engine and Docker Compose v2
- A static LAN address for the host
- TCP and UDP port 53 available on the host
- TCP ports 80 and 443 available on the host
- An existing media directory readable by the chosen `PUID` and `PGID`

Back up the persistent data before replacing an existing Pi-hole, Jellyfin,
Caddy, or Portainer installation. This project stores its state in bind mounts
under `./data`.

## Configure

```sh
cp .env.example .env
```

Set these values in `.env`:

- `SERVER_IP` to the server's static LAN address
- `MEDIA_PATH` to the media directory on the host
- `PIHOLE_PASSWORD` to a unique password
- `PUID` and `PGID` to the account that can read the media directory

Create the persistent directories, then give Jellyfin ownership of its config
and cache. Replace `1000:1000` if `.env` uses a different `PUID` or `PGID`:

```sh
mkdir -p data/{caddy/config,caddy/data,jellyfin/cache,jellyfin/config,pihole,portainer}
sudo chown -R 1000:1000 data/jellyfin
```

This command generates a password without saving it in shell history:

```sh
openssl rand -base64 32
```

Git ignores `.env`. Keep it out of commits and backups that other people can
read.

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

The NVIDIA overlay requires a working NVIDIA driver and NVIDIA Container
Toolkit. Run `nvidia-smi` on the host before starting the stack.

## Switch the network to Pi-hole

The project does not change router settings. Test Pi-hole directly before
advertising it to the rest of the network:

```sh
docker compose ps
dig @SERVER_IP pi.hole
dig @SERVER_IP watch.home.arpa
```

Replace `SERVER_IP` with the address from `.env`. When both lookups succeed,
set the router's LAN DHCP/DNS option to advertise that address as the DNS
server.

Wait until Pi-hole has survived a reboot before making it the Docker host's
only resolver. If the container is unavailable, the host still needs to resolve
image registries and package mirrors.

Pi-hole v6 includes its own dnsmasq-compatible DNS engine. The Compose file
passes local service records through `FTLCONF_misc_dnsmasq_lines`. Add other
dnsmasq-compatible files under `dnsmasq.d/`.

This project leaves DHCP to the router. Running a second DHCP server can break
client addressing, and a containerized DHCP server needs a different network
configuration.

See the [Pi-hole Docker documentation](https://docs.pi-hole.net/docker/) for
Pi-hole v6 settings and DHCP options.

## Service addresses

After Pi-hole is the LAN DNS server:

- Jellyfin: `https://watch.home.arpa`
- Pi-hole: `https://dns.home.arpa/admin/`
- Portainer: `https://portainer.home.arpa`

Caddy signs these private names with its internal certificate authority. Import
the following root certificate into each client trust store to remove browser
warnings:

```text
data/caddy/data/caddy/pki/authorities/local/root.crt
```

The root certificate is safe to distribute. The private key stored beside it
is not.

## Jellyfin hardware acceleration

The overlays give the Jellyfin container access to the GPU. They do not change
Jellyfin's playback settings. Open Dashboard, Playback, Transcoding and choose
the matching method:

- Intel: Quick Sync or VA-API
- NVIDIA: NVENC

Enable only the codecs supported by the installed GPU. Force a transcode and
check Jellyfin's FFmpeg log before relying on hardware acceleration.

See the [Jellyfin hardware acceleration guide](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/)
for vendor-specific setup and codec support.

## Updates and backups

Update and recreate the containers:

```sh
docker compose pull
docker compose up -d
```

Include the same GPU overlay in every Compose command used for that deployment.

View status and recent logs:

```sh
docker compose ps
docker compose logs --tail=100
```

Stop the relevant container before backing up its directory:

```text
data/pihole
data/jellyfin/config
data/portainer
data/caddy
```

Portainer belongs to the project it manages. Keep command-line Docker access
available because removing the project also removes the Portainer interface.
Start the project from the command line to recover it.

## Security

- Restrict the administration services to a trusted LAN.
- Do not forward this stack's DNS or web ports from the internet.
- Protect `.env` and the complete `data/` directory as sensitive backup material.
- The Docker socket gives Portainer administrative control of the host.
- Pin image tags or digests before using the project where updates require review.

## Port conflicts

If Pi-hole cannot bind port 53, identify the current listener before changing
anything:

```sh
sudo ss -lntup '( sport = :53 )'
```

On systems using `systemd-resolved`, follow the operating system's instructions
for freeing port 53. Do not disable the resolver during a remote session unless
the host already has another working resolver.

