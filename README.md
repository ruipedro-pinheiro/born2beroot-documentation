# Born2beroot - Configuration serveur Debian

Projet 42 Lausanne : Configuration et sécurisation d'un serveur Debian sous VirtualBox.

## Description

Born2beroot est un projet d'administration système visant à configurer un environnement serveur sécurisé depuis zéro. Le projet ne peut pas être uploadé sur GitHub car il s'agit d'une machine virtuelle (.vdi), mais cette documentation retrace les étapes et compétences acquises.

## Objectifs du projet

- Installer et configurer Debian sur une machine virtuelle
- Mettre en place un partitionnement LVM chiffré
- Configurer SSH avec sécurité renforcée
- Implémenter un pare-feu (UFW)
- Établir une politique de mots de passe stricte
- Configurer sudo avec restrictions
- Créer un script de monitoring système

## Configuration réalisée

### Partitionnement LVM
- Partition boot séparée
- Volumes logiques chiffrés (root, swap, home, var, tmp)
- Gestion LVM pour flexibilité

### Sécurité SSH
- Port personnalisé (non-standard)
- Authentification par clé publique
- Désactivation du login root
- Configuration stricte des permissions

### Pare-feu UFW
- Règles entrantes/sortantes configurées
- Ports autorisés : SSH uniquement
- Politique par défaut : deny all

### Politique de mots de passe
- Longueur minimale : 10 caractères
- Expiration : 30 jours
- Avertissement avant expiration
- Complexité requise (majuscules, minuscules, chiffres)

### Configuration sudo
- Log de toutes les commandes
- Limitation des tentatives
- Messages personnalisés
- TTY obligatoire
- Chemins sécurisés

### Script de monitoring
Script bash affichant toutes les 10 minutes :
- Architecture OS et version kernel
- CPU physiques/virtuels
- RAM utilisée/disponible
- Disque utilisé/disponible
- % utilisation CPU
- Dernier boot
- LVM actif
- Connexions TCP actives
- Utilisateurs connectés
- Adresse IP et MAC
- Nombre de commandes sudo exécutées

## Commandes clés utilisées
```bash
# LVM
lsblk
vgdisplay
lvdisplay

# Utilisateurs et groupes
usermod -aG sudo username
getent group sudo

# SSH
systemctl status ssh
nano /etc/ssh/sshd_config

# UFW
ufw status
ufw allow 4242

# Monitoring
wall < monitoring.sh
crontab -e
```

## Compétences acquises

- Administration système Linux
- Gestion LVM et partitionnement
- Sécurisation serveur
- Configuration SSH
- Pare-feu et réseau
- Scripting bash
- Gestion utilisateurs et permissions
- Politiques de sécurité

## Projet 42

Réalisé dans le cadre du cursus de 42 Lausanne.
Score : [ton score]/100 (si validé)
