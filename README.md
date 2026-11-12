# SOLEA 2 – Infrastructure réseau et systèmes multi-sites

Projet de fin d'année de Licence 3 Télécommunications et Réseaux Informatiques
(Université Savoie Mont Blanc, 2025/2026) : concevoir, déployer et exploiter de bout en bout
l'infrastructure informatique d'une entreprise fictive répartie sur deux sites.

## Contexte

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

### Architecture de l'infrastructure
- Architecture physique et logique des deux sites sur un seul serveur : **7 ponts virtuels Proxmox**
  (vmbr10, vmbr30, vmbr40 pour Aix, vmbr20 et vmbr200 pour Valence, vmbr140 et vmbr170 vers le
  réseau de l'université) qui isolent les VLAN en mutualisant le matériel
- **Plan d'affectation des VLAN sur le commutateur d'Aix** (12 ports) : 2 ports trunk
  (hyperviseur et routeur de routage inter-VLAN), ports access pour les postes admin (VLAN 10),
  les serveurs (VLAN 30), les postes employés (VLAN 40) et la borne Wi-Fi invités (VLAN 70)
- **Plan d'adressage IPv4** : un /24 par VLAN, passerelle toujours en `.1`, plages DHCP définies
  par VLAN ; plan IPv6 associé
- **Inventaire des machines virtuelles de production** (plus de 20 VM) servant de périmètre à la recette
- Schéma global de l'infrastructure (deux sites, routeur central de l'université, routeurs Cisco,
  tunnel inter-sites, Proxmox, VLAN)

### Windows Server 2022 et Active Directory
- Déploiement de **Windows Server 2022** sur une VM Proxmox (`Aix-AD-serveur`, VLAN 30, `192.168.30.7`)
- Promotion en **contrôleur de domaine** : domaine `ad.solea2.local`
- Structure de l'annuaire : **unités d'organisation par site** (OU Aix, OU Valence), comptes
  administrateurs par site (`admin.aix`, `admin.valence`) et compte employé standard pour tester
  les permissions
- **Serveur de fichiers SMB** : partage `\\192.168.30.7\Partage_Solea2`, avec un fichier sensible
  pour valider la sécurité
- **Sécurité NTFS** : rupture d'héritage (`icacls /inheritance:r`) et application du principe de
  moindre privilège. Seuls les administrateurs ont accès au fichier sensible ; l'employé standard
  est refusé. Le partage SMB est ouvert en lecture et la vraie sécurité est portée par NTFS.
- **Tests et audit depuis le site de Valence** : accès refusé pour l'employé, accès autorisé pour
  l'administrateur, avec captures à l'appui
- **Troubleshooting** : cache d'identifiants SMB de Windows qui faussait les tests (résolu avec
  `net use * /delete` et `cmdkey`), règle de pare-feu Windows pour autoriser le ping ICMPv4,
  vérification du routage inter-sites et de l'état des comptes AD

## Services déployés sur le projet

| Brique | Technologies |
|---|---|
| Virtualisation | Dell PowerEdge T340, iDRAC, RAID 1, Proxmox VE, alertes mail |
| Réseau | Cisco IOS, VLAN, routage inter-VLAN, GRE, NAT/PAT, DHCP relayé |
| IPv6 | Dual-stack, RIPng, routage statique inter-sites |
| Services réseau | DHCP (isc-dhcp-server), DNS interne BIND9 (`solea.local`), Nginx + HTTPS |
| Téléphonie | Asterisk (PJSIP), plan de numérotation, appels inter-sites |
| Administration | Bastion SSH, rsyslog centralisé, Zabbix |
| Identité | Active Directory, serveur de fichiers SMB, permissions NTFS |
| Industrialisation | Scripts shell, sauvegardes cron, page d'administration intranet, recette |

## Difficultés rencontrées

- **Filtrage IPv6 de l'université** : le préfixe alloué était bloqué silencieusement par le routeur
  de bordure, ce qui a conduit à une architecture hybride avec deux préfixes
- **Autoconfiguration IPv6 (SLAAC) / Netplan** sur les hôtes Linux
- **Cache d'identifiants SMB Windows** qui masquait les résultats des tests de droits

## Résultats

- Recette technique automatisée : **72 tests exécutés, 70 OK**
- Sauvegarde automatique des configurations critiques, avec test de restauration
- Page d'administration intranet affichant l'état de l'infrastructure

## Perspectives

Finalisation de la supervision Zabbix, de la mise en service complète de l'Active Directory et du
Wi-Fi sécurisé (RADIUS 802.1X + VLAN invité isolé).

## Compétences mises en œuvre

Architecture réseau multi-sites · VLAN · Cisco IOS · Proxmox VE · Windows Server 2022 ·
Active Directory · SMB / NTFS · Debian · BIND9 · Nginx · Asterisk · IPv6 · Bash ·
Troubleshooting · Documentation technique

## Documentation

Rapport complet (101 pages) : `Rapport_final_Solea_2.pdf`
