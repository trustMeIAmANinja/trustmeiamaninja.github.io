---
layout: post
title: Deploying Sish
categories: self-hosted
date: 2021-07-23 08:00:00
---

[Sish](https://github.com/antoniomika/sish) is a great `ngrok` alternative to expose any locally hosted app over the web. With Sish you create temporary secure tunnels to locally hosted applications and allow others to intereact with the app with a publicly accessible web address. Its great to quickly expose a development app server without needing to deploy it and setup reverse proxies or mess around with firewalls. Sish can also be used to expose any TCP connection - e.g MySQL DB.

`sish` uses `ssh` and its port-forwarding features to establish the required tunnels to expose the internal app. This is a big difference from `ngrok` where you don't need a dedicated client to make the tunnel work. 

## Requirements
1. A registered domain.
   *I will be using mysish.co for my deployment.*
1. A DNS Hosting service thats support wildcard subdomain.
   *I will be using cloudflare*
1. API Access to the DNS records to automate the generation of LetsEncrypt SSL Certs.
1. A Linux VPS or some kind of cloud instance with a public IP address.
   * `docker` and `docker-compose` installed on the linux server.
   * Inbound access to port 443 and one port for SSH (typically 22 but any other port will work)


## Setup

`ssh` to the VPS or cloud host where you plan to deploy `sish`

### Create Working directory
```bash
$ mkdir ~/sish
$ cd ~/sish
```
All other commands below will be relative to this working directory.

### Checkout the Sish Git 
```bash
$ git clone https://github.com/antoniomika/sish src
```
This will create a new directory `src` under our working directory `~/sish`

### Prepare working directory
```bash
$ mkdir dnsrobocert keys letsencrypt log pubkeys ssl
$ cp src/deploy/docker-compose.yml .
$ cp src/deploy/le-config.yml dnsrobocert/dnsrobocert-config.yml
```

### Dnsrobocert setup
[DnsRobocert](https://github.com/adferrand/dnsrobocert) is a dockeried app to request and manage LetsEncrypt SSL certs using the DNS Challenge method. For our use-case we will use DnsRobocert to get wild-card certs for the domain `mysish.co`. 

*NOTE: Sish has support for getting certs from LetsEncrypt, but it requests certs everytime someone uses the service to setup a tunnel with a new subdomain. Depending on how heavy your deployment gets used, you may run into LetsEncrypt quota limits with this method. So its better to get wild-card certs for your domain and not have to go back to LetsEncrypt for every new sub-domain*

Since we are using Cloudflare for DNS, we first need to generate an API Token that has Zone Edit access for our domain. Follow the [Cloudflare Docs](https://developers.cloudflare.com/api/tokens/create) to create the token using the "Edit Zone DNS" template.

Edit the file `dnsrobocert/dnsrobocert-config.yml` to look like this
```yaml
acme:
  email_account: no-reply@mysish.co
  staging: false
certificates:
- autorestart:
  - containers:
    - sish
  domains:
  - mysish.co
  - '*.mysish.co'
  name: mysish.co
  profile: cloudflare
profiles:
- name: cloudflare
  provider: cloudflare
  provider_options:
    auth_token: <CLOUDFLARE API TOKEN>
```
The email address `no-reply@mysish.co` doesn't necessarily have to be a valid email address.

### Create cert symlinks
This part is important to ensure that `sish` can find the right certs generated by `dnsrobocert`. 

```bash
$ ln -sf /etc/letsencrypt/live/mysish.co/fullchain.pem ssl/mysish.co.crt
$ ln -sf /etc/letsencrypt/live/mysish.co/privkey.pem ssl/mysish.co.key
```
The symlinks will not make sense on the host filesystem. But our `docker-compose.yml` will mount certain folders inside each container and this will resolve correctly inside the container.

### Add your initial SSH public key
We will be setting up `sish` to only use public key authentication. The app automatically loads any public keys it finds in a designated `pubkeys` folder. To start off copy your ssh public key to the pubkey folder

```bash
$ cp /path/to/ssh_key.pub pubkeys
### OR you can get pubkeys from github if you have added them there
### Replace antoniomika with the github username you want to give access to
$ curl https://github.com/antoniomika.keys > pubkeys/antoniomika
```

`sish` doesn't really care about the username you use for the `ssh` auth. It only authenticates the ssh key. At any time you can add additional ssh keys to `pubkeys` folder and `sish` will automatically load them.

### Setup DNS Records
* Add a DNS A record for `mysish.co` to point to the public IP address of your VPS or cloud instance.
* Add a DNS A record for `*.mysish.co` to point to the same IP address.

### Update `docker-compose.yml`

For `sish` to work it needs to be able to listen on three ports for the three services
* http: typically port 80, but can be changed to any other port. Its best to block inbound access to this port at the firewall and only use https.
* https: typically port 443, but can be changed.
* ssh: typically port 22. But since the host `ssh` daemon is already using this we will have to use an alternative port. In our case will be using 2222.

For docker apps, there are typically two ways for a container to receive outside traffic on a port
* Port forwarding from the host to the container using docker args
* Run the container in host network mode and have it directly bind to the ports on the host.

For this deployment I went with the latter since this is the only service I plan to run on this host.

Update `docker-compose.yml` to look like this. 
```yaml
version: '3.7'

services:
  letsencrypt:
    image: adferrand/dnsrobocert:latest
    container_name: dnsrobocert
    volumes:
      - ./letsencrypt:/etc/letsencrypt
      - ./dnsrobocert/dnsrobocert-config.yml:/etc/dnsrobocert/config.yml
    restart: always

  sish:
    image: antoniomika/sish:main
    container_name: sish
    depends_on:
      - letsencrypt
    volumes:
      - ./letsencrypt:/etc/letsencrypt
      - ./log:/var/log/sish
      - ./pubkeys:/pubkeys
      - ./keys:/keys
      - ./ssl:/ssl
    command: |
      --ssh-address=:2222
      --http-address=:80
      --https-address=:443
      --https=true
      --https-certificate-directory=/ssl
      --log-to-file
      --log-to-file-path=/var/log/sish/sish.log
      --log-to-file-max-size=200
      --authentication=true
      --authentication-keys-directory=/pubkeys
      --private-key-location=/keys/ssh_key
      --bind-random-ports=false
      --bind-random-subdomains=false
      --service-console
      --domain=mysish.co
    network_mode: host
    restart: always
```

### Start it up
```bash
$ docker-compose up -d
```

To check the logs from the app
```
$ tail -f log/sish.log
```

### Test it out.
Lets says you have a simple python http server running on port 4000 on your laptop that would like to give access to someone. First ensure that you have the private key corresponding to one of the public keys published in the `pubkeys` folder on the `sish` host.

On your laptop where you have the development app server running
```
$ ssh -p 2222 -R hostit:80:localhost:4000 mysish.co

```

The important part is this `-R hostit:80:localhost:4000`. Lets break this down:
- `-R` is telling ssh to create a reverse port forward over the ssh connection
- `hostit` is the subdomain we are requesting. 
- `80` indicates that the app on the laptop is using http and not https. This has nothing to do with the final public address that `sish` will provide. That can still be accessed over https.
- `localhost` is pointing to your laptop. This can also be pointing to some other host thats
reachable from your laptop. For e.g. you may have a RaspberryPi running the http server at `192.168.10.23:4000` and this can be accessed from your laptop. To have `https://hostit.mysish.so` map to this, change `localhost` to `192.168.10.23`
- `4000` is the port on which the local http server is listening on.

If everything was setup correctly the ssh command will spit out something like this
```text

Press Ctrl-C to close the session.

Starting SSH Forwarding service for https:443. Forwarded connections can be accessed via the following methods:
Service console can be accessed here: https://hostit.mysish.co/_sish/console?x-authorization=akdjfasdFHJh34785
HTTP: http://hostit.mysish.co
HTTPS: https://hostit.mysish.co
```

Now anyone can access https://hostit.mysish.co and to the local http server.


### Fail2ban
TODO: Custome `fail2ban` filter to ban bots and scanners

## Closing Thoughts
I deployed `sish` mostly as a fun challenge to see if it can be done. If you use `ngrok` a lot then maybe running `sish` might be worth your time. 
Another option that be used the same result is Cloudflare's Argo Tunnel.
