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


### 3.1 Déploiement de la machine virtuelle avec l'outil Vagrant

Dans un fichier nommmé Vagrantfile copier et coller le script ci-dessous puis éxecuter la commande `vagrant up`

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

# Installation du systeme d'exploitation Linux Ubuntu qui acceuillera le cloud privé Openstack

Vagrant.configure("2") do |config|
   
   # installer le plugin vagrant plugin install vagrant-disksize
  config.disksize.size = '100GB'
  config.vm.define "openstack" do |os|
    os.vm.box = "bento/ubuntu-20.04"
    os.vm.hostname = "openstack"
    #os.vm.provision "docker"
    os.vm.box_url = "bento/ubuntu-20.04"
	os.vm.disk :disk, size: "80GB", primary: true
	os.vm.disk :disk, size: "40GB", name: "extra_storage"
    os.vm.network :private_network, ip: "192.168.33.16"
    os.vm.provider :virtualbox do |v|
      v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      v.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
      v.customize ["modifyvm", :id, "--memory", 10000]
      v.customize ["modifyvm", :id, "--name", "openstack"]
      v.customize ["modifyvm", :id, "--cpus", "12"]
    end
	os.vm.provision "shell", path: "scripts/openstack.sh"
  end
  
  ```


### 3.2 Installation d’Openstack

De preference créer un fichier bash openstack.sh copier et coller le script ci-dessous

```
#!/bin/bash

echo "###################################################################"
echo "# Bienvenue dans l'installation automatisé de l'infra Medicarche  #"
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
#git clone https://gitlab.com/Genuiz/medicopen.git
#cd medicopen
#source auto.sh

```

### 3.3 Création des gabarits

```
#Creation des gabarits qui acceuilleront les vms contenant des applications
microstack.openstack flavor create m3.custom --id auto --ram 1024 --disk 5 --vcpus 2
microstack.openstack flavor create m4.custom --id auto --ram 1024 --disk 8 --vcpus 2
microstack.openstack flavor create m5.custom --id auto --ram 1024 --disk 9 --vcpus 2
microstack.openstack flavor list

```

### 3.4 Création des groupes de sécurité

```
#Creation du groupe de securité qui autorise un certain nombre des ports permettant aux applications d'etre atteint depuis le réseau privée ou internet  
microstack.openstack security group create private-sg
microstack.openstack security group rule create private-sg --ingress --protocol tcp --dst-port 8085:8085 --remote-ip 0.0.0.0/0
microstack.openstack security group rule create private-sg --ingress --protocol tcp --dst-port 8086:8086 --remote-ip 0.0.0.0/0
microstack.openstack security group rule create private-sg  --ingress --protocol tcp --dst-port 8087:8087 --remote-ip 0.0.0.0/0
microstack.openstack security group rule create private-sg  --ingress --protocol tcp --dst-port 8088:8088 --remote-ip 0.0.0.0/0
microstack.openstack security group rule create private-sg  --ingress --protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0
microstack.openstack security group rule create private-sg  --ingress --protocol tcp --dst-port 80:80 --remote-ip 0.0.0.0/0

microstack.openstack security group rule create private-sg --protocol icmp private-sg
#microstack.openstack security group rule create private-sg --protocol tcp --src-ip 0.0.0.0/0 --dst-port 1:65525
microstack.openstack security group rule create private-sg --protocol tcp --remote-ip 0.0.0.0/0 --dst-port 1:65525
microstack.openstack security group rule list

```

### 3.5 Uploader l'image ubuntu depuis le depot git

```
#Clonage du systeme d'exploitation Ubuntu dans un repos dinstant afin de televerser l'OS dans Microstack
git clone https://gitlab.com/Genuiz/os-ubuntu-openstack.git
microstack.openstack image create --container-format bare --disk-format qcow2 --file ./os-ubuntu-openstack/focal-server-cloudimg-amd64.img ubuntu
microstack.openstack image list

```

### 3.6 Création de la paire de clé

```
#Creation de la pair de cle rsa pour securiser la connexion vms tournant sur Microstack de puis l'hote. 
ssh-keygen -q -C "" -N ""  -f open_key
sudo chown vagrant open_key
sudo chmod 400 open_key
microstack.openstack keypair create --public-key open_key.pub sto4_key
microstack.openstack keypair list

```




