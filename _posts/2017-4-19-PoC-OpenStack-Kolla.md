---
layout: post
title: "Proof of Concept : OpenStack Kolla avec Ansible et Docker"
---

OpenStack est un assemblage complexe de plusieurs projets. Il est difficile de maîtriser tous ses modules, de les maintenir en condition opérationnelle, et d'intégrer les nouvelles fonctionnalités arrivant tous les six mois.

Il  est vital que l'nfrastructure devienne aussi agile que les applications qu'elle héberge. OpenStack doit rentrer dans le modèle micro-services et DevOps.
Tout cela est possible avec Kolla.

Kolla est le projet qui simplifie le déploiement et la gestion d'OpenStack.
Monter une démo, tester une nouvelle configuration, mettre à jour un module ou une dépendance, tout cela devient simple.

Kolla se base sur deux technologie **Docker** et **Ansible**.

**Docker** permet de packager l'ensemble des services OpenStack. La force de Docker est de pouvoir fournir un système d'isolation entre les services ainsi qu'un moyen simple de les installer sans se soucier des machines cibles. Il est possible de préparer une maquette sur  un environnement A, puis la répliquer facilement sur un environnement B.

**Ansible** est l'outil qui permet d'orchestrer le déploiement, la configuration et la mise à jour. Le plus d'Ansible est qu'il n'y a pas d'agent à installer sur les machines cibles. Tout s'éxécute via SSH.

Kolla peut être considéré comme la suite du projet [TripleO](http://tripleo.org/). Mais les conteneurs Docker remplacent les images RAW installées via Ironic.
Dans les versions futurs, Kolla utilisera Kubernetes pour assurer la redondance et la disponibilité des services.

Dernier point, il ne faut pas confondre les projets Kolla et Magnum.

 - Kolla permet de déployer OpenStack sur un environnement Docker
 - Magnum permet de déployer un environnement Docker depuis OpenStack

![Kolla vs Magnum](/images/poc-openstack-kolla/Kolla_Magnum.png)

### Proof of Concept

Ce PoC a été créé pour la présentation Meetup Cloud Infra Talk #3.
 - Retrouvez les futurs [Meetup Cloud Infra Talk](https://www.meetup.com/fr-FR/Cloud-Infra-Talk/)
 - Slides de la présentation : [Cloud-InfraTalk-Kolla_v1.pdf](/images/poc-openstack-kolla/Cloud-InfraTalk-Kolla_v1.pdf)

Le but de ce PoC est d'installer sur une même machines tous les services OpenStack, puis relancer des services avec une configuration.

#### Installation de Kolla et préparation des serveurs cibles

La première phase consiste à installer le projet Kolla et _bootstraper_ les serveurs cibles.

    yum -y install epel-release
    yum -y install python-pip ansible
    yum -y install python-devel libffi-devel openssl-devel gcc
    pip install -U pip
    curl -sSL https://get.docker.io | bash
    pip install kolla
    pip install kolla-ansible

    cp -r /usr/share/kolla-ansible/etc_examples/kolla /etc/kolla/
    cp /usr/share/kolla-ansible/ansible/inventory/all-in-one .
    vim /etc/kolla/globals.yml
    kolla-genpwd
    kolla-ansible bootstrap-servers -i all-in-one

[![asciicast](https://asciinema.org/a/eotfz5o8p18wz9q22bt37cu6p.png)](https://asciinema.org/a/eotfz5o8p18wz9q22bt37cu6p)

#### Déploiement des services Openstack

Deuxième phase, récupérer les images Docker et les pousser vers les serveurs cibles.  
La phase de post déploiement sert à générer un fichier de connexion _admin-openrc.sh_.

    kolla-ansible pull -i all-in-one
    kolla-ansible deploy -i all-in-one
    kolla-ansible post-deploy -i all-in-one
    pip install python-openstackclient

[![asciicast](https://asciinema.org/a/ccdzabbzv8i7r58ihcpqyx6dn.png)](https://asciinema.org/a/ccdzabbzv8i7r58ihcpqyx6dn)

#### Reconfiguration d'un service

Dans ce PoC nous avons volontairement monter un défaut de configuration du service _nova-compute_. Dans cette dernière phase nous reconfigurons le service en erreur pour qu'il utilise la technologie Qemu au lieu de KVM.

    source /etc/kolla/admin-openrc.sh
    /usr/share/kolla-ansible/init-runonce

[![asciicast](https://asciinema.org/a/601apgfwhmav66m0es29rxard.png)](https://asciinema.org/a/601apgfwhmav66m0es29rxard)

### Liens

 - [Site Officiel](https://wiki.openstack.org/wiki/Kolla)
 - [Docker Hub](https://hub.docker.com/u/kolla/)
