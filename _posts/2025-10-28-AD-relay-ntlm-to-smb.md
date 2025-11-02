---
title: "Attack de relai - vers SMB et d'autres protocols"
categories:
  - Active Directory
tags:
  - Active Directory
  - relai
  - SMB
layout: post
---

Pour ce poste, je vais parler des relai vers le protocol SMB. Le SMB n'est pas comme du HTTP, il a la capacité de signer des paquets, ce qui nous oblige de chercher des "destinataires" qui n'obligent pas cette signature. 


## Prérequis
- un accès en interne
- LLMNR/NBTNS ou IPv6 actif
- un peu de chance et de patience


## Attaque 

### Enumérer les posts qui ne demandent pas la signature SMB
[netexec](https://www.netexec.wiki/) est un magnifique tool. On peut énumérer les postes donc la signature SMB n'est pas obligatoire avec nxc

Dans mon lab, je n'ai 1 seul serveur que le SMB n'est pas obligatoire. (mon Serveur s'appelle WIN-blabla car je n'ai pas fait le renommage quand je le joins à mon domain. )
![gen_relay_list](/assets/images/AD/from smb to smb not sign/nxc gen relay list.png)

### Une authentification chanceuse


Start responder et ntlmrelayx en mode socks
![ntlmrelayx](/assets/images/AD/from smb to smb not sign/start ntlmrelayx.png)

Mon proxychains utilise déjà le port 1080 - port par défaut pour les proxy de ntlmrelayx, je n'ai pas besoin de changer mes configs. Ici on voir que j'ai capturé une authentification d'admin du dom

![socks_admin_auth](/assets/images/AD/from smb to smb not sign/socks auth smb admin.png)

Grâce à ça, je peux relayer vers le serveur cible et dump la base sam local de ce serveur

![secretsdumps](/assets/images/AD/from smb to smb not sign/secretdumps with admin session.png)

Nous pouvons ensuite soit faire le mouvement latéral, si on a de la chance, on peut capturer une session d'admin sur ce serveur, et donc capturer son hash NT. 

### Une authentification d'un utilisateur classique

Si nous n'avons pas de chance, faut tout simplement chercher plus dans les shares, peut-être il y a des informations intéressantes dedans ! 


