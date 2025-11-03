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

Pour ce poste, je vais parler des relai vers le protocol SMB. Le SMB a la capacité de signer des paquets, ce qui nous oblige de chercher des "destinataires" qui n'obligent pas cette signature. 
Dans mon lab, mon Windows server ne demande pas de signature SMB, ça me suffit pour les tests. Je peux faire le relai depuis mon CLIENT01 vers mon serveur. 
Pour les flux vennant des protocols qu'ils n'ont pas la capacité de signé comme le HTTP, le relai passe sans soucis. 


## Prérequis
- un accès en interne
- LLMNR/NBTNS ou IPv6 actif
- un peu de chance et de patience

## Attaque 

### Enumérer les posts qui ne demandent pas la signature SMB
[netexec](https://www.netexec.wiki/) est un magnifique tool. On peut énumérer les postes donc la signature SMB n'est pas obligatoire avec nxc

Dans mon lab, je n'ai 1 seul serveur que le SMB n'est pas obligatoire. (mon Serveur s'appelle WIN-blabla car je n'ai pas fait le renommage quand je le joins à mon domain. )
![gen_relay_list](/assets/images/AD/to smb not sign/nxc gen relay list.png)

### Une authentification chanceuse


Start responder et ntlmrelayx en mode socks
![ntlmrelayx](/assets/images/AD/to smb not sign/start ntlmrelayx.png)

Mon proxychains utilise déjà le port 1080 - port par défaut pour les proxy de ntlmrelayx, je n'ai pas besoin de changer mes configs. Ici on voir que j'ai capturé une authentification d'admin du dom

![socks_admin_auth](/assets/images/AD/to smb not sign/socks auth smb admin.png)

Grâce à ça, je peux relayer vers le serveur cible et dump la base sam local de ce serveur

![secretsdumps](/assets/images/AD/to smb not sign/secretdumps with admin session.png)

Nous pouvons ensuite soit faire le mouvement latéral, si on a de la chance, on peut capturer une session d'admin sur ce serveur, et donc capturer son hash NT. 

### Une authentification d'un utilisateur classique

Si nous n'avons pas de chance, faut tout simplement chercher plus dans les shares, peut-être il y a des informations intéressantes dedans ! 

Dans mon lab, j'ai crée un dossier partage qui se situe sur le bureau de mon utilistateur adm_antinio. ce dossier est partagé avec l'utilisateur f.dejong. Un [mini blog poste](/posts/AD-config-share-folder/) est réservé juste pour la création de ce share. 

On lance responder et ntlmrelayx en mode socks pour on fait une petite manipulation depuis CLIENT01, nous capturons une session de f.dejong, et comme d'habitude, on va faire le relai vers smb de mon serveur. Comme on peut voir à droite, j'ai réussi à capturer puis relayer l'authentification de f.dejong vers le serveur avec ntlmrelayx et proxychains
![relai_vers_smb_with_normal_account](/assets/images/AD/to smb not sign/smbclient relay with f_dejong auth.png)


![contenu](/assets/images/AD/to smb not sign/content of secret.png)

## Vers les autres protocols ? 

Parfois on peut avoir des sessions liés à un autre protocol comme MSSQL, cequi nous permettre de faire quelques requêtes en bdd. Mais dans mon lab, malheureusement je n'ai pas réussit à configurer un serveur mssql pour pouvoir tester.... Promis, je ferai quand j'aurai le temps ! 

## Conclusion

En filtrant sur les postes qui ne demandent pas de la signature SMB, nous pouvons relayer l'authentification et de pouvoir avancer dans nos tests. 

Si vous avez des remarques, des choses que je n'ai pas bien compris, je suis preneur d'avoir vos avis ! 