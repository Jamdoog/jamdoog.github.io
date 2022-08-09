---
title: Taking advantage of free S3 compatible storage on a £0.30 VPS
slug: scaleway-s3-stardust
date_published: 2021-09-30T22:50:46.000Z
date_updated: 2021-09-30T22:52:58.000Z
tags: Container, Docker, Linux, Networking, Web
excerpt: Taking a look (and taking advantage of) Scaleway's cheap compute and storage offerings
---

---

### This post is not meant to be used in a production workflow. I just wanted to show how you can host some data for practically free while being under your OWN terms

---

I have a problem with renting too many virtual servers purely because they are cheap and after recently discovering [Scaleway's offer of a 1GB VPS for £0.30 a month](https://www.scaleway.com/en/stardust-instances/), I had to find a use for some of the idling compute. 

These "Stardust" instances are little KVM boxes with a 100M port and the optional ability to include IPv4 at an extra 1 euro or so. Having previously used (and still do) their [dedicated server offerings](https://www.scaleway.com/en/dedibox/), it was incredible deal that I haven't seen mentioned much. IPv6 adoption may still be a dream ~ but for now, it's good enough.
![A wall of text from Scaleway's page on Stardust](__GHOST_URL__/content/images/2021/09/stardust-1.png)
Cheap enough as it is, but they don't tell you about removing the "Flexible IP" option. Pay hourly for a floating IP? Awesome.
![Image of billing breakdown depicting £0.37/m for a 1GB VPS](__GHOST_URL__/content/images/2021/09/stardustipv6-1.png)Seriously, that's a monthly price. For 1GB of ram!
The thing which really caught my eye though was the free object storage. A whole 75GB! Not only can you host HTTP sites from them, but you can interact with them completely free via this instance. If it was a 1 gigabit port I wouldn't want to imagine the abuse this would cause.
![](__GHOST_URL__/content/images/2021/09/75gb.png)
---

## Penalties of going IPv6 only

Of course without proper adoption, IPv6 presents natural challenges. For instance, I couldn't download docker-compose because it's stored on Github. Seriously? 

Translation mechanisms such as DNS64 exist but they're not good enough. In this case, the best bet is to rent a floating IP for an hour if needed while configuring and later remove it once done. Save the headache, please...

One of the things I found really interesting about Scaleway's approach to floating IP's was that they NAT your traffic. That's right, not hacky scripts to mess with. Your traffic just ✨ works ✨ (NAT's a good thing???)

Anyway, the main thing I was looking for was that Scaleway would support IPv6 on their S3-compatible object storage which they did! But now the next problem was Docker. Incase you aren't aware (blissful ignorance), docker was not built up on the foundation of IPv6 and thus has lacking support. Infact, people conciously chose to NAT IPv6, me included. I'd say I'm sorry but well, I'm not. 

[Wido den Hollander wrote a blog post about a makeshift IPv6 NAT solution](https://blog.widodh.nl/2017/04/docker-containers-with-ipv6-behind-nat/) with ip6tables. Quite frankly, this was the fastest and easiest method of achieving this. The headache of using a real address was not something I was prepared to do. 

---

## Installing Docker 

Installing docker is made pain free by a quick bash script they provide. This is perhaps the lazy way but I've yet to find it has failed me.

`curl https://get.docker.com | bash`

That's it.

---

## Configuring Docker

The method to NAT IPv6 I chose meant that it wouldn't support docker-compose without some work. Quite honestly after debugging this for maybe 4 hours it was already more time then I was wanting to spend on this science experiment. As such, I created manual docker files. But before this I had to configure docker.

As a quick note, this "tutorial" is taking place on Debian 11. I can't speak for Podman on RHEL based distro's but I would imagine it's a similar ordeal?

There's two things we need to do, modify the docker startup parameters and create a ip6tables rules to NAT the traffic. 

To begin with, I modified `/usr/lib/systemd/system/docker.service` and appended the following flags to `ExecStart`:

`--ipv6 --fixed-cidr-v6="fd00::/64"`

The whole command then looked like:

`ExecStart=/usr/bin/dockerd --ipv6 --fixed-cidr-v6="fd00::/64" -H fd:// --containerd=/run/containerd/containerd.sock`

After this, we need to make systemd see the change:

`systemctl daemon-reload`

Then finally we can restart the docker service:

`systemctl restart docker`

Next, we need to just create the NAT rules on ip6tables:

`ip6tables -t nat -A POSTROUTING -s fd00::/64 -j MASQUERADE`

> Again, this should **ONLY** be used for testing purposes. For production [IPv6 Prefix Delegation](https://blog.widodh.nl/2016/03/docker-and-ipv6-prefix-delegation/) is *the* route to go down.

Wido himself mentions that this is not for production. I recommend you read his mentioned post on prefix delegation if you was to use this for cirtical applications. 

---

## Dockerfiles

I found there was a lot of issues while trying to deploy Nextcloud with Docker. Genuinely, the official docker-compose files were not working for me out the box but eventually I got a working configuration.

### Redis

The most simple one:

`docker run -itd --name redis --restart=always redis`

### MariaDB

The most recent version of MariaDB broke support for Nextcloud. It's something to do with an InnoDB incompatability. I'll be honest, I'm not a database person. The fix was just to run an older version, 10.5 specifically

`docker run -itd --name db -v nextcloud_db:/var/lib/mysql -e "MYSQL_ROOT_PASSWORD=snip" -e "MYSQL_PASSWORD=snip" -e "MYSQL_DATABASE=nextcloud" -e "MYSQL_USER=nextcloud" --restart=always mariadb:10.5`

### Nextcloud 

When looking about the best way to link the databases to Nextcloud, apparently the `--link` flag is deprecated right now with potential to be removed in the future. However, the [official Nextcloud docker-compose](https://github.com/nextcloud/docker/blob/master/.examples/docker-compose/insecure/mariadb/apache/docker-compose.yml) images still use it so I based it off this:

`docker run -itd --name nextcloud -v nextcloud_nextcloud:/var/www/html -p 80:80 --link db --link redis --restart always nextcloud`

You can bind the instance to a specific IP as opposed to 0.0.0.0 by modifying the port parameter with an IP. For example, I bound this to ONLY listen on my wireguard instance.

`docker run -itd --name nextcloud -v nextcloud_nextcloud:/var/www/html -p 192.168.183.2:80:80 --link db --link redis --restart always nextcloud`

After all of this has been ran, we will be presented with the nextcloud page. All you need to do is input the credentials you create above and you're good to go!

---

## Adding the Object Storage to Nextcloud

By default, Nextcloud does not support external storage which I find quite strange. But thankfully it is easy enough to add it, without any hacks as a bonus. 

We need to enable the "app" and then configure the bucket details in settings.
![](__GHOST_URL__/content/images/2021/09/nextcloud1.png)Head to app![](__GHOST_URL__/content/images/2021/09/nextcloud2.png)Click "Enable" on the External Storage Support index![](__GHOST_URL__/content/images/2021/09/nextcloud3.png)Head to settings![](__GHOST_URL__/content/images/2021/09/nextcloud4.png)External Storage![](__GHOST_URL__/content/images/2021/09/nextcloud5.png)Input appropriate data
At this point we need to get the information for our bucket and credentials to use. This will use API keys which you can [generate on the Scaleway dashboard ](https://console.scaleway.com/project/credentials)

The information for the bucket will depend on the region, you can find it on your bucket settings. For example, my Paris instance uses the following:

`Bucket: <bucketname>`

`Hostname: s3.fr-par.scw.cloud`

`Port: 443`

`Region: fr-par`

`Enable SSL: yes`

Followed by the API keys you generated on the dashboard. 
![](__GHOST_URL__/content/images/2021/09/nextcloud6.png)
Assuming everything goes well, a green circle should appear to the left which of course means you have authenticated correctly. 

---

## Using the object storage

Simple, go to your files tab and you should see a folder with the name you called the external storage:
![](__GHOST_URL__/content/images/2021/09/nextcloud7-1.png)
This even works on mobile which is quite neat actually. 

Scaleway charges 1 eurocent per GB, which isn't quite as good a rate as Backblaze but we have the added benefit of [no egress fee's](https://images-www.scaleway.com/wp-content/uploads/2020/12/09110105/Object-Storage-ProductSheet-EN2.pdf) (damn you AWS) due to being on their cloud platform. However, with 75GB of free storage it's hard to look away from Scaleway for small-time data storage. This whole experiment could cost you literally 37 cent a month! `seriously, why is this so cheap`
![A diagram explaining costs](__GHOST_URL__/content/images/2021/09/freeegress.png)
