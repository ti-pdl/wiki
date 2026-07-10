---
title: Docker
description: 
published: true
date: 2026-07-10T14:11:25.755Z
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

## Traefik (eeverse proxy)

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
      - "--entryPoints.web.http.redirections.entrypoint.to=websecure"
      - "--entryPoints.web.http.redirections.entrypoint.scheme=https"
      - "--entryPoints.websecure.address=:443"
    networks:
      - traefik
    ports:
      # The HTTP port
      - "80:80"
      # The HTTPS port
      - "443:443"
      # The Web UI (enabled by --api.insecure=true)
      #- "9001:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    extra_hosts: 
      - host.docker.internal:172.17.0.1

networks:
  traefik:
    external: true
```

## Nginx traefik test

```yaml
services:
  nginx:
    image: nginx:latest
    networks:
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nginx.rule=Host(`nginx.srv-docker.home`)"
      - "traefik.http.services.nginx.loadbalancer.server.port=80"
      - "traefik.http.routers.nginx.entrypoints=websecure"
      - "traefik.http.routers.nginx.tls=true"
      - "traefik.http.routers.nginx.service=nginx"

networks:
  traefik:
    external: true
```