---
title: Access servers using teleport, tailscale, cloudflare
date: 2022-06-19 17:18:00 +0530
categories: [homelab, domain, cloudflare]
tags: [homelab, domain, cloudflare, sso, totp, security] # tag names should always be lowercase
---

## Cloudflared Proxy

- Check this post [here]({% link _posts/2022-06-04-connect-services-to-internet.md%}#1-cloudflare-tunnel) for more details.

- Point your **domain or sub-domain** to **http://localhost:3080**

## Tailscale

- Check this post [here]({% link _posts/2022-06-17-using-tailscale-to-access-services.md%}) for more details.

- Setup **tailscale** for **all** the **servers** which you want to access using **teleport**.

# Installing Teleport

- **Directory** structure will be as follows:

```text
teleport
├── docker-compose.yml
├── config
│   └── teleport.yaml
└── data
```

- Create a folder **teleport** and add this _docker-compose.yml_ file.

```yaml
---
version: "3"

services:
  teleport:
    image: quay.io/gravitational/teleport:9.3.2
    container_name: teleport
    entrypoint: /bin/sh
    hostname: teleport.example.com # Change this to your domain name
    command: -c "sleep 1 && /bin/dumb-init teleport start -c /etc/teleport/teleport.yaml"
    environment:
      - TZ=Asia/Kolkata
    ports:
      - "3023:3023"
      - "3024:3024"
      - "3025:3025"
      - "3080:3080"
    volumes:
      - ./config:/etc/teleport
      - ./data:/var/lib/teleport
    restart: unless-stopped
    networks:
      - caddy_default

networks:
  caddy_default:
    external: True
```

- Add this _teleport.yaml_ file in /teleport/config folder. For more details, check [this link](https://goteleport.com/docs/setup/reference/config/)

```yaml
version: v2
teleport:
  nodename: teleport.example.com # Change this to your domain name
  data_dir: /var/lib/teleport
  log:
    output: stderr
    severity: INFO
    format:
      output: text
  ca_pin: sha256:<your-ca-pin>
  diag_addr: ""
auth_service:
  enabled: "yes"
  listen_addr: 0.0.0.0:3025
  public_addr: teleport.example.com:3025 # Replace this with your domain name:3025
  proxy_listener_mode: multiplex
ssh_service:
  enabled: "yes"
  labels:
    env: example
  commands:
    - name: hostname
      command: [hostname]
      period: 1m0s
proxy_service:
  enabled: "yes"
  web_listen_addr: 0.0.0.0:3080
  # public address which caddy hosts in front of Teleport
  listen_addr: 0.0.0.0:3023
  tunnel_listen_addr: 0.0.0.0:3024
```

- Run the following command to start **Teleport**:

  ```bash
  docker-compose up -d --force-recreate
  ```

- Get `ca_pin` running the following command:

  ```bash
  docker-compose exec teleport tctl status
  ```

- Replace `sha256:<your-ca-pin>` with the CA pin you got from the above command.

- Run the following command to create user:

  ```bash
  docker-compose exec teleport tctl users add <user> --logins=<user>,root --roles=access,editor
  ```

- This command will **generate** a link to create **password** and **TOTP** secret for the `<user>`.

- The link will contain **3080** port number, change this to **443** or remove **:3080** from the link if you are using it over your **cloudflare** DNS.

- Use the link _as it is_ if using **tailscale DNS**.

### Adding Nodes

- Run the following command to add nodes:

  ```bash
  docker exec teleport tctl nodes add
  ```

- This will show you **auth_token** and **ca_pin** for the **node**. Copy as we will need this later.

- Install **Teleport** on the **node** using the installation [page](https://goteleport.com/docs/setup/admin/daemon/).

- Now create _teleport.yaml_ file on the **node** in _/etc_ folder.

```yaml
teleport:
  nodename: <Any name of the node>
  data_dir: /var/lib/teleport
  auth_token: <auth_token>
  auth_servers: # Replace this with the IP address of the node which will able to access the Teleport cluster
    - <teleport.example.com_or_tailscale-ip_or_tailscale_DNS>:3025
  log:
    output: stderr
    severity: INFO
  ca_pin: sha256:<your-ca-pin>
  #advertise_ip: This server IP address
auth_service:
  enabled: no
ssh_service:
  enabled: yes
proxy_service:
  enabled: no
```

- If you **don't** want to **open** any **port** then use **tailscale IP or DNS** name instead of **cloudflare** registered domain name. If port **3025** is open on the main teleport server (**cluster**) then use **teleport.example.com**.

- **Remove** _/var/lib/teleport/_ if it already **exists** using the following command:

  ```bash
  sudo rm -rf /var/lib/teleport/
  ```

- Open **3022** port on this server (**node**) by running the following command:

  ```bash
  sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 3022 -j ACCEPT

  sudo netfilter-persistent save
  ```

  - Can also **open** port using **ufw**:

    ```bash
    sudo ufw allow 3022
    ```

- We want to use _/etc/teleport.yaml_ for our **config**.

- Check _/usr/lib/systemd/system/teleport.service_ file for the following line, if something **different** then **replace** it with the following:

  ```bash
  ExecStart=/usr/local/bin/teleport start --config=/etc/teleport.yaml --pid-file=/run/teleport.pid
  ```

- Run the following commands to start **Teleport** as a **systemd** service:

  ```bash
  sudo systemctl enable teleport.service
  ```

- **Start** the **service**, replace start with **restart** if you want to restart the service.

  ```bash
  sudo systemctl start teleport
  ```

- Check the **status** of **Teleport** using the following command:

  ```bash
  sudo systemctl status teleport
  ```

- Now check on your teleport **dashboard** you will able to connect to your servers.

## Using Caddy for SSL, Teleport over tailscale for accessing it over private vpn network

- Install **Caddy** from [page]({% link _posts/2022-06-04-connect-services-to-internet.md%}#1-caddy}).

- Add the following line to the _Caddyfile_ file:

```text
machine-name.domain-alias.ts.net {
	reverse_proxy teleport:3080 {
		transport http {
                        tls
                        tls_insecure_skip_verify
                }
	}
}
```

- You can run both **cloudflared** and **caddy** on the same machine **simultaneously** to access the **Teleport cluster**.

## Setup without opening or forwarding any ports

- Use **cloudflared** which **don't** require any **port** to be **open** or any port to be **forwarded**. So **no** port **opening** of **3080** is required.

- Use **tailscale DNS** for accessing the Teleport **cluster auth** service. So **no** port **opening** of **3025** is required.
