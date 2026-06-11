#  Ubuntu

Facile à utiliser et populaire

Beaucoup de documentation et de tutoriels disponibles.

Gestion simple des paquets

Installer SSH, firewall, Docker, Nginx… c’est rapide avec apt.

Réseau et IP faciles à configurer

Netplan permet de mettre des IP statiques facilement.

Sécurisé

Mises à jour automatiques, firewall intégré (UFW).

Compatible avec les conteneurs

Docker et Docker Compose fonctionnent parfaitement pour déployer vos services.


# 🧱 VM1 – FIREWALL / RELAIS (SRV-FW)

➡️ C’est la porte d’entrée du réseau.

## Rôles :

- Firewall nftables +

- NAT + Routage entre NAT et réseau interne +

- DHCP +

- Serveur DNS forwarder (DNSMasq optionnel) +

- Filtrage du trafic (ports autorisés) +

- Accès SSH uniquement depuis LAN +

## Interfaces réseau :

- enp0s3 (NAT VirtualBox) → Internet 

- enp0s8 (Interne LAN) → 192.168.10.1/24


# 🧱 VM2 – CORE SERVICES (SRV-CORE)

➡️ Héberge les services de l’entreprise.

## Rôles :

- DNS interne (Bind9) +

- Serveur web HTTPS (Nginx) + 

- Serveur complémentaire : Nextcloud +

- Accès SSH +

- Graphana et promete +

- IP : 192.168.10.69

# 🧱 VM3 – BACKUP SERVER (SRV-BACKUP)

➡️ Gère toutes les sauvegardes + restauration.

## Rôles :

- Serveur Rsync +

- Stockage des snapshots -

- Script de sauvegarde automatisé (cron) +

- Système de restauration fonctionnel +

- IP : 192.168.10.73

/////////////////////////////////////////////////////////////////////////

# Sétup du projet

## 🧱 Étape 1 -  Préparer la VM-FIREWALL (SRV-FW)

### Config réseau

```
sudo nano /etc/netplan/01-netcfg.yaml
```

```
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true          # NAT = Internet
    enp0s8:
      addresses:
        - 192.168.10.1/24  # Passerelle du réseau interne
```

```
sudo netplan apply
```

### Activer le routage IP

```
sudo nano /etc/sysctl.conf
```
### Décommente 
```
net.ipv4.ip_forward=1
```
```
sudo sysctl -p
```
### Configurer le NAT avec nftables

Installer nftables :

```
sudo apt install nftables -y
```

Crée la config NAT :

```
sudo nano /etc/nftables.conf
```

```
table ip nat {
    chain prerouting {
        type nat hook prerouting priority -100;
    }

    chain postrouting {
        type nat hook postrouting priority 100;
        oifname "enp0s3" masquerade
    }
}
```

Applique :

```
sudo systemctl enable nftables
sudo systemctl restart nftables
```

## Activer le routage IP sur SERV-FW

```
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

```
sudo sysctl -p
```

## Configurer iptables pour NAT

```
sudo iptables -t nat -A POSTROUTING -o enp0s3 -s 192.168.10.0/24 -j MASQUERADE
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Sauvegarder les règles iptables :

```
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

## configuration dnsmasq sur VM SERV-FW pour gérer à la fois le DHCP interne et le DNS forwarding

Installer dnsmasq

```
sudo apt install dnsmasq -y
```

Sauvegarder la configuration existante

```
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.bak
```

Créer une nouvelle configuration

```
Créer une nouvelle configuration
```

```
# Interface LAN
interface=enp0s8
listen-address=192.168.10.1

# Plage DHCP pour les clients internes
dhcp-range=192.168.10.50,192.168.10.100,12h

# Passerelle et DNS par défaut pour les clients
dhcp-option=3,192.168.10.1
dhcp-option=6,192.168.10.1

# DNS forwarding
server=8.8.8.8
server=1.1.1.1
```

Redémarrer dnsmasq et enable

```
sudo systemctl restart dnsmasq
sudo systemctl enable dnsmasq
```

Vérifier que dnsmasq écoute correctement sur le port 53

```
sudo lsof -i :53

mael@SRV-FW:~/Desktop$ sudo lsof -i :53
COMMAND  PID    USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
dnsmasq 7006 dnsmasq    6u  IPv4  38659      0t0  UDP *:domain 
dnsmasq 7006 dnsmasq    7u  IPv4  38660      0t0  TCP *:domain (LISTEN)
dnsmasq 7006 dnsmasq    8u  IPv6  38661      0t0  UDP *:domain 
dnsmasq 7006 dnsmasq    9u  IPv6  38662      0t0  TCP *:domain (LISTEN)

```

Configurer une VM cliente pour le LAN interne

```
sudo nano /etc/netplan/01-netcfg.yaml
```

```
network:
  version: 2
  ethernets:
    enp0s3:   # carte du réseau interne
      dhcp4: true
```
```
sudo netplan apply
```

LETS GO MAIS PUTAIN DE VM ON ENFIN UNE PUTAIN DE PLAGE D'IP DE MERDE !!! 
PS : penser a changer l'address MAC des VM ;-;

## configuration nftables pour SRV-FW afin de gérer le firewall + NAT

```
sudo nano /etc/nftables.conf
```

```
#!/usr/sbin/nft -f

flush ruleset

# Table principale
table inet filter {
    # Chaîne INPUT : trafic entrant
    chain input {
        type filter hook input priority 0;
        policy drop;

        # Loopback
        iif lo accept

        # ICMP (ping)
        ip protocol icmp accept

        # SSH depuis le LAN
        iif "enp0s8" tcp dport 22 accept

        # HTTP / HTTPS depuis le LAN
        iif "enp0s8" tcp dport { 80, 443 } accept

        # DNS depuis le LAN
        iif "enp0s8" tcp dport 53 accept
        iif "enp0s8" udp dport 53 accept

        # DHCP (UDP 67/68)
        udp dport 67 accept
        udp dport 68 accept

        # Trafic déjà établi ou lié
        ct state established,related accept
    }

    # Chaîne FORWARD : trafic transitant par le firewall
    chain forward {
        type filter hook forward priority 0;
        policy drop;

        # LAN -> WAN
        iif "enp0s8" oif "enp0s3" ct state new,established,related accept
        # WAN -> LAN
        iif "enp0s3" oif "enp0s8" ct state established,related accept
        # LAN -> LAN (optionnel)
        iif "enp0s8" oif "enp0s8" ct state new,established,related accept
    }

    # Chaîne OUTPUT : trafic sortant
    chain output {
        type filter hook output priority 0;
        policy accept;
    }
}

# Table NAT pour masquerading
table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100;
        oif "enp0s3" masquerade
    }
}

```

Charger la configuration et verif

```
sudo nft -f /etc/nftables.conf
sudo nft list ruleset
```

## 2 ➡️ SRV-CORE 

Utile
```
sudo apt install -y curl wget nano git
```

Installer Bind9 (DNS interne) :
```
sudo apt install bind9 bind9utils bind9-doc -y
```

Installer NGINX (serveur web HTTPS) :
```
sudo apt install nginx -y
```

### PETIT BUG DE MON COTE :
le firewall de SERV-CORE me bloqué http et https bruh
```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
sudo ufw status
```

### config nginx : 
```
sudo nano /etc/nginx/sites-available/default
```

```
server {
    listen 443 ssl;
    server_name _;   # accepte toutes les IP

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    root /var/www/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}

server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}
```
reload :
```
sudo nginx -t
sudo systemctl restart nginx
```
et puis petit test :
```
curl -k https://<IP_DHCP_du_SRV-CORE>
```

### config Bind9 pour eviter de devoir ce rap des IP 

Activer :
```
sudo systemctl enable named
```
Démarrer le service :
```
sudo systemctl start named
```
Vérifier qu’il tourne correctement :
```
sudo systemctl status named
```

### Configurer un domaine interne

```
sudo nano /etc/bind/named.conf.local
```

```
zone "entreprise.local" {
    type master;
    file "/etc/bind/db.entreprise.local";
};
```

Créer la zone DNS :

```
sudo cp /etc/bind/db.local /etc/bind/db.entreprise.local
sudo nano /etc/bind/db.entreprise.local
```

Et adapter le contenu avec les machines du réseau (ADAPTER LES IP), par exemple : 

```
@       IN      SOA     ns.entreprise.local. admin.entreprise.local. (
                          2025120701 ; Serial
                          604800     ; Refresh
                          86400      ; Retry
                          2419200    ; Expire
                          604800 )   ; Negative Cache TTL
;
@       IN      NS      ns
ns      IN      A       192.168.10.69
srv-core IN      A       192.168.10.69
srv-backup IN    A       192.168.10.73
srv-fw IN        A       192.168.10.1
```
Redémarrer Bind9 :
```
sudo systemctl restart named
```

### Pour tester autoriser les resuete dig sur SERV-FW

```
iif "enp0s8" tcp dport 53 accept
iif "enp0s8" udp dport 53 accept
```

BON CA FONCTIONNE PAS ON PASS POUR L'INSTANT

## Nextcloud

Installer les dépendances :

```
sudo apt update
sudo apt install -y nginx mariadb-server php-fpm php-mysql php-gd php-json php-curl php-mbstring php-intl php-bcmath php-imagick php-xml php-zip php-gmp unzip curl
```
Installer MariaDB (base pour Nextcloud) : 

```
sudo apt install mariadb-server -y
sudo systemctl enable --now mariadb
```

Configurer la base :
```
sudo mysql -u root
```

Dans MariaDB :

```
CREATE DATABASE nextcloud;
CREATE USER 'ncuser'@'localhost' IDENTIFIED BY 'motdepassefort';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'ncuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Télécharger Nextcloud : 
```
cd /var/www/
sudo wget https://download.nextcloud.com/server/releases/latest.zip
sudo apt install unzip -y
sudo unzip latest.zip
sudo chown -R www-data:www-data nextcloud
```

Créer la config Nginx :
```
sudo nano /etc/nginx/sites-available/nextcloud.conf
```

```
# Redirection HTTP → HTTPS
server {
    listen 80;
    server_name 192.168.10.69;
    return 301 https://$host$request_uri;
}

# Serveur Nextcloud HTTPS
server {
    listen 443 ssl;
    server_name 192.168.10.69;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    root /var/www/nextcloud/;
    index index.php;

    client_max_body_size 512M;

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

Activer :
```
sudo ln -s /etc/nginx/sites-available/nextcloud.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

Installe PHP 8.3 FPM + modules pour Nextcloud :
```
sudo apt install php8.3-fpm php8.3-mysql php8.3-zip php8.3-xml php8.3-gd php8.3-curl php8.3-mbstring php8.3-intl php8.3-bz2 php8.3-imagick php8.3-gmp php8.3-cli
```

Active + démarre PHP-FPM :
```
sudo systemctl enable --now php8.3-fpm
sudo systemctl status php8.3-fpm
```
Redémarre nginx :
```
sudo systemctl restart nginx
```


Accéder à Nextcloud depuis ton réseau interne :
```
https://192.168.10.69/nextcloud
```

# sauvegarde !!!!

Sur SRV-BACKUP :

```
sudo nano /usr/local/bin/backup_core.sh
```
```
#!/bin/bash

# ==============================================================================
# SCRIPT DE SAUVEGARDE AUTOMATISÉE - SRV-BACKUP
# Cible : SRV-CORE (192.168.10.69)
# ==============================================================================

# 1. Variables de configuration
SRC_USER="mael"
SRC_HOST="192.168.10.69"
# On ajoute les configs DNS et Nginx pour un backup complet
SRC_DIRS="/home /var/www /etc/bind /etc/nginx"
DEST_DIR="/backup/core"
DATE_SUFFIX=$(date +%F)
LOGFILE="/var/log/backup_core_${DATE_SUFFIX}.log"
RETENTION_DAYS=7
EXCLUDE="--exclude='*.log' --exclude='*cache*' --exclude='node_modules'"

# 2. Création du dossier de destination et du log
mkdir -p "$DEST_DIR"
echo "--- Début de la sauvegarde le $(date) ---" > "$LOGFILE"

# 3. Fonction pour envoyer un mail en cas d'erreur (Nécessite mailutils)
send_mail() {
    echo -e "Erreur lors du backup de SRV-CORE le $(date)" | mail -s "🚨 ALERT: Backup FAILED" mael@entreprise.local
}

# 4. Exécution du backup avec rsync
# -a : mode archive (conserve droits/dates)
# -v : verbeux
# -z : compression pendant le transfert
# -R : CONSERVE L'ARBORESCENCE RELATIVE (ex: /backup/core/etc/bind/...)
# --delete : supprime sur la destination les fichiers disparus à la source
echo "Transfert en cours..." >> "$LOGFILE"

for DIR in $SRC_DIRS; do
    echo "Sauvegarde du répertoire : $DIR" >> "$LOGFILE"
    rsync -avzR --delete $EXCLUDE -e "ssh" "$SRC_USER@$SRC_HOST:$DIR" "$DEST_DIR" >> "$LOGFILE" 2>&1
    
    if [ $? -ne 0 ]; then
        echo "❌ ERREUR détectée sur $DIR" >> "$LOGFILE"
        # send_mail  # Décommente cette ligne si mailutils est configuré
    else
        echo "✅ Succès pour $DIR" >> "$LOGFILE"
    fi
done

# 5. Nettoyage de la rétention (Supprime les fichiers vieux de plus de 7 jours)
echo "Nettoyage des anciens fichiers (Rétention: $RETENTION_DAYS jours)..." >> "$LOGFILE"
find "$DEST_DIR" -type f -mtime +$RETENTION_DAYS -delete
# Supprime les dossiers vides
find "$DEST_DIR" -type d -empty -delete

echo "--- Sauvegarde terminée le $(date) ---" >> "$LOGFILE"

```

Rendre le script exécutable :
```
sudo chmod +x /usr/local/bin/backup_core.sh
```

Ajouter une tâche cron pour exécuter le backup à 3h du matin :
```
sudo crontab -e
```

Puis ajouter :
```
0 3 * * * /usr/local/bin/backup_core.sh
```

mail : 

```
sudo apt install mailutils
```
## TEST

```
sudo /usr/local/bin/backup_core.sh
```

### sur serv-core

```
echo "TEST BACKUP" | sudo tee /home/mael/test_backup.txt
```

```
mael@SRV-CORE:~/Desktop$ sudo nano /etc/nginx/sites-available/nextcloud.conf
[sudo] password for mael: 
mael@SRV-CORE:~/Desktop$ sudo systemctl restart nginx
mael@SRV-CORE:~/Desktop$ echo "TEST BACKUP" | sudo tee /home/mael/test_backup.txt
[sudo] password for mael: 
TEST BACKUP
mael@SRV-CORE:~/Desktop$ sudo /usr/local/bin/backup_core.sh
mael@192.168.10.69's password: 
mael@192.168.10.69's password: 
mael@SRV-CORE:~/Desktop$ ls /srv/backups/core/2025-12-10/home/mael/
Desktop    Downloads  Pictures  snap       test_backup.txt
Documents  Music      Public    Templates  Videos
mael@SRV-CORE:~/Desktop$ sudo rm /home/mael/test_backup.txt
mael@SRV-CORE:~/Desktop$ sudo rsync -av /srv/backups/core/2025-12-10/home/mael/test_backup.txt mael@192.168.10.69:/home/mael/
mael@192.168.10.69's password: 
sending incremental file list
test_backup.txt

sent 124 bytes  received 35 bytes  63.60 bytes/sec
total size is 12  speedup is 0.08
```
