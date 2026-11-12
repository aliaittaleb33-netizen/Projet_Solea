# SOLEA 2 – Infrastructure réseau et systèmes multi-sites

Projet de fin d'année de Licence 3 Télécommunications et Réseaux Informatiques
(Université Savoie Mont Blanc, 2025/2026) : concevoir, déployer et exploiter de bout en bout
l'infrastructure informatique d'une entreprise fictive répartie sur deux sites.

## Contexte
Schéma de l'infrastructure logique du réseau Solea
<img width="1207" height="701" alt="image" src="https://github.com/user-attachments/assets/104d6633-0f0c-4c08-b0ab-c7a9705f9981" />

Schéma de l'infrastructure physique du réseau Solea
<img width="596" height="342" alt="image" src="https://github.com/user-attachments/assets/81d05ca0-5ba9-497a-891e-1d5407fd3345" />


**SOLEA** est une PME fictive spécialisée dans la vente et l'installation de panneaux solaires :

- **Aix-les-Bains** : siège social, environ 300 collaborateurs. Il centralise les services
  (DHCP, DNS, Web, téléphonie, journalisation, supervision, Active Directory) et héberge
  l'hyperviseur.
- **Valence** : antenne d'environ 30 collaborateurs, reliée au siège par un tunnel et utilisant
  les services centralisés.

Point de départ du scénario : une infrastructure vieillissante, sans segmentation, peu sécurisée,
avec des services dispersés et une forte dépendance aux opérateurs pour relier les sites.
Le projet consiste à la remplacer par une architecture professionnelle, documentée et testable.

## Objectifs

- Segmentation par VLAN selon les rôles (administration, serveurs, clients, invités, VoIP)
- Interconnexion sécurisée des deux sites (tunnel GRE, puis IPv6 dual-stack)
- Services centralisés : DHCP, DNS interne, serveur Web HTTPS, publication via NAT statique
- Téléphonie sur IP entre les deux sites
- Gestion des identités et partage de fichiers sécurisé (Active Directory, SMB, NTFS)
- Administration et sécurité : bastion SSH, journalisation centralisée, supervision
- Industrialisation : sauvegardes automatiques et cahier de recette exécutable

## Démarche

1. **Étude et simulation sous Cisco Packet Tracer** : validation de l'architecture avant tout
   déploiement (VLAN, routage inter-VLAN, DHCP relayé, NAT/PAT, tunnel GRE, IPv6).
2. **Déploiement physique** sur un serveur Dell PowerEdge T340 (iDRAC, RAID 1) avec l'hyperviseur
   **Proxmox VE**, un commutateur Cisco et des routeurs Cisco.
3. **Mise en production des services** sur des VM Debian et Windows Server.
4. **Industrialisation** : scripts d'automatisation, sauvegardes, recette automatisée.

Le groupe de six étudiants était réparti en deux sous-groupes qui relisaient et reprenaient
le travail de l'autre (revue croisée).

## Ma contribution (Ali Aittaleb)

### Architecture logique et physique
- Conception de l'architecture des deux sites sur un seul serveur physique : **7 ponts virtuels
  Proxmox** (vmbr10, vmbr30, vmbr40 pour Aix ; vmbr20 et vmbr200 pour Valence ; vmbr140 et vmbr170
  vers le réseau de l'université) qui isolent les VLAN en mutualisant le matériel
- **Plan d'affectation des VLAN sur le commutateur d'Aix** (12 ports) : 2 ports trunk (hyperviseur
  et routeur), ports access pour les postes admin (VLAN 10), les serveurs (VLAN 30), les postes
  employés (VLAN 40) et la borne Wi-Fi invités (VLAN 70)
- **Plan d'adressage IPv4** (un /24 par VLAN, passerelle en `.1`, plages DHCP par VLAN) et
  plan d'adressage IPv6
- **Inventaire des machines virtuelles de production** (plus de 20 VM)
- Schéma global de l'infrastructure : deux sites, routeur central de l'université, routeurs Cisco,
  tunnel inter-sites, Proxmox, VLAN

### Virtualisation : serveur Dell T340, iDRAC et Proxmox VE
- Prise en main et configuration de l'**iDRAC** (accès distant, mise à jour du firmware,
  alertes mail SMTP)
  <img width="426" height="295" alt="image" src="https://github.com/user-attachments/assets/23282f75-f303-4ade-a3e2-c47c22281ecc" />

- Configuration du **RAID 1** et installation de **Proxmox VE**
- Configuration réseau dynamique de l'hyperviseur sur le VLAN de management de l'université
- Création d'un **template de VM Debian**, de comptes Proxmox par étudiant, des bridges réseau
  pour les VLAN, et test de sauvegarde ZSTD sur un second SSD

### Configuration réseau : commutateur et routeurs
- Configuration du **commutateur** (VLAN, ports trunk / access) et des **routeurs Cisco** d'Aix
  et de Valence
- **Routage inter-VLAN** par sous-interfaces et **DHCP relayé** (`ip helper-address`)
- **Tunnel GRE** entre les deux sites
- **NAT/PAT** pour l'accès Internet et **NAT statique** pour la publication du serveur Web
- **IPv6 dual-stack** : plan d'adressage, routage dynamique RIPng et routage statique inter-sites
- Configurations d'abord validées en simulation sous Packet Tracer, puis transposées en production

### Téléphonie sur IP
- Déploiement d'un **IPBX Asterisk** sur Debian 13, centralisé à Aix
- Comptes **PJSIP** avec séparation entre numéro d'extension et identifiants d'authentification
  (bonne pratique contre l'énumération)
- **Plan de numérotation** : extensions 100 (Aix), 200 (Valence) et 999 (test d'écho)
- **Provisionnement des téléphones IP par DHCP (option 150)**
- Validation : appels inter-sites Aix ↔ Valence, flux RTP, rechargement à chaud de la configuration

### Postes de travail, terminaux et Wi-Fi
- Configuration des **postes clients** Linux et Windows sur les deux sites (adressage DHCP / statique,
  Netplan, résolution DNS, softphones Linphone)
- Raccordement du **point d'accès Wi-Fi invités** (VLAN 70) et définition de son scope DHCP
- Tests de connectivité, de résolution de noms et d'appels depuis les postes des deux sites

### Windows Server 2022 et Active Directory
- Déploiement de **Windows Server 2022** sur une VM Proxmox (`Aix-AD-serveur`, VLAN 30)
- Promotion en **contrôleur de domaine** : domaine `ad.solea2.local`
- **Unités d'organisation par site** (Aix, Valence), comptes administrateurs et utilisateur standard
- **Serveur de fichiers SMB** (`Partage_Solea2`) avec **sécurité NTFS** : rupture d'héritage
  (`icacls /inheritance:r`), moindre privilège, accès refusé à l'employé standard
- **Audit depuis Valence** : accès refusé / accès autorisé selon le compte, avec captures
- Troubleshooting : cache d'identifiants SMB de Windows (`net use`, `cmdkey`), règle de pare-feu
  ICMPv4, vérification du routage inter-sites

## Services déployés sur le projet

| Brique | Technologies |
|---|---|
| Virtualisation | Dell PowerEdge T340, iDRAC, RAID 1, Proxmox VE |
| Réseau | Cisco IOS, VLAN, routage inter-VLAN, GRE, NAT/PAT, DHCP relayé |
| IPv6 | Dual-stack, RIPng, routage statique inter-sites |
| Services réseau | DHCP (isc-dhcp-server), DNS BIND9 (`solea.local`), Nginx + HTTPS |
| Téléphonie | Asterisk (PJSIP), Linphone, option DHCP 150 |
| Administration | Bastion SSH, rsyslog centralisé, Zabbix |
| Identité | Active Directory, SMB, permissions NTFS |
| Industrialisation | Scripts shell, sauvegardes cron, page d'administration intranet, recette automatisée |

## Difficultés rencontrées

- **Filtrage IPv6 de l'université** : le préfixe alloué était bloqué silencieusement par le routeur
  de bordure, ce qui a conduit à une architecture hybride avec deux préfixes
- **Autoconfiguration IPv6 (SLAAC) / Netplan** sur les hôtes Linux
- **Cache d'identifiants SMB Windows** qui faussait les tests de droits

## Résultats

- Recette technique automatisée : **72 tests exécutés, 70 OK**
- Appels VoIP inter-sites fonctionnels, services réseau accessibles depuis les deux sites
- Sauvegarde automatique des configurations critiques, avec test de restauration

## Perspectives

Finalisation de la supervision Zabbix, de la mise en service complète de l'Active Directory et du
Wi-Fi sécurisé (RADIUS 802.1X + VLAN invité isolé).

## Compétences mises en œuvre

Architecture réseau multi-sites · Cisco IOS · VLAN · GRE · NAT · IPv6 · Proxmox VE · iDRAC ·
Windows Server 2022 · Active Directory · SMB / NTFS · Asterisk / VoIP · Debian · BIND9 · Nginx ·
Bash · Troubleshooting · Documentation technique

## Documentation

Rapport complet (101 pages) : `Rapport_final_Solea_2.pdf`
