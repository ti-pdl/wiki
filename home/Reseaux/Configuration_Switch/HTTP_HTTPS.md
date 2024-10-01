---
title: Disable HTTP & HTTPS Switch Cisco
description: 
published: true
date: 2024-10-01T19:21:56.088Z
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