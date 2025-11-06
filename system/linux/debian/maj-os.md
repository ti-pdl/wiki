---
title: Mise à jour debian
description: 
published: true
date: 2025-11-06T10:55:48.015Z
tags: 
editor: markdown
dateCreated: 2025-11-06T10:55:48.015Z
---

# Tips
- Désactiver le boot graphique
> sudo systemctl set-default multi-user.target
> sudo reboot
{.is-info}

- Réactiver le boot graphique
> sudo systemctl set-default graphical.target
> sudo reboot
{.is-info}

- Trouver et supprimer le proxy
> sudo grep -r eple /root
> sudo grep -r eple /etc
{.is-info}


# Mise à jour debian 9 > 10
 > Ne pas effectuer une mise à jour du système via SSH
{.is-warning}

- Mettre à jour le système
> sudo apt install -y debian-archive-keyring
> sudo apt update
> sudo apt dist-upgrade -y
> sudo apt autoremove
{.is-info}

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
