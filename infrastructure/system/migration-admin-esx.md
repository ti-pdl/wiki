---
title: Migration Admin ESX
description: 
published: true
date: 2026-10-01T12:46:06.483Z
tags: 
editor: markdown
dateCreated: 2026-10-01T12:41:15.893Z
---

# Migration admin - ESX

Afin d'améliorer la bande passante, le vlan administratif est basculé sur le vswitch "pédagogique". Deux cas se posent à nous : 

- Infra en 10Gb: attribuer deux liens 10Gb pour le vswitch pédagogique (profs, élèves, admin...) et les deux liens restants (1Gb) pour le vswitch management.
- Infra en 1Gb: attribuer les 4 liens pour sur un seul vswitch (péda, management, vmotion).

## Préparation

Nous allons procéder un ESX après l'autre. Par conséquent l'opération peut se faire en production. Nous devons donc tout d'abord :

- Migrer toutes les machines virtuelles sur un seul ESX.
- Passer l'ESX sur lequel nous allons travailler en mode maintenance.

## Infrastructure en 1Gb
<br>

#### 1 - Identifier les interfaces connectées au cœur de réseau

- Sélectionner l'hôte 
- Cliquer sur `Configurer`
- Sélectionner `Commutateurs virtuels`
- Sélectionner un adaptateur physique puis cliquer sur les 3 petits points puis `Afficher les paramètres`
- Sur l'onglet `CDP` récupérer l'ID du port de la carte (vmnicX)
- Répéter l'opération pour les quatre adaptateurs physiques.
<br>
<img title="" src="/media/system/vswitch-0.png" alt="" width="600">
<img title="" src="/media/system/vswitch-1.png" alt="" width="300">
<br>

#### 2 - Préparation du vswitch / port channel pédagogique

- Récupérer la configuration des quatre interfaces
`sh run int TenGigabitEthernetX/X/X`
- Identifier les interfaces du réseau pédagogique ainsi que du port channel correspondant

```
interface TenGigabitEthernet2/0/11
 description virt2-peda-vmnic2
 switchport trunk allowed vlan 50,51,60,100,204,205,210,220,310,320,365,380,400
 switchport mode trunk
 switchport nonegotiate
 channel-group 2 mode on
end
```

```
interface TenGigabitEthernet2/0/10
 description virt2-peda-vmnic3
 switchport trunk allowed vlan 50,51,60,100,204,205,210,220,310,320,365,380,400
 switchport mode trunk
 switchport nonegotiate
 channel-group 2 mode on
end
```

- Ajouter l'id du vlan de management (205) au trunk des deux interfaces s'il n'est pas déjà défini

```
conf t
int TenGigabitEthernet2/0/11
switchport trunk allowed vlan add 205
```

- Ajouter l'id du vlan de management (205) au port channel s'il n'est pas déjà défini

```
conf t
int po2
switchport trunk allowed vlan add 205
```

> Ne pas toucher aux deux interfaces / vmnic de management pour l'instant, sinon perte de connexion avec l'hôte (esx)
{.is-warning}


#### 3 - Migration du VMkernel de management




