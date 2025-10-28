---
title: "Attack mot de passe"
categories:
  - Active Directory
tags:
  - Active Directory
  - Kerberoasting
  - AsrepRoasting
  - Reused password
  - password spray
---
On commence avec quelques attaques sur les mots de passe.

Il existe plusieurs méthode sur les attaques de mot de passe mais rien n'est très compliqué. Et afin de tester les mots de passe, il nous faut d'abord une liste de l'utilisateur valide. Et oui. 

## Récupération une liste de l'utilisateur valide
De ma connaissance, il y a 2 méthodes pour le faire: 
- tester avec un wordlist
- faire du relai pour pouvoir dump LDAP.

La partie de relai dump LDAP, je vais présenter dans les prochains postes. Pour la partie de tester les username avec un wordlist, on peut utiliser kerbrute. Ici j'ai utilisé un wordliste avec seulement 6 usernames. Mais on peut tester avec un wordlist beaucoup plus grande

![Kerbrute](/assets/images/AD/Password attack/kerbrute.png)


## Récupération le mot de passe avec AsrepRoasting

Il y a plusieurs articles sur ce sujet, cet [article](https://beta.hackndo.com/kerberos-asrep-roasting/) de pixis est vraiment bien détaillé. En gros, pour demander un TGT, le client doit d'abord envoyer son identifiant et son mot de passe - cette phase s'apelle Pré-auth, ce qui est logique. Pourtant il existe une option de Windows qui permet de ne pas passer de cette phase, et donc n'importe qui peut demander le TGT pour le compte donc l'option préauth est désactivé. 

Dans mon lab, j'ai setup comme ci : 

![Do not require Preauth](/assets/images/AD/Password attack/AsrepRoasting/config asreproasting.png)

Et on peut récupérér le TGT avec le module GetNPUSers d'impacket. Ici j'ai utilisé mon wordlist, on peut voir que f.dejong et d.olmo existent bien dans le domain mais leurs comptes ne peuvent pas bypass la phase préauth, par contre le compte r.araujo y est. La dernière ligne corresponde à un username qui n'existe pas dans le domaine  

![Asrep_without_account](/assets/images/AD/Password attack/AsrepRoasting/asrep without account.png)

Une fois récupérée le TGT, on peut lancer hashcat et de croiser les doigts que le secret se trouve dans nos wordlists. (A noter que le TGT est chiffré avec le secret de l'utilisateur)

![Asrep_hashcat](/assets/images/AD/Password attack/AsrepRoasting/hashcat.png)

Et voilà ! 

## Password spray

Une technique redoutable ! Et ça marche très bien ! 

![Password_spray](/assets/images/AD/Password attack/password spray.png)

Et voilà 
## Kerberoasting

Au final, le kerberoasting. cet [article](https://en.hackndo.com/kerberoasting/) de pixis est aussi très bien ! En gros, un compte utilisateur avec un attribut SPN - cequi lui permettre d'indiquer que ce compte est lié à un service. Ce fois ci, on peut demander un ticket TGS. Qui dit ticket TGS, dit faut d'abord avoir un ticket TGT valide. 

Comme le TGS est chiffré avec le secret du service, nous pouvons ce fois ci tenter nos chances. 
Et pourquoi un compte utilisateur et pas un compte de machine ? parce que les comptes de machine ont des mot de passe fort ! A noter aussi il y a le compte de krbtgt, mais ce compte est inactif, et son mot de passe est ultra complexe. 

Voilà comment j'ai pu paramétrer un compte SPN qui tourne dans mon lab.
![config_user_SPN](/assets/images/AD/Password attack/Kerberoasting/config User SPN.png)

Récupérer le TGS avec un compte valide (f.dejong)
![getUSerSPN](/assets/images/AD/Password attack/Kerberoasting/GetUserSPN.png)

Et cracker avec hashcat
![getUSerSPN](/assets/images/AD/Password attack/Kerberoasting/hashcat cracked.png)

## Pre2k
Les Pre2k reste aussi une possibilité. En gros, les comptes de machines de type "pre-Windows 2000" ont leurs mot de passe qui est aussi leurs nom (sans $ à la fin). Une fois connectée pour la première fois, ce mot de passe doit être changé. La page the [thehacker.recipes](https://www.thehacker.recipes/ad/movement/builtins/pre-windows-2000-computers) détaille très bien !

Voici comment j'ai setup un compte de machine de type pre-Windows 2000. Juste coché l'option Pré2K ! Je ne sais pas pourquoi Windows laisse le groupe "Domain Admins" par défaut quand on crée un nouveau computer, pour le coup j'ai changé le groupe dans mon lab, mais je testerai avec le config par défaut....  
![Pre2k](/assets/images/AD/Password attack/pre2k/setup pre2k computer.png)

ET là c'est bon ! je peux commencer mon attaque ! Avec un compte valide, je peux faire de l'énumération avec nxc.
![nxc_enum_pre2k](/assets/images/AD/Password attack/pre2k/pre2k enum.png)

Test avec le mot de passe 
![nxc_enum_pre2k](/assets/images/AD/Password attack/pre2k/STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT.png)
Selon thehackerrecipes, qui lui cite cet [article](https://www.trustedsec.com/blog/diving-into-pre-created-computer-accounts), le status STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT est juste pour dire que le mot de passe est bon mais personne n'a utilisé. 

On peut demander directement le TGT de ce compte sans avoir besoin de changer son mot de passe
![getTGT](/assets/images/AD/Password attack/pre2k/getTGT without change password.png)
![nxc_connect](/assets/images/AD/Password attack/pre2k/nxc test connection.png)