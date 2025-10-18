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
1. **Partition /boot** : 524MB (non chiffrée, nécessaire pour le boot)
2. **Partition principale** : Reste de l'espace (chiffrée avec LUKS)
   - Nom du volume chiffré : `sda5_crypt`
   - Volume Group : `LVMGroup`

### Volumes logiques dans la partition chiffrée
- root
- swap
- home
- var
- srv
- tmp
- var-log

### Étapes de création
1. Créer partition /boot (524MB, recommandation Debian)
2. Créer partition principale (reste de l'espace)
3. Chiffrer la partition avec LUKS (mot de passe de déchiffrement au boot)
4. Créer le Volume Group LVM dans la partition chiffrée
5. Nommer le groupe "LVMGroup"
6. Créer les volumes logiques selon la structure demandée
7. Monter les volumes et assigner les mount points (filesystem ext4)

## Configuration système

### Sécurité SSH
- Port personnalisé (4242)
- Désactivation du login root (`PermitRootLogin no`)
- Configuration dans `/etc/ssh/sshd_config`

### Pare-feu UFW
```bash
ufw enable
ufw allow 4242
ufw status
```

### Politique de mots de passe
Fichiers modifiés :
- `/etc/login.defs` : Expiration, durée de validité
- `/etc/pam.d/common-password` : Complexité (minuscules, majuscules, chiffres, longueur min 10)

### Configuration sudo
Fichier `/etc/sudoers.d/sudo_config` :
- Log de toutes les commandes sudo dans `/var/log/sudo/`
- Limitation à 3 tentatives (`Defaults passwd_tries=3`)
- Messages personnalisés en cas d'erreur
- TTY obligatoire (`Defaults requiretty`)
- Restriction des chemins sécurisés (`Defaults secure_path`)

### Script de monitoring
Script bash (`/usr/local/bin/monitoring.sh`) diffusant via `wall` toutes les 10 minutes :
- Architecture OS et kernel
- CPU physiques/virtuels
- RAM/Disque utilisés
- % utilisation CPU
- Dernier boot
- LVM actif (oui/non)
- Connexions TCP
- Utilisateurs connectés
- IP et MAC
- Nombre de commandes sudo

Configuration cron : `*/10 * * * * /usr/local/bin/monitoring.sh`

## Commandes clés
```bash
# Vérification LVM
lsblk
vgdisplay
lvdisplay

# Gestion utilisateurs
useradd -m username
usermod -aG sudo username
getent group sudo

# SSH
systemctl status ssh
systemctl restart ssh

# AppArmor
aa-status

# Hostname
hostnamectl set-hostname new-hostname

# Cron
crontab -e
```

## Compétences acquises

- Administration système Linux (Debian)
- Partitionnement LVM avec chiffrement LUKS
- Sécurisation serveur SSH
- Configuration pare-feu UFW
- AppArmor (MAC - Mandatory Access Control)
- Politiques de sécurité PAM
- Scripting bash système
- Gestion utilisateurs/groupes
- Cron et automatisation

## Projet 42

Réalisé dans le cadre du cursus de 42 Lausanne.
Score : A valider
