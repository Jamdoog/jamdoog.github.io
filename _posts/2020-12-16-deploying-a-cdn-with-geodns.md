---
title: Deploying a "CDN" with GeoDNS
slug: deploying-a-cdn-with-geodns
date_published: 2020-12-16T08:11:45.000Z
date_updated: 2020-12-27T10:31:25.000Z
tags: Linux, Web
---

### Introduction

When wondering how to pass the time I decided to optimise my website. I took initial steps such as minimising CSS and JS however incase you haven't seen, [my base site](https://jamdoog.com) actually is actually very small (and in reality serves no purpose). 

At the time my website was powered by 2 loadbalancers and 3 web servers located around the world. If you was curious, the setup was HAProxy following [this](https://www.tecmint.com/setup-nginx-haproxy-load-balancer-in-centos-8/) tutorial. Also, at this time, I used OpenBSD and had no idea how to use PF so my HTTP connections were left exposed at the origin...

This worked however, it wasn't as satisfying as saying "I deliver content from a local edge node". Not only that, but with big companies like Cloudflare powering my websites since 2016-2019 the idea of creating my own "CDN" really appealed to me from a privacy standpoint but also a learning standpoint. 

As it stood my web payload was already tiny and in practicallity the only issue for loading times was DNS requests and web server location. Infact, Google PageSpeed insights suggested it was practically perfect (100%). But I knew it wasn't optimal for people viewing from Asia or America. 
![Google PageSpeed Insights showing 32/34 page optimisations](__GHOST_URL__/content/images/2020/12/Screen-Shot-2020-12-16-at-04.07.46-1.png)
---

## Servers?

So the next step logically to me was to get some new PoP's. I am very thankful that [LET](https://lowendtalk.com) is well known for there Black Friday deals! The aftermath of that I bought 4 VPS servers, 3 in America and 1 in Singapore. Shoutout to [Racknerd](https://racknerd.com) and [Cam](https://gullo.me) for some awesome deals. At the time of writing this 2 of the America servers are not used (Infact 1 has not been delivered, thanks Virmach). 

### What servers do I currently use?

OVH:

- London, United Kingdom
- Gravelines, France
- Beauharnois, Canada
- Frankfurt, Germany
- Singapore, Singapore

Hetzner:

- Falkenstein, Germany
- Helsinki, Finland

Colocrossing:

- Ashburn, Virginia, United States

Serverius:

- Droten, The Netherlands ***(coming soon)***

## Research

The next step was to figure out how I could utilise these. I actually considered GeoDNS from the very start however I needed to further my research before deciding it would be the route I took. 

My criteria for a self-hosted CDN was the following:

> -     Cannot rely on external providers such as AWS (No thanks Route53)
> -     Had to be configured by myself from origin server to end server
> -     Had to be scalable. ***More on this later***
> -     Some sort of learning involved

Posts such as [this one](https://www.nginx.com/blog/learn-to-stop-worrying-build-cdn/) from NGINX really helped to inspire me, I highly suggest you watch the presentation. Of course they are at a much larger scale, but the principle was the same; serve content faster. Although I quickly found that Google was not that useful as it proposed alot of advertisements and/or solutions outside of my reach. 
How I learned to Stop Worrying and Build My Own CDN | Time Warner
The big one that, to my understanding, is used in Industry is BGP anycast. Of course, this is completely out of reach and not possible for a University Student who doesn't have thousands of pounds available. Also, none of my providers allow BGP announcements either.

After going through multiple pages of search queries I eventually settled on the GeoDNS solution. I came accross a gold mine of a page, [GeoIP.site](https://geoip.site). Who would have thought someone wanted to achieve the same goal? 

---

## How does it work?

The author of this page explains that they wanted a in-house soltion to move away from UltraDNS and that this was the solution they utilised. While I encourage you to read the original post, to abbreviate it, the scripts provided will pull from Maxmind and other GeoLocation databases and format them to be ACL compatible with BIND.

With these IP ranges matched to countries, it allows for custom DNS responses based on the origin IP. Honestly, such a simple solution amazed me. Of course this isn't a foolproof method and there are some flaws in it but, this will be talked about later.
![GeoDNS diagram outlining how the connection works](__GHOST_URL__/content/images/2020/12/geo-dns-diagram.png)GeoDNS diagram outlining how the connection works
## How do my edge servers provide content?

NGINX. Reverse proxying is a beautiful technology which allows me to cache my website locally on each edge node once they have fetched before. I did look into varnish however after researching the differences between this approach I felt like NGINX was the most appropriate. 

## Speed Statistics?

I unfortauntely didn't note down any previous measurements for speed. However, I can provide some figures of how this preforms as of now.

### Google PageSpeed
![Google PageSpeed - Desktop test image. Image depicts a score of 100 and speed index of 0.2 seconds.](__GHOST_URL__/content/images/2020/12/google-pagespeed-desktop.png)Google PageSpeed - Desktop![Google PageSpeed - Mobile. Image depicts a loading time of 0.8 seconds.](__GHOST_URL__/content/images/2020/12/google-pagespeed-mobile.png)Google PageSpeed - Mobile
### Pingdom - London
![Pingdom Website Speed Test. Total time: 124ms](__GHOST_URL__/content/images/2020/12/pingdom-london.png)Pingdom Speed Test - London
### Pingdom - Japan
![Pingdom Speed Test - Japan. 1 second for DNS and 1.3 seconds for page load](__GHOST_URL__/content/images/2020/12/pingdom-japan.png)Pingdom Speed Test - Japan
### Pingdom - United States (Washington)
![Pingdom Page Speed insights. Washington. 500ms to load page.](__GHOST_URL__/content/images/2020/12/pingdom-washington.png)Pingdom Speed Test - Washington USA
### WebPageTest - United States (Salt Lake City)
![WebPageTest Salt Lake City test. 700ms fully loaded.](__GHOST_URL__/content/images/2020/12/webpagetest-saltlake.png)WebPageTest - Salt Lake City USA
### Uptrends - Netherlands
![Uptrends website speed test image. Depicts 300ms to load page.](__GHOST_URL__/content/images/2020/12/uptrends-nl.png)Uptrends - Netherlands
### GTmetrix - Canada
![GTmetrix website speed test in vancouver. Loaded in 0.7s](__GHOST_URL__/content/images/2020/12/gtmetrix-canada.png)GTmetrix Speed Test - Vancouver
### Site 24x7 - Hong Kong
![Image of Site 24x7 Speed Test using Hong Kong server. Took 3.8s to load fully](__GHOST_URL__/content/images/2020/12/24x7hk.png)Site 24x7 Speedtest - Hong Kong
## Downsides to this approach

DNS isn't a perfect technology and I have noticed that some lookups do have exceedingly high times (6+ seconds). While this is not true accross the board (as seen in above screneshots) it has happened at least a couple times. I believe this is down to my configuration of named and it's ACL pool. 

Of course depending on the DNS resolver that the client is using it can seriously affect the results of this. For example, if someone uses Google or Cloudflare it might has potential to return results based on these resolvers last saved query. However, since the majority of people will utilise ISP nameservers this should not be considered too much.

Another critique of this system is that it does not take advantage of city based responses. For example, a US-EAST client connecting to my US-WEST server would have significantly higher loading times compared to if I had a server in US-EAST

Finally this approach is not great resolution when downtime occurs. I could create a custom script which updates my nameservers to push clients to another sevrer, however, this does not solve the issue of DNS caching. This is where a solution such as a floating IP address could come into use but I do not have any services capable of this.

As a personal issue with this approach, I do not have much content to actually put this system through work. This blog may act as one but it is hardly something to break a sweat over. This is where I expect load balancers in combination with this to be required in ensuring uptime. 

## Summary

In short a Geo aware DNS resolver can absolutely help to resolve page speed times for content, especially if you have lots of PoP's. However, it does also have flaws. I would recommend it in combination with other technologies such as Load Balancing and Anycast. 

As a fun project though? Learning basic technologies? Sure. It kept me entertained for a couple of days. However, with technologies like Kubernetes around I think this could have been optimised better on my edge nodes. 
