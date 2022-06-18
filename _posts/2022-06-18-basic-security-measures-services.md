---
title: Enhancing security on services using crowdsec, watchtower, authelia, traefik
date: 2022-06-18 15:03:00 +0530
categories:
  [homelab, domain, cloudflare, traefik, authelia, watchtower, crowdsec]
tags: [
    homelab,
    domain,
    cloudflare,
    traefik,
    authelia,
    sso,
    totp,
    watchtower,
    crowdsec,
    security,
  ] # tag names should always be lowercase
---

## Ports

- Only **open ports** that are **required** for the **service**.

## Traefik

- Check for traefik installation and setup [here]({% link _posts/2022-06-04-connect-services-to-internet.md %}#3-installing-traefik).

## Authenticating with authelia

- Check this post [here]({% link _posts/2022-06-16-adding-authentication-to-services.md %}).

## Adding watchtower to auto update docker images

- This service will be used to **auto update** the **docker images** of the services every **24** hours (default).

- It will send the necessary logs via notifications. Check this [link](https://containrrr.dev/watchtower/notifications/) for more details.

- Create a folder **watchtower** and add this _docker-compose.yml_ file.

```yaml
version: "3"
services:
  watchtower:
    image: containrrr/watchtower
    container_name: "watchtower"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - TZ=Asia/Kolkata
      - WATCHTOWER_NOTIFICATIONS=slack
      - WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL="https://hooks.slack.com/services/x/y/z"
      - WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER=watchtower
      - WATCHTOWER_NOTIFICATION_SLACK_CHANNEL=#watchtower
      # - WATCHTOWER_RUN_ONCE=true
```

- Update **WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL** with your slack webhook url.

## Adding ratelimit to services

Check this [page]({% link _posts/2022-06-04-connect-services-to-internet.md %}#3-installing-traefik).

## Adding IP whitelist middleware to traefik

- Add **IP whitelist** middleware in traefik to allow only the following cloudflare [**IP's**](https://www.cloudflare.com/en-gb/ips/) to access the services.

- Add this in config.yml file:

```yaml
middlewares:
  cloudflare-ipwhitelist:
    ipWhiteList:
      sourceRange:
        - 173.245.48.0/20
        - 103.21.244.0/22
        - 103.22.200.0/22
        - 103.31.4.0/22
        - 141.101.64.0/18
        - 108.162.192.0/18
        - 190.93.240.0/20
        - 188.114.96.0/20
        - 197.234.240.0/22
        - 198.41.128.0/17
        - 162.158.0.0/15
        - 104.16.0.0/13
        - 104.24.0.0/14
        - 172.64.0.0/13
        - 131.0.72.0/22
```

- Apply this **middleare** to your **external** services in _config.yml_ using below lines.

```yaml
middlewares:
  - cloudflare-ipwhitelist
```

- Apply this **middleware** to **all services** by adding below lines to _traefik.yml_.

```yaml
entryPoints:
  web:
    address: :80
    # (Optional) Redirect to HTTPS
    # ---
    http:
      middlewares:
        - cloudflare-ipwhitelist@file
        - services-ratelimit@file
      redirections:
        entryPoint:
          to: websecure
          scheme: https

  websecure:
    address: :443
    http:
      middlewares:
        - cloudflare-ipwhitelist@file
        - services-ratelimit@file
```

- You can add multiple middlewares to the **entryPoint**. Remember that **order** of the middlewares is **important**.

## Adding crowdsec middleware to traefik

- Create a folder **crowdsec** and add this _docker-compose.yml_ file.

```yaml
version: "3.8"

services:
  crowdsec:
    image: crowdsecurity/crowdsec:latest
    container_name: crowdsec
    environment:
      TZ: "Asia/Kolkata"
      GID: "${GID-1000}"
      COLLECTIONS: "crowdsecurity/linux crowdsecurity/traefik"
    # depends_on:  #uncomment if running traefik in the same compose file
    #   - 'traefik'
    volumes:
      - ./config/acquis.yaml:/etc/crowdsec/acquis.yaml
      - ./crowdsec-db:/var/lib/crowdsec/data/
      - ./crowdsec-config:/etc/crowdsec/
      - traefik_traefik-logs:/var/log/traefik/:ro
    networks:
      - traefik_default
    restart: unless-stopped

networks:
  traefik_default:
    external: true
volumes:
  traefik_traefik-logs: # this will be the name of the volume from trarfic logs
    external: true # remove if traefik is running on same stack
```

- Use [this]({% link _posts/2022-06-04-connect-services-to-internet.md %}#3-installing-traefik) as template to create traefik-logs volume and logs settings in traefik.yml file.

- Create a **config** folder in **crowdsec** folder and add this _acquis.yaml_ file.

  ```yaml
  filenames:
    - /var/log/traefik/*
  labels:
  type: traefik
  ```

- Create a update_and_upgrade.sh file in the **crowdsec** folder and add this content:

  ```bash
  #!/bin/sh

  docker exec crowdsec cscli hub update
  docker exec crowdsec cscli hub upgrade
  ```

- Run this file as a **cron job** to update and upgrade the crowdsec hub. (running every hour)

  ```cron
    0 * * * * /path_to_directory/crowdsec/update_and_upgrade.sh
  ```

- Getting the crowdsec bouncer **api** key, run the following command:

  ```bash
  docker exec crowdsec cscli bouncers add bouncer-traefik
  ```

- Store this **api key** as it will be **used** in the _next step_ and will **not** be shown again.

- Add this **bouncer-traefik** service in the same docker compose file.

```yaml
bouncer-traefik:
  image: docker.io/fbonalair/traefik-crowdsec-bouncer:latest
  container_name: bouncer-traefik
  environment:
    TZ: "Asia/Kolkata"
    CROWDSEC_BOUNCER_API_KEY: "<crowdsec_bouncer_api_key_generated_above>"
    CROWDSEC_AGENT_HOST: crowdsec:8080
  networks:
    - traefik_default # same network as traefik + crowdsec
  depends_on:
    - crowdsec
  restart: unless-stopped
```

- Run the docker-compose up command to start the crowdsec **hub** and **bouncer**.

  ```bash
  docker-compose up -d --force-recreate
  ```

- Add the following lines to _config.yml_ file of **traefik**.

```yaml
middlewares:
  crowdsec-bouncer:
    forwardauth:
      address: http://bouncer-traefik:8080/api/v1/forwardAuth
      trustForwardHeader: true
```

- Add this **middleware** to every service,add below lines to _traefik.yml_ of **traefik**.

```yaml
entryPoints:
  web:
    address: :80
    # (Optional) Redirect to HTTPS
    # ---
    http:
      middlewares:
        - crowdsec-bouncer@file
        - cloudflare-ipwhitelist@file
        - services-ratelimit@file
      redirections:
        entryPoint:
          to: websecure
          scheme: https

  websecure:
    address: :443
    http:
      middlewares:
        - crowdsec-bouncer@file
        - cloudflare-ipwhitelist@file
        - services-ratelimit@file
```

- **Restart** the **traefik** service.

- To see **crowdsec hub metrics** use the following command:

  ```bash
  docker exec crowdsec cscli metrics
  ```

- To see banned IPs use the following command:

  ```bash
  docker exec crowdsec cscli decisions list
  ```

- To ban IPs use the following command:

  ```bash
  docker exec crowdsec cscli decisions add --ip <ip>
  ```

- To unban IPs use the following command:

  ```bash
  docker exec crowdsec cscli decisions delete --ip <ip>
  ```
