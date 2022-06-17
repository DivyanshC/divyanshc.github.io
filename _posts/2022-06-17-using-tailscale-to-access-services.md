---
title: Using tailscale to access services privately on your personal VPN mesh network using SSL
date: 2022-06-17 11:24:00 +0530
categories:
  [homelab, domain, cloudflare, traefik, caddy, tailscale, vaultwarden]
tags: [
    homelab,
    domain,
    cloudflare,
    traefik,
    caddy,
    tailscale,
    VPN,
    private-VPN,
    SSL,
    vaultwarden,
  ] # tag names should always be lowercase
---

## Tailscale

### Setup tailscale

- Install **tailscale** using this [guide](https://tailscale.com/download/).

- Check if **tailscale** is running as a service.

  ```bash
  sudo systemctl status tailscale
  ```

- If not running, **start** it.

  ```bash
  sudo systemctl start tailscale
  ```

- Enable for **automatic** startup.

  ```bash
  sudo systemctl enable tailscale
  ```

- Run the following command to run the **tailscale** service:

  ```bash
  sudo tailscale up
  ```

- Check your ip and all other connected services using this command:

  ```bash
  tailscale status
  ```

### Some features of tailscale

- You can also access services using the [dashboard](https://login.tailscale.com/admin/machines).

- You can now access tailscale server over the ip or name of the server which you get from the **tailscale status** command.

- You don't need to have any **ports open** on the server to communicate with it, you can ssh and access that server without opening port 22 on that server.

- You can access services hosted on other servers using tailscale privately, only accessible through your personal tailscale VPN mesh network.

- You can use some server running tailscale as your VPN server for routing all your traffic through it. Check more on exit node [here](https://tailscale.com/kb/1103/exit-nodes/).

- You can send or receive files using **tailscale** on all your connected devices. Check more on [here](https://tailscale.com/kb/1106/taildrop/).

## Access your service over HTTPS on tailscale

- We are using tailscale to setup **vaultwarden** on our tailscale network.

- It will only be accessible from the devices connected to our tailscale network.

- We will see two ways on how to setup vaultwarden on our tailscale network using reverse proxy.

- You can also use [authelia]({% link _posts/2022-06-16-adding-authentication-to-services.md %}) to connect vaultwarden to internet for extra security. (But by using authelia you will not able to connect your extension or apps to your vaultwarden service.)

### 1. Traefik

- Create a folder **vaultwarden** and add this _docker-compose.yml_ file.

```yaml
version: "3"

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    volumes:
      - ./data:/data/
    # ports:
    #   - 8081:80
    restart: unless-stopped
    networks:
      - traefik_default
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.vaultwarden.entrypoints=websecure"
      - "traefik.http.routers.vaultwarden.rule=Host(`machine-name.domain-alias.ts.net`)"
      - "traefik.http.routers.vaultwarden.tls=true"
      # - "traefik.http.services.vaultwarden.loadbalancer.server.port=80"
      # - 'traefik.http.routers.vaultwarden.middlewares=authelia@docker'

networks:
  traefik_default:
    external: true
```

- **Vaultwarden** on works when using **SSL**. Check this page [here](https://tailscale.com/kb/1153/enabling-https/).

- Enable **HTTPS** by going to this [page](https://login.tailscale.com/admin/settings/features). Also enable **Magic DNS** to use the domain name of the server by going to this [page](https://login.tailscale.com/admin/dns).

- You will get **Tailnet domain alias** for the server in the end of this [page](https://login.tailscale.com/admin/settings/features). Example: `*.domain-alias.ts.net`.

- You can access your service over **https** using this type of domain : `https://machine-name.domain-alias.ts.net`.

- Know your **domain** easily by running just this command, it will show your domain:

```bash
tailscale cert
```

- Get SSL certificate from [Let's Encrypt](https://letsencrypt.org/) using this command:

```bash
sudo tailscale cert "machine-name.domain-alias.ts.net"
```

- You can run the above command again for getting the **updated certificate** again if the certificate gets **expired**.

- This will generate a **SSL cert and key** file for the domain in the current directory and in _/var/lib/tailscale/certs_ (You can also mount this directory to docker as **:ro** also).

- Add the following to _/etc/traefik/config.yml_ file:

```yaml
tls: # Standalone TLS configuration (do not paste this under any section)
  certificates: # traefik will match your domain name with the certificate name
    - certFile: /etc/traefik/tailscale/certs/machine-name.domain-alias.ts.net.crt
      keyFile: /etc/traefik/tailscale/certs/machine-name.domain-alias.ts.net.key
```

### 2. Caddy

- Check this on how to install [Caddy]({% link _posts/2022-06-04-connect-services-to-internet.md %}#1-caddy).

- Add these lines to docker-compose.yml file in caddy folder:

```yaml
volumes:
  - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock:ro
```

- This **tailscaled.sock** will eanble to get SSL **automatically** and **renew** the certificate **automatically**.

- Now the docker-compose.yml for vaultwarden will be the following:

```yaml
version: "3"

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    volumes:
      - ./data:/data/
    # ports:
    #   - 8081:80
    restart: unless-stopped
    networks:
      - caddy_default

networks:
  caddy_default:
    external: true
```

- Add these lines to Caddyfile in caddy folder:

```caddy
machine-name.domain-alias.ts.net {
	reverse_proxy vaultwarden:80
}
```

#### If running caddy without docker

- First check which **user** is running caddy by running this command:

  ```bash
  ps aux | grep caddy
  ```

- If you are running caddy **without docker** then add this line to **/etc/default/tailscaled**:

  ```txt
  TS_PERMIT_CERT_UID=caddy
  ```

- This will allow **caddy** to **access** _tailscaled.sock_ file to obtain **SSL** certificates.

- Add `https://machine-name.domain-alias.ts.net` to your _SELF-HOSTED ENVIRONMENT > Server URL_ on your **vaultwarden apps** and **browser extension** to access your service.
