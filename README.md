# Migration de l'informatique de MedicArche vers le Cloud - Rapport de synthèse

## I.	Introduction

La société Medicarche spécialisée dans la recherche sur les virus, est située sur deux sites, à LA DÉFENSE dans la Grande Arche (site dénommé ARCHE), et à Paris rue de l’Arcade, dans le 8ème arrondissement (site dénommé ARCADE). Sa croissance est actuellement importante en raison des demandes liées à la crise sanitaire. Aussi elle envisage d’ouvrir de nouveaux sites pour répondre aux besoins du marché. Pour répondre au cahier de charge de l’entreprise Medicarche concernant la migration vers le cloud, nous expliquons, dans ce document, comment réaliser un déploiement automatisé de l’infrastructure MedicArche dans le cloud. 

Pour cela, nous déployons, entre autre chose, trois applications emblématiques (Syncthing, Nextcloud et Odoo) et un site Web via le CMS Wordpress dans une solution OpenStack qui est un orchestrateur de Cloud open source. Les différentes applications sont hébergées dans un sous réseau privé de l'orchestrateur dont seuls les collaborateurs de l’entreprise auront accès. Pour rendre cela possible nous avons développé deux types de script. Le premier script se déploie de bout en bout sans interaction humaine en exécutant la commande de déploiement du système OpenStack et le second script, qui est considéré comme le cœur du projet, peut se déployer après le premier et il installe les services sus mentionnés. Dans les lignes qui suivent nous décrivons l’environnement de travail de l’entreprise Medicarche, le processus de déploiement de l’infrastructure, le plan de test, l’accompagnement au changement et enfin nous pourrons conclure. 

## II.	Environnement de travail

L’environnement de travail qui a permis de mettre en place le déploiement de l’infrastructure de l’entreprise Medicarche est composé d’une machine hôte dont le système d’exploitation est Microsoft Windows 10. La machine hôte est une machine dont le type de processeur est un Intel i5 composé de 8 cœurs physique et 4 cœurs logique, d’une mémoire RAM de 16 gigabytes et d’un disque dur de 300 gigabytes. Comme hyperviseur nous avons utilisé VirtualBox qui est un hyperviseur open source. Pour faciliter la communication au niveau réseau nous utilisons deux cartes réseaux. La première carte réseau est configurée en utilisant l’option NAT qui permettra la communication de la machine virtuelle hôte vers internet et la deuxième carte réseau est configurée avec l’option hôte privé pour faciliter la communication entre la machine virtuelle et la machine physique.

Le déploiement a été réalisé grâce à Vagrant qui est un outil de création et de configuration d'environnements de développement virtuels qui facilite aussi l’automatisation. A l’aide de Vagrant nous avons déployé une machine virtuelle avec les caractéristiques suivantes :

-	Système d’exploitation Linux Ubuntu Focal 20.04
-	Disque dur de 100 gigabytes
-	Mémoire RAM de 10 gigabytes
-	12 vCPUs
-	Adresse IP 192.168.33.16 

Ces paramètres se retrouvent dans la partie centrale de la figure 1.

<p align="center">
 <img src="https://github.com/Genuins/cqp_rapports/blob/main/images/virtualbox_ui.JPG?raw=true" alt="Virtualbox" style="width: 45vw; min-width: 230px;" />
 <p align="center"> Figure 1 : Interface graphique de virtual box </p>
</p>
  
Par ailleurs, Visual Code est un environnement de développement intégré qui facilite la création de scripts. Pour administrer la machine hôte qui héberge le cloud privé de l’entreprise Medicarche nous avons installé des plugins. Parmi les extensions que nous avons installées nous avons RemoteSSH une extension qui permet de se connecter en mode SSH dans les machines virtuelles se trouvant dans le cloud privé de l’entreprise Medicarche pour pouvoir effectuer des actions d’administration si cela s’avère nécessaire. 

Nous avons également travaillé avec Packer qui est une solution open source permettant de construire des images système pour de multiples plateforme cloud. Pour construire les images dont nous avons besoin sous Openstack nous avons récupéré une image Linux Ubuntu Bionic 18.04 depuis le site officiel de Openstack avec le format QCOW2. Pour personnaliser l’image nous avons utilisé Packer en définissant un utilisateur par défaut et le mot de passe, la capacité maximale de mémoire RAM, de disque dur, le format, le format du clavier, la langue du système d’exploitation et le nombre de processeurs pouvant être utilisé.

Enfin, nous avons utilisé Git qui est un logiciel de gestion de versions décentralisé. Il s'agit d'un logiciel libre et gratuit. Cet outil nous a été utile pour faire du versioning des scripts utilisés tout au long du déploiement de l’infrastructure. 

## III.	Procédure de déploiement de l’infrastructure Medicarche

Le déploiement de l’infrastructure se fait en utilisant le concept de Infrastructure as a code (IaaS). Pour la mise en place de l’infrastructure Openstack et le déploiement des applications (Syncthing, Nextcloud, Odoo), nous avons utilisé des scripts bash pour automatiser l’installation.

Les scripts sont écrits de sorte à automatiser :
-	Le déploiement de l’infrastructure Openstack ;
-	La création des instances ;
- La création des pairs de clé RSA pour les instances ;
- La création des règles de sécurité ;
- Le clonage et le téléversement de l’image Linux Ubuntu et son déploiement ;
- La configuration de la connexion SSH de chaque instance pour assurer le déploiement des applications qui sont Syncthing, Nextcloud, Odoo et le site de l’entreprise Medicarche.

<p align="center">
 <img src="https://github.com/Genuins/cqp_rapports/blob/main/images/maquet1.png?raw=true" alt="Maquete 1"/ style="width: 45vw; min-width: 330px;">
 <p align="center"> Figure 2 : Maquette de base proposée lors de l'appel d'offre</p>
</p>

<p>Initialement nous avons commencé par une mise en œuvre la maquette de base décrite à la figure 2. À la suite de difficultés rencontrées en rapport avec les performances de la machine virtuelle hôte, nous avons été contraints d'utiliser les services de base que propose Openstack.</p>

<p align="center">
 <img src="https://github.com/Genuins/cqp_rapports/blob/main/images/maquettes_int%C3%A9gr%C3%A9_bis.JPG?raw=true" alt="Maquete 2"/ style="width: 45vw; min-width: 330px;">
 <p align="center"> Figure 3 : Maquette de base proposé lors de l'appel </p>
</p>
  

La maquette réalisée, afin de donner accès aux applications, depuis le Système hôte Windows, est décrite à la figure 3. Nous avons pensé à la mise en place d’un serveur mandataire (reserve proxy).

Avant d’entrer dans le vif du sujet nous aimerons faire une piqure de rappel concernant l’architecture de base présentée lors de la conception de l’infrastructure Medicarche. Cette architecture représente les besoins réels effectués lors de la rédaction du cahier de charge. Lors de la conception de la plateforme de Medicarche, nous avons proposé deux types de solution cloud qui sont AWS un fournisseur public et Openstack un Cloud de type privé. L’entreprise Medicarche avait opté pour Openstack lors de l’appel d’offre. 

En résumé, le déploiement de l’infrastructure Medicarche a été scripté de bout en bout. Pour commencer le déploiement, nous exécutons la commande vagrant up, une commande de l’outil vagrant qui permet de créer une machine virtuelle. Nous avons provisionné le script de déploiement dans le fichier vagrantfile afin d’automatiser l’installation de Openstack ainsi que le déploiement des applications et du site web. La figure 4 détaille l'ensemble des phases de cette partie du déploiement.


<p align="center">
 <img src="https://github.com/Genuins/cqp_rapports/blob/main/images/vagrantfile.JPG?raw=true" alt="Script Vangantfile"/ style="width: 45vw; min-width: 330px;">
 <p align="center"> Figure 4 : Script vagrant qui permet de déployer une machine virtuelle provisionnée. </p>
</p>
  

### 3.1	Installation d’Openstack

OpenStack est un ensemble de logiciels open source permettant de déployer une infrastructure de type cloud privé en suivant le modèle Infrastructure as a Service (IaaS). Une fois l'environnement Openstack installé, des accès (credantials) nous sont fournis afin d'accéder à la console en tant qu’administrateur.  Par défaut, Openstack configure la partie réseau en créant un sous réseau privé, un sous réseau public, la table de routage et d'autres services pour ne citer que ceux-là. La commande iptables nous permet d’autoriser le trafic venant de Openstack vers internet. Cet partie du déploiement correspond à ce qui est présenté à la figure 5.


<p align="center">
 <img src="https://github.com/Genuins/cqp_rapports/blob/main/images/installation_microstack.JPG?raw=true" alt="Script installation Microstack"/ style="width: 45vw; min-width: 330px;">
 <p align="center"> Figure 5 : Script d'installation Microstack</p>
</p>
  
a)	Tableau de bord 

Un tableau de bord permet de visualiser l’ensemble de ressources de l’infrastructure. Cela nous permet aussi de garder un œil sur l’évolution de l’infrastructure. Nous pouvons obtenir une vue sur les instances en cours d’utilisation, la quantité des vCPUs, la quantité de la RAM allouée aux instances en cours d’utilisation, les volumes, les groupes de sécurité, les sous-réseaux, les ports, les routeurs ainsi les adresses IP publiques des instances. 

b)	Topologie réseau dans Openstack

Lors de l’installation de Openstack plusieurs fonctionnalités sont installées par défaut. Une topologie réseau représentant l’architecture du réseau. Nous avons utilisé un sous réseau privé et un sous réseau public.  Pour des raisons de sécurité, nous avons déployé les machines virtuelles contenant les applications web dans le réseau privé et nous avons pensé à leur attribuer une adresse IP privée dont la passerelle est 192.168.222.1. Pour permettre la communication avec internet, un routeur virtuel fait office de passerelle entre le réseau privé et le sous réseau public. L’adresse IP de la passerelle est 10.20.20.224. Cette partie du déploiement est reprise à la figure 6.

<p align="center">
 <img src="https://github.com/Genuins/cqp_rapports/blob/main/images/archi_reseau.JPG?raw=true" alt="Topologie réseau"/ style="width: 45vw; min-width: 330px;">
 <p align="center"> Figure 6 : Topologie du réseau dans Openstack </p>
</p>

c)	Les gabarits

Openstack permet de personnaliser des gabarits qui sont utilisés pour déployer les instances. Le plus petit gabarit par défaut consomme 512 Mo de mémoire par instance. Dans le contexte du déploiement de la plateforme Medicarche nous avons créé différents types de gabarits qui permettent d’accueillir les applications. Lors de la création du gabarit on définit les paramètres suivants :

-	Le nom ; 
-	Le VCPU ;
-	La RAM ;
-	Le disque dur principal ;
-	Possibilité d’ajouter un disque dur éphémère ;
-	La portée public ou privée ;
-	Le swap ;

On a aussi la possibilité d’ajouter des metadatas pour mieux identifier le gabarit. Sur la figure 7, nous avons trois exemples de création de gabarit qui correspondent à trois machines virtuelles identiques sauf pour le disque dont la capacité est de 5, 8 et 9 gigabits.

<p align="center">
 <img src="https://github.com/Genuins/cqp_rapports/blob/main/images/gabarits.JPG?raw=true" alt="Topologie réseau"/ style="width: 45vw; min-width: 330px;">
 <p align="center"> Figure 7 : Script de creation des gabarits </p>
</p>

d)	Les images

Par défaut, Openstack offre une image Cirros qui permet d’effectuer des tests au niveau de la couche réseau. Cette instance ne peut pas être utilisée dans notre contexte car elle est trop limitée. Nous avons automatisé la création et le téléversement de l’image utilisable dans Openstack. Une image Linux Ubuntu Bionic sera utilisée pour toutes les instances. Lors du téléversement de l’image on définit le nom du système d’exploitation, la visibilité et le format du disque (QCOW2).

Pour pouvoir mettre en place cette solution nous avons créé un dépôt git pour stocker les images que nous allons utiliser lors de l’automatisation. On commence par cloner le dépôt à la racine puis on exécute la commande microstack.openstack pour créer l’image ubuntu depuis le dossier os-ubuntu-openstack.

e)	Les instances

Les instances dans Openstack sont des machines virtuelles qui s’exécutent dans un sous réseau. Nous avons déployé les applications "web" Odoo, Nextcloud, Syncthing et un serveur Apache pour le site web de l’entreprise Medicarche. Pour chaque application nous avons utilisé un gabarit spécifique afin de respecter le minimum requis de ressources (CPU, RAM, Disque) tel qu'il est spécifié dans la documentation des applications.

Nous avons pensé à mettre les machines virtuelles dans un même sous réseau privé et avons utilisé un groupe de sécurité pour apporter cette dimension à notre architecture. Le groupe de sécurité a pour objectif de mettre en place le principe de moindre privilège c’est-à-dire de n'autoriser que les ports nécessaires en entrée. Parmi les ports autorisés pour atteindre les machines virtuelles nous avons :
-	Le port 8088 qui permet d’atteindre le site web de l’entreprise ;
-	Le port 8087 qui permet d’atteindre l’application Odoo ;
-	Le port 8806 qui permet de communiquer avec l’application Nextcloud ;
-	Le port 8885 qui permet de communiquer avec l’application Syncthing ;
-	Le port 22 pour assurer une connexion sécurisée en SSH ;
-	Le Port 80 pour autoriser la communication en http.

Les machines virtuelles ont toutes une adresse IP privée et une adresse IP publique afin de communiquer sur le réseau privé (communication entre les machines virtuelles) et sur le réseau externe pour la communication avec l’extérieur. Openstack propose un outil pour le log de toutes les actions effectuées sur la machine virtuelle. Elles peuvent être auditées grâce à cette fonctionnalité. Les machines virtuelles utilisent une clé privée crée lors du déploiement cela permet de mettre un accent sur la sécurité. Le script de la figure 8 donne un aperçu de la création de la machine virtuelle et l'association d'une adresse ip virtuelle.

<p align="center">
 <img src="https://github.com/Genuins/cqp_rapports/blob/main/images/instances.JPG?raw=true" alt="création de la virtuelle machine"/ style="width: 45vw; min-width: 330px;">
 <p align="center"> Figure 8 : Script de création de la virtuelle machine </p>
</p>

f)	Les groupes de sécurité

Un groupe de sécurité agit en tant que pare-feu virtuel pour les instances afin de contrôler le trafic entrant et sortant. Nous avons mis en place un groupe de sécurité nommé private-sg afin d’autoriser que les ports et les adresses IP nécessaires à la communication des instances (voir plus haut). Afin de resserrer le trafic entrant nous avons appliqué la stratégie de moindre privilège en autorisant que le strict nécessaire.
 
Openstack offre une interface d’administration des groupes de sécurité, on peut ajouter, modifier ou supprimer le trafic entrant ou le trafic sortant. En cas de dépannage rapide cela permettra à la DSI de Medicarche de pouvoir intervenir pour pouvoir régler des problèmes. 

g)	Les pairs de clé 

Pour avoir accès à la machine virtuelle et pouvoir faire les opérations d’administration, nous avons, lors de la création des instances, mis en place un script qui permet de créer une paire de clés RSA et de les associer à l’instance. Ce type de connexion permet de renforcer la sécurité ; ainsi seuls les administrateurs de Medicarche pourrons avoir accès aux instances pour effectuer les taches de maintenances. Le script de la figure 9 permet d’automatiser la création d'une clé pour se connecter en SSH dans la machine virtuelle. On crée la paire de clé en attribuant le droit au propiretaire de la clé et en modifiant le fichier en lecture seule.
 
<p align="center">
 <img src="https://github.com/Genuins/cqp_rapports/blob/main/images/key.JPG?raw=true" alt="création de paire de clé"/ style="width: 45vw; min-width: 330px;">
 <p align="center"> Figure 9 : Script de création de la paire de clés. </p>
</p>

h)	Gestions des utilisateurs 

Concernant la création des utilisateurs nous avons jugé plus pertinent de le faire manuellement et non en passant par un script. Openstack offre une interface pour administrer les utilisateurs. La stratégie que nous avons adoptée est celle du moindre privilège. L’administrateur ayant tous les droits sera utilisé qu’en cas de nécessité. Nous créerons les utilisateurs de la DSI qui auront les autorisations pour gérer l’infrastructure. Les rôles pourront être créé en ligne de commande afin d’autoriser les utilisateurs à accéder à des ressources dont ils ont le droit. Trois types de rôle sont créés par défaut : 

-	Membre ;
-	Admin ;
-	Reader.
 
i)	Les adresses IP flottantes

Les adresses IP flottantes permettent aux instances de pouvoir accéder à Internet. Elles ont été créées lors du déploiement en ligne de commande. Une interface de gestion des adresses IP flottantes permet de gérer les adresses IP en mode graphique. On peut les attribuer (ou les retirer) à une instance à tout moment. La commande suivante permet de créer une adresse IP flottante en ligne de commande : `microstack.openstack floating ip create external`. Chaque instance ne peut avoir qu’une seule adresse IP flottante.  Pour en obtenir une nouvelle, l’administrateur système devra libérer l’ancienne adresse IP avant d’en utiliser une nouvelle. 
 
## IV.	Déploiement des applications de l’entreprise Medicarche

Pour assurer le partage des documents, la synchronisation des données entre les services, les gestions des activités de l’entreprise nous avons mise en place un script qui permet de procéder à l’installation, la configuration ainsi que le déploiement des applications dans les machines virtuelles.
Pour répondre à la demande de l’entreprise nous avons proposé les applications suivantes :
-	Odoo ;
-	NextCloud ;
-	Syncthing ;
-	Wordpress.

Nous donnons un aperçu des différentes fonctionnalités de chacune de ces applications un peu plus loin dans le texte et au moment des explications sur leurs déploiements. L’architecture de déploiement reste la même pour toutes les applications. Elle est donc générique. Le déploiement de la base de données et du serveur web se fait dans l’instance. Cela n’est pas une solution qui permet de rendre l’architecture hautement disponible et tolérante aux pannes mais à la suite des contraintes rencontrées en termes de performance de la machine hôte et du nombre réduit de machines virtuelles pouvant être utilisées nous avons préféré déployer les applications en favorisant une architecture monolithe.
 
### 4.1	Installation d’Odoo

Odoo est initialement un progiciel open-source de gestion intégré comprenant de très nombreux modules permettant de répondre à de nombreux besoins de gestion des entreprises, ou de gestion de la relation client. Dans le contexte Medicarche, l’application Odoo utilise une base de données PostgreSQL installée en local dans la machine virtuelle. Pour exposer l’application web et permettre l’accès aux collaborateurs, l’utilisation du reverse proxy ou serveur mandataire a été mis en place afin d’exposer l’application web de manière sécurisée. 

### 4.2	Installation de Nextcloud

Nextcloud est un logiciel libre de site d'hébergement de fichiers et une plateforme de collaboration. La solution NextCloud apporte un haut niveau de protection et de contrôle des informations et communications dans l'entreprise. Les utilisateurs peuvent conserver toutes leurs données sur leurs serveurs internes, y compris les métadonnées. Dans le contexte Medicarche, NextCloud utilise en local un serveur Apache avec une base de données MySQL. Ces services sont tous installés dans une seule machine virtuelle.  De même, nous avons mis en place un serveur mandataire pour exposer l’application web afin de la rendre accessible sur internet de manière sécurisée. 

### 4.3	Installation de Syncthing

Syncthing est une application de synchronisation de fichiers pair à pair open source disponible pour Windows, Mac, Linux, Android, Solaris, Darwin et BSD. Aucun compte ni enregistrement préalable à l'utilisation auprès d'un tiers n'est nécessaire, ni même optionnelle. L’application tourne localement dans la machine virtuelle en utilisant l’adresse IP 127.0.0.1. Pour avoir accès depuis l'extérieur de la machine virtuelle, on doit utiliser le reverse proxy pour pouvoir exposer l’application à tous les collaborateurs de l’entreprise Medicarche.

### 4.4	Site web Medicarche

Wordpress est un système de gestion de contenu (CMS) libre et open-source. Ce logiciel écrit en PHP repose sur une base de données MySQL. Le site de l’entreprise est exposé sur internet permettant à toute personne le désirant d’y avoir accès. Comme pour les autres applications web, le site web de l’entreprise est déployé sur un serveur Apache et utilise une base de données MySQL. Nous avons veillé à appliquer aussi le reverse proxy afin d’exposer le site sur internet.

## V.	Description du plan des tests

Dans le cadre du projet de déploiement il est important de passer des tests. Plus précisément, nous avons effectué un plan de test de capacité du système, un test de montée en charge des applications ou du site web de l’entreprise Medicarche et un test de stress du système. Nous documentons cela dans les lignes qui suivent.

### 5.1	Test de montée en charge 

Le test de montée en charge que nous prévoyons d’effectuer est un test au cours duquel nous aurons simulé un nombre d'utilisateurs sans cesse croissant de manière à déterminer quelle charge limite le système est capable de supporter sans tomber. Nous effectuerons un test sur l’application Odoo, Nextcloud, Syncthing et le site web de l’entreprise Medicarche. Nous avons prévu de mettre en place un script en utilisant l’outil h2load. Une fois l’utilitaire installé, on a à exécuter la commande décrite à la figure 10, que l’on prévoit de scripter, pour toutes les autres applications de la plateforme Medicarche. Dans ce script nous simulons 100 clients qui produiront au total 10000 requêtes HTTP. En résumé, le script de la figure 10 permet de faire le test de montée en charge d'une machine virtuelle en passant en paramètre le nombre de clients et le nombre de requêtes.

<p align="center">
 <img src="https://github.com/Genuins/cqp_rapports/blob/main/images/test_mg.JPG?raw=true" alt="Montée en charge h2load"/ style="width: 45vw; min-width: 330px;">
 <p align="center"> Figure 10 : Test de montée en charge </p>
</p>


### 5.2	Test de stress du système

Pour s’assurer que les gabarits choisis pour accueillir les applications web de l’entreprise Medicarche répondent bien au minimum des capacités requises, nous mettons en place ce type de test qui a pour objectif de stresser le CPU, la mémoire RAM, le I/O et le disque. Pour cela nous utilisons l’outil stress and stress-ng. L’installation s'effectue en ligne de commande et on pourra voir le résultat sur l’interface CLI. Sur la figure 11 nous pouvons voir sur le tableau de bord les points importants comme le traffic du réseau, les nombres des requêtes, le nombre des nouveaux utilisateurs, des tentatives de connexion.

<p align="center">
 <img src="https://github.com/Genuins/cqp_rapports/blob/main/images/supervison_vm.JPG?raw=true" alt="monitoring"/ style="width: 45vw; min-width: 330px;">
 <p align="center"> Figure 11 : monitoring de la machine virtuelle </p>
</p>


### 5.3	Test de performance 

Après avoir installé les applications web de l’entreprise Medicarche, on doit vérifier que le système répond normalement aux attentes du client. Pour s’assurer que c’est réellement le cas, nous avons choisi d'utiliser l’outil Tsung qui est approprié pour ce type de test. Tsung est un outil de test de performances permettant de réaliser des benchmarks massifs. De manière résumée, Tsung est un outil qui nous permettra de faire un rapport sur les statistiques du système, des transactions, du réseau, du débit et même des statuts liés aux requêtes HTTP.

 ## VI.	Supervision avec Grafana et Prometeus
 
 ### 6.1 Introduction à la supervision
 
 La supervision cloud, on parle également de monitoring cloud, consiste en une série d'opérations d'analyse, de recueil d'information et de gestion qui contrôle un flux de travail cloud. Un exemple de flux de travail cloud peut être l'ingestion dans un cloud de données en provenance de capteurs, leur nettoyage et mise en forme avant de les déposer dans un gestionnaire de données à des fins d'analyse. Le monitoring cloud peut utiliser des services ou des outils de monitoring manuels (installés manuellement) et/ou automatisés (installés via une démarche DevOps) pour vérifier qu'un cloud est opérationnel.

En un mot, la surveillance du cloud est une méthode d'examen, d'observation et de gestion du flux opérationnel dans une infrastructure informatique basée sur le cloud. Des techniques de gestion manuelles ou automatisées permettent de confirmer la disponibilité et les performances des sites Web, des serveurs, des applications et d'autres infrastructures en nuage. Cette évaluation continue des niveaux de ressources, des temps de réponse des serveurs et de la vitesse a pour objectif de prévoir un état problématique éventuel d'un service cloud avant que des problèmes plus graves surviennent, par exemple la défaillance totale de tous les services cloud.

Dans ce qui suit nous utilisons des outils bien connus dans la communauté pour réaliser de la supervision cloud, outils déployés soit manuellement soit automatiquement, pour surveiller soit les VM, donc le système d'exploitation de la VM (sous section 6.2), soit de manière automatique (sous section 6.3), pour surveiller le cloud i.e. les services disponibles pour OpenStack / MicroStack. 


### 6.2 Solutions de supervision des VMs

Parmi les solutions le plus utilisée d'après l'outil Google Trend pour faire de la supervision sont Prometheus, Nagios, InfluxDB, Grafana. Dans cette section, nous allons décrire les fonctionnalités de manière generale de ces outils.

 ### 6.2.1 Grafana
 
 Grafana est un outil qui permet de visualiser les données à travers un tableau de bord. Il permet de réaliser des tableaux de bord et des graphiques depuis plusieurs sources dont des bases de données temporelles comme Graphite, InfluxDB, Prometheus et Elasticsearch.
 
 
 ### 6.3 Prometheus
 
 ### 6.4 Conclusion
 
 ## VII- Mise en place d'un annuaire de type LDAP
 
 ### 7.1 Introduction à la problématique
 
 Dire que pour l'instant les comptes sont locaux aux VM (et donc aux applications) et on veut passer à un annuaire centralisé.
 
 ### 7.2 Mise en place d'un annuaire LDAP
 
 ### 7.3 Vérifications fonctionnelles
 
 ### 7.4 Conclusion
 
## VIII. Information d’accompagnement au changement

L’accompagnement au changement se fait en plusieurs étapes pour éviter toute forme de rejet auprès des collaborateurs de l’entreprise Medicarche afin d’accepter facilement le changement. En effet, les employés sont déstabilisés car ils se retrouvent hors de leur zone de confort. L’annonce du changement et de la migration de l’infrastructure Medicarche dans le cloud suscitera un éventail de réactions émotionnelles diverses :

- Le déni ; 
-	La colère ;
-	La peur et la tristesse ;
-	La négociation, le chantage ;
-	La résignation.

En tant que manager, il est essentiel de savoir reconnaître ces réactions, de ne pas les ignorer et d'y apporter une réponse adaptée. 

La première étape lors d'un changement est de communiquer en expliquant les raisons aux collaborateurs de Medicarche et en étant transparent et cohérent.
Le manager doit réunir tout le personnel concerné et impacté par ce changement en exposant les différentes raisons ; les avantages de la migration dans le cloud ou du nouveau logiciel. Les bénéfices pour les salariés, sans pour autant négliger de citer les aspects moins attractifs, doivent être discutés. Une série de questions réponses sera organisée afin de répondre honnêtement aux questions des collaborateurs. 

La deuxième étape est une proposition de formation qui permettra aux collaborateurs de prendre en main la nouvelle infrastructure. Certains collaborateurs refusent le changement simplement parce qu'ils craignent de ne pas être à la hauteur, ou parce qu'ils doutent de leur capacité d'adaptation. Ces craintes concernent tous les niveaux de l'entreprise, du manager à l'opérateur. 

Pour y remédier, nous proposons du coaching, de l'accompagnement et de la formation aux salariés concernés. Le coaching du changement peut aider grandement le manager à motiver et à engager son équipe. Quant aux collaborateurs, ils sont accompagnés dans leur processus d'adhésion au changement.

La troisième étape est de les accompagner en encourageant les collaborateurs, reconnaître leurs efforts, être proche d’eux pour qu'ils ne se sentent pas délaissés. Un compliment est tellement plus fort qu'une critique. 

La quatrième étape est de mettre en place une cellule de suivi pendant une certaine période afin de s'assurer que le système fonctionne correctement et donne satisfaction aux utilisateurs.

## IX.	Conclusion

Pour conclure, OpenStack est une infrastructure open source d'Infrastructure as a Service (IaaS) pour déployer un cloud privé, qui offre un potentiel en évolutivité, en permettant à un grand nombre de nœuds interconnectés de fournir les services nécessaires. OpenStack offre également de la flexibilité en ayant des composants modulaires qui interagissent pour former l'infrastructure finale. Les modules des composants peuvent être ajoutés ou supprimés lorsque la nécessité s'en fait sentir. Dans ce document nous avons considéré une version particulière d'OpenStack, à savoir Microstack, à travers de laquelle nous pouvons facilement déployer l'infrastructure cible. Micostack déploie OpenStack avec des exigences système minimales et gère également le fardeau de la configuration d'OpenStack et de son réseau avant le déploiement. L'intention principale de Microstack est de fournir un environnement OpenStack dans un système du développeur à des fins de test ou de développement. Microstack fait partie de Canonical et cela ne fonctionne que sur Ubuntu. Il existe divers outils disponibles qui aident au déploiement comme Devstack. L'inconvénient d'une telle solution, que nous avons pu mesurer lors du projet, est que cela prend beaucoup de temps et que parfois, la mise au point du déploiement des configurations doit être effectuée manuellement. Le projet nous a permis de simuler le déploiement d’une infrastructure "large échelle" en mettant en place l’automatisation. Le seul inconvénient pour être le plus réaliste possible réside au niveau de la performance de la machine hôte qui ne correspond pas à ce type de projet. 

## X.	Webographie

- Rédiger un plan de test -  https://www.ingenieurtest.fr/2018/04/rediger-un-plan-de-test.html
- Tsung - https://blog.admin-linux.org/administration/tsung-outils-de-benchmark-multi-protocole
- Microstack - https://microstack.run/
- Odoo - https://fr.wikipedia.org/wiki/Odoo
- Nextcloud - https://nextcloud.com/
- Syncthing -https://doc.ubuntu-fr.org/syncthing
- Repository medicarche - https://gitlab.com/Genuiz/medicarche_ostack.git 
- Repository medicarche core - https://gitlab.com/Genuiz/medicopen.git 
