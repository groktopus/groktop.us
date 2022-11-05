---
title: System
description: A little information about how this site is hosted.
toc: false
authors: magnus
tags:
  - homelab
  - docker
  - hugo
categories:
  -
series:
  -
date: 2022-11-05T19:25:12-04:00
lastmod: 2022-11-05T19:25:12-04:00
publishDate: 2022-11-05T19:25:12-04:00
featuredVideo:
featuredImage:
thumbnail: 
images:
  -
draft: false
---

This site is static and built with [Hugo](https://gohugo.io). I'm not in love with this, but it's the devil I know.

I had been hosting it as a DigitalOcean App, which was nice in that it was free and automated. But it couldn't handle things like `git` submodules, or `npm`.

Now I host it out of my homelab on a *Docker Swarm*. Everything in this container is read only, and the builds happen somewhere else.
