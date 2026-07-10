---
title: Docker
description: 
published: true
date: 2026-07-10T14:17:57.405Z
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

## FOG

```yaml
services:
  fog:
    image: cpasjuste/fog
    restart: unless-stopped
    #privileged: true # for nfs-kernel-server mode
    environment:
      FOG_VERSION: 1.5.10.1870
      HTTP_ADDRESS: srv-docker # also used for ftp/tftp/nfs/...
      HTTP_PORT: ${HTTP_PORT:-9445} # you can use a custom http port
      PHP_FPM_PORT: ${PHP_FPM_PORT:-9444} # listen on localhost, no need to open port in firewall
      WEB_PASSWORD: ${WEB_PASSWORD:-password} # web user (fog) password
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:-dockerfogsql} # mysql user (fog) password
      # builtin storage (shell/nfs/ftp/tftp...) password
      # use alphanumeric because it is not handled correctly by fog (php?)
      STORAGE_PASSWORD: ${STORAGE_PASSWORD:-dockerfogstorage}
      USE_UNFS3: true # use userland nfs server (no need for privileged mode)
    volumes:
      - database:/var/lib/mysql
      - /images:/images # on host: sudo mkdir /images && chown -R 1000:1000 /images
      #- /lib/modules:/lib/modules:ro # for nfs-kernel-server mode
    network_mode: host # needed ports: https://github.com/Cpasjuste/docker-fog/blob/main/README.md
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.fog.rule=Host(`fog.srv-docker.home`)"
      - "traefik.http.services.fog.loadbalancer.server.port=9445"
      - "traefik.http.routers.fog.entrypoints=websecure"
      - "traefik.http.routers.fog.tls=true"
      - "traefik.http.routers.fog.service=fog"

volumes:
  database:
```

## Firewall
```
sudo apt install ufw
sudo ufw allow 21     # FTP
sudo ufw allow 22     # SSH
sudo ufw allow 69/udp # TFTP
sudo ufw allow 80     # HTTP
sudo ufw allow 443    # HTTPS
sudo ufw allow 9443   # PORTAINER
sudo ufw allow 110    # NFS
sudo ufw allow 2049   # NFS
sudo ufw allow 20048  # NFS
sudo ufw enable
```