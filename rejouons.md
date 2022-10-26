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


### 3.1 Clonage du dépot Medicarche

Avant de commencer veuillez  cloner le dépot de l'infrastricture Medicarche en éxecutant la commande `git clone https://gitlab.com/Genuiz/medicarche_ostack.git` vous retrouverez un dossier scripts qui contient l'installation du cloud privé Openstack/Microstack et le fichier Vagranfile qui sera detaillé dans les lignes qui suivent.

### 3.2 Déploiement de la machine virtuelle avec l'outil Vagrant

Pour créer la machine virtuelle, il suffit d'éxecuter la commande `vagrant up`

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

# Installation du systeme d'exploitation Linux Ubuntu qui acceuillera le cloud privé Openstack

Vagrant.configure("2") do |config|
   
  # installer le plugin vagrant plugin install vagrant-disksize
  # Definition de la taille du disque de la VM
  config.disksize.size = '100GB'
  config.vm.define "openstack" do |os|
    # Type de systeme d'exploitation utilisé
    os.vm.box = "bento/ubuntu-20.04"
    #nom de la machine virtuelle
    os.vm.hostname = "openstack"
    #Lien vers le depot vagrant contenant le système d'exploitation
    os.vm.box_url = "bento/ubuntu-20.04"
        #Taille du disque primaire et du disque secondaire
	os.vm.disk :disk, size: "80GB", primary: true
	os.vm.disk :disk, size: "40GB", name: "extra_storage"
    # Adresse IP de la machine virtuelle dans le réseau
    os.vm.network :private_network, ip: "192.168.33.16"
    # Proprieté de la machine virtuelle
    os.vm.provider :virtualbox do |v|
      v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      v.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
      v.customize ["modifyvm", :id, "--memory", 10000]
      v.customize ["modifyvm", :id, "--name", "openstack"]
      v.customize ["modifyvm", :id, "--cpus", "12"]
    end
        # Chemin vers le fichier contenant le script d'installation du cloud Microstack
	os.vm.provision "shell", path: "scripts/openstack.sh"
  end
  
  ```


### 3.2 Installation d’Openstack

Le dossier scripts contient le fichier openstack.sh qui est composé du script suivant que nous commenté 

```
#!/bin/bash

echo "###################################################################"
echo "# Bienvenue dans l'installation automatisé de l'infra Medicarche  #"
echo "###################################################################"

# Ces lignes de commande permettent de rendre l'installation non interactive 
echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections
sudo apt-get install -y -q
sudo apt-get install dialog apt-utils -y

# Installation de microstack en mode developpeur avec l'outil snap
sudo snap install microstack --edge --devmode
snap list microstack

#Installation de la configuration initial de microstack
sudo microstack init --auto --control

#Installation de git et sshpass 
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

Dans le fichier `auto.sh` on retrouve les scripts qui  permettent de créer les gabarits des machine virtuelle, des groupes de sécurité, du televersement de l'image ubuntu dans Openstack, de la création des VMs ainsi que l'installation des applications. tous les scripts utilisés seront detaillé dans les lignes qui suivent. 

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

### 3.7 Création des machines virtuelles

```
#Boucle permettant la création des vms 
for machine in `echo vm1 vm2 vm3 vm4`; do

    echo "Debut creation VM $machine"

    case $machine in

        vm1)
	  microstack.openstack server create --network test --security-group private-sg --key-name sto4_key --flavor m4.custom --image ubuntu $machine
	  microstack.openstack floating ip create external
	  foo=`microstack.openstack floating ip list | egrep -e "None.*\|.*None"`

	  if [ ! -z "$foo" ]; then
 	     #$value= echo "$foo" | head -n1 | awk '{print $1;}'
	     bar=`echo " $foo" | cut -d '|' -f 3 | sed 's/ //g'`
	     # La variable bar contient l'adresse ip flottante
	     echo "Floating IP = $bar"
	  else
	     echo "Error"
	     exit 1
	  fi

          ip_vm1=$bar; echo $ip_vm1;;

        vm2)
	  microstack.openstack server create --network test --security-group private-sg --key-name sto4_key --flavor m4.custom --image ubuntu $machine
	  microstack.openstack floating ip create external
          foo=`microstack.openstack floating ip list | egrep -e "None.*\|.*None"`

	  if [ ! -z "$foo" ]; then
             #$value= echo "$foo" | head -n1 | awk '{print $1;}'
	     bar=`echo " $foo" | cut -d '|' -f 3 | sed 's/ //g'`
	     # La variable bar contient l'adresse ip flottante
	     echo "Floating IP = $bar"
	  else
	     echo "Error"
	     exit 1
	  fi

          ip_vm2=$bar; echo $ip_vm2;;

        vm3)
	  microstack.openstack server create --network test --security-group private-sg --key-name sto4_key --flavor m5.custom --image ubuntu $machine
	  microstack.openstack floating ip create external
          foo=`microstack.openstack floating ip list | egrep -e "None.*\|.*None"`

	  if [ ! -z "$foo" ]; then
             #$value= echo "$foo" | head -n1 | awk '{print $1;}'
	     bar=`echo " $foo" | cut -d '|' -f 3 | sed 's/ //g'`
	     # La variable bar contient l'adresse ip flottante
	     echo "Floating IP = $bar"
	  else
	     echo "Error"
	     exit 1
	  fi

         ip_vm3=$bar; echo $ip_vm3;;

        vm4)
          microstack.openstack server create --network test --security-group private-sg --key-name sto4_key --flavor m3.custom --image ubuntu $machine
	  microstack.openstack floating ip create external
          foo=`microstack.openstack floating ip list | egrep -e "None.*\|.*None"`

	  if [ ! -z "$foo" ]; then
	     #$value= echo "$foo" | head -n1 | awk '{print $1;}'
	     bar=`echo " $foo" | cut -d '|' -f 3 | sed 's/ //g'`
	     # La variable bar contient l'adresse ip flottante
	     echo "Floating IP = $bar"
	  else
	     echo "Error"
	     exit 1
	  fi

          ip_vm4=$bar; echo $ip_vm4;;

        *)
            echo "Error";;
    esac
	microstack.openstack server add floating ip $machine $bar
        echo "Je dors pendant 300s"
       sleep 300
       #Ce script permet aux vms d'etre connu sans intervention humaine
       echo "ubuntu@$bar"
       ssh -o StrictHostKeyChecking=no -o PasswordAuthentication=no root@$bar
       sshpass -f password.txt ssh-copy-id  -i ./open_key.pub root@"$bar"
       sudo cat ./open_key.pub | ssh -i open_key ubuntu@"$bar" "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
       ssh -o StrictHostKeyChecking=no -o PasswordAuthentication=no ubuntu@$bar
       sshpass -f password.txt ssh-copy-id  -i ./open_key.pub ubuntu@"$bar"
       sudo cat ./open_key.pub | ssh -f open_key ubuntu@"$bar" "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
 
      echo "Fin de la creation VM $machine"
done

```

### 3.8 Mise en place du riverse proxy des applications

#### 3.8.1 Riverse proxy Syncthing

```
sudo apt update
sudo apt install apache2 -y
sudo sed -i '5,1d' /etc/apache2/ports.conf #Delete Listen 80 in apache
sudo systemctl restart apache2
sudo touch /etc/apache2/sites-available/syncthing.conf
sudo chmod 777 /etc/apache2/sites-available/syncthing.conf
sudo a2ensite syncthing.conf
sudo systemctl restart apache2
sudo sed -i '$ a Listen 8085' /etc/apache2/ports.conf
#sudo ufw allow 8085/tcp
#systemctl status ufw
sudo systemctl restart apache2
sudo lsof -i -P -n | grep LISTEN

echo "<VirtualHost *:8085>
        ServerName localhost
        #ProxyPreserveHost On
        ProxyPass / http://$ip_vm1:9090/
        ProxyPassReverse / http://$ip_vm1:9090/
      </VirtualHost>" > /etc/apache2/sites-available/syncthing.conf

sudo a2ensite syncthing.conf     
sudo a2enmod proxy proxy_http headers proxy_wstunnel
sudo systemctl restart apache2
sudo lsof -i -P -n | grep LISTEN

```

#### 3.8.2 Riverse proxy Nextcloud

```

#mise à jour du systeme et installation d'apache
sudo apt update
#sudo apt install apache2 -y

#Creation du server virtuel apache qui jouera le role de proxy pour que le site puisse etre atteint depuis le port 8086
sudo touch /etc/apache2/sites-available/nextcloud.conf
#En raison de test j'utilise le 777. Dans la prod je le channgerai en 755
sudo chmod 777 /etc/apache2/sites-available/nextcloud.conf
echo "<VirtualHost *:8086>
        ServerName localhost
        #ProxyPreserveHost On
        ProxyPass / http://$ip_vm2:9000/
        ProxyPassReverse / http://$ip_vm2:9000/
      </VirtualHost>"  > /etc/apache2/sites-available/nextcloud.conf

sudo sed -i '$ a Listen 8086' /etc/apache2/ports.conf

#activation des modules apaches et redemarrage du serveur 
sudo service apache2 reload
sudo a2ensite nextcloud.conf
sudo a2enmod proxy proxy_http headers proxy_wstunnel
sudo a2enmod rewrite
sudo a2enmod headers
sudo a2enmod env
sudo a2enmod dir
sudo a2enmod mime
sudo systemctl reload apache2
sudo systemctl restart apache2

#verifier si le port est bien exposé
sudo lsof -i -P -n | grep 8086

```

#### 3.8.3 Riverse proxy Odoo

```
sudo apt update
#sudo apt install apache2 -y
sudo a2enmod proxy proxy_http headers proxy_wstunnel
sudo systemctl restart apache2
sudo touch /etc/apache2/sites-available/odoo.conf
sudo chmod 777 /etc/apache2/sites-available/odoo.conf
sudo a2ensite odoo.conf
sudo sed -i '$ a Listen 8087' /etc/apache2/ports.conf
sudo systemctl restart apache2

#adresse ip à modifier
echo "<VirtualHost *:8087>
        ServerName localhost
        #ProxyPreserveHost On
        ProxyPass / http://$ip_vm3:8069/
        ProxyPassReverse / http://$ip_vm3:8069/
      </VirtualHost>" > /etc/apache2/sites-available/odoo.conf

sudo service apache2 restart
sudo lsof -i -P -n | grep LISTEN

```

#### 3.8.3 Riverse proxy Site Medicarche

```
sudo apt update
#sudo apt install apache2 -y
sudo touch /etc/apache2/sites-available/medicsite.conf
#En raison de test j'utilise le 777. Dans la prod je le channgerai en 755
sudo chmod 777 /etc/apache2/sites-available/medicsite.conf

#Creation du server virtuel apache qui jouera le role de proxy pour que le site puisse etre atteint depuis le port 8088
echo "<VirtualHost *:8088>
        ServerName localhost
        #ProxyPreserveHost On
        ProxyPass / http://$ip_vm4:9100/
        ProxyPassReverse / http://$ip_vm4:9100/
      </VirtualHost>"  > /etc/apache2/sites-available/medicsite.conf

sudo sed -i '$ a Listen 8088' /etc/apache2/ports.conf

#activation des modules apaches et redemarrage du serveur 
sudo service apache2 reload
sudo a2ensite medicsite.conf
sudo a2enmod proxy proxy_http headers proxy_wstunnel
sudo a2enmod rewrite
sudo a2enmod headers
sudo a2enmod env
sudo a2enmod dir
sudo a2enmod mime
sudo systemctl reload apache2
sudo systemctl restart apache2
sudo lsof -i -P -n | grep 8088

```

### 3.9 Installation des applications et services

```
#Installation des applications et mise en place du reverse proxy qui permet requête d'etre acheminé de internet vers la vm
for machine in `echo vm1 vm2 vm3 vm4`; do 

    case $machine in

        vm1) 
           echo "reverse proxy syncthing"
           export ip_vm1;
           #./reverse_proxy_syncthing.sh; 
           echo "Installation Syncthing"
           scp -i open_key scripts/syncthing2.sh  ubuntu@$ip_vm1:/tmp; 
           ssh -i open_key ubuntu@$ip_vm1 "chmod +x /tmp/syncthing2.sh";
           ssh -i open_key ubuntu@$ip_vm1 "source /tmp/syncthing2.sh";;
        vm2) 
           echo "reverse proxy nextcloud"
           export ip_vm2;
           #./reverse_proxy_nextcloud.sh;
           echo  "installation Nextcloud"
           scp -i open_key scripts/nextcloud.sh  ubuntu@$ip_vm2:/tmp; 
           ssh -i open_key ubuntu@$ip_vm2 "chmod +x /tmp/nextcloud.sh";
           ssh -i open_key ubuntu@$ip_vm2 "source /tmp/nextcloud.sh";;
            
        vm3) 
           echo "reverse proxy odoo"
           export ip_vm3;
           #./reverse_proxy_odoo.sh;
           echo  "installation odoo"
           scp -i open_key scripts/odoo2.sh  ubuntu@$ip_vm3:/tmp; 
           ssh -i open_key ubuntu@$ip_vm3 "chmod +x /tmp/odoo2.sh";
           ssh -i open_key ubuntu@$ip_vm3 "source /tmp/odoo2.sh";;
	   
	 vm4) 
	   echo "Reverse proxy Medicarche"
	   export ip_vm4;
	   sudo chmod +x reverse_proxy_medicsite.sh;
	   # ./reverse_proxy_medicsite.sh;
           echo "installation du site medicarche"
           scp -i open_key scripts/sitemedicarche.sh  ubuntu@$ip_vm4:/tmp; 
           ssh -i open_key ubuntu@$ip_vm4 "chmod +x /tmp/sitemedicarche.sh";
           ssh -i open_key ubuntu@$ip_vm4 "source /tmp/sitemedicarche.sh";;
        *)
           echo "Unknown" ;;
    esac
done

```

