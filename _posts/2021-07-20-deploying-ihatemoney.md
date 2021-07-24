---
layout: post
title: Deploying IHateMoney
categories: self-hosted
date: 2021-07-20 08:00:00
---

IHateMoney is a web application made to ease shared budget management. It keeps track of
who bought what, when, and for whom; and helps to settle the bills. Its very to easy to 
deploy in a self-hosting setup and share with friends and family.

### Build the docker image

```bash
### Create a working directory for the app.
$ mkdir ~/ihatemoney
$ cd ~/ihatemoney
$ git clone https://github.com/spiral-project/ihatemoney src
$ cd src
### sudo might be required depending on the context
$ podman build -t github.com/spiral-project/ihatemoney --format docker .
# OR
$ docker build -t github.com/spiral-project/ihatemoney .
```

### Generate Hashed Admin Password
```bash
### Generate a random string and save it.
$ openssl rand -base64 15
$ sudo podman run -it --rm --entrypoint /usr/local/bin/ihatemoney \
    github.com/spiral-project/ihatemoney:latest generate_password_hash
### When prompted for a password enter the string generated above
### Save the hashed admin password response from the command
```

### Create .env file
```bash
$ cd ~/ihatemoney
### Generate a pair of secrets 
$ vim podman.env
```
`podman.env`
```ini
# openssl rand -base64 15
SECRET_KEY=generated_password
# Configure the MAIL* environment variables for your SMTP Relay
MAIL_SERVER=smtp.gmail.com
MAIL_DEFAULT_SENDER=default-sender@mydomain.com
MAIL_PORT=587
MAIL_USERNAME=smtp_user
MAIL_PASSWORD=smtp_password
MAIL_USE_TLS="True"

ACTIVATE_DEMO_PROJECT="False"
ALLOW_PUBLIC_PROJECT_CREATION="True"
# Pasted the hashed admin password generated in the previous step
ADMIN_PASSWORD=generated_password
ACTIVATE_ADMIN_DASHBOARD="True"
# Valid Timzones can be found in /usr/share/zoneinfo
BABEL_DEFAULT_TIMEZONE="America/Los_Angeles"
```

### Test the container
```bash
$ cd ~/ihatemoney
$ touch ihatemoney.db
$ sudo podman run -it --rm -p 8000:8000 \
       -v ./ihatemoney.db:/database/ihatemoney.db \
       --env-file ./podman.env github.com/spiral-project/ihatemoney:latest
```

If everything worked then the app can be accessed on http://&lt;host-ip&gt;:8000.

### Create docker-compose file
Still in the same working directory, add this `docker-compose.yml` file. 

`docker-compose.yml`

```yaml
version: "3.3"
services:
  ihatemoney:
    image: github.com/spiral-project/ihatemoney:latest
    container_name: ihatemoney
    restart: always
    volumes:
      - ./ihatemoney.db:/database/ihatemoney.db
    env_file:
      - podman.env
    networks:
      - net-caddy

networks:
  net-caddy:
    external: true
```

The network `net-caddy` is a container network that I use to keep all containers that
have a UI that I want to access via a reverse proxy. Replace `net-caddy` in the file
to point to the right network for your setup.

### Start the service
Bring up the container.
```bash
$ docker-compose up -d
```


### Setup Reverse-proxy
I want the external subdomain `expenses.mydomain.com` to point to the app. First update DNS 
records for `mydomain.com` to point the `expenses` subdomain to the server's IP hosting the app.
If you are self-hosting in a home network, you would point the subdomain to the public IP
assigned by your ISP to your home connection. 

I already have Caddy running in a docker container to reverse proxy other services. Add this to the Caddyfile

```
expenses.mydomain.com {
  log {
    level INFO
    output file /data/expenses.access.log {
      roll_size 10MB
      roll_keep 10
    }
    format single_field common_log
  }

  tls {$EMAIL}
  encode gzip
  reverse_proxy ihatemoney:8000
}
```
