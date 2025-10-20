# Born2beroot - Configuration serveur Debian

Projet 42 Lausanne : Configuration et sécurisation d'un serveur Debian sous VirtualBox.

## Description

Born2beroot est un projet d'administration système visant à configurer un environnement serveur sécurisé depuis zéro. Le projet ne peut pas être uploadé sur GitHub car il s'agit d'une machine virtuelle (.vdi), mais cette documentation retrace les étapes et compétences acquises.

## Choix de la distribution

**Debian** choisi plutôt que Rocky Linux pour plusieurs raisons :
- Familiarité avec l'environnement Debian
- Configuration LVM plus accessible pour un premier projet sysadmin
- AppArmor plus simple que SELinux pour débuter

### AppArmor vs SELinux
- **AppArmor** (Debian/Ubuntu) : Utilise des profils liés aux répertoires des programmes
- **SELinux** (RHEL) : Plus strict, basé sur des security policies. Si aucune policy n'existe pour un programme, celui-ci n'aura aucune permission

## Partitionnement LVM détaillé

### Structure créée
```
sda (30GB recommandé pour bonus)
├─sda1 (boot, 524MB, non chiffrée)
├─sda2 (extended)
└─sda5 (partition principale chiffrée LUKS)
  └─sda5_crypt
    └─LVMGroup (Volume Group)
      ├─root (10GB)
      ├─swap (2.5GB)
      ├─home (5GB)
      ├─var (3GB)
      ├─srv (3GB)
      ├─tmp (3GB)
      └─var-log (4GB)
```

### Étapes de création
1. Créer partition /boot (524MB, recommandation Debian)
2. Créer partition principale (reste de l'espace)
3. Chiffrer la partition avec LUKS (mot de passe de déchiffrement au boot)
4. Créer le Volume Group LVM dans la partition chiffrée
5. Nommer le groupe "LVMGroup"
6. Créer les volumes logiques selon la structure demandée
7. Monter les volumes et assigner les mount points (filesystem ext4)

### Extension de volumes logiques

Principe de l'extension d'un volume logique avec LVM :

```bash
# 1. Ajouter un nouveau Physical Volume
sudo pvcreate /dev/sdaX

# 2. Étendre le Volume Group
sudo vgextend VGname /dev/sdaX

# 3. Étendre le Logical Volume
sudo lvextend -L +XG /dev/VGname/LVname
# ou utiliser tout l'espace libre :
sudo lvextend -l +100%FREE /dev/VGname/LVname

# 4. Redimensionner le filesystem
sudo resize2fs /dev/VGname/LVname

# 5. Vérifier
df -h
```

## Configuration système

### Sécurité SSH
- Port personnalisé (4242)
- Désactivation du login root (`PermitRootLogin no`)
- Configuration dans `/etc/ssh/sshd_config`

```bash
# Vérifier SSH
sudo systemctl status ssh
sudo ss -tulpn | grep :4242
```

### Pare-feu UFW
```bash
sudo ufw enable
sudo ufw allow 4242
sudo ufw status numbered
sudo ufw delete [number]  # Pour supprimer une règle
```

### Politique de mots de passe

**Fichier `/etc/login.defs`** (expiration) :
```
PASS_MAX_DAYS 30
PASS_MIN_DAYS 2
PASS_WARN_AGE 7
```

**Fichier `/etc/pam.d/common-password`** (complexité) :
```
password requisite pam_pwquality.so retry=3 minlen=10 ucredit=-1 dcredit=-1 maxrepeat=3 reject_username difok=7 enforce_for_root
```

Appliquer aux utilisateurs existants :
```bash
sudo chage -M 30 -m 2 -W 7 username
sudo chage -l username  # Vérifier
```

### Configuration sudo

**Fichier `/etc/sudoers.d/sudo_config`** :
```
Defaults passwd_tries=3
Defaults badpass_message="Mot de passe incorrect. Réessayez."
Defaults logfile="/var/log/sudo/sudo.log"
Defaults log_input, log_output
Defaults iolog_dir="/var/log/sudo"
Defaults requiretty
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
```

Vérifier les logs :
```bash
sudo cat /var/log/sudo/sudo.log
```

### Script de monitoring

**Script `/usr/local/bin/monitoring.sh`** diffusant via `wall` toutes les 10 minutes :
- Architecture OS et kernel (`uname -a`)
- CPU physiques/virtuels (`nproc`, `/proc/cpuinfo`)
- RAM utilisée (`free`)
- Disque utilisé (`df`)
- % utilisation CPU (`top`)
- Dernier boot (`who -b`)
- LVM actif (`lsblk | grep lvm`)
- Connexions TCP établies (`ss -t`)
- Utilisateurs connectés (`users`)
- IP et MAC (`hostname -I`, `ip link`)
- Nombre de commandes sudo (`journalctl _COMM=sudo`)

**Configuration cron** :
```bash
sudo crontab -e
# Ajouter :
*/10 * * * * /usr/local/bin/monitoring.sh
```

Tester le script :
```bash
sudo /usr/local/bin/monitoring.sh
```

Arrêter temporairement :
```bash
sudo systemctl stop cron
```

## Partie Bonus

### WordPress avec Lighttpd, MariaDB, PHP

**Installation des services** :
```bash
sudo apt install lighttpd mariadb-server php-cgi php-mysql
sudo lighttpd-enable-mod fastcgi
sudo lighttpd-enable-mod fastcgi-php
sudo systemctl restart lighttpd
```

**Configuration MariaDB** :
```bash
sudo mysql_secure_installation
sudo mysql

CREATE DATABASE wpdb;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL ON wpdb.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**Installation WordPress** :
```bash
cd /var/www/html
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzvf latest.tar.gz
sudo chown -R www-data:www-data wordpress/
sudo chmod -R 755 wordpress/
```

**Configuration `wp-config.php`** :
```php
define('DB_NAME', 'wpdb');
define('DB_USER', 'wpuser');
define('DB_PASSWORD', 'password');
define('DB_HOST', 'localhost');
```

Accès : `http://localhost:8080/wordpress` (avec port forwarding VirtualBox)

### Service bonus : Serveur FTP (vsftpd)

**Installation** :
```bash
sudo apt install vsftpd
sudo systemctl enable vsftpd
sudo systemctl start vsftpd
```

**Configuration `/etc/vsftpd.conf`** :
```bash
# Désactiver connexion anonyme
anonymous_enable=NO

# Autoriser utilisateurs locaux
local_enable=YES

# Permettre l'écriture
write_enable=YES

# Emprisonner utilisateurs dans leur home
chroot_local_user=YES

# Messages de bienvenue
ftpd_banner=Bienvenue sur le serveur FTP Born2beroot

# Ports passifs (pour pare-feu)
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100

# Logging
xferlog_enable=YES
xferlog_file=/var/log/vsftpd.log
```

**Configuration UFW** :
```bash
sudo ufw allow 21/tcp
sudo ufw allow 40000:40100/tcp
sudo ufw reload
```

**Créer un utilisateur FTP** :
```bash
sudo adduser ftpuser
sudo usermod -d /var/ftp ftpuser
sudo mkdir -p /var/ftp
sudo chown ftpuser:ftpuser /var/ftp
```

**Redémarrer le service** :
```bash
sudo systemctl restart vsftpd
sudo systemctl status vsftpd
```

**Tester la connexion** :
```bash
ftp localhost 21
# Username: ftpuser
# Password: [mot de passe]
```

**Justification du choix FTP** :
- Service réseau traditionnel essentiel en administration système
- Permet le transfert de fichiers entre machines
- Configuration simple mais complète (chroot, logs, sécurité)
- Complémentaire au stack web (WordPress)
- Utile pour déploiement de fichiers sur le serveur

## Commandes clés pour la défense

### Système et virtualisation
```bash
uname -a                    # Kernel et architecture
hostnamectl                 # Hostname et OS
hostnamectl set-hostname X  # Changer hostname
```

### LVM et partitions
```bash
lsblk                       # Arborescence des disques
sudo pvdisplay              # Physical Volumes
sudo vgdisplay              # Volume Groups
sudo lvdisplay              # Logical Volumes
df -h                       # Espace disque utilisé
```

### Utilisateurs et groupes
```bash
sudo adduser username       # Créer utilisateur
sudo userdel -r username    # Supprimer utilisateur
sudo usermod -aG group user # Ajouter au groupe
groups username             # Voir groupes d'un user
getent group groupname      # Membres d'un groupe
id username                 # Info complète user
sudo chage -l username      # Politique mot de passe
```

### Sudo
```bash
sudo visudo                         # Éditer sudoers
sudo cat /var/log/sudo/sudo.log    # Voir logs sudo
```

### SSH
```bash
sudo systemctl status ssh
sudo systemctl restart ssh
sudo cat /etc/ssh/sshd_config
ssh username@localhost -p 4242
```

### UFW
```bash
sudo ufw status numbered
sudo ufw allow port
sudo ufw delete number
```

### AppArmor
```bash
sudo aa-status              # Statut AppArmor
```

### Services
```bash
sudo systemctl status service_name
sudo systemctl start|stop|restart service_name
sudo systemctl enable|disable service_name
```

### Cron
```bash
sudo crontab -l             # Lister cron jobs
sudo crontab -e             # Éditer cron jobs
sudo systemctl stop cron    # Arrêter cron temporairement
```

### FTP
```bash
sudo systemctl status vsftpd
sudo cat /etc/vsftpd.conf
ftp localhost 21
```

## Troubleshooting

### MariaDB ne démarre pas après extension de /var

**Symptômes** : `mariadb.service failed`, erreurs de permissions

**Solution** :
```bash
sudo systemctl stop mariadb
sudo chown -R mysql:mysql /var/lib/mysql
sudo chmod 750 /var/lib/mysql
sudo systemctl start mariadb
```

Si échec, réinitialiser :
```bash
sudo rm -rf /var/lib/mysql/*
sudo mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
sudo chown -R mysql:mysql /var/lib/mysql
sudo systemctl start mariadb
```

### WordPress "Error establishing database connection"

Vérifier `/var/www/html/wordpress/wp-config.php` :
- DB_NAME, DB_USER, DB_PASSWORD correspondent à MariaDB
- Permissions : `sudo chown -R www-data:www-data wordpress/`

### vsftpd : 500 OOPS: vsftpd: refusing to run with writable root

Solution :
```bash
sudo chmod a-w /home/ftpuser
```

## Export de la VM

Pour transférer la VM entre machines :

**Méthode 1 : Export OVA (recommandé)**
- VirtualBox → Fichier → Exporter un appareil virtuel
- Format : OVA (Open Virtualization Format)
- Import : Fichier → Importer un appareil virtuel

**Méthode 2 : Copier le fichier .vdi**
- Localiser le fichier .vdi du disque virtuel
- Copier sur la nouvelle machine
- Créer une VM et attacher le .vdi existant

## Génération de la signature

```bash
# Sur Linux/Mac
sha1sum Born2beroot.vdi > signature.txt

# Sur Windows
certUtil -hashfile Born2beroot.vdi sha1 > signature.txt
```

La signature permet de vérifier l'intégrité du fichier .vdi lors de l'évaluation.

## Compétences acquises

- Administration système Linux (Debian)
- Partitionnement LVM avec chiffrement LUKS
- Extension de volumes logiques LVM
- Sécurisation serveur SSH
- Configuration pare-feu UFW
- AppArmor (MAC - Mandatory Access Control)
- Politiques de sécurité PAM
- Configuration sudo avancée
- Scripting bash système
- Gestion utilisateurs/groupes
- Cron et automatisation
- Installation et configuration LAMP (Lighttpd, MariaDB, PHP)
- Déploiement WordPress
- Configuration serveur FTP (vsftpd)
- Troubleshooting services Linux

## Ressources

- [Documentation Debian](https://www.debian.org/doc/)
- [LVM HOWTO](https://tldp.org/HOWTO/LVM-HOWTO/)
- [Arch Wiki - SSH](https://wiki.archlinux.org/title/OpenSSH)
- [UFW Documentation](https://help.ubuntu.com/community/UFW)
- [vsftpd Documentation](https://security.appspot.com/vsftpd.html)

## Projet 42

Réalisé dans le cadre du cursus de 42 Lausanne.

---

*Note : Ce README documente le processus et les compétences acquises. La VM elle-même n'est pas versionnée sur GitHub en raison de sa nature binaire (.vdi).*