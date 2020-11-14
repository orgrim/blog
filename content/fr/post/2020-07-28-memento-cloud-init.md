---
title: Memento Cloud Init
date: "2020-07-28"
categories:
- Unix
description: Utiliser simplemant Cloud Init avec libvirt
tags:
- unix
- cloud-init
- libvirt
---

Cloud Init permet de configurer des machines virtuelles au boot, en utilissant des méta données provenant de la plateforme ou tourne la VM (AWS, Azure, GCP, KVM Local, etc). Pour une installation sur KVM avec libvirt, il faut utiliser le module NoCloud :

1. Installer le système dans un KVM, avec `virt-manager` et une image ISO
2. Installer le paquet `cloud-init` (et le paquet `sudo` pour que le reste fonctionne)
```console
# yum install cloud-init sudo
```
3. Eteindre la VM
4. Créer un répertoire de travail et s'y déplacer
5. Créer les 3 fichiers :
  * `user-data` : contient la configuration pour ajouter son user, sa clé SSH, configurer sudo, configurer le mot de passe `root`
  * `meta-data` : contient la configuration de la machine, le hostname, etc
  * `network-config` : contient la configuration du réseau
6. Créer l'image ISO :
```console
$ genisoimage -output cloudinit_la_vm.iso -volid cidata -joliet -rock user-data meta-data network-config
```
7. Attacher l'image ISO
8. Démarrer la VM

<!--more-->

Voici les fichiers à placer dans l'image ISO pour avoir le minimum pour finir la configuration de la VM avec Ansible ou autre par la suite :

Fichier `user-data` :

```yaml
#cloud-config
# vim: syntax=yaml
#
# https://cloudinit.readthedocs.io/en/latest/topics/examples.html
#
chpasswd:
  list: |
     root: "Un mot de passe complexe" # Change me
  expire: False

users:
  - name: orgrim # Change me
    ssh_authorized_keys:
      - ssh-rsa AA...= orgrim@hw
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    shell: /bin/bash
    groups: users
```

Fichier `meta-data` :

```yaml
local-hostname: centos77.vz.lo
```

Fichier `network-config` :

```yaml
version: 2
ethernets:
  eth0:
    match:
      name: eth0
    addresses:
      - 10.100.0.59/24
    gateway4: 10.100.0.1
    nameservers:
      search: [vz.lo]
      addresses: [10.100.0.1]
```

La machine est configurée avec le hostname `centos77.vz.lo`, son IP, la gateway, le DNS qui vont bien et un user avec accès root et sa clé SSH pour pouvoir utiliser Ansible pour la suite de la configuration.

La configuration placée dans l'image ISO est appliquée à chaque boot.

La documentation : <https://cloudinit.readthedocs.io/en/latest/topics/datasources/nocloud.html>
