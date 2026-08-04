🏗️ Architecture de l'infrastructure
┌──────────────────────────────────────────────────────────┐
│         PC Physique — Dell Vostro 15 3510                │
│         Intel i7-1165G7 | 32 GB RAM | Windows 11         │
│                                                          │
│  ┌───────────────────────────────────────────────────┐   │
│  │      VMware Workstation (Hyperviseur L1)           │   │
│  │      VMnet8 — NAT | Subnet : 192.168.159.0/24     │   │
│  │                                                    │   │
│  │  ┌─────────────────────────────────────────────┐  │   │
│  │  │   Nutanix CE (Hyperviseur L2 — Nested)       │  │   │
│  │  │   28 GB RAM | 4 vCPUs | 3 × 200 GB Disk     │  │   │
│  │  │                                               │  │   │
│  │  │   AHV   : 192.168.159.10                     │  │   │
│  │  │   CVM   : 192.168.159.11                     │  │   │
│  │  │   Prism : https://192.168.159.11:9440        │  │   │
│  │  │                                               │  │   │
│  │  │  ┌────────────────────────────────────────┐  │  │   │
│  │  │  │   VM : test-RHEL9-Golden               │  │  │   │
│  │  │  │   OS  : RHEL 9.6                       │  │  │   │
│  │  │  │   IP  : 192.168.159.134                │  │  │   │
│  │  │  │   SSH : aliuser@192.168.159.134        │  │  │   │
│  │  │  └────────────────────────────────────────┘  │  │   │
│  │  └─────────────────────────────────────────────┘  │   │
│  └───────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘




Golden Image RHEL 9.6
Prérequis
Nutanix CE opérationnel avec Prism Element accessible
ISO RHEL 9.6 officielle (compte Red Hat Developer)
Subscription Red Hat active (SCA — Simple Content Access)
Installation des packages
bash
# Update complet du système
dnf update -y

# Installation des packages requis
dnf install -y \
  cloud-init \
  python3 \
  chrony \
  openssh-server \
  git \
  vim \
  curl \
  wget
Configuration système
bash
# Timezone
timedatectl set-timezone Africa/Tunis

# Locale
localectl set-locale LANG=en_US.UTF-8

# NTP
systemctl enable --now chronyd

# Firewall
systemctl enable --now firewalld
firewall-cmd --permanent --add-service=ssh
firewall-cmd --permanent --add-service=ntp
firewall-cmd --reload

# SELinux — mode Enforcing
setenforce 1
sed -i 's/SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config

# Journald persistant
sed -i 's/#Storage=auto/Storage=persistent/' /etc/systemd/journald.conf
systemctl restart systemd-journald
Utilisateur d'administration
bash
# Créer l'utilisateur admin
useradd -m -G wheel aliuser
passwd aliuser

# Politique sudo (wheel sans password en lab)
# Éditer /etc/sudoers via visudo
# Décommenter : %wheel ALL=(ALL) NOPASSWD: ALL
Nettoyage avant template (Sysprep)
bash
# Logs
journalctl --rotate && journalctl --vacuum-time=1s
find /var/log -type f -exec truncate -s 0 {} \;

# Historique bash
history -c && history -w
echo "" > ~/.bash_history

# Clés SSH machine
rm -f /etc/ssh/ssh_host_*
systemctl stop sshd

# Cache DNF
dnf clean all

# Cloud-init reset
cloud-init clean --logs

# Identifiant machine
rm -f /etc/machine-id
touch /etc/machine-id
