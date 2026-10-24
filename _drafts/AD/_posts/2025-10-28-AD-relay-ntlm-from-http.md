---
title: "Attack de relai - HTTP vers LDAP(s)"
categories:
  - Active Directory
tags:
  - Active Directory
  - relai
  - HTTP
  - LDAP
layout: post
---

Aujourd'hui, je vous parle sur le relai HTTP vers LDAP et LDAPS dans mon lab. 

Comme écris dans le chapitre sur les notion de base de relai, pour déterminer sur la signature des sessions, Windows se base sur ces 2 questions :
- Question 1 : Est-ce qu'avec le protocol utilisé, le client a la capacité de de signer ? 
- Question 2 : Est-ce que c'est obligatoire côté serveur sur le protocol cible ? 

Lorsqu'on arrive à capturer une authentification NTLM depuis un protocol HTTP, comme le protocol HTTP ne peut pas mettre en place de la signature, la réponse pour la première question est un non sèche. Et en tant qu'attaque, cela nous intéresse car on peut relayer cette session vers n'importe qui et n'importe quoi.


## Prérequis
- un accès en interne
- LLMNR/NBTNS ou IPv6 actif cj
- un peu de chance et de patience


## Attaque

Pendant les tests dans mon lab, j'ai rencontré une difficulté de faire le relai de puis le HTTP. Les testes que j'avais fait n'est pas stable, un jour je teste par exemple avec "http://notexisteeee.io/test" et ça marche, un autre jour je teste avec un autre url bizare mais je ne capte rien sur responder. [topotam](https://x.com/topotam77) m'a conseillé de tester directement avec "http://IP_ATTACK/blabla". C'est smart, et ça marche ! 

### HTTP vers LDAP
A noter que mon IP de la machine attaque est 192.168.56.1

Run responder (mode SMB et HTTP OFF) + ntlmrelayx 
![responder](/assets/images/AD/from http to LDAP and LDAPS/ntlmrelayx dump ldap.png)

Déclencher une connexion HTTP depuis un workstation
![HTTP](/assets/images/AD/from http to LDAP and LDAPS/http request from client.png)

Dump LDAP réussite 
![LDAP_dump](/assets/images/AD/from http to LDAP and LDAPS/ntlmrelayx dump ldap 2.png)

### HTTP vers LDAPs
Pareil que vers LDAP, mais ce fois ci on essaie d'ajouter un compte de machicne. Avant ça, faut vérifier si le MAQ (Machine Account Quota) est supérieur à 0. On peut faire ça avec les informations LDAP qu'on vient de récupérer
![check_MAQ](/assets/images/AD/from http to LDAP and LDAPS/MAQ ldap.png)

ET puis relayer et c'est bon on a désormais un compte de machine valide, que nous pouvons utiliser par la suite pour l'énumération ou des attaques de type délégation. 
![check_MAQ](/assets/images/AD/from http to LDAP and LDAPS/add_computer_ok.png). 

### IPv6
Pareil pour IPv6.
![mitm6](/assets/images/AD/from http to LDAP and LDAPS/IPv6/mitm6.png)

Et dump LDAP
![dumpLDAP](/assets/images/AD/from http to LDAP and LDAPS/IPv6/dump ldap.png)

## Conclusion

Voilà on a pu faire nos premiers attaque de relai pour lire les informations du LDAP et aussi créer un compte dans le domain. 
