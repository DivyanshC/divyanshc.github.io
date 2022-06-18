---
title: Adding authentication to services using authelia, traefik
date: 2022-06-16 16:18:00 +0530
categories: [homelab, domain, cloudflare, traefik, authelia]
tags: [homelab, domain, cloudflare, traefik, authelia, sso, totp] # tag names should always be lowercase
---

# Adding Single Sign On and 2 Factor Auth (TOTP) using Authelia middleware for traefik

## Traefik

- You can check this [section]({% link _posts/2022-06-04-connect-services-to-internet.md %}#3-installing-traefik) on how to install **traefik**.

## Authelia

- Add a DNS record for _auth.example.com_ pointing to the server running **traefik**.

- This is how your directory for authelia will look like:

  ```
  authelia/
  ├── config
  │   ├── configuration.yml
  │   ├── db.sqlite3
  │   ├── notification.txt
  │   └── users_database.yml
  └── docker-compose.yml

  1 directory, 5 files
  ```

- Use _docker-compose.yml_ as mentioned below. (Make a directory named authelia and copy the file to that directory)

```yaml
version: "3"

services:
  authelia:
    image: authelia/authelia
    container_name: authelia
    volumes:
      - ./config:/config
    networks:
      - traefik_default
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.authelia.rule=Host(`auth.example.com`)"
      - "traefik.http.routers.authelia.entrypoints=websecure"
      - "traefik.http.routers.authelia.tls=true"
      - "traefik.http.middlewares.authelia.forwardauth.address=http://authelia:9091/api/verify?rd=https://auth.example.com"
      - "traefik.http.middlewares.authelia.forwardauth.trustForwardHeader=true"
      - "traefik.http.middlewares.authelia.forwardauth.authResponseHeaders=Remote-User,Remote-Groups,Remote-Name,Remote-Email"
    # expose:
    #   - 9091
    restart: unless-stopped
    environment:
      - TZ=Asia/Kolkata
    healthcheck:
      disable: true
networks:
  traefik_default:
    external: true
```

- Create a config folder inside authelia main folder.

- Add the below yaml to configuration.yml file in config folder.

```yaml
---
###############################################################
#                   Authelia configuration                    #
###############################################################

server:
  host: 0.0.0.0
  port: 9091
log:
  level: debug
theme: dark
# This secret can also be set using the env variables AUTHELIA_JWT_SECRET_FILE
jwt_secret: a_long_character_string_like_this_with_any_number_and_characters
default_redirection_url: https://auth.example.com
totp:
  issuer: authelia.com

# duo_api:
#  hostname: api-123456789.example.com
#  integration_key: ABCDEF
#  # This secret can also be set using the env variables AUTHELIA_DUO_API_SECRET_KEY_FILE
#  secret_key: 1234567890abcdefghifjkl

authentication_backend:
  file:
    path: /config/users_database.yml
    password:
      algorithm: argon2id
      iterations: 1
      salt_length: 16
      parallelism: 8
      memory: 64

access_control:
  default_policy: deny
  rules:
    # Rules applied to everyone
    - domain: a.example.com
      policy: bypass
    - domain:
        - example.com
      subject:
        - "user:<Username>"
        - "user:<Username>"
        #- "group:<Group_Name>"
        #- "group:<Group_Name>"
        # [,] this is AND above one is OR
      policy: one_factor
    - domain:
        - b.example.com
        - c.example.com
      subject:
        - "user:<Username>"
      policy: two_factor
    # - domain: pve1.local.example.com
    #   policy: two_factor

session:
  name: authelia_session
  # This secret can also be set using the env variables AUTHELIA_SESSION_SECRET_FILE
  secret: unsecure_session_secret
  expiration: 3600 # 1 hour
  inactivity: 300 # 5 minutes
  domain: example.com # Should match whatever your root protected domain is

  # redis:
  #   host: redis
  #   port: 6379
  #   # This secret can also be set using the env variables AUTHELIA_SESSION_REDIS_PASSWORD_FILE
  #   # password: authelia

regulation:
  max_retries: 3
  find_time: 120
  ban_time: 300

storage:
  encryption_key: a_long_character_string_like_this_with_any_number_and_characters # Now required
  local:
    path: /config/db.sqlite3

notifier:
  #smtp:
  #  username: name
  #  # This secret can also be set using the env variables AUTHELIA_NOTIFIER_SMTP_PASSWORD_FILE
  #  password: password
  #  host: smtp.gmail.com
  #  port: 25
  #  sender: authelia@example.com
  filesystem:
    filename: /config/notification.txt
```

- You will get forgot password, **TOTP** genration link in your email if you have configured an email notifier. If you have not configured an email notifier, you will get a link in your _/config/notification.txt_. Same goes for the other **notifications** like _2 Factor Auth (TOTP)_.

- Add the below yaml to users_database.yml file to config folder.

```yaml
---
###############################################################
#                         Users Database                      #
###############################################################

# This file can be used if you do not have an LDAP set up.

# List of users
users:
  username_1:
    displayname: "name_1"
    # Password is Authelia, generate your own hash using argon2id algorithm
    password: "$argon2id$v=19$m=65536,t=1,p=8$cUI4a0E3L1laYnRDUXl3Lw$ZsdsrdadaoVIaVj8NltA8x4qVOzT+/r5GF62/bT8OuAs"
    email: <email>
      - admins_group
      - dev_group

  username_2:
    displayname: "name_2"
    # Password is Authelia, generate your own hash using argon2id algorithm
    password: "$argon2id$v=19$m=65536,t=1,p=8$cUI4a0E3L1laYnRDUXl3Lw$ZsdsrdadaoVIaVj8NltA8x4qVOzT+/r5GF62/bT8OuAs"
    email: <email>
      - dashboard
```

- Generate the **hashed password** for _users_ using the following command.

```bash
docker run authelia/authelia:latest authelia hash-password 'yourpassword'
```

- Replace the **password** with the _generated_ hash.

## Enable authelia for services

### <u>Docker</u>

- Add the below line to docker-compose.yml file for the docker service you want to get behind authentication.

```yaml
labels:
  - "traefik.http.routers.traefik.middlewares=authelia@docker"
```

### <u>Services outside docker</u>

- Below shown is an example of how to enable authelia for services outside docker. You have to add the below line to your traefik's config file.

```yaml
middlewares:
  #order matters
  authelia:
    forwardAuth:
      address: "http://authelia:9091/api/verify?rd=https://auth.example.com"
```

- Now add this middleware to your service's routers section. Check this [section]({% link _posts/2022-06-04-connect-services-to-internet.md %}#3-installing-traefik) for more details.

- Example:

```yaml
http:
  routers:
    # Define a connection between requests and services
    to-whoami:
      entryPoints:
        - "websecure"
      rule: "Host(`example.com`) && PathPrefix(`/whoami/`)"
      # If the rule matches, applies the middleware
      middlewares:
        - authelia
        - services-user
      tls: {}
      service: whoami
```
