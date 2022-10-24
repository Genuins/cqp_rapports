# Migration de l'informatique de MedicArche vers le Cloud - Rejouer le déploiement de A à Z


## I.	Introduction

Dans cette partie nous mettons à disposition des lecteurs les scripts qui permettent d'installer le cloud privé Openstack et de déployer les services cités dans la [page précedente](https://github.com/Genuins/cqp_rapports/edit/main/README.md). Pour le déploiement de l'infrastructure et l'installation des services de base, nous avons utilisés des scripts bash. Les interventions lié à la maintenance ou au déploiement des nouveaux services pourra s'éffectuer avec l'outil Terraform.

## II.	Environnement de travail


Tout ce qui suit a été réalisé et testé sur une machine Windows 10 équipé de 16Gb de mémoire RAM, de 300Gb de disque dur, d'un processeur i5 9500T. A priori un environnement Linux ou MacOs pourrait convenir car tous les outils listé ci-dessous pourrait convenir. Pour éxecuter le script d'installation du cloud privé Openstack nous avons besoin des prérequis suivantes : 

* Installer [Virtualbox](https://www.virtualbox.org/) sur une machine dont le système d'exploitation est Windows 10.
* Installer [Vagrant](https://www.vagrantup.com/downloads) sur la machine Windows 10
* Installer le plugin Vagrant qui permet d'augmenter la taille du disque `vagrant plugin install vagrant-disksize `
* Installer [Packer](https://www.packer.io/downloads) sur la machine Windows 10.
* Installer l'outil [Terraform](https://www.terraform.io/downloads) de même sur la machine Windows 10.
* Installer l'IDE Visualcode ou MobaXterm pour faciliter la connexion en SSH et l'écriture et le déploiement du code en bash ou avec l'outil Terraform.

## III.	Procédure de déploiement de l’infrastructure Medicarche

### 3.1	Installation d’Openstack

A little intro about the installation.
```
#!/bin/bash

echo "###################################################################"
echo "# Bienvenue dans l'installation automatisé de l'infra medicarche  #"
echo "###################################################################"

echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections
sudo apt-get install -y -q
sudo apt-get install dialog apt-utils -y
sudo snap install microstack --edge --devmode
snap list microstack
sudo microstack init --auto --control
sudo apt-get install git sshpass -y

#Autoriser les machines virtuelles tournant dans microstack de se connecter à internet 
# Le NAT est souvent implémenté par des routeurs, dans notre contextl'hôte effectuant le NAT comme un routeur NAT.
sudo iptables -t nat -A POSTROUTING -s 10.20.20.1/24 ! -d 10.20.20.1/24 -j MASQUERADE
sudo sysctl net.ipv4.ip_forward=1

#Recuperation des access pour se connecter à la console
sudo snap get microstack config.credentials.keystone-password

#Recuperation des scripts d'installation des vms et l'installation des applications
git clone https://gitlab.com/Genuiz/medicopen.git
cd medicopen
source auto.sh

```
