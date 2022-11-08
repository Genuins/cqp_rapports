# Migration de l'informatique de MedicArche vers le Cloud - Rejouer le déploiement de A à Z

## I.	Introduction

Dans cette partie nous mettons à disposition les scripts qui permettent d'installer le cloud privé Openstack (MicroStack) et de déployer les services cités dans le [document](https://github.com/Genuins/cqp_rapports/edit/main/README.md) de présentation du projet. Nous vous invitons à lire ce document dans le détail avant de parcourir cette page. Pour le déploiement de l'infrastructure et l'installation des services de base, nous avons utilisé des scripts Bash. Les interventions liées à la maintenance ou au déploiement des services s'effectue avec l'outil Terraform.

## II.	Environnement de travail

Tout ce qui suit a été réalisé et testé sur une machine Windows 10 équipée de 16Gb de mémoire RAM, de 300GB de disque dur, d'un processeur Intel i5 9500T. A priori un environnement Linux ou MacOs pourrait convenir car tous les outils listés ci-dessous sont également disponibles pour ces distributions de système d'exploitation. Cependant nous n'avons pas testé. Pour éxecuter le script d'installation du cloud privé Openstack nous avons besoin des prérequis suivants : 

* Installer [Virtualbox](https://www.virtualbox.org/) sur une machine dont le système d'exploitation est Windows 10 ;
* Installer [Vagrant](https://www.vagrantup.com/downloads) sur la machine Windows 10 ;
* Installer le plugin Vagrant qui permet d'augmenter la taille du disque `vagrant plugin install vagrant-disksize` ;
* Installer [Packer](https://www.packer.io/downloads) sur la machine Windows 10 ;
* Installer l'outil [Terraform](https://www.terraform.io/downloads) de même sur la machine Windows 10 ;
* Installer l'IDE Visualcode ou MobaXterm pour faciliter la connexion en SSH et l'écriture et le déploiement du code en bash ou avec l'outil Terraform.

## III.	Procédure de déploiement de l’infrastructure Medicarche

### 3.1 Clonage du dépot Medicarche

Avant de commencer veuillez  cloner le dépot qui permet de construire l'infrastructure Medicarche en éxecutant la commande `git clone https://gitlab.com/Genuiz/medicarche_ostack.git`. Vous obtiendrez un dossier `scripts` qui contient l'installation du cloud privé Openstack/Microstack et le fichier `Vagranfile` qui sera detaillé dans les lignes qui suivent.

### 3.2 Déploiement de la machine virtuelle avec l'outil Vagrant

Pour créer la machine virtuelle, il suffit d'éxecuter la commande `vagrant up` qui provoque l'analyse et l'exécution du fichier `Vagrantfile`suivant : 

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

# Installation du système d'exploitation Linux Ubuntu qui acceuillera le cloud privé Openstack

Vagrant.configure("2") do |config|
   
  # au préalable vous avez à installer le plugin permettant de redimensioner le disque par la commande : 
  # vagrant plugin install vagrant-disksize 
  # depuis l'hote Windows. N'oubliez pas ce préalable.
  
  # Definition de la taille du disque de la VM
  config.disksize.size = '100GB'
  config.vm.define "openstack" do |os|
    # Type de systeme d'exploitation utilisé
    os.vm.box = "bento/ubuntu-20.04"
    # Nom de la machine virtuelle
    os.vm.hostname = "openstack"
    # Lien vers le depot vagrant contenant le système d'exploitation
    os.vm.box_url = "bento/ubuntu-20.04"
        # Taille du disque primaire et du disque secondaire
	os.vm.disk :disk, size: "80GB", primary: true
	os.vm.disk :disk, size: "40GB", name: "extra_storage"
    # Adresse IP de la machine virtuelle dans le réseau
    os.vm.network :private_network, ip: "192.168.33.16"
    # Proprietés de la machine virtuelle
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
Le script précédent est commenté pour détailler les differentes étapes de construction. Cela est fait à titre d'information.

### 3.3 Installation d’Openstack

Le fichier `Vagrantfile` précédent lance l'éxecution du script `openstack.sh`, donc il n'y a rien à faire manuellement. C'est dans le dossier `scripts` que l'on trouve le fichier `openstack.sh` qui est présenté ci-dessous : 

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

# Installation de la configuration initial de microstack
sudo microstack init --auto --control

# Installation de git et sshpass 
sudo apt-get install git sshpass -y

# Autoriser les machines virtuelles tournant dans microstack de se connecter à Internet 
# Le NAT est souvent implémenté par des routeurs, dans notre contexte l'hôte effectuant le NAT comme un routeur NAT.
sudo iptables -t nat -A POSTROUTING -s 10.20.20.1/24 ! -d 10.20.20.1/24 -j MASQUERADE
sudo sysctl net.ipv4.ip_forward=1

# Récupération des accès pour se connecter à la console
sudo snap get microstack config.credentials.keystone-password

# Récupération des scripts d'installation des vms et l'installation des applications
git clone https://gitlab.com/Genuiz/medicopen.git
cd medicopen
source auto.sh
```

Dans le fichier `auto.sh` on retrouve les scripts qui  permettent de créer les gabarits des machines virtuelles, des groupes de sécurité, du téléversement de l'image Ubuntu dans Openstack, de la création des VMs ainsi que l'installation des applications. Tous les scripts utilisés sont detaillés dans les lignes qui suivent. Ils sont tous commentés, donc ils n'appelent pas de commentaires particuliers. Nous les listons de manière brute.

### 3.4 Création des gabarits

```
# Creation des gabarits qui acceuilleront les vms contenant des applications
microstack.openstack flavor create m3.custom --id auto --ram 1024 --disk 5 --vcpus 2
microstack.openstack flavor create m4.custom --id auto --ram 1024 --disk 8 --vcpus 2
microstack.openstack flavor create m5.custom --id auto --ram 1024 --disk 9 --vcpus 2
microstack.openstack flavor list
```

### 3.5 Création des groupes de sécurité

```
# Creation du groupe de securité qui autorise un certain nombre des ports permettant aux applications 
# d'être atteintes depuis le réseau privé ou depuis Internet

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

### 3.6 Uploader l'image Ubuntu depuis le dépôt git

```
# Clonage du systeme d'exploitation Ubuntu dans un repos distant afin de téléverser cet OS dans Microstack
git clone https://gitlab.com/Genuiz/os-ubuntu-openstack.git
microstack.openstack image create --container-format bare --disk-format qcow2 --file ./os-ubuntu-openstack/focal-server-cloudimg-amd64.img ubuntu
microstack.openstack image list
```

### 3.7 Création de la paire de clé

```
# Creation de la pair de clé rsa pour sécuriser la connexion des VMs, tournant dans Microstack, depuis l'hôte. 
ssh-keygen -q -C "" -N ""  -f open_key
sudo chown vagrant open_key
sudo chmod 400 open_key
microstack.openstack keypair create --public-key open_key.pub sto4_key
microstack.openstack keypair list
```

### 3.8 Création des machines virtuelles

```
# Boucle permettant la création des vms 
for machine in `echo vm1 vm2 vm3 vm4`; do

    echo "Début creation VM $machine"

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
        # Ce script permet aux vms d'etre connues sans intervention humaine
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

### 3.9 Mise en place du reverse proxy pour les applications que l'on déploie

#### 3.9.1 Reverse proxy Syncthing

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

#### 3.9.2 Reverse proxy Nextcloud

```
# Mise à jour du système et installation d'apache
sudo apt update
sudo apt install apache2 -y

# Création du server virtuel apache qui jouera le role de proxy pour que le site puisse etre atteint depuis le port 8086
sudo touch /etc/apache2/sites-available/nextcloud.conf
# En raison de test j'utilise les droits 777. Dans la prod je le changerai en 755
sudo chmod 777 /etc/apache2/sites-available/nextcloud.conf
echo "<VirtualHost *:8086>
        ServerName localhost
        #ProxyPreserveHost On
        ProxyPass / http://$ip_vm2:9000/
        ProxyPassReverse / http://$ip_vm2:9000/
      </VirtualHost>"  > /etc/apache2/sites-available/nextcloud.conf

sudo sed -i '$ a Listen 8086' /etc/apache2/ports.conf

# Activation des modules apache et redemarrage du serveur 
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

#### 3.9.3 Reverse proxy Odoo

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

#### 3.9.4 Reverse proxy Site Medicarche

```
sudo apt update
#sudo apt install apache2 -y
sudo touch /etc/apache2/sites-available/medicsite.conf
#En raison de test j'utilise le 777. Dans la prod je le channgerai en 755
sudo chmod 777 /etc/apache2/sites-available/medicsite.conf

# Création du server virtuel apache qui jouera le role de proxy pour que le site puisse etre atteint depuis le port 8088
echo "<VirtualHost *:8088>
        ServerName localhost
        #ProxyPreserveHost On
        ProxyPass / http://$ip_vm4:9100/
        ProxyPassReverse / http://$ip_vm4:9100/
      </VirtualHost>"  > /etc/apache2/sites-available/medicsite.conf

sudo sed -i '$ a Listen 8088' /etc/apache2/ports.conf

# Activation des modules apache et redemarrage du serveur 
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

### 4.0 Installation des applications et services

```
# Installation des applications et mise en place du reverse proxy qui permet aux requêtes
# d'etre acheminées de internet vers la vm
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
### 4.1 Installation des applications de supervision

#### 4.1.1 Installation de prometheus

```
#Creation user and group user
sudo useradd --no-create-home --shell /bin/false prometheus
sudo useradd --no-create-home --shell /bin/false node_exporter

sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus

sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus

#Prometheus installation

#Download Prometheus Binary File
wget https://github.com/prometheus/prometheus/releases/download/v2.39.1/prometheus-2.39.1.linux-amd64.tar.gz
sha256sum prometheus-2.39.1.linux-amd64.tar.gz
tar -xvf prometheus-2.39.1.linux-amd64.tar.gz


#Copy Prometheus Binary files
sudo cp prometheus-2.39.1.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.39.1.linux-amd64/promtool /usr/local/bin/

#Update Prometheus user ownership on Binaries
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool

#Copy Prometheus Console Libraries
sudo cp -r prometheus-2.39.1.linux-amd64/consoles /etc/prometheus
sudo cp -r prometheus-2.39.1.linux-amd64/console_libraries /etc/prometheus
sudo cp -r prometheus-2.39.1.linux-amd64/prometheus.yml /etc/prometheus

#Update Prometheus ownership on Directories
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
sudo chown -R prometheus:prometheus /etc/prometheus/prometheus.yml

#Check Prometheus Version
prometheus --version
promtool --version

#Editer /etc/prometheus/prometheus.yml lors de la modification de ce fichier ne pas oblier de restart le service prometheus

#Creating Prometheus Systemd file
sudo touch /etc/systemd/system/prometheus.service
sudo chmod 556 /etc/systemd/system/prometheus.service
sudo echo "[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries
[Install]
WantedBy=multi-user.target" > /etc/systemd/system/prometheus.service


sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
#sudo systemctl status prometheus

#Accessing Prometheus
sudo ufw allow 9090/tcp
```

#### 4.1.2 Installation de grafana

```
sudo apt-get install -y apt-transport-https
sudo apt-get install -y software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
                   
echo "deb https://packages.grafana.com/enterprise/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install grafana-enterprise -y
sudo systemctl daemon-reload
sudo systemctl start grafana-server
#sudo systemctl status grafana-server
sudo systemctl enable grafana-server.service

#Node_exporter installation
wget https://github.com/prometheus/node_exporter/releases/download/v1.4.0/node_exporter-1.4.0.linux-amd64.tar.gz
sudo tar -xvf node_exporter-*.*-amd64.tar.gz

#cd node_exporter-1.4.0.linux-amd64
sudo cp node_exporter-1.4.0.linux-amd64/node_exporter /usr/local/bin

# Creating Node Exporter Systemd service
#cd /lib/systemd/system
sudo touch /lib/systemd/system/node_exporter.service
sudo chmod 556 /lib/systemd/system/node_exporter.service
sudo echo "[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target
[Service]
Type=simple
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter
Restart=always
RestartSec=10s
[Install]
WantedBy=multi-user.target" > /lib/systemd/system/node_exporter.service

sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

```
#### 4.1.3 Installation de ELK

```
echo "##########################################################"
echo "# Etape 1 Installation des prerequis et de elasticsearch #"
echo "##########################################################"

curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update
sudo apt install elasticsearch -y

sudo sed -i '/#network.host: 192.168.0.1/c\network.host: 0.0.0.0' /etc/elasticsearch/elasticsearch.yml  
sudo sed -i '/#node.name: node-1/c\node.name: node-1' /etc/elasticsearch/elasticsearch.yml   
sudo sed -i '75i cluster.initial_master_nodes: ["node-1"]' /etc/elasticsearch/elasticsearch.yml


sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch

curl -X GET "localhost:9200"

echo "##########################################################"
echo "# Etape 2 Installation de Kibana                         #"
echo "##########################################################"

sudo apt install kibana -y
sudo systemctl enable kibana
sudo systemctl start kibana
sudo apt install nginx -y
echo "MyStrongCryptedPassword" > filepass.txt
echo "kibanaadmin:`openssl passwd -in filepass.txt -apr1`" | sudo tee -a /etc/nginx/htpasswd.users

sudo touch /etc/nginx/sites-available/elk_medicarche
sudo chmod 777 /etc/nginx/sites-available/elk_medicarche

sudo echo "
server {
    listen 90;

    server_name elk_medicarche;

    #auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/htpasswd.users;

    location / {
        proxy_pass http://localhost:5601;
        proxy_http_version 1.1;
        #proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        #proxy_set_header Host $host;
        #proxy_cache_bypass $http_upgrade;
    }
}
" > /etc/nginx/sites-available/elk_medicarche
sudo ln -s /etc/nginx/sites-available/elk_medicarche /etc/nginx/sites-enabled/elk_medicarche
sudo nginx -t
sudo systemctl reload nginx

echo "##########################################################"
echo "# Etape 3 Installation de Logstash                       #"
echo "##########################################################"
sudo apt install logstash -y
sudo touch /etc/logstash/conf.d/02-beats-input.conf
sudo chmod 777 /etc/logstash/conf.d/02-beats-input.conf
sudo echo "
input {
  beats {
    port => 5044
  }
}
" > /etc/logstash/conf.d/02-beats-input.conf
sudo touch  /etc/logstash/conf.d/30-elasticsearch-output.conf
sudo chmod 777 /etc/logstash/conf.d/30-elasticsearch-output.conf
sudo echo '
output {
  if [@metadata][pipeline] {
elasticsearch {
  hosts => ["localhost:9200"]
  manage_template => false
  index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  pipeline => "%{[@metadata][pipeline]}"
}
  } else {
elasticsearch {
  hosts => ["localhost:9200"]
  manage_template => false
  index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
}
  }
}
' > /etc/logstash/conf.d/30-elasticsearch-output.conf
sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
sudo systemctl start logstash
sudo systemctl enable logstash

echo "##########################################################"
echo "# Etape 4 Installation de l'agent Filebeat               #"
echo "##########################################################"
sudo apt install filebeat -y
# sudo nano /etc/filebeat/filebeat.yml Edit fileBeat à la mano
sudo filebeat modules enable system
sudo filebeat modules list
sudo filebeat setup --pipelines --modules system
sudo filebeat setup --index-management -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["localhost:9200"]'
sudo filebeat setup -E output.logstash.enabled=false -E output.elasticsearch.hosts=['localhost:9200'] -E setup.kibana.host=localhost:5601
sudo systemctl start filebeat
sudo systemctl enable filebeat
curl -XGET 'http://localhost:9200/filebeat-*/_search?pretty'

echo "##########################################################"
echo "# Etape 5 Installation de l'agent PacketBeat             #"
echo "##########################################################"

sudo apt install apt-transport-https -y
sudo apt update && sudo apt install packetbeat -y
sudo systemctl start packetbeat
sudo packetbeat setup

```
#### 4.1.4 Installation de GLPI

```
#Etape 1 Installation de GLPI
sudo apt update
sudo apt install php apache2 mariadb-server -y
sudo systemctl enable apache2 mariadb
sudo systemctl reload apache2

#sudo mysql_secure_installation

sudo mysql -u root --execute  "UPDATE mysql.user SET plugin = 'mysql_native_password' WHERE User = 'root'; FLUSH PRIVILEGES;"

sudo mysql -u root --execute  "CREATE USER 'glpi'@'localhost' IDENTIFIED BY 'StrongDBPassword';
CREATE DATABASE IF NOT EXISTS glpi CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
GRANT ALL PRIVILEGES ON glpi.* TO 'glpi'@'localhost';
FLUSH PRIVILEGES; "

sudo apt install perl -y
sudo apt install php-ldap php-imap php-apcu php-xmlrpc php-cas php-mysqli php-mbstring php-curl php-gd php-simplexml php-xml php-intl php-zip php-bz2 -y

sudo systemctl restart apache2

sudo apt-get -y install libapache2-mod-php
sudo apt-get -y install wget
export VER="9.3.2"
wget https://github.com/glpi-project/glpi/releases/download/$VER/glpi-$VER.tgz

tar xvf glpi-$VER.tgz
sudo mv glpi /var/www/html/
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 775 /var/www/html/glpi


# Etape 2 Installation du plugin fusionInventory

sudo apt-get update
sudo wget -P /usr/src https://github.com/fusioninventory/fusioninventory-for-glpi/archive/glpi9.3+1.3.tar.gz
sudo tar -zxvf /usr/src/glpi9.3+1.3.tar.gz -C /var/www/html/glpi/plugins
sudo chown -R www-data /var/www/html/glpi/plugins

cd /var/www/html/glpi/plugins
sudo mv fusioninventory-for-glpi-glpi9.3-1.3/ fusioninventory/
cd ~

#Installation Agent fusion Inventory dans le poste client

sudo apt-key adv --keyserver keyserver.ubuntu.com --recv 049ED9B94765572E
wget -O - http://debian.fusioninventory.org/debian/archive.key | apt-key add -
sudo apt-get install lsb-release -y 
sudo echo "deb http://debian.fusioninventory.org/debian/ `lsb_release -cs` main" >> /etc/apt/sources.list
sudo apt-get update
sudo apt-get install fusioninventory-agent -y
sudo sed -i "$ a server = http://ip_srv/glpi/plugins/fusioninventory" /etc/fusioninventory/agent.cfg

#Remonter les metrics
sudo fusioninventory-agent

```
