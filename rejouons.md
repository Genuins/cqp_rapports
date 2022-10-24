# Migration de l'informatique de MedicArche vers le Cloud - Rejouer le déploiement de A à Z


## I.	Introduction

Dans cette partie nous mettons à disposition des lecteurs les scripts qui permettent d'installer le cloud privé Openstack et de déployer les services cités dans la [page précedente](https://github.com/Genuins/cqp_rapports/edit/main/README.md). Pour le déploiement de l'infrastructure et l'installation des services de base, nous avons utilisés des scripts bash. Les interventions lié à la maintenance ou au déploiement des nouveaux services pourra s'éffectuer avec l'outil Terraform.

## II.	Environnement de travail


Tout ce qui suit a été réalisé et testé sur une machine Windows 10 équipé de 16Gb de mémoire RAM, de 300Gb de disque dur, d'un processeur i5 9500T. Pour éxecuter le script d'installation du cloud privé Openstack nous avons besoin des prérequis suivantes : 

* Installer [Virtualbox](https://www.virtualbox.org/) sur une machine dont le système d'exploitation est Windows 10.
* Installer [Vagrant](https://www.vagrantup.com/downloads) sur la machine Windows 10
* Installer le plugin Vagrant qui permet d'augmenter la taille du disque `vagrant plugin install vagrant-disksize `
* Installer [Packer](https://www.packer.io/downloads) sur la machine Windows 10.
* Installer l'outil [Terraform](https://www.terraform.io/downloads) de même sur la machine Windows 10.

## III.	Procédure de déploiement de l’infrastructure Medicarche

### 3.1	Installation d’Openstack


