---
title: Connecting services to internet using reverse proxy
date: 2022-06-04 16:33:00 +0530
categories:
  [
    homelab,
    domain,
    cloudflare,
    services,
    internet,
    traefik,
    nginx,
    letsencrypt,
    tunnel,
    ssl,
    tls,
    email,
    forward,
  ]
tags: [
    homelab,
    domain,
    cloudflare,
    services,
    internet,
    traefik,
    nginx,
    letsencrypt,
    tunnel,
    ssl,
    tls,
    email,
    forward,
    caddy,
    reverse,
    proxy,
    connect,
    cloudflared-tunnel,
    tunnel,
  ] # tag names should always be lowercase
---

# Exposing your services to the internet using SSL

We have a domain but the **tricky** part is how to **connect** that domain to our service.

Here comes the part of **reverse proxy** and **cloudflare tunnel**.

We have different **services** running at different ports and we want to expose them to the internet on different apex domains and sub domains.

### <u>DNS Entry</u>

You will have to **point** your domain or subdomain on your DNS to the server on which you will run your services using reverse proxy. No need to do this if you are using a **cloudflare tunnel**.

Add a DNS **A** record for the apex domain and point it to your server's IP address.

If your apex domain point to a server, you also want your subdomain to point to that same server then add a **CNAME** record for the subdomain and point it to the apex domain.

# 1. Cloudflare Tunnel

It is the easiest of them all.

- Go to Cloudflare [dashboard](https://dash.cloudflare.com/)

- Select your domain > click on access tab and then click on [**Launch Zero Trust**](https://dash.teams.cloudflare.com/).

- This will open a zero trust dashboard, then got to Access > Tunnels > Create a tunnel.

- Give you tunnel a name then click on next, choose your **os** environment then appropriately install cloudflared **connector** and click on next. (Do not install docker for **arm64**)

- Click on Add a public hostname,then provide your subdomain name, select your subdomain.

- In service section, select http if service don't have SSL certificate, then enter **localhost:<PORT_NUMBER>** of your application. Then click on save hostname. (You can add multiple hostnames for different services).

- It will add **subdomain** to your **DNS** automatically.

### <u>Check if cloudflared service running : </u>

Run the following command in your terminal.

```bash
sudo systemctl status cloudflared
```

If not running then run the following command in your terminal.

```bash
sudo systemctl start cloudflared
```

Check if service is enabled for running on boot.

```bash
sudo systemctl is-enabled cloudflared
```

If not enabled then run the following command in your terminal.

```bash
sudo systemctl enable cloudflared
```

Try restarting the service

```bash
sudo systemctl restart cloudflared
```

If it does not start then we have create a cert.pem file which is the origincert flag file.

## There are two ways to create a cert.pem file :

### 1. Through Terminal

Type the following command in your terminal.

```bash
cloudflared tunnel login
```

It will open a browser window showing **Authorize Cloudflare Tunnel**.

Click on your domain name and then click on **Authorize**.

It will automatically add a cert.pem file in your /home/\<USER>/.cloudflared/ directory.

Then try restarting the cloudflared service again.

### 2. Through Browser and downloading the cert.pem file

Open this [Argo Tunnel link](https://dash.cloudflare.com/argotunnel) > Select your domain > Authorize > Download the certificate.

Then copy the cert.pem file to your **/etc/cloudflared** directory.

Then try restarting the cloudflared service again.

# 2. Reverse Proxy Manager

There are many reverse proxy managers available like **caddy**, **nginx proxy manager**, **traefik** etc.

For this you will need your domain or subdomain to point to your server's IP address. See this [DNS Entry section](#dns-entry).

## 1. Caddy

- Open the installation page [here](https://caddyserver.com/docs/install). If you want to install on the host machine. We will use docker for this.

- Use _docker-compose.yml_ as mentioned below. (Make a directory named caddy and copy the file to that directory)

```yaml
version: "3.7"

services:
  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      # - $PWD/site:/srv
      - ./caddy_data:/data
      - ./caddy_config:/config
      - ./certs:/etc/certs/origin_cloudflare
```

It's very simple with caddy just add the following to your **Caddyfile**.

```
#{
#	acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
#}

caddy.example.com {
	#tls /etc/certs/origin_cloudflare/ssl_origin_certificate.cert /etc/certs/origin_cloudflare/ssl_origin_certificate_key.pem
	reverse_proxy nginx:80
}
```

- Run command in your terminal to run the caddy docker.

  ```bash
  docker-compose up -d --force-recreate
  ```

- We will use **nginx** as an example here. First we will create a nginx docker.

```yaml
version: "3"

services:
  nginx:
    image: nginx:latest
    container_name: nginx
    restart: unless-stopped
    # ports:
    #   - "1080:80"
    networks:
      - caddy_default

networks:
  caddy_default:
    external: True
```

- Don't use **proxy** when adding **DNS record** to **cloudflare** when first getting **letsenrypt** certificate. (It needs to check if domain is _pointing_ to the server ip or not, You can after certificate is obtained)

- Caddy will **automatically** generate a SSL certificate for you from the **Let's Encrypt** server. (Make sure you have ports 80 and 443 open in your firewall or your server vnc because by default it uses http challenge to get the certificate.)

- First try with **staging SSL** certificate by using the following command at the top of caddyfile. If everything work fine then comment out the following line as showing in our caddyfile. (We are using staging SSL certificate here because there is a _rate limit_ for _production SSL_ certificate.)

  ```
  {
    acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
  }
  ```

- If you want to use **custom SSL** certificate then use the following line.

  ```
  tls /etc/certs/origin_cloudflare/ssl_origin_certificate.cert /etc/certs/origin_cloudflare/ssl_origin_certificate_key.pem
  ```

- To get _custom_ certificates from cloudflare check [this](#2-through-cloudflare)

## 2. Installing Nginx Proxy Manager

- Open the [ installation guide](https://nginxproxymanager.com/guide/).

- Use _docker-compose.yml_ as mentioned below. (Make a directory named nginx_proxy_manager and copy the file to that directory)

```yaml
version: "3"
services:
  app:
    image: "jc21/nginx-proxy-manager:latest"
    container_name: "nginx-proxy-manager"
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    networks:
      - npm

networks:
  npm:
```

Now run the following command in your terminal.

```bash
docker-compose up -d
```

Then check if docker created our network.

```bash
docker network list
```

It will show the network name as **\<FOLDER_NAME>\_npm**.

```
name : nginx_proxy_manager_npm
driver : bridge
scope : local
```

Now for example lets say you want to host vaultwarden on port 8080.

Use the docker network command as follows in docker-compose.yml file of that service. See the networks part.

```yml
version: "3"
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    networks:
      - nginx_proxy_manager_npm
    volumes:
      - ./data:/data/
    ports:
      - 8080:80
    restart: unless-stopped

networks:
  nginx_proxy_manager_npm:
    external: true
```

Here **external flag** is used to connect to a network which is already created.

Now go to the **http://\<localhost_or_your_ip>:81** for nginx proxy manager dashboard.

## <u>Adding SSL Certificate</u>

- Got to SSL Certificates tab. Then click on **Add SSL Certificate**.

There are two ways to get SSL certificate :

### 1. <u>Through Letsencrypt</u>

- Enter the **apex** domain, **wildcard** subdomain name in the **Domain Names** field. (eg. **example.com**, **\*.example.com**)

- Enter any email address from which you want to receive the certificate.

- Check on DNS Challenge checkbox. Choose DNS provider as cloudflare. (You have to use DNS challenge if you want wildcard subdomain.)

It will show
Credentials file content field.

You want an API token from cloudflare:

- Go to [cloudflare dashboard](https://dash.cloudflare.com/) > Overview > Get your API token (bottom right hand side).

- Click on create token >use template of edit zone DNS > Under zone resources change specific zone to all zones > click on continue to summary > click on create token.

- Copy the token and save it (it will not be shown later or just create a new one if lost).

- Copy this edit zone dns api token here as shown below:

```
# Cloudflare API token
dns_cloudflare_api_token = <ADD_YOUR_API_TOKEN>
```

- Click on I Agree then Save. It will take some time to generate the certificate.

This certificate will only last upto 3 months then we have to renew it.

### 2. <u>Through Cloudflare</u>

- Go to [cloudflare dashboard](https://dash.cloudflare.com/) > SSL/TLS > Origin Server > Create Certificate.

- Select 15 years for certificate validity > Create.

- Save **Origin Certificate** as _\<filename>.pem_ and **Private Key** as _\<filename>.key_

- Now go to nginx proxy manager dashboard and click on **SSL Certificates** > Add SSL Certificate > Custom.

- Give any name, **certificate key** will be _\<filename>.key_ and **certificate** will be _\<filename>.pem_ > Click on Save. It will take some time to generate the certificate.

### <u>Adding a service to a domain or subdomain</u>

- Go to nginx proxy manager dashboard > Hosts > Proxy Hosts > Add proxy host.

- Under domain names enter the domain name or subdomain name.

- Select http.

- Hostname/IP will be **name of our docker container** which is on the same network as nginx proxy manager or IP of your server when not running in docker. (You can also provide IP of your docker container but it will change after every restart.)

To Check IP of your docker container use the following command in your terminal :

```bash
docker inspect <CONTAINER_NAME_OR_ID> | grep '"IPAddress"' | tail -n1
```

- Port will be the **internal** port of the service which is running in docker (8080:80 then **80**) or the service port when not running in docker.

- Tick all the checkboxes > Got to SSL Certificates > Select the appropriate certificate > Tick all the checkboxes > Click on Save.

## 3. Intalling Traefik

This one is bit complicated because Traefik allows various implementations of configuration.

First create a traefik folder in which we will install traefik.

The overall file structure is as follows.

<!-- ![traefik_file_heirarchy](/assets/traefik_file_heirarchy.png) -->

```
traefik/
├── docker-compose.yml
└── etc
    └── traefik
        ├── certs
        │   ├── acme.json
        │   ├── cert-key.pem
        │   └── cert.cert
        ├── config.yml
        └── traefik.yml

3 directories, 6 files
```

Then create a docker-compose.yml file in that folder.

```yaml
version: "3"

services:
  traefik:
    image: "traefik:latest"
    container_name: "traefik"
    restart: "unless-stopped"
    environment:
      - TZ=Asia/Kolkata
      # - CF_API_EMAIL=your_cloudflare_email
      # - CF_DNS_API_TOKEN=your_cloudflare_api_token
      # - CF_API_KEY=YOU_API_KEY
      # be sure to use the correct one depending on if you are using a token or key
    ports:
      - "80:80"
      - "443:443"
      # (Optional) Expose Dashboard
      # - "8082:8080"  # Don't do this in production!
    volumes:
      - ./etc/traefik:/etc/traefik
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik-logs:/var/log/traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.entrypoints=websecure"
      - "traefik.http.routers.traefik.rule=Host(`traefik.example.com`)"
      - "traefik.http.middlewares.traefik-auth.basicauth.users=USER:BASIC_AUTH_PASSWORD"
      - "traefik.http.routers.traefik.middlewares=traefik-auth"
      - "traefik.http.routers.traefik.tls=true"
      - "traefik.http.services.traefik.loadbalancer.server.port=8080"

      # - "traefik.http.routers.traefik-secure.tls=true"
      # - "traefik.http.routers.traefik-secure.tls.certresolver=cloudflare"
      # - "traefik.http.routers.traefik-secure.tls.domains[0].main=example.com"
      # - "traefik.http.routers.traefik-secure.tls.domains[0].sans=*.example.com"
      # - "traefik.http.routers.traefik-secure.service=api@internal"

volumes:
  traefik-logs:
```

Adding authentication for the traefik dashboard in above yml.

1. Install :

```bash
sudo apt update
sudo apt install apache2-utils
```

2. Create a user:password for the dashboard.

```bash
echo $(htpasswd -nb <USER> <PASSWORD>) | sed -e s/\\$/\\$\\$/g
```

Here <USER> is the username and <PASSWORD> is the password.

Replace **USER:BASIC_AUTH_PASSWORD** with the generated string.

If you want letsenrypt SSL certificates then you can use appropriate certresolver.

I am using cloudflare as my DNS provider.

Then create etc > traefik folder in traefik folder.

```bash
mkdir -p etc/traefik
```

Then create a traefik.yml file in /traefik/etc/traefik folder.

This is a **static configuration** for traefik. It will use this at the start of traefik container.

```yaml
global:
  checkNewVersion: true
  sendAnonymousUsage: false # true by default

# (Optional) Log information
# ---
# log:
#   level: DEBUG # DEBUG, INFO, WARNING, ERROR, CRITICAL
#   format: common  # common, json, logfmt
#   filePath: /var/log/traefik/traefik.log

# (Optional) Accesslog
# ---
# accesslog:
# format: common  # common, json, logfmt
# filePath: /var/log/traefik/access.log

# (Optional) Enable API and Dashboard
# ---
api:
  dashboard: true # true by default
  insecure: true # Don't do this in production!

# Entry Points configuration
# ---
entryPoints:
  web:
    address: :80
    # (Optional) Redirect to HTTPS
    # ---
    http:
      middlewares: # middleware that works for every request
        - test-ratelimit@file
      redirections:
        entryPoint:
          to: websecure
          scheme: https

  websecure:
    address: :443
    http:
      middlewares: # middleware that works for every request
        - test-ratelimit@file

# Configure your CertificateResolver here...
# ---
certificatesResolvers: # Don't use proxy when adding DNS record to cloudflare when first getting letsenrypt certificate (You can after certificate is obtained)
staging: # Open ports 80 and 443 in firewall
  acme:
    email: your_email_address_for_letsencrypt
    storage: /etc/traefik/certs/acme.json
    caServer: "https://acme-staging-v02.api.letsencrypt.org/directory"
    httpChallenge:
      entryPoint: web

production: # Open ports 80 and 443 in firewall
  acme:
    email: your_email_address_for_letsencrypt
    storage: /etc/traefik/certs/acme.json
    caServer: "https://acme-v02.api.letsencrypt.org/directory"
    httpChallenge:
      entryPoint: web

cloudflare: # Open ports 80 and 443 in firewall
  acme:
    email: your_email_address_for_letsencrypt
    storage: /etc/traefik/certs/acme.json
    dnsChallenge:
      provider: cloudflare
      resolvers:
        - "1.1.1.1:53"
        - "1.0.0.1:53"

# (Optional) Overwrite Default Certificates
tls:
  stores:
    default:
      defaultCertificate:
        certFile: /etc/traefik/certs/cert.cert
        keyFile: /etc/traefik/certs/cert-key.pem
  # (Optional) Disable TLS version 1.0 and 1.1
  options:
    default:
      minVersion: VersionTLS12

providers:
  docker:
    exposedByDefault: false # Default is true
  file:
    # watch for dynamic configuration changes
    directory: /etc/traefik
    watch: true
    # filename: /etc/traefik/config.yml

log:
  level: "INFO"
  filePath: "/var/log/traefik/traefik.log"
accessLog:
  filePath: "/var/log/traefik/access.log"
```

If you want any **dynamic configuration** then add a _config.yml_ file in /traefik/etc/traefik folder. This is used for any other networking services that we need. (Also to communicate with services that don't use docker cotainer.)

This is an example of _config.yml_ file. Traefik will watch for changes in this file and reload the configuration. This file is not necessary.

```yaml
# http routing section
http:
  routers:
    # Define a connection between requests and services
    to-whoami:
      entryPoints:
        - "websecure"
      rule: "Host(`example.com`) && PathPrefix(`/whoami/`)"
      # If the rule matches, applies the middleware
      middlewares:
        - test-user
      tls: {} # SSL termination configuration leave empty to send this service request to http endpoint
      # If the rule matches, forward to the whoami service (declared below)
      service: whoami

  services:
    # Define how to reach an existing service on our infrastructure
    whoami:
      loadBalancer:
        servers: # use host.docker.internal instead of <private-ip> for accessing the service from the host machine
          - url: http://<private-ip>:8000

  middlewares:
    # Here, an average of 100 requests per second is allowed.
    # In addition, a burst of 50 requests is allowed.
    test-ratelimit:
      rateLimit:
        average: 100
        burst: 50

    # Define an authentication mechanism
    test-user:
      basicAuth:
        users: # set password the same as traefik dashboard
          - test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/
```

Now add a certs folder in this /traefik/etc/traefik folder.

```bash
mkdir certs
```

Then create acme.json file. It has to be created for traefik to run. It will be used by traefik to store certificates as shown in yml.

```yaml
{
  "email": "your_email_address_for_letsencrypt",
  "storage": "/etc/traefik/certs/acme.json",
}
```

```bash
touch acme.json
chmod 600 acme.json
```

- Don't use **proxy** when adding **DNS record** to **cloudflare** when first getting **letsenrypt** certificate. (It needs to check if domain is _pointing_ to the server ip or not, You can after certificate is obtained)

- If you want to use **letsencrypt SSL** certificates then you can use appropriate certresolver as _staging_ or _production_. (These will be stored in acme.json, if using staging as your certresolver then **delete** all contents of _acme.json_ file first then use production certresolver.)

- If you want to use custom certificates then add the following lines in config.yml file.

```yaml
# Standalone TLS configuration (do not paste this under any section)
tls: # Use this for custom SSL certs other than default one set in traefik.yml
  certificates: # traefik will match your domain name with the certificate name (without .com)
    - certFile: /path/to/other-domain-name.cert
      keyFile: /path/to/other-domain-name.key
```

- If you have obtained your coudflare SSL certificates then you can store them in the certs folder. For more info to get your cloudflare SSL certificates read this [cloudflare ssl certificate](#2-through-cloudflare).

- `cert.cert`: contains the certificate
- `cert-key.pem`: contains the private key

Then start your traefik container by running the following command by going to the /traefik folder where docker compose file is added.

```bash
docker-compose up -d --force-recreate
```

### Example of a docker container using traefik.

We will use nginx as our example container.

First create a new folder named nginx on your machine. Then add docker-compose.yml file in this folder.

```yaml
version: "3"

services:
  nginx:
    image: nginx:latest
    container_name: nginx
    restart: unless-stopped
    networks:
      - traefik_default
    # ports:
    #   - "80:80"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nginx.entrypoints=websecure"
      - "traefik.http.routers.nginx.rule=Host(`nginx.example.com`)"
      - "traefik.http.routers.nginx.tls=true"
      # - "traefik.http.routers.nginx.tls.certresolver=production" # use this if you want to use LetsEncrypt SSL certificates
      - "traefik.http.services.nginx.loadbalancer.server.port=80"

networks:
  traefik_default:
    external: true
```

Things to remember while creating docker-compose.yml file for a container which uses traefik.

- `traefik.enable=true`: This is required to enable traefik to know which container wants to use traefik.

- `traefik.http.routers.nginx.entrypoints=websecure`: This is required to enable traefik to watch https (:443) port as entrypoint for this service.

- `traefik.http.routers.nginx.rule=Host(`nginx.example.com`)`: This is required to enable traefik to watch nginx.example.com as hostname for this service.

- `traefik.http.routers.nginx.tls=true`: This is required to enable traefik to use SSL.

- `traefik.http.routers.nginx.tls.certresolver=production`: This is required to enable traefik to use LetsEncrypt SSL certificates. (staging or production)

- `traefik.http.services.nginx.loadbalancer.server.port=80`: This is required to enable traefik to send traffic to this docker container at port 80.

This is the format traeffic uses to send traffic to
your container.

![traefik map](/assets/traefik_map.png)

If you want to add a middleware like authelia for authentication before accessing the service then you can add it as a label to the nginx docker-compose.yml file as shown below.

```yaml
- "traefik.http.routers.nginx.middlewares=authelia@docker"
```

Now the traefik map will look like this.

![traefik_map_with_middleware](/assets/traefik_map_with_middleware.png)

# Connect services running on host machine from reverse proxy running in docker container

- Add an _environment variable_ to your docker-compose.yml file.

  ```yaml
  extra_hosts:
    - host.docker.internal:$HOST_GATEWAY
  ```

- Adding the following in **script.sh** file.

  ```bash
  #!/bin/bash

  export HOST_GATEWAY=$(ip -4 addr show docker0 | grep -Po 'inet \K[\d.]+')
  # docker compose up -d --force-recreate
  ```

- Run the script.sh using the following command.

  ```bash
  source ./script.sh
  ```

- Now you can access the service on your host machine from the docker container using **host.docker.internal** instead of **localhost**.
