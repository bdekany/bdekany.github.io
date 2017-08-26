---
layout: post
title: "Azure Network Security Group : Comment vraiment sécuriser vos réseaux"
---

Le _Network Security Group_ (NSG) est la première brique de sécurité réseau utilisée sur Azure. Il permet de créer des règles de pare-feu sur les machines virtuelles et les réseaux virtuels. Une ressource simple mais pas très efficace.

C'est en mettant en place une démo sur l'utilisation de plusieurs réseaux Azure que j'ai découvert cette inefficacité.
Grace au _peering_ (Homologations sur le portail en français) il est possible de d'interconnecter deux VNet (réseaux virtuels).
Cette fonction permet de créer des architectures avec des réseaux segmentés comme une DMZ faisant face à Internet et une zone d'administration protégée.

### La démo

Me voilà donc parti à créer une démo en IaaS avec un VNet pour les services contrôleur de domaine et un VNet pour les VM accessibles depuis Internet.
Je déploie deux machines, une VM contrôleur de domaine (AD DC) via le [template Microsoft](https://github.com/Azure/azure-quickstart-templates/tree/master/active-directory-new-domain) et une VM standard.
Comme d'habitude, je place sur les interfaces réseaux un NSG par défaut avec comme seule règle "Autoriser l'accès en RDP" depuis n'importe quelle source (gag en approche).

Je me connecte à la VM et tente de rejoindre le domaine : tout fonctionne.

Phase 1 : euphorie

> "Ca alors le NSG contiendrait des règles par défaut autorisant DNS et Kerberos, fantastique."  

Phase 2 : paranoïa

> "Mais quelles sont les autres règles ?"

### Pourquoi ?

Tous les NSG crées sur Azure ont des règles par défaut qui ne peuvent pas être supprimées. Il est possible de les afficher en cliquant sur "Default Rules" depuis le panneau "Inbound security rules".

![default-nsg](/images/azure-securiser-les-nsg/default-nsg.PNG)

La règle 65000 utilise en source et destination le mot clé _VirtualNetwork_ qui représente __tous les VNet sur Azure__ et pas seulement le VNet où se trouve la machine. Pourquoi ne pas avoir utilisé _VirtualNetwork**s**_ ?

Bonus : si on fait du VPN Site à Site, le mot clé _VirtualNetwork_ représente aussi les plages d'adresses du site local.

Il est possible de lister tous les VNet dernière ce mot clé via le panneau "Effective security rules" dans la ressource "Network security Group".

![NSG Pane](/images/azure-securiser-les-nsg/networkSecurityGroup.PNG) ![Effective Rules Pane](/images/azure-securiser-les-nsg/EffectiveRulesPane.PNG)

Sélectionnez une machine virtuelle via la liste déroulante et le calcul se lance.

![Effective Rules Pane](/images/azure-securiser-les-nsg/EffectiveRules.PNG)

En cliquant sur Virtual Network, on ouvre une fenêtre avec les plages d'adresses cibles.

![Prefix Pane](/images/azure-securiser-les-nsg/VirtualNetworks.PNG)

En plus des VNet créés sur Azure et les plages d'adresses du Site VPN, on trouve l'adresse [168.63.129.16/32](https://blogs.msdn.microsoft.com/mast/2015/05/18/what-is-the-ip-address-168-63-129-16/).
C'est une IP virtuelle représentant l'hôte hyper-V. C'est la même pour tout le monde.

### Solution

Pour rendre les NSG éfficaces, et pas seulement filtrer le trafic venant d'Internet, il faut ajouter une règle.

Ajouter une vraie règle DenyAll avec la priorité 4096 qui bloque toutes les sources et tous les ports (Any/Any).

![Prefix Pane](/images/azure-securiser-les-nsg/DenyAll.PNG)
