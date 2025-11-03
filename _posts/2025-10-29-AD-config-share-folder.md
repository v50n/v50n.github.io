---
title: "Créer un dossier partagé"
categories:
  - Active Directory
tags:
  - Active Directory
  - relai
  - share
layout: post
---

Pendant que je configure un répertoire de partage pour les tests de relai vers SMB, je me trouve dans une situation "bizarre". C'est pour cela que je veux partager avec vous ici. 

Au début, j'ai essayé avec cette commande en pensant que j'ai donné aussi le droit de full control à tout le monde.
```powershell
net share share=driver:"PATH_to_share" /grant:everyone,full
```

Mais quand je vérifie les droits d'accès vers ce folder, je vois bien le répertoire mais je n'ai aucun droit dessus. 
![nxc_check_before](/assets/images/AD/to smb not sign/config share/nxc enum dont have permission.png)

Très bizarre ! Après des [recherches](https://anadoxin.org/blog/windows-permissions-cheat-sheet/) et de l'aide par un ami, il s'avère quand on crée un share avec net share, on ouvre tout simplement le network vers ce share, et le ```/grant:everyone,full``` est juste pour dire que le 1er filtre de droit d'accès. Mais pour vraiment donner les droits d'écriture/lecture à une personne spécifique ou un groupe de personne, il faut modifer l'ACL (Access Control List) de ce répertoire ! On va faire ça avec icals
```
icacls "C:PATH_to_share" /grant Everyone:(OI)(CI)F
```
- OI : object Inherit ==> Tous les fichiers dedans ce répertoire
- CI : Container Inherit ==> Tous les sous répertoires 
- F : full control

Avec ça, maintenant nous pouvons voir que nos cher f.dejong a droit de lecture/écriture dans ce répertoire ! 
![nxc_check_after](/assets/images/AD/to smb not sign/config share/nxc enum share after add write permission to folder.png)

Voilà, quelques choses que je pense que c'est simple à faire, et ça m'a pris du temps de comprendre pourquoi ça ne marche pas ! Depuis l'interface Windows, les choses semblent facile, mais depuis le cmd, c'est une autre histoire. 

En résumé, il faut à la fois crée un accès de réseau pour le répertoire, ainsi de changer les ACL sur ce répertoire pour donner l'accès à une personne ou un groupe. 

