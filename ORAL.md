# ORAL

## 🧱 VM1 – FIREWALL / RELAIS (SRV-FW)

Le but est de vérifier que le routage fonctionne et que les machines internes ont accès à Internet via cette VM.

SRV-FW : `192.168.10.1`
```
ip a ✅
```
```
ping 8.8.8.8 ✅
ping google.com ✅
```
Vérifier le routage : **net.ipv4.ip_forward = 1**
```
sysctl net.ipv4.ip_forward ✅
```
Vérifier le statut du **DHCP** :
```
systemctl status dnsmasq ✅
```
Tester le **DNS** Forwarder :
```
dig https://www.google.com/search?q=google.com @127.0.0.1 ✅
```
Sécurité (Filtrage) :
- Test de connexion **SSH** depuis le **LAN** :
```
ssh 192.168.10.69 ✅
```
Test de blocage **SSH** depuis l'extérieur : **refusé**
```
ssh 192.168.10.69 ✅
```
## 🧱 VM2 – CORE SERVICES (SRV-CORE)

**Rôle** : Hébergement des applications métier. C'est ici que tu dois intégrer la **Conteneurisation**.

DNS Interne (Bind9) :
```
curl -Ikv https://srv-core.entreprise.local Sur SRV-FW et SRV-BACKUP
dig @127.0.0.1 srv-core.entreprise.local +short ✅
Résultat attendu : 192.168.10.69 ✅
```
🌐 Service Web & Sécurité (Nginx) :

**Rôle** : Reverse Proxy et terminaison **SSL**.

**HTTPS** : Implémentation du protocole **TLS** via un certificat auto-signé (OpenSSL).

**Reverse Proxy** : Nginx redirige les flux entrants (443) vers le conteneur Nextcloud (8080).

**Sécurité** : Redirection automatique du port 80 (HTTP) vers le port 443 (HTTPS).

Test de validation :
```
Accès à http://192.168.10.69 -> Redirection auto vers HTTPS. ✅

Accès à https://192.168.10.69 -> Chargement de l'interface Nextcloud sécurisée. ✅
```

Accès **SSH** :
```
systemctl status ssh ✅
```
🐳 Conteneurisation :

```
sudo docker ps ✅
# Résultat attendu : 2 conteneurs actifs (nextcloud-stack-app et nextcloud-stack-db)
```

🚀 Automatisation :

```
ls -l ~/deploy_stack.sh ✅
# Test de reconstruction :
~/deploy_stack.sh ✅
Résultat attendu : "Infrastructure déployée et sécurisée !"
```
📈 Surveillance :

**Interface Grafana** : Aller sur ```http://192.168.10.69:3000``` (Login: admin / admin123).

**Dashboard** : Montrer le Dashboard ID 1860.

**Test en direct** : Lancer ```timeout 30s cat /dev/zero > /dev/null``` sur la VM2 et montrer le pic de CPU sur Grafana. ✅


## 🧱 VM3 – BACKUP SERVER (SRV-BACKUP) :

**Rôle** : Sauvegarde centralisée.

Vérifier la connectivité sans mot de passe : Pour que le cron fonctionne, VM3 doit pouvoir se connecter à VM1 et VM2 via des clés Depuis SRV-BACKUP, cette commande ne doit PAS demander de mot de passe :
```
Depuis SRV-BACKUP
ssh mael@192.168.10.69 "hostname"
Résultat attendu : SRV-CORE ✅
```
📜 Script de Sauvegarde (backup_core.sh)

**Emplacement** : `/usr/local/bin/backup_core.sh`

**Extraction** à distance (Pull Mode) via rsync.

**Synchronisation** des dossiers : `home/mael, /etc/nginx, /etc/bind.`

**Journalisation** complète dans `/var/log/backup_core.log.`

**Test de validation** :
```
sudo /usr/local/bin/backup_core.sh
sudo ls -l /backup/core/home/mael/test_backup.txt
Résultat : Présent ✅
```
**Restauration** : Test de validation

**Action** : Suppression manuelle de `test_backup.txt` sur **SRV-CORE**.

**Restauration** : Exécution de la commande suivante depuis **SRV-BACKUP** :

```
sudo rsync -avz --rsync-path="sudo rsync" /backup/core/home/mael/test_backup.txt mael@192.168.10.69:/home/mael/
Résultat : Le fichier a été réintégré immédiatement sur le serveur source avec des permissions identiques. ✅
```
