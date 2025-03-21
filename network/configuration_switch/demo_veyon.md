---
title: demo_veyon
description: 
published: true
date: 2024-10-01T19:30:29.138Z
tags: 
editor: markdown
dateCreated: 2024-09-27T23:27:15.342Z
---

# Activer la fonction DEMO sur Veyon
Pour que la fonction "démo" de Veyon fonctionne entre le poste Prof (Vlan Prof) et les postes des élèves (Vlan Eleves), il faut ajouter une règle dans l'ACL Eleves.
Il faut placer cette règle avant celle qui interdit la communication entre les deux VLAN
deny ip reseau_eleves wildcard_mask_reseau_eleves reseau_profs wildcard_mask_reseau_profs

**Pour connaitre le numéro des règles :** 
---

   ```
   show access-lists
  ```

> Extended IP access list Eleves
>     180 permit tcp 172.20.40.0 0.0.3.255 172.20.44.0 0.0.0.255 established (461 matches)
>     **189 permit tcp 172.20.40.0 0.0.3.255 172.20.44.0 0.0.0.255 eq 11400**
>     190 deny ip 172.20.40.0 0.0.3.255 172.20.44.0 0.0.0.255 (566262 matches)
> 
{.is-info}

**Commande pour ajouter la règle:**
---

```
conf t
ip access-list extended Eleves
189 permit tcp 172.20.40.0 0.0.3.255 172.20.44.0 0.0.0.255 eq 11400
```