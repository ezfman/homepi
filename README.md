# homepi
This repository assumes you have a Pi 4 with:
1. Raspberry Pi OS Lite (64 bit)
2. WiFi/SSH configured
3. Optional USB storage

## Docker

To set up docker, use their install script:
```
curl -fsSL https://get.docker.com -o get-docker.sh | bash
```

This will enable the use of `docker compose` to spin up (and down) various images.

## DuckDNS

DuckDNS provides free subdomains for personal use.  The are various pros/cons to using DuckDNS vs. a paid DNS provider like CloudFlare, which you should evaluate on your own.

An account is required, and your account will have a unique API token.  Each account can have up to five free subdomains (YOURDOMAIN.duckdns.org).  Each subdomain requires an IP to bind to (IPv4/IPv6 supported).  If your ISP does not give you a static IP, there is a script available to periodically update your IPs via curl using an API token as follows:

```
crontab -e
*/20 * * * * curl -v https://duckdns.org/update/YOUR_DOMAIN/API_TOKEN
```

Note that you should validate that the correct (public) IP is being added by this script.

After claiming a subdomain and giving it a valid public IP to route to your LAN, you can create SSL certificates via letsencrypt.  This is as simple as populating the `SUBDOMAINS` and `TOKEN` fields as follows:
```
---
version: "2.1"
services:
  ddns:
    image: lscr.io/linuxserver/duckdns:latest
    container_name: ddns
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - SUBDOMAINS=YOUR_SUBDOMAIN
      - TOKEN=YOUR_TOKEN
      - LOG_FILE=false
    volumes:
      - ./config:/config

```

I have not tested multiple subdomains yet, but it is likely a simple comma-separated or indented list syntax.

After changing these values in the cloned repository, run the following:
```
cd ~/containers/ddns
docker compose up -d
```

## Traefik

Traefik will act as the reverse proxy to allow access to your services over HTTPS from outside your LAN.

Installing this repository will place traefik's required configuration under the `container/traefik` 
You will need to manually edit the `docker-compose.yml` file to change 5 values.

On lines 20 and 51, add your `SUBDOMAIN` as set up via DuckDNS.  On lines 24 and 54, add your `AUTH_STR` values.  On line 28, add your DuckDNS API token.  In a future patch, these will be abstracted to a `.env` file.

To generate `AUTH_STR` values, you can use `htpasswd`, which may need to be installed on your system via `sudo apt install htpasswd`, then call:
```
echo $(htpasswd -nb USER PASSWORD) | sed -e s/\\$/\\$\\$/g
```

It is important to use the exact command above.  You can add multiple user/password pairs via comma-separated `AUTH_STR` values.

Once configured, run the following to spin up traefik:
```
cd ~/containers/traefik
docker compose up -d
```

You can then navigate to `SUBDOMAIN.duckdns.org` to see if traefik is correctly configured.  There should be a dashboard available which requires your specified login.

As a secondary test, you can navigate to the `whoami` service by visiting `whoami.SUBDOMAIN.duckdns.org`.  If you are unable to visit these sites, try explicitly using `https`.

## Immich

To set up Immich, you can either use the configuration I've provided here or install from [their website](https://documentation.immich.app/docs/install/docker-compose) then add in config for traefik.

Random passwords should be used for the `TYPSESENSE_API_KEY` and the `DB_PASSWORD` environmental variables in the provided `.env` file.
To generate random passwords, use:
```
openssl rand -bas64 24
```

If you have USB storage, that can be linked to Immich via a symbolic link:
```
mkdir /PATH/TO/STORAGE/library
ln -s /PATH/TO/STORAGE/library ~/container/immich-app/library
```

After replacing the `SUBDOMAIN` field in the `docker-compose.yml` file, you can spin up Immich with:
```
cd ~/containers/immich
docker compose up -d
```

After a few seconds, your Immich instance should be available remotely via your DuckDNS subdomain.

NOTE: On first login, an admin account can be created by anyone!  In future versions, pre-configure the admin password.  For now, just log in right away and make the account yourself.
