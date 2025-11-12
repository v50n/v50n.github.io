---
title: "Active Directory - Kerberos Délégation"
categories:
  - Active Directory
tags:
  - Active Directory
  - Delegation
  - Kerberos
layout: post
---

Aujourd'hui dans ce mini poste je vais parler des délégations kerberos. Il est important pour moi d'écrire sur ce sujet avant de faire quelques attaques liés à ce sujet.

Le kerberos, j'ai expliqué rapidement dans [mon poste](/posts/AD-relay-notion-de-base/). Mais vous pouvez trouver les détails dans cette [article de pixis](https://beta.hackndo.com/kerberos/). 

En gros, le système Kerberos gère la partie d'authentification d'un compte dans l'Active Directory. Une fois un compte réussi à s'authentifier et obtenir son TGT, pour accéder à un service, il demande Kerberos d'un ticket TGS lié à ce service. Kerberos lui fournir le TGS, mais c'est au service lui même de vérifier si l'utilisateur a bien le droit d'accéder (vérification avec le fameux PAC - Privilege Attribute Certificate). Une fois présenter ce TGS et de pouvoir accéder à un service, pour ensuite usurper l'identité de l'utilisateur et d'accéder à une autre service, Kerberos a besoin un système de délegation.

Nous avons 3 types de délegations: 
- Délégation sans contraint (Unconstrained)
- Délégation avec contraint (Constrained)
- Délégation basé sur les contraints de ressource (Resource-based Constrained Delegation) - pardon pour la traduction trop nulle ! 

Avant de rentrer dans les détails de chaque type de délégation, je vous présente 3 amis qui vont nous suivre par la suite. Les 3 amis sont : 
- User 1
- Service A
- Service B
- Service C et D


## Unconstrained delegation

![Unconstrained_delegation](/assets/images/AD/delegation_attack/basic notion/unconstrained.png)

Dans le délégation sans contraint, une fois que l'User 1 accède au Service A, le Service A peut usurper son identité et se connecter à n'importe quel service. Il n'y a pas de limite. 

## Constrained delegation

![constrained](/assets/images/AD/delegation_attack/basic notion/constrained.png)

Dans le cadre de délégation avec contraint, les services A ne peut uniquement ce fois ci usurper l'identité de l'user 1 sur les services B et D

## Resource-based Constrained Delegation (RBCD)

![RBCD](/assets/images/AD/delegation_attack/basic notion/RBCD.png)

Dans le cas de RBCD, ce fois ci, ce n'est plus au service A de définir quel service qu'il peut faire de la délégation, mais au tour du service B qui défini que je n'accepte uniquement les délégations vennant de service A. 

