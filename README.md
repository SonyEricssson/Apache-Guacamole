<img src="https://github.com/user-attachments/assets/93a9444b-5790-49e6-b5a0-809acd499a06" alt="Apache Guacamole Logo" width="100">

# Apache-Guacamole
Apache Guacamole : Un Bastion d'Administration
# Apache Guacamole : Un Bastion d'Administration

Apache Guacamole est un outil permettant de gérer des connexions distantes (RDP, SSH, VNC) via un navigateur web, sans nécessiter de client logiciel. Ce guide détaille l'installation et la configuration de Guacamole sur un serveur Debian.

---

## Table des Matières

1. [Introduction](#introduction)
2. [Installation des Prérequis](#installation-des-prérequis)
3. [Compilation et Installation de Guacamole Server](#compilation-et-installation-de-guacamole-server)
4. [Installation de Guacamole Client (Web App)](#installation-de-guacamole-client-web-app)
5. [Configuration de MariaDB pour l'Authentification](#configuration-de-mariadb-pour-lauthentification)
6. [Premiers Pas avec Apache Guacamole](#premiers-pas-avec-apache-guacamole)
7. [Créer un Nouveau Compte Administrateur](#créer-un-nouveau-compte-administrateur)
8. [Dépannage et Erreurs Courantes](#dépannage-et-erreurs-courantes)

---

## Introduction

Apache Guacamole est une solution open-source permettant de gérer des connexions RDP, SSH ou VNC à travers une interface web sécurisée. Il s'agit d'un outil idéal pour unifier les connexions administratives au sein d'une organisation.

---

## Installation des Prérequis

### Mettez à jour votre système et installez les paquets nécessaires :

```bash
sudo apt-get update
sudo apt-get install build-essential libcairo2-dev libjpeg62-turbo-dev libpng-dev libtool-bin uuid-dev libossp-uuid-dev libavcodec-dev libavformat-dev libavutil-dev libswscale-dev freerdp2-dev libpango1.0-dev libssh2-1-dev libtelnet-dev libvncserver-dev libwebsockets-dev libpulse-dev libssl-dev libvorbis-dev libwebp-dev
```
On attend gentiment la fin de l'installation.

La partie "cliente" d'Apache Guacamole nécessite d'installer un serveur Tomcat, mais nous allons effectuer cette opération plus tard.

Pour effectuer l'installation depuis un compte utilisateur, sans utiliser le compte "root" directement, pensez à installer "sudo" et à ajouter un utilisateur au groupe correspondant. L'exemple ci-dessous donne les permissions à l'utilisateur "flo" :

```bash
apt-get install sudo
usermod -aG sudo flo
```
Ensuite, préfixez par "sudo" les commandes qui nécessitent une élévation de privilèges.

```bash
sudo apt-get update
```
## Compiler et installer Apache Guacamole "Server"
On va se positionner dans le répertoire "/tmp" et télécharger l'archive tar.gz :

```bash
cd /tmp
wget https://downloads.apache.org/guacamole/1.5.5/source/guacamole-server-1.5.5.tar.gz
```
Une fois le téléchargement terminé, on décompresse l'archive tar.gz et on se positionne dans le répertoire obtenu :

```bash
tar -xzf guacamole-server-1.5.5.tar.gz
cd guacamole-server-1.5.5/
```
On exécute la commande ci-dessous pour se préparer à la compilation, ce qui va permettre de vérifier la présence des dépendances :

```bash
sudo ./configure --with-systemd-dir=/etc/systemd/system/
```
Regardez bien la sortie de la commande précédente, afin de vérifier la présence éventuelle d'une erreur. Si vous obtenez une erreur qui spécifie "guacenc_video_alloc", c'est lié au composant "guacenc" qui est utilisé pour créer les enregistrements au format vidéo (lié à FFmpeg). Dans ce cas, vous pouvez relancer la commande précédente en désactivant ce composant :
```bash
sudo ./configure --with-systemd-dir=/etc/systemd/system/ --disable-guacenc
```
Ensuite, poursuivez avec la compilation du code source de guacamole-server :
```bash
sudo make
```
```bash
sudo make install
```
La commande ci-dessous sert à mettre à jour les liens entre guacamole-server et les librairies (cette commande ne retourne aucun résultat) :

```bash
sudo ldconfig
```

Ensuite, on va démarrer le service "guacd" correspondant à Guacamole et activer son démarrage automatique. La première commande sert à prendre en compte le nouveau service: 

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now guacd
```

Enfin, on vérifie le statut d'Apache Guacamole Server :

```bash
sudo systemctl status guacd
```
## Créer le répertoire de configuration
Dernière étape avant de passer à la partie client d'Apache Guacamole, on crée l'arborescence pour la configuration d'Apache Guacamole. Cela va donner le répertoire "/etc/guacamole" avec les sous-répertoires "extensions" et "lib". Nous en aurons besoin par la suite pour mettre en place le stockage des données dans une base de données MariaDB / MySQL.

```bash
sudo mkdir -p /etc/guacamole/{extensions,lib}
```

## Installer Guacamole Client (Web App)

Pour la Web App correspondante à Apache Guacamole, et donc à la partie cliente, nous avons besoin d'un serveur Tomcat 9. J'insiste sur le fait que Tomcat 10, distribué par défaut via les dépôts de Debian 12, n'est pas pris en charge par Apache Guacamole. Nous devons ajouter le dépôt de Debian 11 sur notre machine Debian 12 afin de pouvoir télécharger les paquets correspondants à Tomcat 9.

Nous allons ajouter un nouveau fichier source pour Apt. Créez le fichier suivant :

```bash
sudo nano /etc/apt/sources.list.d/bullseye.list 
```
```bash
deb http://deb.debian.org/debian/ bullseye main
```
Mettez à jour le cache des paquets :

```bash
sudo apt-get update
```
Effectuez l'installation des paquets Tomcat 9 sur Debian 12 avec cette commande :

```bash
sudo apt-get install tomcat9 tomcat9-admin tomcat9-common tomcat9-user
```
Puis, nous allons télécharger la dernière version de la Web App d'Apache Guacamole depuis le dépôt officiel (même endroit que pour la partie serveur). On se positionne dans "/tmp" et on télécharge la Web App, ce qui revient à télécharger un fichier avec l'extension ".war". Ici, la version 1.5.5 est téléchargée.

```bash
cd /tmp
wget https://downloads.apache.org/guacamole/1.5.5/binary/guacamole-1.5.5.war
```
Une fois que le fichier est téléchargé, on le déplace dans la librairie de Web App de Tomcat9 avec cette commande :

```bash
sudo mv guacamole-1.5.5.war /var/lib/tomcat9/webapps/guacamole.war
```
Puis, on relance les services Tomcat9 et Guacamole :

```bash
sudo systemctl restart tomcat9 guacd
```
**Voilà, Apache Guacamole Client est installé !**
 
## Base de données MariaDB pour l'authentification

On commence par installer le paquet MariaDB Server :

```bash
sudo apt-get install mariadb-server
```
Puis, on exécute le script ci-dessous pour sécuriser un minimum notre instance (changer le mot de passe root, désactiver les accès anonymes, etc...)

```bash
sudo mysql_secure_installation
```

**Réponses aux questions de l'assistant :**

Enter current password for root (enter for none) ➡️ ne rien renseigner et faire entrer

Set root password ? ➡️ Entrer le mot de passe du futur admin de 
mariaDB (exemple : "Adm.2020") (**c'est le mot de passe root du serveur mysql !**)

Remove anonymous user ➡️ yes

Disallow root login remotely ? ➡️ Yes (on ne pourra se connecter en root qu'en local)

Remove test database and access to it ? ➡️ yes

Reload privilege tables now ? ➡️ yes

Une fois cette étape effectuée, on va se connecter en tant que root à notre instance MariaDB :

```bash
mysql -u root -p
```
Ceci est utile pour créer une base de données et un utilisateur dédié pour Apache Guacamole. Les commandes ci-dessous permettent de créer la base de données "guacadb", avec l'utilisateur "guaca_nachos" **associé au mot de passe** "P@ssword!" (adaptez ces valeurs). Cet utilisateur dispose de quelques droits sur la base de données.

```bash
CREATE DATABASE guacadb;
CREATE USER 'guaca_nachos'@'localhost' IDENTIFIED BY 'P@ssword!';
GRANT SELECT,INSERT,UPDATE,DELETE ON guacadb.* TO 'guaca_nachos'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
La suite va consister à ajouter l'extension MySQL à Apache Guacamole ainsi que le connecteur correspondant. Encore quelques fichiers à télécharger depuis Internet.

Toujours depuis le dépôt officiel, on télécharge cette extension :

```bash
cd /tmp
wget https://downloads.apache.org/guacamole/1.5.5/binary/guacamole-auth-jdbc-1.5.5.tar.gz
```
Puis, on décompresse l'archive tar.gz obtenue :

```bash
tar -xzf guacamole-auth-jdbc-1.5.5.tar.gz
```
On déplace le fichier ".jar" de l'extension dans le répertoire "/etc/guacamole/extensions/" créé précédemment :

```bash
sudo mv guacamole-auth-jdbc-1.5.5/mysql/guacamole-auth-jdbc-mysql-1.5.5.jar /etc/guacamole/extensions/
```
Télécharger le connecteur MySQL
On lance le téléchargement :

```bash
cd /tmp
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-9.1.0.tar.gz
```
Puis, on décompresse l'archive tar.gz :

```bash
tar -xzf mysql-connector-j-9.1.0.tar.gz
```
On copie (ou déplace) le fichier .jar du connecteur vers le répertoire "lib" d'Apache Guacamole :

```bash
sudo cp mysql-connector-j-9.1.0/mysql-connector-j-9.1.0.jar /etc/guacamole/lib/
```
Les dépendances sont déployées, mais nous n'avons pas encore fini cette intégration avec MariaDB.

En effet, il faut importer la structure de la base de données Apache Guacamole dans notre base de données "guacadb". Pour cela, on va importer tous les fichiers SQL situés dans le répertoire "guacamole-auth-jdbc-1.5.5/mysql/schema/". Le mot de passe root de MariaDB doit être saisit pour effectuer l'import.

```bash
cd guacamole-auth-jdbc-1.5.5/mysql/schema/
cat *.sql | mysql -u root -p guacadb
```
Une fois que c'est fait, on va créer et éditer le fichier "guacamole.properties" pour déclarer la connexion à MariaDB. Ce fichier peut être utilisé pour d'autres paramètres, selon vos besoins.

```bash
sudo nano /etc/guacamole/guacamole.properties
```
Dans ce fichier, insérez les lignes ci-dessous en adaptant les trois dernières lignes avec vos valeurs :

```bash
# MySQL
mysql-hostname: 127.0.0.1
mysql-port: 3306
mysql-database: guacadb
mysql-username: guaca_nachos
mysql-password: P@ssword!
```
Enregistrez et fermez le fichier.

Tant que l'on est dans la configuration, éditez le fichier "guacd.conf" pour déclarer le serveur Guacamole (ici, on déclare une connexion locale sur le port par défaut, à savoir 4822).

```bash
sudo nano /etc/guacamole/guacd.conf
```
Voici le code à intégrer :

```bash
[server] 
bind_host = 0.0.0.0
bind_port = 4822
```
On enregistre et on termine par redémarrer les trois services liés à Apache Guacamole :

```bash
sudo systemctl restart tomcat9 guacd mariadb
```
**Voilà, l'installation de base est terminée !**

## Premiers pas avec Apache Guacamole

On va pouvoir se connecter à Apache Guacamole pour effectuer nos premiers pas sur l'interface de la Web App.

```bash
http://<Adresse IP>:8080/guacamole/
```
**Pour se connecter, on va utiliser les identifiants par défaut :**

```bash
Utilisateur : guacadmin

Mot de passe : guacadmin
```
## Créer un nouveau compte admin
Tout d'abord, nous allons créer un nouveau compte d'administration et supprimer le compte par défaut, pour des raisons de sécurité.

Notre objectif est le suivant :

Créer un nouveau compte administrateur (avec un nom personnalisé)
Se déconnecter du compte **"guacadmin"**
Se reconnecter avec le nouveau compte administrateur
Supprimer le compte **"guacadmin"** par défaut (ou à minima changer son mot de passe et le désactiver)
Pour accéder aux paramètres, il faut cliquer sur le nom d'utilisateur en haut à droite puis sur **"Paramètres".**

Ensuite, sur l'onglet **"Utilisateurs"** et sur **"Nouvel utilisateur".**

Un formulaire s'affiche. Indiquez un nom d'utilisateur, en évitant les traditionnels **"Administrateur"**, **"Admin"**, etc.... Et choisissez un mot de passe robuste. Cochez l'ensemble des permissions pour que cet utilisateur soit administrateur de la plateforme Guacamole.

## Ajouter une connexion RDP

Nous allons créer notre première connexion dans Apache Guacamole, de manière à se connecter à un serveur en RDP ! Pour créer une connexion avec un autre protocole tel que SSH, le principe reste le même.

Pour créer une nouvelle connexion : **Paramètres > Connexion > Nouvelle connexion**

Mais avant cela, on va créer un nouveau groupe, car ces groupes vont permettre d'organiser les connexions : **Paramètres > Connexion > Nouveau groupe**

Dans cet exemple, je crée un groupe nommé **"Serveurs applicatifs"**. Il sera positionné sous le lieu **"ROOT"** qui est la racine de l'arborescence. Le type de groupe "Organizationel" doit être sélectionné pour tous les groupes qui ont pour vocation à organiser les connexions.

On enregistre et on clique sur le bouton "Nouvelle connexion". On commence par nommer la connexion, choisir le groupe et le protocole. Ici, c'est sur le serveur "SRV-APPS" que je souhaite me connecter, associé à l'adresse IP **"192.168.100.12".**

Ensuite, il y a un ensemble de paramètres à renseigner :

- **Nom d'hôte** : le nom DNS du serveur (si le serveur Apache Guacamole est capable de résoudre le nom), sinon l'adresse IP

- **Port** : le numéro de port du RDP, par défaut 3389 (pas utile de le préciser si c'est le port par défaut)

- **Identifiant** : compte avec lequel s'authentifier sur le serveur

- **Mot de passe** : mot de passe du compte spécifié ci-dessus

- **Nom de domaine** : nom du domaine Active Directory, si besoin

- **Mode de sécurité** : par défaut, c'est en détection automatique (vous pouvez également choisir le NLA)
- **Ignorer le certificat du serveur** : cochez cette option si vous 
n'avez pas déployé de certificat pour vos connexions RDP et si vous utilisez une adresse IP pour la connexion

- **Agencement clavier** : choisissez "Français (Azerty)", ou adaptez selon votre configuration

- **Fuseau horaire** : choisissez "Europe / Paris", ou adaptez selon votre configuration

## Apache Guacamole : erreur de connexion en RDP

Que faire si la connexion RDP ne se lance pas ou qu'elle affiche une erreur ?

Retournez sur la ligne de commande de votre serveur et vérifiez les dernières lignes de logs qui s'affichent lorsque l'on regarde le statut du service guacd : 

```bash
sudo systemctl status guacd
```
Par exemple, on peut trouver ceci :

```bash
juin 14 20:15:29 srv-guacamole guacd[31120]: Certificate validation failed
juin 14 20:15:29 srv-guacamole guacd[31120]: RDP server closed/refused connection: SSL/TLS connection failed (untrusted/self-signed certificate?)
```
Si le certificat RDP ne peut pas être vérifié (auto-signé par exemple) et que l'option "Ignorer le certificat du serveur" n'est pas cochée dans les paramètres de la connexion Guacamole, alors cette erreur se produira.

Une autre erreur que vous pourriez rencontrer si vous avez besoin d'établir des connexions en RDP, c'est celle-ci :

```bash
RDP server closed/refused connection: Security negotiation failed (wrong security type?)
```

Nous devons créer un nouvel utilisateur, lui associer les permissions nécessaires sur les données d'Apache Guacamole, puis mettre à jour le service et enfin le relancer.

Voici la série de commandes à exécuter, dans l'ordre :

```bash
sudo useradd -M -d /var/lib/guacd/ -r -s /sbin/nologin -c "Guacd User" guacd
sudo mkdir /var/lib/guacd
sudo chown -R guacd: /var/lib/guacd
sudo sed -i 's/daemon/guacd/' /etc/systemd/system/guacd.service
sudo systemctl daemon-reload
sudo systemctl restart guacd
```

Puis, vérifiez l'état du service :

```bash
sudo systemctl status guacd
```

## Sources
[it-connect.fr](https://www.it-connect.fr/tuto-apache-guacamole-bastion-rdp-ssh-debian/)
