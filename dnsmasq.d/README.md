# Additional dnsmasq configuration

Pi-hole v6 contains its own dnsmasq-compatible DNS engine. Place optional
`*.conf` files in this directory when configuration beyond the local service
records in `.env` is required.

For example, `10-router-reverse.conf` could contain:

```ini
rev-server=192.168.1.0/24,192.168.1.1
```

Validate custom lines before restarting Pi-hole:

```sh
docker compose exec pihole pihole-FTL dnsmasq-test
```

