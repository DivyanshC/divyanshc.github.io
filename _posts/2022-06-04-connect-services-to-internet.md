---
title: Connecting services to internet using encrypted methods
date: 2022-06-04 16:33:00 +0530
categories: [homelab, domain, cloudflare, services, internet]
tags: [homelab, domain, cloudflare, reverse-proxy, tunnel] # tag names should always be lowercase
---

# Exposing your services to the internet using SSL

We have a domain but the **tricky** part is how to **connect** that domain to our service.

Here comes the part of **reverse proxy** and **cloudflare tunnel**.

We have different **services** running at different ports and we want to expose them to the internet on different apex domains and sub domains.

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

There are many reverse proxy managers available like **nginx proxy manager**, **traefik** etc.

We will use **nginx proxy manager**.

### Installing Nginx Proxy Manager

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

It will show the network name as **<FOLDER_NAME>\_npm**.

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

Now go to the **http://<localhost_or_your_ip>:81** for nginx proxy manager dashboard.

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

- Give any name, **certificate key** will be _<filename>.key_ and **certificate** will be _<filename>.pem_ > Click on Save. It will take some time to generate the certificate.

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
