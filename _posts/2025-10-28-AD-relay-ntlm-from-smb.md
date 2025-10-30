---
title: "Attack de relai - depuis SMB"
categories:
  - Active Directory
tags:
  - Active Directory
  - relai
  - SMB
layout: post
---

Pour ce poste, je vais parler des relai à partir des flux de SMB. Le SMB n'est pas comme du HTTP, il a la capacité de signer des paquets, ce qui nous oblige de chercher des "destinataires" qui n'obligent pas cette signature


## Prérequis
- un accès en interne
- LLMNR/NBTNS ou IPv6 actif
- un peu de chance et de patience


## Attaque

### Enumérer les posts qui ne demandent pas la signature SMB
[netexec](https://www.netexec.wiki/) est un magnifique tool. On peut énumérer les postes donc la signature SMB n'est pas obligatoire avec nxc

Dans mon lab, je n'ai 1 seul serveur que le SMB n'est pas obligatoire. (mon Serveur s'appelle WIN-blabla car je n'ai pas fait le renommage quand je le joins à mon domain. )
![gen_relay_list](/assets/images/AD/from smb to smb not sign/nxc gen relay list.png)

### Une session chanceuse

Une session admin du domaine! 


### Une session d'utilisateur classique


## Conclusion


