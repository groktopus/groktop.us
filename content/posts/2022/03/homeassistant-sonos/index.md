---
title: Homeassistant and Sonos vs. Docker Bridge Network
description: Running Home Assistant inside of a Docker bridge network can cause issues with its integrations like Sonos. But these aren't insurmountable. I'll show you how I got it working. 
toc: true
authors: magnus
tags:
  - Home Assistant
  - home automation
  - donos
  - docker
  - homelab
  - traefik
categories:
  - Home Assistant
  - home automation
  - homelab
series: [Stupid Home Assistant Tricks, Stupid Docker Tricks]
date: 2022-03-20T22:06:59-04:00
lastmod: 2022-03-21T12:13:59-04:00
publishDate: 2022-03-20T22:06:59-04:00
featuredVideo:
featuredImage: posts/2022/03/homeassistant-sonos/sonos.png
thumbnail: posts/2022/03/homeassistant-sonos/sonos.png
images:
  - posts/2022/03/homeassistant-sonos/sonos.png
draft: false
---

{{% notice warning %}}
This is not for casual users. Container networking is not for the faint of heart. We're going to be breaking some rules. And doing some reverse proxying. This isn't a beginner tutorial, but rather some notes for those who are brave & foolish enough to wander in this direction.
{{% /notice %}}

The official Home Assistant [documentation](https://www.home-assistant.io/installation/linux#install-home-assistant-container) suggests that when running inside of Docker, `host` networking should be enabled. But since this creates a situation where the container network cannot be... *contained*, I decided to try running Home Assistant inside of a `bridge` network.

The same documentation says that one must use `privileged` mode for Home Assistant to work. I'm not finding that to be true.

I did use a bridge network, no privileged mode, and I've got Home Assistant working pretty well. There were maybe *two* hiccups to contend with. Let me tell you about one of them.

# Sonos

The Home Assistant documentation for the [Sonos integration](https://www.home-assistant.io/integrations/sonos/#network-requirements) does offer a warning about what happens if Sonos devices are running behind a NAT or are otherwise on a different subnet.

> *To work optimally, the Sonos devices must be able to connect back to the Home Assistant host on TCP port 1400. This will allow the push-based updates to work properly. If this port is blocked or otherwise unreachable from the Sonos devices, the integration will fall back to a polling mode which is slower to update and much less efficient. The integration will alert the user if this problem is detected.*

Sure enough, even though my speakers worked, I got a notification that something was amiss with the Sonos integration and that I should check my logs. Here's what I found:

```
2022-03-20 22:02:28 WARNING (MainThread) [homeassistant.components.sonos.entity] Office cannot reach 172.15.0.26:1400, falling back to polling, functionality may be limited, see https://www.home-assistant.io/integrations/sonos/#network-requirements for more details
```

The Sonos integration document gives a good clue about a potential remedy here:

> The Sonos speakers will attempt to connect back to Home Assistant to deliver change events. By default, Home Assistant will listen on port 1400 but will try the next 100 ports above 1400 if it is in use. You can change the IP address that Home Assistant advertises to Sonos speakers. This can help in NAT scenarios such as when not using the Docker option --net=host.

```yaml
# Example configuration.yaml entry modifying the advertised host address
sonos:
  media_player:
    advertise_addr: 192.0.2.1
```

So we need to do a few things:

1. Have Home Assistant listening on port **1400/tcp** on the LAN. (But I don't want to expose a port directly because then I have to traffic shape ingress on the Docker host itself.)
2. Make a change to Home Assistant config to advertise to Sonos speakers that we're listening on the Docker host IP.

Let's break it down.

```yaml {hl_lines="26-30"}
---
version: "2.1"
services:
  homeassistant:
    image: homeassistant/home-assistant:stable
    environment:
      - TZ=America/New_York
    volumes: 
      - homeassistant:/config
    restart: unless-stopped
    network_mode: bridge
    depends_on:
      - zwavejs
      - deconz
      - mqtt
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.${TRAEFIK_ROUTER}.entrypoints=websecure"
      - "traefik.http.routers.${TRAEFIK_ROUTER}.rule=Host(`${TRAEFIK_ROUTER}.example.com`)"
      - "traefik.http.routers.${TRAEFIK_ROUTER}.service=${TRAEFIK_SERVICE}"
      - "traefik.http.routers.${TRAEFIK_ROUTER}.tls=true"
      - "traefik.http.routers.${TRAEFIK_ROUTER}.tls.certresolver=myresolver"
      - "traefik.http.routers.${TRAEFIK_ROUTER}.middlewares=default@file"
      - "traefik.http.services.${TRAEFIK_SERVICE}.loadbalancer.server.port=8123"
      - "traefik.http.services.${TRAEFIK_SERVICE}.loadbalancer.server.scheme=http"
      - "traefik.tcp.routers.sonos.entrypoints=sonos"
      - "traefik.tcp.routers.sonos.rule=HostSNI(`*`)"
      - "traefik.tcp.routers.sonos.service=sonos"
      - "traefik.tcp.routers.sonos.middlewares=homelan@file"
      - "traefik.tcp.services.sonos.loadbalancer.server.port=1400"
```      

I was already using [Traefik](https://traefik.io/traefik/) for ingress, traffic shaping, security middlewares, etc. You can see I was already exposing the Home Assistant web interface with Traefik. You'll notice *no ports are exposed* directly to the container itself.

We are using a `bridge` type network. Not the `host` type network that the documentation prescribes.

On the `config.yaml` for Home Assistant, I added a few lines straight from the doc (change IP to suit your own Docker host).

```yaml
sonos:
  media_player:
    advertise_addr: 192.168.42.42
```    

And also Traefik's `traefik.yaml` (though I'm probably going to have a do-over with Traefik and shove as much config into the runtime as I can).

```yaml {hl_lines="3-4"}
entryPoints:
# [...]
  sonos:
    address: :1400
```

Get everything back up & running with its new config and *bam* it works. No more errors. 