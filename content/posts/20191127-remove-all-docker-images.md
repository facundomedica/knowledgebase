---
title: "Remove all Docker images and containers"
author: "Facundo Medica"
type: "post"
date: 2019-11-27T16:42:42-03:00
subtitle: ""
image: ""
tags: ["snippets", "docker"]
draft: false
---


```
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
docker rmi $(docker images -a -q)
```
