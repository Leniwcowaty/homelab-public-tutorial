# Full homelab config tutorial
## Bitwarden, Nextcloud, Jellyfin, Heimdall and many more hosted on premise

*Version for ARM processors, like RaspberryPi or Oracle Cloud A1 architecture*

### This is also available on my [Gitea Repository](https://gitea.kacper.stream/leniwcowaty/homelab-public-tutorial)

---

This repository contains docker files and instructions needed to run your very own on premise homelab, complete with:
- Bitwarden - password manager
- NextCloud - alternative to Google Drive, complete with Kanban boards, office suite, callendars
- Privatebin - minimalistic and private Pastebin alternative
- Jellyfin - your private Netflix
- Kasm Workspaces - a way to launch virtual machines/applications in your browser
- Gitea - locally hosted Github alternative
- Mailu - fully featured e-mail server, complete with webmail application
- Heimdall - a nice landing page

Also for management it includes:
- Portainer - service to manage Docker containers via WebUI
- Nginx Porxy Manager - reverse proxy and Let's Encrypt SSL certificate provider

All websites are secured with Let's Encrypt SSL certificate, proxied and encrypted, e-mails have DKIM and DMARC enabled, preventing them from going to spam. This however requires a small investment (about 4-5 USD). Everything will be dockerized and as closed as possible, with only essential ports open.

---

**IMPORTANT NOTICE**

Although it's the setup I personally use and am pretty confident with it (SSL secured connections, proxy and no needlessly opened ports) **I DO NOT HOLD ANY RESPONSIBILITY ABOUT YOUR DATA SECURITY**! Everything you do here is done at your own risk!

---

With that out of the way, first we will need suitable machine to run our homelab on. You can do it on your hardware, in this case recommended specs are:

- CPU: Decent CPU with at least 4 cores (something like i5 4th Gen would be suitable)
- RAM: At least 8 GB, the more the better
- Disk: SSD would be best, about 20-30 GB for system + as much as you want for your Nextcloud and Jellyfin storage
- Network: the faster, the better, 1 Gbps cable connection would be ideal
- OS: I recommend Ubuntu Server, but pick your poison

I personally didn't run this on hardware, so the part about static public IP, NAT forwarding and mouting your physical drives for Docker and Nextcloud to use you have to figure yourself. I used Oracle Cloud Always Free Tier instance. Specs I chose for it:

- CPU: Ampere 2 CPU units
- RAM: 12 GB
- Disk: 150 GB boot volume
- Network: 2 Gbps link
- OS: Ubuntu Server 22.04 (latest LTS at the moment of writing this)

With that everything is fast and snappy, uploading files I am able to reach up to 200-300 Mbps upload via Wi-Fi. Also this configuration can run 24/7 and is good for Always Free Tier, and leaves 2 CPU, 12 GB RAM and 50 GB of storage for your other instances.

---

## Prerequisites
0. Basic knowledge about using Linux command line

Without it you won't be able to do this. I will try to hold your hand, but will use a lot of shortcuts and skip obvious things. In places where here or in config files i use <this format>, it means that's something you have to change. For example <your domain> has to be changed to the domain you own.

**NOTE FOR ORACLE CLOUD!**

Before anything you need to configure your VCN and subnet! Without it you won't be able to do anything. I won't be explaining it here, as it's easy, you do it in "Networking" section of your Oracle Cloud dashboard.

1. Updating system

It will prevent you from frustration later:

```bash
apt update && apt upgrade -y
```

Then reboot. Of course, if you're using something else than Ubuntu - figure that out.

2. Changing password

This step is CRUCIAL if you are on Oracle Cloud. You need to set up password for user `ubuntu`. Why is that important? Becouse if you screw something up with ports, firewall or ssh daemon and won't be able to access server via SSH you could still access it via online console. Without it, you would have to delete whole instance and start over. So after connecting with SSH simply run:

```bash
sudo passwd ubuntu
```

Also important - all of things here will be done on root acount, so:

```bash
sudo su
```


3. Preparing firewall

First of all, you will need to pick 4 ports - SSH, Heimdall, Bitwarden and Nextcloud. They need to be outside of well-known TCP ports. For more info go [here](https://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers). Next you have to edit `/etc/iptables/rules.v4` to add these lines:

```bash
-A INPUT -p tcp -m state --state NEW -m tcp --dport <ssh-port> -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 81 -j ACCEPT #temporary
-A INPUT -p tcp -m state --state NEW -m tcp --dport 587 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 993 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 465 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 25 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
```

Then simply:

```bash
iptables-restore < /etc/iptables/rules.v4
```

It's recommended to reboot after. You will also need to open these ports for mask `0.0.0.0/0` in Security List of your subnet in Oracle Cloud dashboard. It's easy, you can do it. If you're using something else, or do it on hardware - keep that in mind, these ports have to be available from outside.

4. Changing SSH port

First order of business is changing default SSH port from 22 to something else. To do this in OS you can either edit `/etc/ssh/sshd_config` file or create new file, like `/etc/ssh/sshd_config.d/new.conf` (name is not important, but it must have `.conf` extension). there you edit/add this line:

```bash
Port <ssh-port>
```

Then you have to restart ssh daemon:

```bash
systemctl restart sshd
```

**VERY IMPORTANT!**

BEFORE YOU DO ANYTHING ELSE, OPEN SECOND SSH CONNECTION WITH THIS NEW PORT AND MAKE SURE YOU CAN CONNECT! IF NOT, SOMETHING'S WRONG WITH EITHER FIREWALL ON INSTANCE OR ON YOUR PROVIDER (ORACLE, SHELLS, LINODE OR YOUR ROUTER). IF YOU CAN'T ACCESS, GO BACK TO PREVIOUS CONSOLE AND DELETE THAT LINE WITH PORT, RESTART DAEMON AND FIGURE OUT WHAT'S WRONG!

If you really f up, and completly lose access to server, you can always use cloud console on Oracle, using password we set in 1st step.

5. Installing Docker

I won't be going into details here, Docker has great instruction on their website, so I will just point you to [THAT](https://docs.docker.com/engine/install/). If you want rootless Docker, add user `ubuntu` to `docker` group.

You will also need to create Docker network. You can name it however you want, also you can specify whatever subnet you want:

```bash
docker network create --driver=bridge --subnet=<subnet CIDR> <subnet name>
```

6. Configuring domain

While we could use something like duckdns.org, this is just a half-solution. We cannot set DKIM or DMARC, no encryption, DDoS protection, and we only rent a duckdns subdomain, so our address is quite long (eg. <service>.<domain>.duckdns.org). Instead it's best to buy a cheap domain with obscure TLD, as they can be quite cheap. I personally use Cloudflare, as it provides full DNS configuration options, end-to-end encryption, proxy by default, DDoS protectio and both buying and renewing domains is cheap. I paid about 6 USD/2 years for my domain. It's not that much, and it's worth added security and convinience.

After you buy your domain enter DNS settings and add A records pointing to your instance's public IP. These are basically subdomains for your services. First you need to set root domain (in Cloudflare by providing @ in subdomain). In Cloudflare you have the option to make it proxied - this essentially forwards all traffic through Cloudflare servers, adding a layer of security and hiding your IP address. Add as many as you need.

If you are planning to create email server, keep in mind to create subdomain for this, that IS NOT proxied. If it's proxied while it will send messages to you, it will not recieve any. Let's call this <mail subdomain>

---

## Installing services

Here comes this repository. Just clone it anywhere on your system. I personally preffer rootless Docker and keeping compose files in `ubuntu` user's home directory under something like `/home/ubuntu/compose`:

```bash
cd /home/ubuntu/compose
git clone <repo-url>
```

Now you have `compose` directory and in that directories for all services

For the rest of tutorial I will be refering to this scheme, where all files are under `/home/ubuntu/compose/`. If your's is diferent remember to change it in compose and env files.

1. Nginx Proxy Manager

[Nginx Proxy Manager GitHub](https://github.com/NginxProxyManager/nginx-proxy-manager)

Let's start with the backbone of our operation. This service will allow us to keep all of our Docker ports closed and connect to our services via HTTPS. This is really straightforward, no configuration needed in docker-compose.yml, except for providing Docker network you created earlier in `network_mode` by replacing <docker network> with your network name. Edit this and run it with:

```bash
docker compose up -d
```

Only after this there comes to some configuration. Connect to your domain to port 81 and login with initial credentials, that you can find on NPM Github. After this you will be asked to create admin account.

Further configuration is quite easy. First you need to generate Let's Encrypt certificate, so go to "SSL Certificates" tab and create one, select Let's Encrypt. For domain type:

```
*.<your domain>
<your domain>
```

The * is crucial, as it generates wildcard cert, to use with main domain and subdomains. Select "Use a DNS Challenge". Select your domain provider (for me it's Cloudflare) and perform needed actions. For Cloudflare you need to create API key to edit DNS for all zones in dashboard and paste it in. Click "Save" and wait a bit. It may fail once or twice, but just try again. You have 5 tries until your domain is blocked from Let's Encrypt for 60 days.

After you get your certificate go to "Hosts > Proxy Hosts" and "Add Proxy Host". Here comes a scheme we will use for the whole process. In "Domain Names" provide your full subdomain, eg. for proxy it can be `npm.<your domain>`. Next "Scheme" http, and in "Forward Hostname" provide NAME of service container, in this case "nginx-proxy-app". "Port" should be internal port of container - 81 in this case. You can check both name and ports container uses with `docker ps` command. It's recommended to enable Websockets Support just in case.

This works becouse when containers are in the same Docker network, it creates a simple DNS resolver, and treats container names as hostnames. So Docker automatically translates it to IP that was given for this container internally (these IPs can change). 

Next go to SSL tab in Proxy Host creation window and select certificate you just created. Tick `Force HTTPS` and `HTTP/2 Support`. Click Save.

After you did that (remember to also add the same subdomain as A record in your domain) you can go back to Firewall section of this tutorial and delete line containing port 81. We won't need it anymore. Apply iptables rules, reboot and you can now connect to your proxy through https://<npm subdomain>.<your domain>.

2. Heimdall

[Heimdall GitHub](https://github.com/linuxserver/Heimdall)

Heimdall is also a "quality of life" type of thing. It provides us with customizable dashboard, where we can pin our applications, like a starting page of a browser, for many, many services, that you would want to use in the future. Added bonus above typical starting page (becouse you can also just pin your services addresses to your Chrome/Firefox starting page) is that it integrates with services. For example, for Nextcloud it can show available or used space, right there on the dashboard.

Same as Nginx Proxy Manager, no configuration is needed, except for providing Docker network in `network_mode` (replace <docker network> with your network name), and maybe changing timezone. Edit this, run with `docker compose` and you're up.

Now add Proxy Host to Nginx Proxy Manager (and A record if you didn't already) using container name, just like in previous step. It should be `heimdall-heimdall-1` with port 80, but verify this with `docker ps`. Add SSL certificate to the mix and your landing page should now work.

3. Portainer

[Portainer GitHub](https://github.com/portainer/portainer)

This is extremally useful tool to graphically manage your Docker, check logs, add containers to networks etc.

Again - absolutely no changes in docker-compose.yml needed except for changing network name in `network_mode` (again replace <docker network> with your network name). Edit this, run with `docker compose` and that's done. 

Next - standard procedure of adding Proxy Host to NPM and A record to domain. Nothing out of ordinary here.

4. Mailu

[Mailu GitHub](https://github.com/Mailu/Mailu)

Mailu is in my opinion the best full-featured email server. It comes with WebUI admin panel as well as Webmail service you can access from your browser. With DKIM and DMARC it's really great and you can even use it as your own mail address. We will however only use it as automated mailing server for our services (like forgotten password or 2FA authentication).

I really, really, really strongly recommend you [read the docs](https://mailu.io/2.0/) as they are filled to the brim with information, useful features and additional configuration parameters. What I have here is absolute minimum and you should set up your Mailu server to your liking by using their [Docker Compose setup utility](https://setup.mailu.io/). But if you don't want to use that, here's a quick rundown what you have to change in files I provided in this repo.

Absolute first order of business, if you're using Oracle Cloud is to get SMTP credentials and create email domain. This is becouse Oracle is blocking any outgoing traffic to port 25, which means you are not able to just send an email from your VPS. You need a relay server. To mitigate this, Oracle provides a free email relay service, limited to 200 emails per 24 hours. You have to go to your account options, scroll to `SMTP Credentials` and generate one. Next, to go `Email Delivery` in OCI dashboard and create an Email Domain. Provide <your domain> and click okay.

One last thing for Oracle - it needs to have senders be approved. This is done simply by going to your Email Domain in OCI dashboard, going to `Approved Senders` tab. Add at least `admin@<yourdomain>` and click okay.

If you're using different VPS provider - sorry, you're on your own.

Now to configuration. First `docker-compose.yml`. Here you only have to change two things. One - in `front` service in `networks` section FOR THE SERVICE replace <docker network> with your network name. Next scroll to the bottom and in GLOBAL `networks` section replace <docker network> with your network name.

Second the `mailu.env` file. Here we have A LOT to change, so I will explain it really quickly. Comments in the file should help you.

- `SECRET_KEY=` set to randomly generated 16 characters string (only capital letters and numbers are allowed)
- `DOMAIN=` set this to <your domain>
- `HOSTNAMES=` remember when I told you to create one A record, that was NOT proxied? Paste it here as <mail subdomain>.<yourdomain>
- `RELAYHOST=` here you paste relay host corresponding to your region in Oracle, these can be found [HERE](https://docs.oracle.com/en-us/iaas/Content/Email/Reference/gettingstarted_topic-Configure_the_SMTP_connection.htm)
- `RELAYUSER=` here you paste user you got when generating SMTP Credentials in Oracle
- `RELAYPASSWORD=` password for said user, you also get this when generating SMTP Credentials
- `SITENAME=` set this to anything, how you want your welcome page to be named
- `WEBSITE=` set this to subdomain you intent to use as your webmail UI `https://<subdomain>.<domain>
- `TZ=` set this to your [time zone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)

Ufff, that was a lot. Now run it. Because Mailu is creating many containers, you need to specify the project, so it will be nicely organized in Portainer:

```bash
docker compose -p mailu up -d
``` 

This will take some time, but after that you can go ahead and add it to Nginx Proxy Manager. Two things to remember here:

- use the same <subdomain> as you set in `mailu.env` file
- point to `mailu-front-1` container, port 80 (you can verify port with `docker ps`)

One last step is to create admin user. Simply run:

```bash
docker compose exec admin flask mailu admin me <your domain> <admin password>
```

Now you should be able to access both webmail and admin panels (under `/webmail` and `/admin` respectively) and log in with `admin@<yourdomain>`. From admin panel you can create separate users/email addresses for Nextcloud, Bitwarden, Gitea, or you can just use admin address everywhere.

5. Nextcloud

[Nextcloud AIO GitHub](https://github.com/nextcloud/all-in-one)

Nextcloud is too big to explain here, so just read about it [HERE](https://nextcloud.com/). We will be using Nextcloud AIO image, which provides us with easy and fast process of setting everything. If you want to use full manual version, find it on [GitHub](https://github.com/nextcloud/server). And good luck.

Only config we have to make is change port on which Apache container serving our actual drive will be run. To change this we need to open `docker-compose.yml` and replace `<nextcloud-port>` in enviroments of Docker file to whatever port you want (Nextcloud recommends 11000, but choice is yours).

```bash
environment:
	- APACHE_PORT=<nextcloud-port>
```

Before you're able to access the AIO panel, you need to add it to Nginx Proxy Manager. Host `nextcloud-aio-mastercontainer`, port 8080. IMPORTANT - here you set `Scheme` to https.

Important notice - after you use AIO interface to deploy all containers, you HAVE TO add `nextcloud-aio-apache` to your docker network! Without that Nginx Proxy Manager will not be able to resolve its name and connect to it. This can be done via command line, but I prefer to do it via Portainer. So `nextcloud-aio-apache` should be in two networks - the one Nextcloud created by itself and your docker network.

Add `nextcloud-aio-apache` to Nginx Proxy Manager as Proxy Host (and A record for domain) with port you specified above. Again, this can be checked with `docker ps`.

6. Bitwarden

[Bitwarden GitHub](https://github.com/bitwarden/server)

Bitwarden is also big, so read about it [HERE](https://bitwarden.com/). We'll be using docker-unified method, since it's the only one that works both on x86_64 and ARM processors. Important note here - docker-unified is still in beta, so minor bugs can occur. Should be good, but no guarantee. If you're using x86_64 architecture, you probably should use "classic" method.

Here we have a but things to configure. First let's hop to `docker-compose.yml`. We will be using MariaDB database for our passwords, with password protected user and random root password. We need to edit several lines:

- first add your docker network to `network_mode` for both bitwarden and database services

- next change `<db-password>` to strong, complicated, random password. Don't worry, you won't have to remember it, just paste it in other config file, so I recommend 32 characters random generated password

```bash
MARIADB_PASSWORD: "<db-password>"
```

Write, quit. Open env file - `settings.env` and change following lines:

```bash
BW_DOMAIN=<bitwarden-domain>
.
.
.
BW_DB_PASSWORD=<db-password>
```

I'm pretty sure you know what to put there. Next you need to go to https://bitwarden.com/host/, provide your e-mail address and get an installation ID and installation key. These are random and e-mail connected. Why they require those? Read the docs ;)

After you have ID and key, you need to put them in `settings.env` file:

```bash
BW_INSTALLATION_ID=<id>
BW_INSTALLATION_KEY=<key>
```

Last step is to configure SMTP e-mail. If you don't want to create your own email server following this tutorial, you can use your private mail, like GMail, Yahoo or whatever really. You just need to know a few configuration variables. This is example for GMail:

- SMTP Host: smtp.gmail.com
- SMTP Port: 587
- SMTP Username: your username
- SMTP Password: your password (if using 2FA you need to go to your Google Account settings and generate an Application Password)

Just paste these parameters to your `settings.env`. However, if you did create your Mailu service, it should look like this:

```bash
globalSettings__mail__replyToEmail=<account>@<your domain>
globalSettings__mail__smtp__host=<mail subdomain>.<your domain>
globalSettings__mail__smtp__port=465
globalSettings__mail__smtp__startTls=true
globalSettings__mail__smtp__ssl=true
globalSettings__mail__smtp__username=<account>@<your domain>
globalSettings__mail__smtp__password=<password>
```

You know what comes next - adding this to Nginx Proxy Manager. Host `bitwarden-bitwarden-1`, port 8080. Verify with `docker ps`.

7. Privatebin

[Pastebin GitHub](https://github.com/PrivateBin/docker-nginx-fpm-alpine)

Privatebin is a nice, tiny alternative to Pastebin. You can set files to expire after several days, or right after reading. I use it a lot at work, to send quick snippets, passwords etc in office. 

We will although use [Nginx Unit + Alpine](https://github.com/PrivateBin/docker-unit-alpine) image due to its smaller footprint.

Here for a change we don't need any configuration, except for standard changing network name in `network_mode` (again replace <docker network> with your network name). After you start it with `docker compose` you also need to change permissions of `data` directory this created:

```bash

sudo chown -R 65534:82 /home/ubuntu/compose/privatebin/data
```

Why these specific owner? Don't ask me, ask project creators. Add it to Nginx Proxy Manager as per usual, checking container name and port with `docker ps` and you're off to the races.

8. Jellyfin

[Jellyfin GitHub](https://github.com/jellyfin/jellyfin)

Jellyfin is alternative to Plex and essentially a private Netflix. It takes in your **legally owned** copies of movies, tv series, music etc, displays them in nice folders, downloads thumbnails, descriptions, cast, reviews, even subtitles. It can also track which episode of series you finished on, where did you stop and really a lot of nice features.

This is yet another service, where except for - you guessed it - replacing <docker network> with your network name you don't need to do anything. Run it with `docker compose`, add to Nginx Proxy Manager and that's all.

9. Gitea

[Gitea Gitea](https://gitea.com/gitea)

This is just a nice, small git service you can use to host your own projects, or version stuff like dotfiles for your Linux machines. I keep here for example this repo. Here again, nothing to configure, except managing network. However, similarly to Mailu, you look for section `networks` in `gitea` service and replace <docker network> with your network. Then on the top of the file this time, in global `networks` section you do the same.

Run with `docker compose`, add to Nginx Proxy Manager, Bob's your uncle, done.

10. Kasm Workspaces

[Kasm GitHub](https://github.com/kasmtech/workspaces-images)

Kasm is really nice, but very specific software. It essentially nests itself and creates docker images with GUI applications in your browser. This ranges from Discord, Firefox, Steam, Audacity, LibreOffice to fully-featured operating systems, like Ubuntu, Fedora, Kali, Rocky. It has its uses, for example its safer to connect to your bank account via Kasm Firefox, than normally when on public hotspot - the bank session cannot be hijacked. 

In terms of installation, Kasm is a bit finnicky, first it spits out a dashboard, that you have to connect to, accept license agreement, and then create passwords for both admin and first user. In terms of configuration you don't need to do anything, just replace <docker network> with your network name, run with `docker compose`.

As for Nginx Proxy Manager, I recommend creating two entries, like in Nextcloud. One for dashboard, pointing to port 3000, second for application itself on port 443. Note, that you need to set `Scheme` to https, just like in Nextcloud AIO mastercontainer.

---

## Starting and managing Docker services

First of all - useful commands:

```bash
docker compose up -d #start Docker compose file
docker compose down #stop Docker compose file
docker ps #list running containers
docker system prune -a #clean up - remove everything not connected to running containers INCLUDING stopped containers
docker volume prune #clean up Docker volumes not connected to any running containers
```

Docker compose must be always run from directory where `docker-compose.yml` is located. You can change any setting and play with it, but after every change you have to stop container with `docker compose down` and start it back again.

---

## Further configuraion

Almost every service here allows you to just start and go, or has a really nice introduction wizard, that you can follow with minimal to no brain cells active. Some require additional configuration, and I will describe it here.

### Services that run "out of the box":
- Heimdall (only needs you to add links to services)
- Nginx Proxy Manager (no configuration needed except the ones described in it's section in this tutorial)
- Mailu (you just login and go)
- Bitwarden (create account and go)
- Portainer (just create password for admin user when you first open it)
- Privatebin (open and use)

### Services that need minimal post-install configuration:
- Kasm Workspaces - after you open dashboard you have to accept license terms, then create passwords for admin and user. Please note, that Kasm is very picky about passwords, so it's best NOT to use any special characters. After that you click Launch and it installs and configures itself. From there you open your main subdomain pointing to port 443 and you're good.
- Jellyfin - it has REALLY NICE first-time wizard, where you configure everything to your liking, create your first library and everything. I can also recommend going to `Settings` > `Plugins`, installing "OpenSubtitles" plugin and logging in, so Jellyfin can download subtitles automatically. It's not reqiured, but recommended.

### Services that require larger post-install configuration
- Nextcloud - "larger" is not the right word, you have to go to Personal Settings, set email for your account, 2FA if you want etc. After that you want to go to `Administration Settings` > `Basic Settings` and set up Email server. You do this exactly how you configured it in Bitwarden env file:

Property | Value
---|---
Encryption | None/STARTTLS
From address | \<account>@\<your domain>
Server address | \<mail subdomain>.\<your domain>:465
Authentication | &#x2611;
Credentials | \<email username> \<email password>

You can test it by clicking `Send email`, it will send a test message to mail you set in your Nextcloud account.

- Gitea - here you have a really nice wizard, that will mostly fill for you by itself, but you want to edit few things. First select `Email Settings` and fill out exactly like you did above for Nextcloud:

Property | Value
---|---
SMTP Host | \<mail subdomain>.\<your domain>
SMTP Port | 465
Send Email As | \<account>@\<your domain>
SMTP Username | \<email username>
SMTP Password | \<email password>

You can select to require email confirmation, enable email notifications. Below that you have `Server and Third-Party Service Settings`, you can configure them as you wish. Next you have two `Install Gitea` buttons. Select the one at the bottom, as it saves your configuration to file, so you can change email server withour redoing whole service.

---

## Final product

If you done everything correctly (and I didn't mess up in this tutorial) now you should be able to just open your browser, type `https://<subdomain>.<your domain>` and go to your desired service. All that was left to do is add your apps to Heimdall main dashboard, upload some files to Nextcloud, paste some code to Gitea and upload some movies to Jellyfin.

One final request - please don't create Issues asking me to help you with setting up. Better go to each project's respective GitHub repo and ask there, you will get answer from software creators. I am just some guy, who after 2 weeks managed to make this and wanted to share with you.

---

## DKIM and DMARC

In order to have your emails delivered to you and not get trapped in SPAM folder you need to have them secured and verified. First step is to encrypt them via SSL, which we done already. Two next steps are adding DKIM and DMARC.

DMARC is quite simple, you just need to head to your domain provider DNS settings (I use Cloudflare) and add a TXT new record.

Name | Value
---|---
_dmarc.\<your domain> | v=DMARC1; p=none; rua=mailto:admin@\<your domain>

As for DKIM, it can be quite tricky, and you need to refer to your relay/VPS provider. As stated before - Oracle provides you with Email Relay. When you go to this section and select your `Email Domain`. There you can see status of your DKIM - it should be non present. Then you just click `Add DKIM`. Fill your `DKIM Selector` - the convention is to use format like `<name>-<country>-<date as YYMMDD>`. Click `Generate DKIM Record` and you will get two values - `CNAME Record` and `CNAME Value`. As you can imagine - go to your domain provider DNS settings and fill those in, adding CNAME.

**Important!** Both DKIM and DMARC records HAVE TO be DNS Only (not proxied) in Cloudflare. Without it this won't work. After that to verify you can use something like [mail-tester](https://www.mail-tester.com/) to verify if DKIM and DMARC are working. As long as your score is above 6.5 your emails shouldn't go to SPAM.

---

## ALL USED CODE IS OWNED BY ITS RESPECTIVE CREATOR. I DIDN'T MAKE, CREATE ANY OF SOFTWARE USED HERE. LINKS TO EACH PROJECT'S RESPECTIVE GITHUB REPO IS PROVIDED.
