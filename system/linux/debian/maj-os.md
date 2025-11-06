---
title: Mise à jour debian
description: 
published: true
date: 2025-11-06T13:05:49.085Z
tags: 
editor: markdown
dateCreated: 2025-11-06T10:55:48.015Z
---

# Tips
- Désactiver le boot graphique
	```
	sudo systemctl set-default multi-user.target
	sudo reboot
	```

- Réactiver le boot graphique
	```
	sudo systemctl set-default graphical.target
	sudo reboot
  ```

- Trouver et supprimer le proxy
	```
  sudo grep -r eple $HOME
	sudo grep -r eple /root
	sudo grep -r eple /etc
	```

# Mise à jour debian
 > Ne pas effectuer une mise à jour du système via SSH
{.is-warning}

- Mettre à jour le système
	```
	sudo apt install -y debian-archive-keyring
	sudo apt update
	sudo apt -y dist-upgrade
	sudo apt -y autoremove
	```

- Effectuer l'upgrade
	- reboot
  - remplacer le contenu du fichier `/etc/apt/sources.list` par les sources de la version cible
  - procéder à l'upgrade du système en conservant les fichiers de configuration des paquets
    ```
  	sudo apt update
  	sudo DEBIAN_FRONTEND=noninteractive apt -o Dpkg::Options::="--force-confold" -y dist-upgrade
    sudo apt -y autoremove
    ```

# sources.list
- debian 11
> deb http://deb.debian.org/debian/ bullseye main contrib non-free
> deb http://deb.debian.org/debian-security/ bullseye-security main contrib non-free
> deb http://deb.debian.org/debian/ bullseye-updates main contrib non-free
{.is-info}
- debian 10
> deb http://archive.debian.org/debian/ buster main contrib non-free
> deb http://archive.debian.org/debian/ buster-proposed-updates main contrib non-free
> deb http://archive.debian.org/debian-security buster/updates main contrib non-free
{.is-info}
- debian 9
> deb http://archive.debian.org/debian/ stretch main contrib non-free
> deb http://archive.debian.org/debian/ stretch-proposed-updates main contrib non-free
> deb http://archive.debian.org/debian-security stretch/updates main contrib non-free
{.is-info}
