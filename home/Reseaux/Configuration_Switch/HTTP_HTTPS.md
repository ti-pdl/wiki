---
title: Disable HTTP & HTTPS Switch Cisco
description: 
published: true
date: 2024-09-27T23:19:09.138Z
tags: 
editor: markdown
dateCreated: 2024-09-27T23:19:07.394Z
---

# Disable HTTP & HTTPS Switch Cisco
Pour désactiver le HTTP et le HTTPS sur les switch Cisco suite à la faille zero-day
  ```
conf t
no ip http server
no ip http secure-server
end
wr
  ```