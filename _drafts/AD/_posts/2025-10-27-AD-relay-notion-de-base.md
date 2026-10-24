---
title: "Attack de relai - Les notions de bases"
categories:
  - Active Directory
tags:
  - Active Directory
  - relai
---

Les attaques de relais sont souvent des attaques de type "from 0 to hero". Ils sont très opportunistes, on lance attaque, on prends un café (ou un thé, ça depénd de votre goût !) et hop, on possède une session valide (c'est un peu ça !). 
Personnellement je trouve qu'il est plus dur à comprendre qu'à lancer les attaques. 
Aujourd'hui dans cette article je vais parler quelques notions que je pense il est nécessaire de comprendre avant de faire des attaques de relai. 

## Comment les DNS fonctionnent ? 
En gros, quand on tape "https://cettepagenexistepas.com" dans votre navigateur ou "\\server_not_existe\share_file" pour la barre de recherche de l'explorateur, vous commencez à faire des requêtes DNS. 
- Tout d'abord il cherche dans le DNS cache local (hosts)
- Si il ne trouve pas, il cherche sur le serveur DNS
- si il ne trouve rien, il tombe sur le fallback NBT-NS en broadcast. Lui, il existe uniquement pour IPv4
- Et si pas de bol, il tombe sur le LLMNR en multicast. Lui, il existe en IPv4 et IPv6

Cet [article](https://www.it-connect.fr/active-directory-comment-et-pourquoi-desactiver-les-llmnr-et-netbios/) décris bien comment fonctionne. 

Tous ça pour dire que si les 2 protocols NBT-NS et LLMNR ou l'IPv6 ne sont pas désactivé par Microsoft, et malheureusement c'est le cas par défaut quand vous créez votre domain. En tant qu'attaquant, nous pouvons se faire passer par un MITM pour pouvoir capturer l'authentification et relayer des sessions vers n'importe qui (il y a quelques contraints, on verra un peu plus bas)


## Kerberos et NTLM 

OK, donc on peut faire le relai. Mais on relai quoi ? 
Souvent, quand on se connecte sur une machine dans un Active Directory, on a 2 manières d'**authentifier**, soit par kerberos, soit en NTLM. 
Pour les connexions de type de réseau, on utilise souvent le NTLM (à l'exeption des comptes dans le groupe "protected users").

Mais comment ça marche ce NTLM ? 

En gros, le système de NTLM se base sur un système de negociation/challenge-response/authentification. 
- Un client se veut se connecter à un serveur, il dit que moi je veux connecter à toi. (phase négociation)
- Le serveur lui réponde : bah super, puisque tu veux authentifier avec moi, je te donne un challenge que je suis le seul à connaitre. Comme ça nous sommes sûr que personne d'autres peuvent lire tes secrets (phase challenge)
- Le client chiffre ses infos avec le challenge. Et renvoie aux serveurs. Voilà. (phase réponse)
- Si le serveur a le hash du client, il peut comparer, sinon il renvoie tout au DC (phase authentification) 

Et le Kerberos ? 

Le kerberos permet authentifier mais avec un système de 2 phases seulement :
- Coucou c'est moi ! (Et il envoie des informations pour prouver que c'est bien lui biensûr). Donne-moi mon TGT ! 
- Une fois obtenir le TGT :  coucou c'est mon TGT, donne moi un TGS pour un tel service

Le relay Kerberos, j'ai lu dans un article de Dirkjan, il dit que ce n'est pas vraiment du relai, mais juste sur le fait que ça re-utilise les mêmes modules du tool ntlmrelayx qu'il appelle comme ça (peut-être que ma mémoire n'est pas bonne !). En tous cas, on va aborder le relai kerberos dans le chapitre dédié aux délégations. 

Aujourd'hui dans ce poste, je ne vais parler uniquement sur le NTLM

## Session et Signature de NTLM
Comme dit avant, le NTLM se permet d'authentifier. Pour pouvoir "communiquer" après, on se base sur des session (comme le principe d'authentification web). Et dans cette phase de session, **chaque requête** dans cette phase doit contenir ou non une signature. Cette signature permet de prouver que l'émetteur est bien à l'origine des échanges, car la signature est calculé par le message et le secret du client.  Mais comment définir si on peut signer ou pas une communication ?  

Et bah pendant la 1ère phase (phase de négociation), non seulement envoyé un message au serveur de ce type : "je veux communiquer avec toi", le client lui envoie aussi les informations sur sa capacité de signer une communication, et pendant la phase challenge, le serveur lui envoie la sienne. Cette négociation se joue principalement sur 2 grandes questions : 
- Question 1 : Est-ce qu'avec le protocol utilisé, le client a la capacité de de signer ? 
- Question 2 : Est-ce que c'est obligatoire côté serveur sur le protocol cible ? 

En répondant à ses 2 questions comme un jeu logique (oui ET non, oui ET oui, non ET non, non ET oui), on aura la décision finale. Il y a aussi les notions de chanel binding, mais je ne maitrise pas encore. Pour ne pas raconter de bêtise, je préfère de ne pas érire sur ce sujet.

Mais on est fou, pour la première question, si on réponde systématiquement NON à chaque fois qu'on capture une authentification ? Comme ça nous pouvons relayer n'importe quoi vers n'importe qui et quoi. La réponse est non on ne peut pas faire ça. Parce que pendant la dernière phase d'authentification, pour rassurer sur l'intégrité du contenu, Windows utilise le MIC (Message Intigrity Code) - une sorte de chiffrement qui chiffre le contenu du message avec le secret du client. Et pour informer au serveur que ce MIC existe, il utilise un drapeau (flag). La vie est belle puisque l'attaquant ne connait pas le secret du client, donc il ne peut pas déchiffrer le MIC, changer la capacité de signature et puis rechiffrer le message une fois terminée. Sauf si .... le message authentification est vulnérable au Drop The Mic (1 et 2), qui permet tout simplement de virer ce MIC. 

Une fois l'authentification est abouti, et que les 2 parties sont en accord pour signer, la session sera signer avec le secret du client. ET encore une fois ça rendra fou l'attaquant puisqu'il ne connait pas le secret du client, et donc impossible de relayer les sessions capturés. Sauf que, il y a un monde où les échanges utilisent encore le vieux protocol NTLMv1. Et RIP ! 

## NTLMv1 et NTLMv2

On a parlé de signature et de MIC, mais c'est pour du NTLMv2. En NTLMv1, le MIC n'existe pas ! Donc on peut relayer n'importe quoi vers n'importe qui en disant juste que "coucou ! c'est moi mais je ne peux pas signer ! désolé !". Je vais vous présenter ça dans un poste dédié à l'attaque NTLMv1 

A noter aussi que les hashs NTLMv1 sont plus simple à casser aussi par rapport au v2. ça ouvre beaucoup de porte là ! 

## Conclusion
Sans rentrée dans les détails techniques, je pense qu'il est important de revenir sur les notions clés.
Dans ce poste je vous ai parlé de DNS,NTLM,signature, session,MIC, NTLMv1. 

A noter que : 
- Les protocols LLMNR/NBT-NS sont des protocols de fallback DNS, qui permet les attaquants de faire du MITM
- Si la signature existe pour les sessions, le relai NTLM est impossible
- NTLMv1 = compromission de domaine

Il y a plusieurs formidables article sur ce sujet si vous voulez creuser un peu plus.  
- [pixis](https://beta.hackndo.com/ntlm-relay/)
- [Login sécurité](https://blog.login-securite.com/les-faiblesses-du-protocole-ntlm#heading-quest-ce-que-cest)
- [thehacker.recipes](https://www.thehacker.recipes/ad/movement/ntlm/relay)