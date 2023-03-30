
# Projet Sur la Mise en Place d'un Reverse Proxy Local

Dans ce Projet il sera question pour nous de mettre en place un reverse Proxy
nous permettant de mettre à disposition des ressources heberger sur des serveurs interne
vers l'exterieur.

***NB: Ce projet est basé sur une implémentation en local***



## Installation
### Préréquis

- Un ordinateur (PC/Desktop)
- Une connexion Internet
- D'un Système d'exploitation Linux (CentOs7 version minimal dans notre cas)
**Link: https://ftp.riken.jp/Linux/centos/7.9.2009/isos/x86_64/**
- D'un outil de virtualisation de niveau 2 (VirtualBox dans notre cas)
**Link: https://www.virtualbox.org/wiki/Downloads**
- D'un gestionnaire de code source (Git)
- D'un serveur web (Nginx)

## Mise en place de l'environnement
### Caracteristique de nos machines Virtuelles
- Système d'exploitation : CentOs7 
- CPU: 1
- RAM: 1
- Disque: 20 Go
- Carte réseaux:
    + NAT 
    + Réseaux privé hôte
### Configuration de la carte réseaux
Notre machine virtuelle est constituee de 2 cartes reseaux, une NAT(Network Address Translation) et une en état de réseaux privé hôte.
- La carte NAT nous permettra d'avoir un accès à internet et sera configurer de manière dynamique par notre Hyperviseur VirtualBox.
- La seconde carte sera configurer par nous car elle nous permettra d'avoir un accès distant/externe au terminal de notre machine virtuelle.

Pour se faire: \
Se rendre dans le repertoire de configuration des cartes réseaux:

```bash
[root@reverse-proxy ~]# cd /etc/sysconfig/network-scripts/
[root@reverse-proxy network-scripts]# ls
ifcfg-enp0s3      ifdown-post      ifup-eth     ifup-Team
ifcfg-enp0s8      ifdown-ppp       ifup-ippp    ifup-TeamPort
```
Nous allons voir deux fichiers portant le nom de nos interfaces(Cartes réseaux):
- ifcfg-enp0s3 qui correspond à notre carte NAT
- ifcfg-enp0s8 qui correspond à notre carte Privé hôte
Nous modifions les fichiers de configuration de tous nos serveurs selon se format:
```bash
[root@reverse-proxy network-scripts]# vi ifcfg-enp0s8
NM_CONTROLLED=yes
NAME=enp0s8
UUID=91f44fa7-4a5b-42ff-a39a-403102c2db7e
DEVICE=enp0s8
ONBOOT=yes
IPADDR=192.168.56.20
NETMASK=255.255.255.0
PEERDNS=no
```
Une fois la modification effectue il faut enregistrer le fichier et redemarrer le service reseau pour que les modifcations soient prise en compte.
```bash
[root@reverse-proxy network-scripts]# service network restart
Restarting network (via systemctl):                        [  OK  ]
```
Notre configuration IP :
| Nom du serveur  | Addresse IP       | Description/ressource |
| :---------------|:---------------:  | -----:                |
| reverse-proxy   |   192.168.56.20/24|  serveur proxy        |
| server1         | 192.168.56.21/24  |   Site web 1          |
| server2         | 192.168.56.22/24  |    Site web 2         |

### Installation des utilitaires

- Mise à Jour de notre Système:
```bash
[root@reverse-proxy ~]# yum update -y
```
- Installation de Git
```bash
[root@reverse-proxy ~]# yum install git -y
```
- Installation de Nginx
```bash
[root@reverse-proxy ~]# yum install epel-release nginx -y
```
### Configuration de Nginx
- Activation de Nginx
```bash
[root@reverse-proxy ~]# systemctl enable --now  nginx
```
- Démarrage de Nginx
```bash
[root@reverse-proxy ~]# systemctl start nginx
```
### Configuration du Parefeu Linux (Firewalld)
- Nous allons autoriser le protocole de communication web
```bash
[root@reverse-proxy ~]# firewall-cmd --zone=public --permanent --add-service=http
success
[root@reverse-proxy ~]# firewall-cmd --zone=public --permanent --add-service=https
success
[root@reverse-proxy ~]# firewall-cmd --reload
success
[root@reverse-proxy ~]#
```
- Une fois la configuration terminee nous verifions que les ports sont bien ouverts:
```bash
[root@reverse-proxy ~]# telnet 192.168.56.22 80
Trying 192.168.56.22...
Connected to 192.168.56.22.
Escape character is '^]'.
```
Le port 80 est bien opérationnel dans notre environnement.

### Téléchargement de nos ressources
- server1:
```bash
[root@server1 ~]# git clone https://github.com/MrUSN/sun-website.git
Cloning into 'sun-website'...
remote: Enumerating objects: 25, done.
remote: Counting objects: 100% (25/25), done.
remote: Compressing objects: 100% (24/24), done.
remote: Total 25 (delta 0), reused 25 (delta 0), pack-reused 0
Unpacking objects: 100% (25/25), done.
[root@server1 ~]#
```
- server2:
```bash
[root@server2 ~]# git clone https://github.com/diranetafen/static-website-example.git
Cloning into 'static-website-example'...
remote: Enumerating objects: 75, done.
remote: Total 75 (delta 0), reused 0 (delta 0), pack-reused 75
Unpacking objects: 100% (75/75), done.
[root@server2 ~]#
```
### Copie de nos ressources vers les repertoires appropriés 
- server1:
```bash
[root@server1 ~]# cp -r sun-website/* /usr/share/nginx/html/
```
- server2:
```bash
[root@server2 ~]# cp -r static-website-example/* /usr/share/nginx/html/
cp: overwrite ‘/usr/share/nginx/html/index.html’? yes
[root@server2 ~]#
```
### Configuration de notre DNS  interne
Pour se faire il suffit de se rendre dans le fichier host de notre machine 
et de faire un enregistrement DNS.
```bash
192.168.56.20	lab.io
192.168.56.20	server1.lab.io
192.168.56.20	server2.lab.io
```
### Configuration du serveur nginx (serveur reverse Proxy)
Nous allons créer 02 fichiers de configuration nginx au sein de notre serveur reverse.
- se rendre dans le repertoire conf.d 
```bash
[root@reverse-proxy ~]# cd /etc/nginx/conf.d/
```
- créer le fichier de redirection du serveur 1
```bash
[root@reverse-proxy conf.d]# vi server1.lab.io.conf
```
- insérer les informations suivante
```bash
server {
        listen 80;
        server_name server1.lab.io;
        location / {
                proxy_pass http://192.168.56.21;
        }
}
```
- créer le fichier de redirection du serveur 2
```bash
[root@reverse-proxy conf.d]# vi server2.lab.io.conf
```
- insérer les informations suivante
```bash
server {
        listen 80;
        server_name server2.lab.io;
        location / {
                proxy_pass http://192.168.56.22;
        }
}
```
Une fois les fichiers créés il nous faut recharger le serveur nginx pour qu'il prenne en considération les nouvelles configurations
```bash
[root@reverse-proxy conf.d]# systemctl reload nginx
```

### Vérification de l'accès à nos ressources depuis le serveur reverse proxy via les sous domaines
Il vous suffit d'ouvrir un navigateur web et d'entrer les différents nom de domaines configurés
- server1
![Site web 1](/image/server1.png)
- server2
![Site web 2](/image/server2.png)





## Authors

- [@Ulrich](https://www.github.com/ulrichnoumsi)
- [@Kevin](https://github.com/kev-skywalker)

