---
title: Pilotes
description: Windows et les pilotes...
published: true
date: 2025-06-13T12:25:30.721Z
tags: windows, serveurs, pilotes, drivers
editor: markdown
dateCreated: 2024-09-27T11:56:14.383Z
---

## Déploiement de pilotes via GPO (Windows 11)

Un [script](https://github.com/ti-pdl/deploy_drivers/blob/main/deploy_drivers.ps1) PowerShell a été écrit afin de faciliter le déploiement des pilotes manquants via GPO sur certains modèles d'ordinateurs compatibles. 
> Sur ces modèles, la carte Ethernet a été vérifié fonctionnelle avec une installation par défaut de Windows 11
{.is-success}

> Pour l'instant, seules les cartes graphiques `NVIDIA T400`, `Radeon RX550/550 Series` ainsi que `AMD Radeon R7 450` sont supportées. Les autres modèles devront être ajoutés à la [base de donnée des pilotes](#base-de-données-des-pilotes)
{.is-warning}

### Les avantages

- Réduction importante de la taille du master (pas de "drivers packs")
- Installation des pilotes nécessaires uniquement (pas de pilotes en double)
- Fichiers compressés (gain de temps, de bande passante)
- Mise à jour simplifiée (pas de master à refaire)

### Fonctionnement

- Exécution du script sur le serveur avec le paramètre `-init`
  - Téléchargement de la [base de donnée des pilotes](#base-de-données-des-pilotes)
  - Téléchargement de tous les pilotes de la base de donnée
- Exécution du script sur le client via GPO
  - Mappage d'un lecteur réseau contenant la [base de donnée des pilotes](#base-de-données-des-pilotes) (et les pilotes précédemment téléchargés)
  - Téléchargement (depuis le serveur) et installation du pilote si le modèle correspond et que celui-ci n'est pas encore installé
  
### Mise en place

- Télécharger le script [deploy_drivers.ps1](https://github.com/ti-pdl/deploy_drivers/blob/main/deploy_drivers.ps1) et le placer sur un serveur du domaine dans un dossier qui sera partagé ultérieurement
- Éxécuter le [script](https://github.com/ti-pdl/deploy_drivers/blob/main/deploy_drivers.ps1) avec l'option `-init` afin de télécharger la [base de donnée](#base-de-données-des-pilotes) et les pilotes sur le serveur:
  ```
  .\deploy_drivers.ps1 -init
  ```
> Si le téléchargement n'est pas possible sur le serveur (proxy, etc), exécuter le script depuis un autre poste puis copier les fichiers `deploy_drivers.ps1`, `pilotes.md` et le dossier `drivers` sur le serveur.
{.is-warning}

- Créer un compte spécifique sur le domaine pour le partage du dossier sur le serveur
<details>
  <summary>Voir les captures d'écran</summary>
  <img src="/media/system/windows/pilotes/create-account.png"/>
</details>

- Partager le dossier contenant la base de donnée (`pilotes.md`) et le dossier des pilotes précédemment téléchargés (`drivers`) en lecture seule sur le serveur pour le rendre accessible depuis une GPO ordinateur
<details>
  <summary>Voir les captures d'écran</summary>
  <img src="/media/system/windows/pilotes/partage2.png"/> <img src="/media/system/windows/pilotes/partage1.png"/>
</details>

- Copier le script [deploy_drivers.ps1](https://github.com/ti-pdl/deploy_drivers/blob/main/deploy_drivers.ps1) sur le contrôleur du domaine (dossier `netlogon`)
<details>
  <summary>Voir les captures d'écran</summary>
  <img src="/media/system/windows/pilotes/netlogon.png"/>
</details>

- Créer une GPO ordinateur pour l'exécution du script avec les paramètres suivants:
  - `srv_path`: dossier partagé contenant `pilotes.md` et `drivers`
  - `srv_username`: nom d'utilisateur du compte créé précédemment
  - `srv_password`: mot de passe du compte créé précédemment
  ```
  -srv_path "\\srv-xxx\drivers" -srv_username "xxx.local\svc_drivers" -srv_password "xxx"
  ```
<details>
  <summary>Voir les captures d'écran</summary>
  <img src="/media/system/windows/pilotes/gpo.png"/>
</details>

> Par défaut, **le [script](https://github.com/ti-pdl/deploy_drivers/blob/main/deploy_drivers.ps1) ne s'exécute qu'une seule fois sur les postes**. Pour forcer l'exécution de celui-ci à chaque démarrage (mise à jours de la [base de donnée](#base-de-données-des-pilotes) sur le serveur par exemple), ajouter le paramètre `-force`
{.is-info}

> Si le [script](https://github.com/ti-pdl/deploy_drivers/blob/main/deploy_drivers.ps1) est activé pour des postes sur lesquels des drivers plus récent que ceux de la [base de donnée](#base-de-données-des-pilotes) sont déjà installés, **et** que vous utilisez l'option `-force`, une tentative d'installation (téléchargement) des drivers plus anciens de la [base de donnée](#base-de-données-des-pilotes) sera effectuée à chaque exécution de celui-ci.
{.is-danger}

> Par défaut, les logs sont consultables sur le poste déployé à l'emplacement suivant: `C:\deploy_drivers.log`
{.is-info}

<br>

### Tester le déploiement des pilotes sur un poste

- Déployer une image (sans pilotes additionnels) sur un poste compatible (Fog)
- Le joindre au domaine (Fog)
- Se connecter en admin du domaine sur le poste
- Ouvrir une console powershell ainsi que le gestionnaire de périphériques
<details>
  <summary>Voir les captures d'écran</summary>
  <img src="/media/system/windows/pilotes/test-01.png"/>
</details>

- Se dépacer dans le répertoire partagé des pilotes via **le chemin complet** et non le "partage" (sinon le script ne pourra mapper le partage sur un lecteur)
<details>
  <summary>Voir les captures d'écran</summary>
  <img src="/media/system/windows/pilotes/test-02.png"/>
</details>

- Exécuter le script avec les arguments indiqués précédemment lors de la création de la GPO
<details>
  <summary>Voir les captures d'écran</summary>
  <img src="/media/system/windows/pilotes/test-03.png"/>
</details>

- Examiner le gestionnaire de périphériques ainsi que les logs
<details>
  <summary>Voir les captures d'écran</summary>
  <img src="/media/system/windows/pilotes/test-04.png"/>
</details>
<details>
  <summary>Voir les logs</summary>
  
  ```
[2024-10-13 12:23:15] [Info] GetLocalDrivers: skipping local drivers, directory not found (C:\drivers\OptiPlex 3060)
[2024-10-13 12:23:15] [Info] MapDrive: mapping \\srv-xxx.xxx.local\Drivers to r:
[2024-10-13 12:23:15] [Info] MapDrive: checking connexion to srv-xxx.xxx.local
[2024-10-13 12:23:19] [Info] LoadDriverDb: loading database from r:\pilotes.md
[2024-10-13 12:23:19] [Info] LoadDriverDb: loaded 61 drivers
[2024-10-13 12:23:19] [Info] GetDrivers: local_driver_path set to C:\Users\xxx\AppData\Local\Temp\deploy_drivers
[2024-10-13 12:23:20] [Info] GetDrivers: skipping NVIDIA - Display - 32.0.15.6109 (device not found: PCI\VEN_10DE&DEV_1FF2&SUBSYS_16131028&REV_A1\4&50383AA&0&0008)
[2024-10-13 12:23:20] [Info] GetDrivers: skipping Advanced Micro Devices, Inc. - Display - 31.0.21912.14 (device not found: PCI\VEN_1002&DEV_699F&SUBSYS_17121028&REV_C7\4&95ED034&0&0008)
[2024-10-13 12:23:20] [Info] GetDrivers: downloading Advanced Micro Devices, Inc. - Display - 31.0.21912.14 (r:\drivers\whql-amd-software-adrenalin-edition-24.3.1-win10-win11-mar20-vega-polaris.exe)...
[2024-10-13 12:24:16] [Info] GetDrivers: installing Advanced Micro Devices, Inc. - Display - 31.0.21912.14...
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Advanced Micro Devices, Inc. - Display - 27.20.20913.2000 (device not found: PCI\VEN_1002&DEV_682B&SUBSYS_10041028&REV_87\4&2A6FB196&0&0008)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel - System - 1.0.11406.42226 (OptiPlex SFF 7010 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.37.7 (OptiPlex SFF 7010 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel - System - 2406.5.5.0 (OptiPlex SFF 7010 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel(R) Corporation - HIDClass - 2.2.2.10 (OptiPlex SFF 7010 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel Corporation - System - 30.100.2417.30 (OptiPlex SFF 7010 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.37.7 (OptiPlex SFF 7010 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel Corporation - System - 3.5.0.1578 (OptiPlex SFF 7010 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel Corporation - Display - 31.0.101.4953 (OptiPlex SFF 7010 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel - System - 1.0.11406.42226 (OptiPlex 3000 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.37.7 (OptiPlex 3000 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel - System - 2406.5.5.0 (OptiPlex 3000 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel Corporation - Display - 31.0.101.4953 (OptiPlex 3000 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel Corporation - System - 30.100.2417.30 (OptiPlex 3000 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel Corporation - Bluetooth - 23.60.5.10 (OptiPlex 3000 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.37.7 (OptiPlex 3000 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel Corporation - System - 3.5.0.1578 (OptiPlex 3000 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel - System - 1.0.11406.42226 (ThinkCentre neo 50s Gen 3 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.37.7 (ThinkCentre neo 50s Gen 3 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel - System - 2406.5.5.0 (ThinkCentre neo 50s Gen 3 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.37.7 (ThinkCentre neo 50s Gen 3 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.37.7 (ThinkCentre neo 50s Gen 3 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.37.7 (ThinkCentre neo 50s Gen 3 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.37.7 (ThinkCentre neo 50s Gen 3 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.37.7 (ThinkCentre neo 50s Gen 3 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.37.7 (ThinkCentre neo 50s Gen 3 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.37.7 (ThinkCentre neo 50s Gen 3 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.37.7 (ThinkCentre neo 50s Gen 3 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel Corporation - System - 3.5.0.1578 (ThinkCentre neo 50s Gen 3 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel Corporation - Display - 31.0.101.4953 (ThinkCentre neo 50s Gen 3 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.33.3 (OptiPlex 3080 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.33.3 (OptiPlex 3080 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.33.3 (OptiPlex 3080 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel - System - 2406.5.5.0 (OptiPlex 3080 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.7.3 (OptiPlex 3080 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel Corporation - Display - 31.0.101.2127 (OptiPlex 3080 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.33.3 (OptiPlex 3070 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.33.3 (OptiPlex 3070 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel - System - 2406.5.5.0 (OptiPlex 3070 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.16.7 (OptiPlex 3070 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping INTEL - System - 10.1.7.3 (OptiPlex 3070 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: skipping Intel Corporation - Display - 31.0.101.2127 (OptiPlex 3070 != OptiPlex 3060)
[2024-10-13 12:28:09] [Info] GetDrivers: downloading INTEL - System - 10.1.33.3 (r:\drivers\74395ee6-64fc-400a-8c1c-f305545e8e91_ca678480483052caa7f24332acc9ba1c4a368e19.cab)...
[2024-10-13 12:28:09] [Info] GetDrivers: extracting INTEL - System - 10.1.33.3 from C:\Users\xxx\AppData\Local\Temp\deploy_drivers\74395ee6-64fc-400a-8c1c-f305545e8e91_ca678480483052caa7f24332acc9ba1c4a368e19.cab...
[2024-10-13 12:28:12] [Info] GetDrivers: skipping INTEL - System - 10.1.33.3 (duplicate driver)...
[2024-10-13 12:28:12] [Info] GetDrivers: downloading Intel - System - 2406.5.5.0 (r:\drivers\6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab)...
[2024-10-13 12:28:12] [Info] GetDrivers: extracting Intel - System - 2406.5.5.0 from C:\Users\xxx\AppData\Local\Temp\deploy_drivers\6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab...
[2024-10-13 12:28:13] [Info] GetDrivers: downloading INTEL - System - 10.1.16.7 (r:\drivers\aba36574-3af7-4c06-9dbf-99f958205e71_34932f041b994a914218407603299b18c0c5ba80.cab)...
[2024-10-13 12:28:13] [Info] GetDrivers: extracting INTEL - System - 10.1.16.7 from C:\Users\xxx\AppData\Local\Temp\deploy_drivers\aba36574-3af7-4c06-9dbf-99f958205e71_34932f041b994a914218407603299b18c0c5ba80.cab...
[2024-10-13 12:28:15] [Info] GetDrivers: downloading INTEL - System - 10.1.7.3 (r:\drivers\6205aa6b-12e9-4298-ba23-aa06e5a8ef4d_f3604bed1e4b9176da6f7a045e5a76f46f506be9.cab)...
[2024-10-13 12:28:15] [Info] GetDrivers: extracting INTEL - System - 10.1.7.3 from C:\Users\xxx\AppData\Local\Temp\deploy_drivers\6205aa6b-12e9-4298-ba23-aa06e5a8ef4d_f3604bed1e4b9176da6f7a045e5a76f46f506be9.cab...
[2024-10-13 12:28:16] [Info] GetDrivers: downloading Intel Corporation - Display - 31.0.101.2127 (r:\drivers\09e28ba3-4673-4e30-a6aa-51f0c7db008c_9a95d25c9c842facf7749e414b2ec35338e8de83.cab)...
[2024-10-13 12:28:44] [Info] GetDrivers: extracting Intel Corporation - Display - 31.0.101.2127 from C:\Users\xxx\AppData\Local\Temp\deploy_drivers\09e28ba3-4673-4e30-a6aa-51f0c7db008c_9a95d25c9c842facf7749e414b2ec35338e8de83.cab...
[2024-10-13 12:29:18] [Info] GetDrivers: skipping INTEL - System - 10.1.1.44 (OptiPlex 3050 != OptiPlex 3060)
[2024-10-13 12:29:18] [Info] GetDrivers: skipping INTEL - System - 10.1.1.44 (OptiPlex 3050 != OptiPlex 3060)
[2024-10-13 12:29:18] [Info] GetDrivers: skipping INTEL - System - 10.1.1.44 (OptiPlex 3050 != OptiPlex 3060)
[2024-10-13 12:29:18] [Info] GetDrivers: skipping Intel - System - 2406.5.5.0 (OptiPlex 3050 != OptiPlex 3060)
[2024-10-13 12:29:18] [Info] GetDrivers: skipping Intel Corporation - Display - 31.0.101.2114 (OptiPlex 3050 != OptiPlex 3060)
[2024-10-13 12:29:18] [Info] GetDrivers: skipping INTEL - System - 10.1.1.44 (HP ProDesk 600 G2 SFF != OptiPlex 3060)
[2024-10-13 12:29:18] [Info] GetDrivers: skipping INTEL - System - 10.1.1.44 (HP ProDesk 600 G2 SFF != OptiPlex 3060)
[2024-10-13 12:29:18] [Info] GetDrivers: skipping INTEL - System - 10.1.1.44 (HP ProDesk 600 G2 SFF != OptiPlex 3060)
[2024-10-13 12:29:18] [Info] GetDrivers: skipping Intel - System - 2406.5.5.0 (HP ProDesk 600 G2 SFF != OptiPlex 3060)
[2024-10-13 12:29:18] [Info] GetDrivers: skipping Intel Corporation - Display - 30.0.101.1339 (HP ProDesk 600 G2 SFF != OptiPlex 3060)
[2024-10-13 12:29:18] [Info] GetDrivers: installing all drivers in C:\Users\xxx\AppData\Local\Temp\deploy_drivers...
[2024-10-13 12:32:19] [Info] UnMapDrive: unmapping r:
[2024-10-13 12:32:19] [Info] All done...
  ```
</details>

- Exécuter le script une seconde fois avec l'option `-force` pour s'assurer que tous les pilotes ont bien étés installés

## Inventaire des pilotes manquants (Windows 11)

### Ajout d'un nouveau modèle à la base de donnée

- Trouver le pilote d'un poste dans le catalogue microsoft via l'instance de périphérique (à exécuter sur le poste en question avec accès au site https://catalog.update.microsoft.com):
  ```
  .\deploy_drivers.ps1 -search "PCI\VEN_10DE&DEV_2560&SUBSYS_3A8117AA&REV_A1\4&2CAE475F&0&0009"
  ```
<details>
  <summary>Voir le résultat</summary>

  ```
  QueryMsCatalog: searching windows 11 (22H2/23H2) driver for "PCI\VEN_10DE&DEV_2560&SUBSYS_3A8117AA" (NVIDIA GeForce RTX 3060 Laptop GPU)
  QueryMsCatalog: found windows 11 (22H2) driver

  Title          : NVIDIA - Display - 31.0.15.4630
  Products       : Windows 11 Client, version 22H2 and later, Servicing Drivers, Windows 11 Client, version 22H2 and later, Upgrade & Servicing Drivers
  Classification : Drivers (Video)
  LastUpdated    : 11/29/2023
  Size           : 765.2 MB 802348236
  Link           : https://www.catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_10DE%26DEV_2560%26SUBSYS_3A8117AA

  Markdown:
  | Legion 5 15ACH6H | NVIDIA GeForce RTX 3060 Laptop GPU | PCI\VEN_10DE&DEV_2560&SUBSYS_3A8117AA&REV_A1\4&2CAE475F&0&0009 | [NVIDIA - Display - 31.0.15.4630](https://www.catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_10DE%26DEV_2560%26SUBSYS_3A8117AA) | [:floppy_disk:](TODO) | [:floppy_disk:](TODO) | NON |
  ```
</details>

- Scanner un poste à la recherche de pilotes manquants dans le catalogue microsoft (à exécuter sur le poste en question avec accès au site https://catalog.update.microsoft.com):
  ```
  .\deploy_drivers.ps1 -scan
  ```

<br>

### Base de données des pilotes

Liste des pilotes manquant sous Windows 11 (23h2)

> :floppy_disk: <button id="downloadCSV">Exporter la base de donnée en .csv</button>
{.is-info}

| MODEL | NAME | ID | DRIVER | DDL | MIRROR | HQ |
|-------|------|----|--------|-----|--------|----|
| ANY | Carte vidéo de base Microsoft (NVIDIA T400 4GB) | PCI\VEN_10DE&DEV_1FF2&SUBSYS_16131028&REV_A1\4&50383AA&0&0008 | [NVIDIA - Display - 32.0.15.6109](https://www.nvidia.com/fr-fr/drivers) | [:floppy_disk:](https://fr.download.nvidia.com/Windows/561.09/561.09-desktop-win10-win11-64bit-international-nsd-dch-whql.exe) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/561.09-desktop-win10-win11-64bit-international-nsd-dch-whql.exe) | NON |
| ANY | Carte vidéo de base Microsoft (AMD Radeon RX 6500) | PCI\VEN_1002&DEV_743F&SUBSYS_00411028&REV_C3\6&109BD3E5&0&00000008 | [Adrenalin 25.6.1](https://www.amd.com/fr/support/downloads/drivers.html/graphics/radeon-rx/radeon-rx-6000-series/amd-radeon-rx-6500-xt.html) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/whql-amd-software-adrenalin-edition-25.6.1-win10-win11-june5-rdna.exe) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/whql-amd-software-adrenalin-edition-25.6.1-win10-win11-june5-rdna.exe) | NON |
| ANY | Carte vidéo de base Microsoft (Radeon RX550/550 Series) | PCI\VEN_1002&DEV_699F&SUBSYS_17121028&REV_C7\4&95ED034&0&0008 | [Advanced Micro Devices, Inc. - Display - 31.0.21912.14](https://www.amd.com/fr/support/downloads/previous-drivers.html/graphics/radeon-600-500-400/radeon-rx-500-series/radeon-rx-580.html) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/whql-amd-software-adrenalin-edition-24.3.1-win10-win11-mar20-vega-polaris.exe) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/whql-amd-software-adrenalin-edition-24.3.1-win10-win11-mar20-vega-polaris.exe) | NON |
| ANY | Carte vidéo de base Microsoft (Radeon RX550/550 Series) | PCI\VEN_1002&DEV_699F&SUBSYS_17121028&REV_C7\4&3318CD7F&0&0008 | [Advanced Micro Devices, Inc. - Display - 31.0.21912.14](https://www.amd.com/fr/support/downloads/previous-drivers.html/graphics/radeon-600-500-400/radeon-rx-500-series/radeon-rx-580.html) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/whql-amd-software-adrenalin-edition-24.3.1-win10-win11-mar20-vega-polaris.exe) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/whql-amd-software-adrenalin-edition-24.3.1-win10-win11-mar20-vega-polaris.exe) | NON |
| ANY | Carte vidéo de base Microsoft (AMD Radeon R7 450) | PCI\VEN_1002&DEV_682B&SUBSYS_10041028&REV_87\4&2A6FB196&0&0008 | [Advanced Micro Devices, Inc. - Display - 27.20.20913.2000](https://www.amd.com/fr/support/downloads/previous-drivers.html/graphics/radeon-600-500-400/radeon-rx-500-series/radeon-rx-580.html) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/radeon-software-adrenalin-2020-22.6.1-win10-win11-64bit-legacyasics-june23-2022-legacy.exe) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/radeon-software-adrenalin-2020-22.6.1-win10-win11-64bit-legacyasics-june23-2022-legacy.exe) | NON |
| OptiPlex SFF 7010 | Acquisition de données PCI et contrôleur de traitement du signal | PCI\VEN_8086&DEV_461D&SUBSYS_0BD21028&REV_05\3&11583659&0&20 | [Intel - System - 1.0.11406.42226](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_461D) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/02/5080a456-7f3d-4bf2-8f71-13a5bf33d75f_ccae92e0d6f3005e63f4a45cf0943823c3f84ccf.cab) |[:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/5080a456-7f3d-4bf2-8f71-13a5bf33d75f_ccae92e0d6f3005e63f4a45cf0943823c3f84ccf.cab) | NON |
| OptiPlex SFF 7010 | Contrôleur de bus SM | PCI\VEN_8086&DEV_7AA3&SUBSYS_0BD21028&REV_11\3&11583659&0&FC | [INTEL - System - 10.1.37.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AA3) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2022/08/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | NON |
| OptiPlex SFF 7010 | Contrôleur PCI de communications simplifiées | PCI\VEN_8086&DEV_7AE8&SUBSYS_0BD21028&REV_11\3&11583659&0&B0 | [Intel - System - 2406.5.5.0](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AE8) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2024/05/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | NON |
| OptiPlex SFF 7010 | Périphérique inconnu | ACPI\INTC1070\2&DABA3FF&1 | [Intel(R) Corporation - HIDClass - 2.2.2.10](https://catalog.update.microsoft.com/Search.aspx?q=ACPI%5CINTC1070) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/07/118f10cf-b66d-4106-8fad-feef05881973_75687b96195a598c58240d3e37dace5bf1674f9f.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/118f10cf-b66d-4106-8fad-feef05881973_75687b96195a598c58240d3e37dace5bf1674f9f.cab) | NON |
| OptiPlex SFF 7010 | Périphérique inconnu | ACPI\INTC1056\2&DABA3FF&1 | [Intel Corporation - System - 30.100.2417.30](https://catalog.update.microsoft.com/Search.aspx?q=ACPI%5CINTC1056) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/09/47849bb9-1389-4e3f-bea3-72702452a1bb_80adb2ab1a4de24d24be3d515ce8b9a65d8f06e8.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/47849bb9-1389-4e3f-bea3-72702452a1bb_80adb2ab1a4de24d24be3d515ce8b9a65d8f06e8.cab) | NON |
| OptiPlex SFF 7010 | Périphérique inconnu | ACPI\INTC1070\2&DABA3FF&2 | [Intel(R) Corporation - HIDClass - 2.2.2.10](https://catalog.update.microsoft.com/Search.aspx?q=ACPI%5CINTC1070) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/07/118f10cf-b66d-4106-8fad-feef05881973_75687b96195a598c58240d3e37dace5bf1674f9f.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/118f10cf-b66d-4106-8fad-feef05881973_75687b96195a598c58240d3e37dace5bf1674f9f.cab) | NON |
| OptiPlex SFF 7010 | Périphérique inconnu | ACPI\INTC1056\2&DABA3FF&2 | [Intel Corporation - System - 30.100.2417.30](https://catalog.update.microsoft.com/Search.aspx?q=ACPI%5CINTC1056) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/09/47849bb9-1389-4e3f-bea3-72702452a1bb_80adb2ab1a4de24d24be3d515ce8b9a65d8f06e8.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/47849bb9-1389-4e3f-bea3-72702452a1bb_80adb2ab1a4de24d24be3d515ce8b9a65d8f06e8.cab) | NON |
| OptiPlex SFF 7010 | Périphérique PCI | PCI\VEN_8086&DEV_7AA4&SUBSYS_0BD21028&REV_11\3&11583659&0&FD | [INTEL - System - 10.1.37.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AA4) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2022/08/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | NON |
| OptiPlex SFF 7010 | Périphérique système de base | PCI\VEN_8086&DEV_464F&SUBSYS_0BD21028&REV_05\3&11583659&0&40 | [Intel Corporation - System - 3.5.0.1578](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_464F) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/04/a5379279-e16b-4cc2-be9f-8ae603a066d6_e7f8b82bc82b92fa33f1c1f4774f49827b5d7d15.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/a5379279-e16b-4cc2-be9f-8ae603a066d6_e7f8b82bc82b92fa33f1c1f4774f49827b5d7d15.cab) | NON |
| OptiPlex SFF 7010 | Contrôleur vidéo (Intel(R) UHD Graphics 710) | PCI\VEN_8086&DEV_4693&SUBSYS_0BD21028&REV_0C\3&11583659&0&10 | [Intel Corporation - Display - 31.0.101.4953](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_4693%26SUBSYS_0AC51028) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/02/ac12bccd-37d2-4732-bed1-505a14f77085_78cb28c6e0f2b79c109b99e7af335d6ee85fa9dc.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/ac12bccd-37d2-4732-bed1-505a14f77085_78cb28c6e0f2b79c109b99e7af335d6ee85fa9dc.cab) | NON |
| OptiPlex 3000 | Acquisition de données PCI et contrôleur de traitement du signal | PCI\VEN_8086&DEV_461D&SUBSYS_0AC51028&REV_05\3&11583659&0&20 | [Intel - System - 1.0.11406.42226](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_461D) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/02/5080a456-7f3d-4bf2-8f71-13a5bf33d75f_ccae92e0d6f3005e63f4a45cf0943823c3f84ccf.cab) |[:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/5080a456-7f3d-4bf2-8f71-13a5bf33d75f_ccae92e0d6f3005e63f4a45cf0943823c3f84ccf.cab) | NON |
| OptiPlex 3000 | Contrôleur de bus SM | PCI\VEN_8086&DEV_7AA3&SUBSYS_0AC51028&REV_11\3&11583659&0&FC | [INTEL - System - 10.1.37.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AA3) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2022/08/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | NON |
| OptiPlex 3000 | Contrôleur PCI de communications simplifiées | PCI\VEN_8086&DEV_7AE8&SUBSYS_0AC51028&REV_11\3&11583659&0&B0 | [Intel - System - 2406.5.5.0](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AE8) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2024/05/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | NON |
| OptiPlex 3000 | Contrôleur vidéo (Intel(R) UHD Graphics 710) | PCI\VEN_8086&DEV_4693&SUBSYS_0AC51028&REV_0C\3&11583659&0&10 | [Intel Corporation - Display - 31.0.101.4953](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_4693%26SUBSYS_0AC51028) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/02/ac12bccd-37d2-4732-bed1-505a14f77085_78cb28c6e0f2b79c109b99e7af335d6ee85fa9dc.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/ac12bccd-37d2-4732-bed1-505a14f77085_78cb28c6e0f2b79c109b99e7af335d6ee85fa9dc.cab) | NON |
| OptiPlex 3000 | Périphérique inconnu | ACPI\INTC1056\2&DABA3FF&1 | [Intel Corporation - System - 30.100.2417.30](https://catalog.update.microsoft.com/Search.aspx?q=ACPI%5CINTC1056) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/09/47849bb9-1389-4e3f-bea3-72702452a1bb_80adb2ab1a4de24d24be3d515ce8b9a65d8f06e8.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/47849bb9-1389-4e3f-bea3-72702452a1bb_80adb2ab1a4de24d24be3d515ce8b9a65d8f06e8.cab) | NON |
| OptiPlex 3000 | Périphérique inconnu | USB\VID_8087&PID_0032\5&386D88AE&0&14 | [Intel Corporation - Bluetooth - 23.60.5.10](https://catalog.update.microsoft.com/Search.aspx?q=USB%5CVID_8087%26PID_0032) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2024/08/3ee65fef-122c-4a31-8e29-e4408b19a663_6f8fa2cb7b62b1689ce0f147a5f398340efdb796.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/3ee65fef-122c-4a31-8e29-e4408b19a663_6f8fa2cb7b62b1689ce0f147a5f398340efdb796.cab) | NON |
| OptiPlex 3000 | Périphérique PCI | PCI\VEN_8086&DEV_7AA4&SUBSYS_0AC51028&REV_11\3&11583659&0&FD | [INTEL - System - 10.1.37.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AA4) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2022/08/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | NON |
| OptiPlex 3000 | Périphérique système de base | PCI\VEN_8086&DEV_464F&SUBSYS_0AC51028&REV_05\3&11583659&0&40 | [Intel Corporation - System - 3.5.0.1578](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_464F) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/04/a5379279-e16b-4cc2-be9f-8ae603a066d6_e7f8b82bc82b92fa33f1c1f4774f49827b5d7d15.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/a5379279-e16b-4cc2-be9f-8ae603a066d6_e7f8b82bc82b92fa33f1c1f4774f49827b5d7d15.cab) | NON |
| ThinkCentre neo 50s Gen 3 | Acquisition de données PCI et contrôleur de traitement du signal | PCI\VEN_8086&DEV_461D&SUBSYS_32E617AA&REV_05\3&11583659&0&20 | [Intel - System - 1.0.11406.42226](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_461D) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/02/5080a456-7f3d-4bf2-8f71-13a5bf33d75f_ccae92e0d6f3005e63f4a45cf0943823c3f84ccf.cab) |[:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/5080a456-7f3d-4bf2-8f71-13a5bf33d75f_ccae92e0d6f3005e63f4a45cf0943823c3f84ccf.cab) | NON |
| ThinkCentre neo 50s Gen 3 | Contrôleur de bus SM | PCI\VEN_8086&DEV_7AA3&SUBSYS_32E617AA&REV_11\3&11583659&0&FC | [INTEL - System - 10.1.37.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AA3) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2022/08/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | NON |
| ThinkCentre neo 50s Gen 3 | Contrôleur PCI de communications simplifiées | PCI\VEN_8086&DEV_7AE8&SUBSYS_32E617AA&REV_11\3&11583659&0&B0 | [Intel - System - 2406.5.5.0](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AE8) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2024/05/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | NON |
| ThinkCentre neo 50s Gen 3 | Périphérique PCI | PCI\VEN_8086&DEV_7ACE&SUBSYS_32E617AA&REV_11\3&11583659&0&AA | [INTEL - System - 10.1.37.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AA4) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2022/08/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | NON |
| ThinkCentre neo 50s Gen 3 | Périphérique PCI | PCI\VEN_8086&DEV_7AAB&SUBSYS_32E617AA&REV_11\3&11583659&0&F3 | [INTEL - System - 10.1.37.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AA4) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2022/08/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | NON |
| ThinkCentre neo 50s Gen 3 | Périphérique PCI | PCI\VEN_8086&DEV_7ACD&SUBSYS_32E617AA&REV_11\3&11583659&0&A9 | [INTEL - System - 10.1.37.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AA4) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2022/08/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | NON |
| ThinkCentre neo 50s Gen 3 | Périphérique PCI | PCI\VEN_8086&DEV_7AA4&SUBSYS_32E617AA&REV_11\3&11583659&0&FD | [INTEL - System - 10.1.37.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AA4) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2022/08/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | NON |
| ThinkCentre neo 50s Gen 3 | Périphérique PCI | PCI\VEN_8086&DEV_7AFD&SUBSYS_32E617AA&REV_11\3&11583659&0&C9 | [INTEL - System - 10.1.37.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AA4) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2022/08/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | NON |
| ThinkCentre neo 50s Gen 3 | Périphérique PCI | PCI\VEN_8086&DEV_7ACF&SUBSYS_32E617AA&REV_11\3&11583659&0&AB | [INTEL - System - 10.1.37.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AA4) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2022/08/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | NON |
| ThinkCentre neo 50s Gen 3 | Périphérique PCI | PCI\VEN_8086&DEV_7ACC&SUBSYS_32E617AA&REV_11\3&11583659&0&A8 | [INTEL - System - 10.1.37.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AA4) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2022/08/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | NON |
| ThinkCentre neo 50s Gen 3 | Périphérique PCI | PCI\VEN_8086&DEV_7AFC&SUBSYS_32E617AA&REV_11\3&11583659&0&C8 | [INTEL - System - 10.1.37.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AA4) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2022/08/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/4b8b75db-f86c-40de-a51b-f731b69129a0_d3788d72f83e419edebb61f51f10eb382ac33ff7.cab) | NON |
| ThinkCentre neo 50s Gen 3 | Périphérique système de base | PCI\VEN_8086&DEV_464F&SUBSYS_32E617AA&REV_05\3&11583659&0&40 | [Intel Corporation - System - 3.5.0.1578](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_464F) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/04/a5379279-e16b-4cc2-be9f-8ae603a066d6_e7f8b82bc82b92fa33f1c1f4774f49827b5d7d15.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/a5379279-e16b-4cc2-be9f-8ae603a066d6_e7f8b82bc82b92fa33f1c1f4774f49827b5d7d15.cab) | NON |
| ThinkCentre neo 50s Gen 3 | Contrôleur vidéo (Intel(R) UHD Graphics 710) | PCI\VEN_8086&DEV_4693&SUBSYS_32E617AA&REV_0C\3&11583659&0&10 | [Intel Corporation - Display - 31.0.101.4953](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_4693%26SUBSYS_0AC51028) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/02/ac12bccd-37d2-4732-bed1-505a14f77085_78cb28c6e0f2b79c109b99e7af335d6ee85fa9dc.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/ac12bccd-37d2-4732-bed1-505a14f77085_78cb28c6e0f2b79c109b99e7af335d6ee85fa9dc.cab) | NON |
| OptiPlex 3080 | Acquisition de données PCI et contrôleur de traitement du signal | PCI\VEN_8086&DEV_A3B1&SUBSYS_09A81028&REV_00\3&11583659&0&A2 | [INTEL - System - 10.1.33.3](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A3B1) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2020/07/74395ee6-64fc-400a-8c1c-f305545e8e91_ca678480483052caa7f24332acc9ba1c4a368e19.cab) |[:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/74395ee6-64fc-400a-8c1c-f305545e8e91_ca678480483052caa7f24332acc9ba1c4a368e19.cab) | NON |
| OptiPlex 3080 | Contrôleur de bus SM | PCI\VEN_8086&DEV_A3A3&SUBSYS_09A81028&REV_00\3&11583659&0&FC | [INTEL - System - 10.1.33.3](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A3B1) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2020/07/74395ee6-64fc-400a-8c1c-f305545e8e91_ca678480483052caa7f24332acc9ba1c4a368e19.cab) |[:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/74395ee6-64fc-400a-8c1c-f305545e8e91_ca678480483052caa7f24332acc9ba1c4a368e19.cab) | NON |
| OptiPlex 3080 | Contrôleur de mémoire PCI | PCI\VEN_8086&DEV_A3A1&SUBSYS_09A81028&REV_00\3&11583659&0&FA | [INTEL - System - 10.1.33.3](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A3B1) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2020/07/74395ee6-64fc-400a-8c1c-f305545e8e91_ca678480483052caa7f24332acc9ba1c4a368e19.cab) |[:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/74395ee6-64fc-400a-8c1c-f305545e8e91_ca678480483052caa7f24332acc9ba1c4a368e19.cab) | NON |
| OptiPlex 3080 | Contrôleur PCI de communications simplifiées | PCI\VEN_8086&DEV_A3BA&SUBSYS_09A81028&REV_00\3&11583659&0&B0 | [Intel - System - 2406.5.5.0](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AE8) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2024/05/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | NON |
| OptiPlex 3080 | Périphérique système de base | PCI\VEN_8086&DEV_1911&SUBSYS_09A81028&REV_00\3&11583659&0&40 | [INTEL - System - 10.1.7.3](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_1911) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2018/12/6205aa6b-12e9-4298-ba23-aa06e5a8ef4d_f3604bed1e4b9176da6f7a045e5a76f46f506be9.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/6205aa6b-12e9-4298-ba23-aa06e5a8ef4d_f3604bed1e4b9176da6f7a045e5a76f46f506be9.cab) | NON |
| OptiPlex 3080 | Contrôleur vidéo (Intel(R) UHD Graphics 610) | PCI\VEN_8086&DEV_9BA8&SUBSYS_09A81028&REV_03\3&11583659&0&10 | [Intel Corporation - Display - 31.0.101.2127](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_9BA8) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/04/09e28ba3-4673-4e30-a6aa-51f0c7db008c_9a95d25c9c842facf7749e414b2ec35338e8de83.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/09e28ba3-4673-4e30-a6aa-51f0c7db008c_9a95d25c9c842facf7749e414b2ec35338e8de83.cab) | NON |
| OptiPlex 3070 | Acquisition de données PCI et contrôleur de traitement du signal | PCI\VEN_8086&DEV_A379&SUBSYS_09301028&REV_10\3&11583659&0&90 | [INTEL - System - 10.1.33.3](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A3B1) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2020/07/74395ee6-64fc-400a-8c1c-f305545e8e91_ca678480483052caa7f24332acc9ba1c4a368e19.cab) |[:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/74395ee6-64fc-400a-8c1c-f305545e8e91_ca678480483052caa7f24332acc9ba1c4a368e19.cab) | NON |
| OptiPlex 3070 | Contrôleur de bus SM | PCI\VEN_8086&DEV_A323&SUBSYS_09301028&REV_10\3&11583659&0&FC | [INTEL - System - 10.1.33.3](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A3B1) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2020/07/74395ee6-64fc-400a-8c1c-f305545e8e91_ca678480483052caa7f24332acc9ba1c4a368e19.cab) |[:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/74395ee6-64fc-400a-8c1c-f305545e8e91_ca678480483052caa7f24332acc9ba1c4a368e19.cab) | NON |
| OptiPlex 3070 | Contrôleur PCI de communications simplifiées | PCI\VEN_8086&DEV_A360&SUBSYS_09301028&REV_10\3&11583659&0&B0 | [Intel - System - 2406.5.5.0](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AE8) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2024/05/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | NON |
| OptiPlex 3070 | Périphérique PCI | PCI\VEN_8086&DEV_A324&SUBSYS_09301028&REV_10\3&11583659&0&FD | [INTEL - System - 10.1.16.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A324) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2019/08/aba36574-3af7-4c06-9dbf-99f958205e71_34932f041b994a914218407603299b18c0c5ba80.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/aba36574-3af7-4c06-9dbf-99f958205e71_34932f041b994a914218407603299b18c0c5ba80.cab) | NON |
| OptiPlex 3070 | Périphérique système de base | PCI\VEN_8086&DEV_1911&SUBSYS_09301028&REV_00\3&11583659&0&40 | [INTEL - System - 10.1.7.3](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_1911) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2018/12/6205aa6b-12e9-4298-ba23-aa06e5a8ef4d_f3604bed1e4b9176da6f7a045e5a76f46f506be9.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/6205aa6b-12e9-4298-ba23-aa06e5a8ef4d_f3604bed1e4b9176da6f7a045e5a76f46f506be9.cab) | NON |
| OptiPlex 3070 | Contrôleur vidéo (Intel(R) UHD Graphics 630) | PCI\VEN_8086&DEV_3E91&SUBSYS_09301028&REV_00\3&11583659&0&10 | [Intel Corporation - Display - 31.0.101.2127](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_9BA8) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/04/09e28ba3-4673-4e30-a6aa-51f0c7db008c_9a95d25c9c842facf7749e414b2ec35338e8de83.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/09e28ba3-4673-4e30-a6aa-51f0c7db008c_9a95d25c9c842facf7749e414b2ec35338e8de83.cab) | NON |
| OptiPlex 3060 | Acquisition de données PCI et contrôleur de traitement du signal | PCI\VEN_8086&DEV_A379&SUBSYS_085C1028&REV_10\3&11583659&0&90 | [INTEL - System - 10.1.16.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A324) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2019/08/aba36574-3af7-4c06-9dbf-99f958205e71_34932f041b994a914218407603299b18c0c5ba80.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/aba36574-3af7-4c06-9dbf-99f958205e71_34932f041b994a914218407603299b18c0c5ba80.cab) | OUI |
| OptiPlex 3060 | Contrôleur de bus SM | PCI\VEN_8086&DEV_A323&SUBSYS_085C1028&REV_10\3&11583659&0&FC | [INTEL - System - 10.1.16.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A324) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2019/08/aba36574-3af7-4c06-9dbf-99f958205e71_34932f041b994a914218407603299b18c0c5ba80.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/aba36574-3af7-4c06-9dbf-99f958205e71_34932f041b994a914218407603299b18c0c5ba80.cab) | OUI |
| OptiPlex 3060 | Contrôleur PCI de communications simplifiées | PCI\VEN_8086&DEV_A360&SUBSYS_085C1028&REV_10\3&11583659&0&B0 | [Intel - System - 2406.5.5.0](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AE8) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2024/05/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | OUI |
| OptiPlex 3060 | Périphérique PCI | PCI\VEN_8086&DEV_A324&SUBSYS_085C1028&REV_10\3&11583659&0&FD | [INTEL - System - 10.1.16.7](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A324) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2019/08/aba36574-3af7-4c06-9dbf-99f958205e71_34932f041b994a914218407603299b18c0c5ba80.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/aba36574-3af7-4c06-9dbf-99f958205e71_34932f041b994a914218407603299b18c0c5ba80.cab) | OUI |
| OptiPlex 3060 | Périphérique système de base | PCI\VEN_8086&DEV_1911&SUBSYS_085C1028&REV_00\3&11583659&0&40 | [INTEL - System - 10.1.7.3](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_1911) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2018/12/6205aa6b-12e9-4298-ba23-aa06e5a8ef4d_f3604bed1e4b9176da6f7a045e5a76f46f506be9.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/6205aa6b-12e9-4298-ba23-aa06e5a8ef4d_f3604bed1e4b9176da6f7a045e5a76f46f506be9.cab) | OUI |
| OptiPlex 3060 | Contrôleur vidéo (Intel(R) UHD Graphics 630) | PCI\VEN_8086&DEV_3E91&SUBSYS_085C1028&REV_00\3&11583659&0&10 | [Intel Corporation - Display - 31.0.101.2127](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_9BA8) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/04/09e28ba3-4673-4e30-a6aa-51f0c7db008c_9a95d25c9c842facf7749e414b2ec35338e8de83.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/09e28ba3-4673-4e30-a6aa-51f0c7db008c_9a95d25c9c842facf7749e414b2ec35338e8de83.cab) | OUI |
| OptiPlex 3050 | Acquisition de données PCI et contrôleur de traitement du signal | PCI\VEN_8086&DEV_A2B1&SUBSYS_07A31028&REV_00\3&11583659&0&A2 | [INTEL - System - 10.1.1.44](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A131) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2020/08/9972fce6-371c-4c3b-9d4c-07b35e1e339a_5c9187451b6c2ff2a499957a8be234c8c115e64e.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/9972fce6-371c-4c3b-9d4c-07b35e1e339a_5c9187451b6c2ff2a499957a8be234c8c115e64e.cab) | OUI |
| OptiPlex 3050 | Contrôleur de bus SM | PCI\VEN_8086&DEV_A2A3&SUBSYS_07A31028&REV_00\3&11583659&0&FC | [INTEL - System - 10.1.1.44](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A131) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2020/08/9972fce6-371c-4c3b-9d4c-07b35e1e339a_5c9187451b6c2ff2a499957a8be234c8c115e64e.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/9972fce6-371c-4c3b-9d4c-07b35e1e339a_5c9187451b6c2ff2a499957a8be234c8c115e64e.cab) | OUI |
| OptiPlex 3050 | Contrôleur de mémoire PCI | PCI\VEN_8086&DEV_A2A1&SUBSYS_07A31028&REV_00\3&11583659&0&FA | [INTEL - System - 10.1.1.44](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A131) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2020/08/9972fce6-371c-4c3b-9d4c-07b35e1e339a_5c9187451b6c2ff2a499957a8be234c8c115e64e.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/9972fce6-371c-4c3b-9d4c-07b35e1e339a_5c9187451b6c2ff2a499957a8be234c8c115e64e.cab) | OUI |
| OptiPlex 3050 | Contrôleur PCI de communications simplifiées | PCI\VEN_8086&DEV_A2BA&SUBSYS_07A31028&REV_00\3&11583659&0&B0 | [Intel - System - 2406.5.5.0](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_7AE8) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2024/05/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | OUI |
| OptiPlex 3050 | Contrôleur vidéo (Intel(R) HD Graphics 530) | PCI\VEN_8086&DEV_1912&SUBSYS_07A31028&REV_06\3&11583659&0&10 | [Intel Corporation - Display - 31.0.101.2114](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_1912%26SUBSYS_07A31028) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/06/56b50043-be12-4105-8983-0b428ef78f8e_b11f6386790dc1fdde4a06ead5766329afb31540.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/56b50043-be12-4105-8983-0b428ef78f8e_b11f6386790dc1fdde4a06ead5766329afb31540.cab) | OUI |
| HP ProDesk 600 G2 SFF | Acquisition de données PCI et contrôleur de traitement du signal | PCI\VEN_8086&DEV_A131&SUBSYS_805D103C&REV_31\3&11583659&0&A2 | [INTEL - System - 10.1.1.44](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A131) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2020/08/9972fce6-371c-4c3b-9d4c-07b35e1e339a_5c9187451b6c2ff2a499957a8be234c8c115e64e.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/9972fce6-371c-4c3b-9d4c-07b35e1e339a_5c9187451b6c2ff2a499957a8be234c8c115e64e.cab) | OUI |
| HP ProDesk 600 G2 SFF | Contrôleur de bus SM | PCI\VEN_8086&DEV_A123&SUBSYS_805D103C&REV_31\3&11583659&0&FC | [INTEL - System - 10.1.1.44](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A131) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2020/08/9972fce6-371c-4c3b-9d4c-07b35e1e339a_5c9187451b6c2ff2a499957a8be234c8c115e64e.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/9972fce6-371c-4c3b-9d4c-07b35e1e339a_5c9187451b6c2ff2a499957a8be234c8c115e64e.cab) | OUI |
| HP ProDesk 600 G2 SFF | Contrôleur de mémoire PCI | PCI\VEN_8086&DEV_A121&SUBSYS_805D103C&REV_31\3&11583659&0&FA | [INTEL - System - 10.1.1.44](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A131) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2020/08/9972fce6-371c-4c3b-9d4c-07b35e1e339a_5c9187451b6c2ff2a499957a8be234c8c115e64e.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/9972fce6-371c-4c3b-9d4c-07b35e1e339a_5c9187451b6c2ff2a499957a8be234c8c115e64e.cab) | OUI |
| HP ProDesk 600 G2 SFF | Contrôleur PCI de communications simplifiées | PCI\VEN_8086&DEV_A13A&SUBSYS_805D103C&REV_31\3&11583659&0&B0 | [Intel - System - 2406.5.5.0](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_A13A) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2024/05/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | OUI |
| HP ProDesk 600 G2 SFF | Carte vidéo de base Microsoft (Intel(R) HD Graphics 510) | PCI\VEN_8086&DEV_1902&SUBSYS_805D103C&REV_06\3&11583659&0&10 | [Intel Corporation - Display - 31.0.101.2114](https://catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_1912%26SUBSYS_07A31028) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2024/06/56b50043-be12-4105-8983-0b428ef78f8e_b11f6386790dc1fdde4a06ead5766329afb31540.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/56b50043-be12-4105-8983-0b428ef78f8e_b11f6386790dc1fdde4a06ead5766329afb31540.cab) | OUI |
| Latitude 3520 | Périphérique PCI | PCI\VEN_8086&DEV_34A4&SUBSYS_0A851028&REV_30\3&11583659&0&FD | [INTEL - System - 10.1.12.3](https://www.catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_34A4) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2020/04/fd41d84a-8c90-4502-b066-ceaab8d3e955_23c80e098c5b0f2ee1a8a89e045e05cd93798220.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/fd41d84a-8c90-4502-b066-ceaab8d3e955_23c80e098c5b0f2ee1a8a89e045e05cd93798220.cab) | NON |
| Latitude 3520 | Contrôleur PCI de communications simplifiées | PCI\VEN_8086&DEV_34E0&SUBSYS_0A851028&REV_30\3&11583659&0&B0 | [Intel - System - 2406.5.5.0](https://www.catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_34E0) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2024/05/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/6b4eea96-19d1-4e45-8184-b15e69e7d61b_342c6ed92ad70a59065201bacbef8f85fc8413ca.cab) | NON |
| Latitude 3520 | Carte vidéo de base Microsoft | PCI\VEN_8086&DEV_8A56&SUBSYS_0A851028&REV_07\3&11583659&0&10 | [Intel Corporation - Display - 31.0.101.2134](https://www.catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_8A56%26SUBSYS_0A851028) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2025/05/d665b283-1cb9-4bfe-b17f-e12a9455c21d_afa1cc768119e434961f5b19fba2174badd05a31.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/d665b283-1cb9-4bfe-b17f-e12a9455c21d_afa1cc768119e434961f5b19fba2174badd05a31.cab) | NON |
| Latitude 3520 | Contrôleur de bus SM | PCI\VEN_8086&DEV_34A3&SUBSYS_0A851028&REV_30\3&11583659&0&FC | [INTEL - System - 10.1.12.2](https://www.catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_34A3) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2019/08/7cd77a37-4c6f-409a-b432-d97699b09192_f362eb9b86a3b3f531e9cfc1897b2a5dc2669719.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/7cd77a37-4c6f-409a-b432-d97699b09192_f362eb9b86a3b3f531e9cfc1897b2a5dc2669719.cab) | NON |
| Latitude 3520 | Port série PCI | PCI\VEN_8086&DEV_34FC&SUBSYS_0A851028&REV_30\3&11583659&0&90 | [Intel - System - 3.1.0.4349](https://www.catalog.update.microsoft.com/Search.aspx?q=PCI%5CVEN_8086%26DEV_34FC) | [:floppy_disk:](https://catalog.s.download.windowsupdate.com/d/msdownload/update/driver/drvs/2021/01/7ac611f7-8a7e-4f98-8f38-ced6bb394ffb_1c7dcd29d1137e00169cbd665525b248812580ea.cab) | [:floppy_disk:](https://files.mydedibox.fr/files/work/wiki/pilotes/7ac611f7-8a7e-4f98-8f38-ced6bb394ffb_1c7dcd29d1137e00169cbd665525b248812580ea.cab) | NON |
{.dense}
