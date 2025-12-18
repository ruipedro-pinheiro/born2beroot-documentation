# Born2beroot - System Administration & Virtualization

**Score:** 125/100 (with bonus)

## About

Born2beroot is the first system administration project at 42. The goal is to set up a secure server environment from scratch using a Virtual Machine — strict partitioning with LVM, firewall configuration, strong password policies, and system monitoring.

*Note: Since the project result is a binary VM file (.vdi), this repository contains documentation and scripts, not the VM itself.*

## OS Choice: Rocky vs Debian

I initially leaned towards **Rocky Linux** — I wanted to explore the Red Hat ecosystem (RHEL), widely used in enterprise. I was already comfortable with Debian/Ubuntu from daily use, so Rocky seemed like a better learning opportunity.

However, I hit a roadblock: creating a partition without a mount point proved difficult with Rocky's installer. Combined with difficulty matching the specific bonus requirements, I switched back to **Debian**.

### AppArmor vs SELinux
- **AppArmor** (Debian/Ubuntu): Uses profiles linked to program directories
- **SELinux** (RHEL): Stricter, based on security policies. If no policy exists for a program, it has no permissions

## LVM Partitioning Structure

```
sda (30GB recommended for bonus)
├─sda1 (boot, 524MB, unencrypted)
├─sda2 (extended)
└─sda5 (main encrypted partition - LUKS)
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

**Mistake learned:** I initially undersized the partitions (<1GB). This came back to bite me during the bonus when installing WordPress, forcing me to extend logical volumes on a live system.

### LVM Extension

```bash
# Add a new Physical Volume
sudo pvcreate /dev/sdaX

# Extend the Volume Group
sudo vgextend VGname /dev/sdaX

# Extend the Logical Volume
sudo lvextend -L +XG /dev/VGname/LVname
# or use all free space:
sudo lvextend -l +100%FREE /dev/VGname/LVname

# Resize the filesystem
sudo resize2fs /dev/VGname/LVname

# Verify
df -h
```

## Security Configuration

### SSH
- **Port:** Changed to `4242` (avoid default port scanning)
- **Root Login:** Disabled (`PermitRootLogin no`)
- **Config:** `/etc/ssh/sshd_config`

```bash
sudo systemctl status ssh
sudo ss -tulpn | grep :4242
```

### UFW Firewall

```bash
sudo ufw enable
sudo ufw allow 4242
sudo ufw status numbered
sudo ufw delete [number]
```

### Password Policy

**`/etc/login.defs`** (expiration):
```
PASS_MAX_DAYS 30
PASS_MIN_DAYS 2
PASS_WARN_AGE 7
```

**`/etc/pam.d/common-password`** (complexity):
```
password requisite pam_pwquality.so retry=3 minlen=10 ucredit=-1 dcredit=-1 maxrepeat=3 reject_username difok=7 enforce_for_root
```

Apply to existing users:
```bash
sudo chage -M 30 -m 2 -W 7 username
sudo chage -l username  # Verify
```

### Sudo Configuration

**`/etc/sudoers.d/sudo_config`**:
```
Defaults passwd_tries=3
Defaults badpass_message="Wrong password. Try again."
Defaults logfile="/var/log/sudo/sudo.log"
Defaults log_input, log_output
Defaults iolog_dir="/var/log/sudo"
Defaults requiretty
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
```

## Monitoring Script

Script at `/usr/local/bin/monitoring.sh` broadcasts via `wall` every 10 minutes:
- OS Architecture and kernel (`uname -a`)
- Physical/virtual CPUs (`nproc`, `/proc/cpuinfo`)
- RAM usage (`free`)
- Disk usage (`df`)
- CPU load (`top`)
- Last boot (`who -b`)
- LVM active (`lsblk | grep lvm`)
- TCP connections (`ss -t`)
- Connected users (`users`)
- IP and MAC (`hostname -I`, `ip link`)
- Sudo commands count (`journalctl _COMM=sudo`)

**Cron configuration:**
```bash
sudo crontab -e
# Add:
*/10 * * * * /usr/local/bin/monitoring.sh
```

## Bonus: WordPress & Lighttpd

### Installation

```bash
sudo apt install lighttpd mariadb-server php-cgi php-mysql
sudo lighttpd-enable-mod fastcgi
sudo lighttpd-enable-mod fastcgi-php
sudo systemctl restart lighttpd
```

### MariaDB Setup

```bash
sudo mysql_secure_installation
sudo mysql

CREATE DATABASE wpdb;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL ON wpdb.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### WordPress Setup

```bash
cd /var/www/html
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzvf latest.tar.gz
sudo chown -R www-data:www-data wordpress/
sudo chmod -R 755 wordpress/
```

## Bonus: FTP Server (vsftpd)

**Why FTP?** Classic network service for file transfers, complements the web stack, simple but complete configuration.

```bash
sudo apt install vsftpd
sudo systemctl enable vsftpd
```

**`/etc/vsftpd.conf`**:
```bash
anonymous_enable=NO
local_enable=YES
write_enable=YES
chroot_local_user=YES
ftpd_banner=Welcome to Born2beroot FTP
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100
xferlog_enable=YES
```

```bash
sudo ufw allow 21/tcp
sudo ufw allow 40000:40100/tcp
```

## Quick Reference

### LVM
```bash
lsblk                       # Disk tree
sudo pvdisplay              # Physical Volumes
sudo vgdisplay              # Volume Groups
sudo lvdisplay              # Logical Volumes
```

### Users
```bash
sudo adduser username
sudo userdel -r username
sudo usermod -aG group user
groups username
```

### Services
```bash
sudo systemctl status/start/stop/restart service_name
sudo systemctl enable/disable service_name
```

## Skills Acquired

- Linux system administration (Debian)
- LVM partitioning with LUKS encryption
- SSH security configuration
- UFW firewall
- AppArmor (MAC)
- PAM security policies
- Advanced sudo configuration
- Bash scripting
- User/group management
- Cron automation
- LAMP stack (Lighttpd, MariaDB, PHP)
- WordPress deployment
- FTP server configuration

---

*Project developed at 42 Lausanne*
