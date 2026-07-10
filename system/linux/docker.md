---
title: Docker
description: 
published: true
date: 2026-07-10T12:33:54.569Z
tags: 
editor: markdown
dateCreated: 2026-07-10T11:20:42.014Z
---

# srv-docker

- Install debian
	- hostname srv-docker
- Install docker
	- https://docs.docker.com/engine/install/debian/
- Install portainer
	- https://docs.portainer.io/start/install-ce/server/docker/linux

## Reverse proxy container

```yaml
services:
  traefik:
    # The official v3 Traefik docker image
    image: traefik:v3.7
    container_name: traefik
    restart: always
    deploy:
      resources:
        limits:
          memory: 1024M
    # Enables the web UI and tells Traefik to listen to docker
    command: 
      - "--api.insecure=true"
      - "--providers.docker"
      - "--providers.docker.exposedbydefault=false"
      - "--entryPoints.web.address=:80"
      - "--entryPoints.websecure.address=:443"
    ports:
      # The HTTP port
      # - "80:80"
      # The HTTPS port
      - "443:443"
      # The Web UI (enabled by --api.insecure=true)
      # - "9444:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"

    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`traefik.srv-docker.home`)"
      - "traefik.http.services.traefik.loadbalancer.server.port=8080"
      - "traefik.http.routers.traefik.entrypoints=websecure"
      - "traefik.http.routers.traefik.tls.certresolver=home"
      - "traefik.http.routers.traefik.middlewares=authentik@docker"

```