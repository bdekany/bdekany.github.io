---
layout: post
title: "Vulnerable Docker VM : le guide de sécurisation"
---

Red Team vs Blue Team, les rouges contre les bleus. Ayant pentesté tout l'été, l'automne venu, il faut sécuriser.
Aujourd'hui on s'attaque à "Vulnerable Docker VM", un lab proposé par NotSoSecure.

Docker et la conteneurisation sont des technologies à la mode.
On s'empresse de les utiliser pour remplacer les systèmes de virtualisation et gagner en performances.

Mais si le gain de performances est énorme, c'est parce qu'il y a une forte mutualisation des ressources. Mutualisation qui est une source de vulnerabilités.

[NotSoSecure](https://www.notsosecure.com/) propose un lab pour tester vos connaissances.
Le but est de retrouver des informations cachées et s'approprier les droits _root_ sur une VM faisant tourner docker.
Si vous voulez tenter l'expérience, téléchargez la [Vulnerable Docker VM](https://www.notsosecure.com/vulnerable-docker-vm/).

Ici je vous propose le guide de sécurisation de ce lab.

![Down whale](/images/vulnerable-docker-vm/drown-whale.png)

### Rapport Red Team

Pour prendre le controle de la machine, la Red Team a utilisé trois attaques connues :

 - une attaque brute-force sur l'authentification Wordpress
 - deux attaques sur docker (user namespace et socket non sécurisé)

Les rapports complets des pentesteurs sont disponibles en bas de la page [Vulnerable Docker VM](https://www.notsosecure.com/vulnerable-docker-vm/).

### Sécurisation de la VM

#### Brute-force WP

Passons rapidement sur l'attaque via brute-force du wordpress. Elle n'est pas liée à docker mais c'est le point d'entrée de l'attaquant. Nous devons donc limiter ce type d'attaque.

Lorsque que docker connecte un conteneur avec le monde exterieur, une chaine _iptables_ nommée "DOCKER" est créée. J'ajoute une règle à cette chaine limitant le nombre de connexion. Pas plus de 30 connexions par minute.

    iptables -I DOCKER -p tcp --dport 8000 -m state --state NEW -m limit --limit 30/minute --limit-burst 5 -j ACCEPT

#### User Namespace

On commence par un gros problème de sécurité. Le daemon docker est lancé par __root__ (UID 0), tous les conteneurs seront aussi exécutés avec cet UID, sauf si on le spécifie à la création (option -u via la cli, tag USER dans le Dockerfile).

Le point important à retenir est : si vous êtes root dans le conteneur, vous l'êtes aussi sur le système hote.

Pour éviter cela et empecher l'attaquant de modifier les fichiers (upload d'un webshell ou de clef ssh dans notre cas) il faut isoler les conteneurs.

Cette isolation des droits se fait via l'option _--usens-remap_ au lancement du daemon docker.

Sur la VM, il existe déjà un utilisateur et un groupe _whale_. les deux fichiers /etc/subuid et /etc/subguid sont déjà renseignés.

Il suffit de modifier le fichier _/etc/default/docker_ et relancer docker. Pour Red Hat et SUSE le bon fichier est _/etc/sysconfig/docker_.

    DOCKER_OPTS="$DOCKER_OPTS --usens-remap=whale"

#### Socket TLS

Deuxième problème, tout utilisateur ayant accès au socket docker (UNIX ou HTTP) peut lancer n'importe quelle commande. Il n'y a aucun système d'authentification.

Il est donc important de forcer l'utilisation de certificats via TLS pour authentifier les clients docker. Surtout si vous avez une deuxième machine dédiée au monitoring, comme [portainer](https://portainer.io/), qui a besoin de récuper les statistiques docker via l'API par le réseau.

Le script ci-dessous génère une autorité de certification, un certificat client pour la connexion et reconfigure le daemon docker.

 - [Script pour ce lab et Ubuntu](https://gist.github.com/bdekany/0d6b1053a07ba15da95a4f10d50c3d54)
 - [Script original pour Red Hat et SUSE](https://gist.github.com/Stono/7e6fed13cfd79598eb15) (attention à la typo ligne 39 s/src/srl/)

### Conclusion

 - Docker est un produit sécurisé, pas indestructible,
 - En production, passez par un orchestrateur (Kubernetes, ...),
 - AppArmor est activé sur la VM, ce qui met en déroute les exploits "standards" de Metasploit
 - SELinux empeche nativement l'injection de clef ssh dans _/root/_ depuis un conteneur
