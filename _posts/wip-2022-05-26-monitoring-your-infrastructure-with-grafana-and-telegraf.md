---
title: Monitoring your infrastructure with Grafana and Telegraf
slug: monitoring-your-infrastructure-with-grafana-and-telegraf
date_published: 2022-05-26 T00:00:00.000Z
date_updated: 2022-05-26T22:29:56.000Z
draft: true
---

Monitoring your infrastructure is something that I have neglected for a while, truthfully there was no need due to not pushing any load on my network (and still don't). However, when I received this message, it was clear monitoring was required:
![](__GHOST_URL__/content/images/2022/05/Screenshot-from-2022-05-26-23-27-20.png)
Turns out OVH don't like neither UDP-based or GRE-based tunnels. 

## Installing Grafana

Grafana is not in any repositories of both APT and RHEL counterparts. As such, we will need to manually add their repo to our Linux boxes. Lucky for BSD people, their is a [port](https://cgit.freebsd.org/ports/tree/www/grafana8) for them to install from, however I will not be covering this.

I would recommend you double check the official Grafana website before proceeding with my tutorial to ensure that this is the recommended and latest method of installing Grafana.

[APT Based Installs](https://grafana.com/docs/grafana/latest/installation/debian/) (Grafana)

[RHEL Based Installs](https://grafana.com/docs/grafana/latest/installation/rpm/) (Grafana)

[Universal Installs ](https://docs.influxdata.com/influxdb/v1.8/introduction/install/)(InfluxDB)

### Debian, Ubuntu & Derivatives

We need to create a file in `/etc/apt/sources.list.d/` which will allow our package list to be updated to pull from there. But first, the official guide recommends us to add their GPG key in order to ensure authenticity. We will need to install `apt-transport-https software-properties-common wget` for this:

`sudo apt install apt-transport-https software-properties-common wget`
![An image of the packages being installed](__GHOST_URL__/content/images/2021/08/installpackages-1.png)
Then we can retreive the GPG key and add it as trusted, this will just return the text "OK".

`wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -`

And finally we can add the repository and update our package store:

`echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list`

`apt update‌`

And finally, we can then install the grafana package:

`apt install grafana`
![An image of Grafana being installed](__GHOST_URL__/content/images/2021/08/installgrafanaapt-1.png)Notice how it doesn't start at boot?
As you may have noticed in the installer text, it says that it will not automatically start by itself. Now, I use systemd (highly controversial, I know) so it's not necessary to do much more however if you use something without systemd then you can do the equivalent commands:

`sudo systemctl daemon-reload`

`sudo systemctl enable --now grafana-server.service`

### RHEL, CentOS & Derivatives

We need to add the Grafana repository for dnf to get its package information from. To do this, we just need to add the following data to `/etc/yum.repos.d/grafana.repo`:

`nano /etc/yum.repos.d/grafana.repo`

    [grafana]
    name=grafana
    baseurl=https://packages.grafana.com/oss/rpm
    repo_gpgcheck=1
    enabled=1
    gpgcheck=1
    gpgkey=https://packages.grafana.com/gpg.key
    sslverify=1
    sslcacert=/etc/pki/tls/certs/ca-bundle.crt

After this, we can install the Grafana package with dnf:

`sudo dnf install grafana`

It should ask you to trust the GPG key. This can be found at `https://packages.grafana.com/gpg.key` if you would like to find out more.

Finally, we can reload the systemd daemon & enable the service on boot:

`sudo systemctl daemon-reload`

`sudo systemctl enable --now grafana-server.service`
