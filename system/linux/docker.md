---
title: Docker
description: 
published: true
date: 2026-07-10T16:06:39.225Z
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
  ```
	docker volume create portainer_data
  docker run -d -p 9000:9000 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:lts
  ```

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

## Portainer traefik proxy
```yaml
services:
  portainer-proxy:
    container_name: portainer-proxy
    image: "marcnuri/port-forward"
    restart: always
    environment:
      - REMOTE_HOST=srv-docker.home
      - REMOTE_PORT=9000
    networks:
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.rule=Host(`portainer.srv-docker.home`)"
      - "traefik.http.services.portainer.loadbalancer.server.port=80"
      - "traefik.http.routers.portainer.entrypoints=websecure"
      - "traefik.http.routers.portainer.tls=true"
      - "traefik.http.routers.portainer.service=portainer"

networks:
  traefik:
    external: true
```

## Firewall
- Add rules
  ```
  # permit already-established flows
  sudo iptables -I DOCKER-USER 1 \
    -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

  # only allow portainer port 9000 from traefik network
  sudo iptables -I DOCKER-USER 2 \
  	-p tcp -s 172.18.0.0/16 --dport 9000 -j ACCEPT
  # allow udp port 69 for tftp
  sudo iptables -I DOCKER-USER 2 \
  	-p udp -m conntrack --ctorigdstport 69 -j ACCEPT
  # allow misc ports
	for port in 21 22 443 111 2049 20048; do
  	sudo iptables -I DOCKER-USER 2 \
			-p tcp -m conntrack --ctorigdstport "$port" -j ACCEPT
	done

  # reject every other new inbound forwarded connection
  sudo iptables -A DOCKER-USER \
    -m conntrack --ctstate NEW -j DROP
  ```
- List rules
	```
	sudo iptables -L DOCKER-USER -n -v --line-numbers
	```
- Delete rules
	```
	sudo iptables -D DOCKER-USER x
	```
- Flush rules (remove all)
	```
	sudo iptables -F DOCKER-USER
	```