# Infrastructure virtualisée Proxmox avec services Docker interconnectés

**TPI Blanc — Antoine BERGER**
EPFL – ENAC-IT | Mars 2026

---

## Table des matières

- [1. Présentation du projet](#1-présentation-du-projet)
- [2. Architecture réseau](#2-architecture-réseau)
- [3. Prérequis](#3-prérequis)
- [4. Procédure d'installation](#4-procédure-dinstallation)
  - [4.1 Installation de Proxmox VE](#41-installation-de-proxmox-ve)
  - [4.2 Configuration réseau (vmbr1 + NAT)](#42-configuration-réseau-vmbr1--nat)
  - [4.3 Création des conteneurs LXC](#43-création-des-conteneurs-lxc)
  - [4.4 Installation de Docker](#44-installation-de-docker)
  - [4.5 Déploiement de PostgreSQL (CT3)](#45-déploiement-de-postgresql-ct3)
  - [4.6 Déploiement de Kanboard (CT2)](#46-déploiement-de-kanboard-ct2)
  - [4.7 Déploiement de Nginx (CT1)](#47-déploiement-de-nginx-ct1)
  - [4.8 Port forwarding](#48-port-forwarding)
  - [4.9 Restriction d'accès à la base de données](#49-restriction-daccès-à-la-base-de-données)
- [5. Fichiers de configuration](#5-fichiers-de-configuration)
- [6. Tests de validation](#6-tests-de-validation)
- [7. Snapshots et restauration](#7-snapshots-et-restauration)

---

## 1. Présentation du projet

Ce projet met en place une infrastructure virtualisée sur Proxmox VE hébergeant 3 conteneurs LXC, chacun faisant tourner un service Docker :

| Conteneur | Service | Rôle |
|-----------|---------|------|
| CT1 (10.10.10.10) | Nginx | Reverse proxy — point d'entrée HTTP |
| CT2 (10.10.10.20) | Kanboard | Application web de gestion de tâches |
| CT3 (10.10.10.30) | PostgreSQL 16 | Base de données |

**Flux applicatif :** Client → Proxmox (:80) → Nginx (CT1) → Kanboard (CT2) → PostgreSQL (CT3)

---

## 2. Architecture réseau

```
Réseau EPFL (128.178.x.x)
        │
   ┌────┴────┐
   │ Proxmox │  IP EPFL : 10.95.48.44
   │  Host   │  Port forward : :80 → CT1:80
   └────┬────┘
        │ vmbr0 (réseau EPFL)
        │
        │ vmbr1 (réseau interne 10.10.10.0/24)
        │  Proxmox = 10.10.10.1 (gateway + NAT)
        │
   ┌────┼────────────┐
   │    │             │
  CT1  CT2          CT3
  .10  .20          .30
 Nginx Kanboard  PostgreSQL
```

- **vmbr0** : bridge connecté au réseau EPFL (existant à l'installation)
- **vmbr1** : bridge interne créé manuellement, pas de port physique
- **NAT** : le host traduit les IPs privées (10.10.10.x) vers son IP EPFL pour l'accès internet
- **Isolation DB** : CT3 n'est accessible que depuis CT2 (iptables)

*Schéma réseau plus détaillé en annexe
---

## 3. Prérequis

- VM xaas EPFL (ou équivalent) avec minimum 16 Go RAM
- Proxmox VE installé en bare-metal depuis l'ISO
- Accès à l'interface web Proxmox (port 8006)
- Accès internet depuis le host (pour télécharger templates et paquets)

---

## 4. Procédure d'installation

### 4.1 Installation de Proxmox VE

L'installation a été réalisée en bare-metal directement depuis l'ISO Proxmox VE 8.x sur la VM xaas de l'EPFL. L'ISO a été ajoutée au catalogue xaas avec l'aide d'un collègue disposant des droits nécessaires.

> **Note :** KVM n'est pas disponible sur cette VM (nested virtualization désactivée). On utilise donc des conteneurs LXC à la place des VMs.

### 4.2 Configuration réseau (vmbr1 + NAT)

Ouvrir le fichier de configuration réseau :

```bash
nano /etc/network/interfaces
```

Ajouter à la fin du fichier (ne pas toucher à la config vmbr0 existante) :

```
# --- Bridge interne pour la communication inter-conteneurs ---
auto vmbr1                                  # Démarrage automatique au boot        
iface vmbr1 inet static
        address 10.10.10.1/24               # IP du host sur le réseau interne (sert de gateway)
        bridge-ports none                   # Pas de port physique — réseau purement virtuel
        bridge-stp off                      # Spanning Tree Protocol désactivé (inutile ici, un seul bridge)
        bridge-fd 0                         # Pas de délai de forwarding
        post-up   echo 1 > /proc/sys/net/ipv4/ip_forward             # Active le routage IP : autorise Proxmox à faire transiter des paquets entre réseaux
        post-up   iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o vmbr0 -j MASQUERADE                  # Active le NAT : remplace l'IP privée (10.10.10.x) par l'IP EPFL (10.95.48.44) en sortie
        post-down iptables -t nat -D POSTROUTING -s 10.10.10.0/24 -o vmbr0 -j MASQUERADE                  # Nettoie la règle NAT quand le bridge s'éteint
        post-up   iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 80 -j DNAT --to-destination 10.10.10.10:80                # Redirige le trafic HTTP entrant (IP EPFL) vers CT1 (Nginx)
        post-down iptables -t nat -D PREROUTING -i vmbr0 -p tcp --dport 80 -j DNAT --to-destination 10.10.10.10:80                # Nettoie la règle de redirection DNAT quand le bridge s'éteint
        post-up   iptables -A FORWARD -s 10.10.10.20 -d 10.10.10.30 -p tcp --dport 5432 -j ACCEPT                         # Isolation DB : autorise CT2 → CT3:5432
        post-down iptables -D FORWARD -s 10.10.10.20 -d 10.10.10.30 -p tcp --dport 5432 -j ACCEPT                         # Nettoie la règle quand le bridge s'éteint
        post-up   iptables -A FORWARD -d 10.10.10.30 -p tcp --dport 5432 -j DROP                                # Isolation DB : refuse Tout le traffic entrant sur CT3:5432 (sauf CT2 car la règle qui autorise est écrite avant)
        post-down iptables -D FORWARD -d 10.10.10.30 -p tcp --dport 5432 -j DROP                                # Nettoie la règle quand le bridge s'éteint
```
Appliquer :

```bash
systemctl restart networking
```

Vérifier :

```bash
ip a show vmbr1
# Doit afficher : inet 10.10.10.1/24
```

### 4.3 Création des conteneurs LXC

Télécharger le template Debian 12 :

```bash
pveam update
pveam available --section system | grep debian-12
pveam download local debian-12-standard_12.12-1_amd64.tar.zst
```

Créer les 3 conteneurs :

```bash
# CT1 — Nginx (reverse proxy)
pct create 101 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname ct1-nginx \
  --memory 512 --cores 1 \
  --net0 name=eth0,bridge=vmbr1,ip=10.10.10.10/24,gw=10.10.10.1 \
  --storage local-lvm --rootfs local-lvm:8 \
  --password --unprivileged 1 --features nesting=1

# CT2 — Kanboard (application web)
pct create 102 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname ct2-kanboard \
  --memory 1024 --cores 2 \
  --net0 name=eth0,bridge=vmbr1,ip=10.10.10.20/24,gw=10.10.10.1 \
  --storage local-lvm --rootfs local-lvm:10 \
  --password --unprivileged 1 --features nesting=1

# CT3 — PostgreSQL (base de données)
pct create 103 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname ct3-postgres \
  --memory 512 --cores 1 \
  --net0 name=eth0,bridge=vmbr1,ip=10.10.10.30/24,gw=10.10.10.1 \
  --storage local-lvm --rootfs local-lvm:10 \
  --password --unprivileged 1 --features nesting=1
```

Démarrer et vérifier la connectivité :

```bash
pct start 101 && pct start 102 && pct start 103

ping -c 2 10.10.10.10
ping -c 2 10.10.10.20
ping -c 2 10.10.10.30
```

### 4.4 Installation de Docker

Répéter cette procédure sur chaque conteneur (101, 102, 103) :

```bash
pct enter <ID>

apt update && apt install -y ca-certificates curl gnupg

install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
> /etc/apt/sources.list.d/docker.list

apt update && apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

docker --version
# Docker version 29.3.0

exit
```

### 4.5 Déploiement de PostgreSQL (CT3)

```bash
pct enter 103
mkdir -p /opt/postgres && cd /opt/postgres
```

Créer le fichier `.env` contenant les credentials (non versionné, séparé du compose) :

```bash
nano .env
```

```
# --- Credentials PostgreSQL (ne pas versionner ce fichier) ---
POSTGRES_DB=dbexemple
POSTGRES_USER=userexemple
POSTGRES_PASSWORD=mdpexemple
```

Créer `docker-compose.yml` (les variables sont lues automatiquement depuis `.env`) :

```yaml
services:
  postgres:
    image: postgres:16                      # Image officielle PostgreSQL version 16
    container_name: kanboard-db             # Nom explicite du conteneur Docker
    restart: unless-stopped                 # Redémarre automatiquement sauf arrêt manuel
    environment:
      POSTGRES_DB: ${POSTGRES_DB}           # Nom de la base créée automatiquement au 1er lancement
      POSTGRES_USER: ${POSTGRES_USER}       # Utilisateur avec accès à la base
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD} # Mot de passe lu depuis .env (pas en clair dans le compose)
    volumes:
      - pgdata:/var/lib/postgresql/data     # Volume persistant — les données survivent aux recréations
    ports:
      - "5432:5432"                         # Expose le port PostgreSQL sur l'IP du CT (10.10.10.30)

volumes:
  pgdata:                                   # Déclaration du volume nommé géré par Docker
```

> **Sécurité :** Les credentials ne sont pas en clair dans le `docker-compose.yml`. Ils sont externalisés dans le fichier `.env`, qui ne doit jamais être partagé ou versionné.

```bash
docker compose up -d
docker compose ps
# kanboard-db doit être Up
```

### 4.6 Déploiement de Kanboard (CT2)

```bash
pct enter 102
mkdir -p /opt/kanboard && cd /opt/kanboard
```

Créer le fichier `.env` contenant les credentials de connexion à la DB :

```bash
nano .env
```

```
# --- URL de connexion à PostgreSQL (ne pas versionner ce fichier) ---
# Format : protocole://user:password@host:port/database
DATABASE_URL=postgres://userexemple:mdpexemple@10.10.10.30:5432/dbexemple
```

Créer `docker-compose.yml` :

```yaml
services:
  kanboard:
    image: kanboard/kanboard:latest         # Image officielle Kanboard (dernière version stable)
    container_name: kanboard-app            # Nom explicite du conteneur Docker
    restart: unless-stopped                 # Redémarre automatiquement sauf arrêt manuel
    environment:
      DATABASE_URL: ${DATABASE_URL}         # URL de connexion lue depuis .env (pas en clair dans le compose)
    volumes:
      - kanboard_data:/var/www/app/data       # Données applicatives persistantes (config, fichiers)
      - kanboard_plugins:/var/www/app/plugins  # Plugins installés, persistent entre mises à jour
    ports:
      - "8080:80"                           # Kanboard écoute sur 80 en interne, exposé sur 8080 du CT

volumes:
  kanboard_data:                            # Volume nommé pour les données applicatives
  kanboard_plugins:                         # Volume nommé pour les plugins
```

> **Sécurité :** Comme pour CT3, les credentials sont externalisés dans `.env` pour ne pas apparaître en clair dans le fichier compose.

```bash
docker compose up -d
docker compose ps
# kanboard-app doit être Up (healthy)
```

### 4.7 Déploiement de Nginx (CT1)

```bash
pct enter 101
mkdir -p /opt/nginx && cd /opt/nginx
```

Créer `nginx.conf` :

```nginx
# --- Configuration Nginx Reverse Proxy ---
# Redirige tout le trafic HTTP entrant vers l'application Kanboard sur CT2

server {
    listen 80;                              # Écoute sur le port 80 (HTTP)
    server_name _;                          # Accepte toutes les requêtes quel que soit le nom de domaine

    location / {
        # Redirige vers Kanboard sur CT2 (10.10.10.20), port 8080
        proxy_pass http://10.10.10.20:8080;

        # --- Headers de forwarding ---
        # Transmet le nom d'hôte original demandé par le client
        proxy_set_header Host $host;
        # Transmet l'IP réelle du client (pas celle de Nginx)
        proxy_set_header X-Real-IP $remote_addr;
        # Liste des proxys traversés (chaîne de confiance)
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # Indique le protocole original utilisé par le client (http ou https)
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Créer `docker-compose.yml` :

```yaml
services:
  nginx:
    image: nginx:stable                     # Image officielle Nginx (version stable)
    container_name: nginx-proxy             # Nom explicite du conteneur Docker
    restart: unless-stopped                 # Redémarre automatiquement sauf arrêt manuel
    volumes:
      # Monte notre fichier de config dans le conteneur en lecture seule (:ro)
      # Remplace la config par défaut de Nginx
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    ports:
      - "80:80"                             # Port 80 exposé sur CT1 (10.10.10.10) — point d'entrée de la stack
```

```bash
docker compose up -d
docker compose ps
# nginx-proxy doit être Up
```

### 4.8 Port forwarding

Pour rendre Kanboard accessible depuis le réseau EPFL, une règle DNAT redirige le trafic HTTP entrant sur l'IP publique du host (10.95.48.44:80) vers CT1 (10.10.10.10:80) :

```bash
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 80 -j DNAT --to-destination 10.10.10.10:80
```

- `-i vmbr0` : uniquement le trafic arrivant depuis le réseau EPFL
- `--dport 80` : port HTTP
- `--to-destination 10.10.10.10:80` : redirige vers Nginx sur CT1

**Vérification :** depuis un navigateur sur le réseau EPFL, accéder à `http://10.95.48.44` — la page de login Kanboard doit s'afficher.

**Persistance :** la règle est ajoutée dans `/etc/network/interfaces` dans le bloc `vmbr1` via une ligne `post-up` (cf. section 4.2).

### 4.9 Restriction d'accès à la base de données

#### Prérequis : br_netfilter

Les conteneurs LXC partagent le même bridge (vmbr1). Par défaut, le trafic entre eux est commuté au niveau L2 (Ethernet) et **ne passe pas par les chaînes iptables FORWARD** du host. Pour que les règles de filtrage s'appliquent au trafic bridgé, il faut activer le module `br_netfilter` :
```bash
# Charger le module
modprobe br_netfilter

# Activer le filtrage iptables sur le trafic bridgé
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
```

Rendre persistant au reboot :
```bash
echo "br_netfilter" > /etc/modules-load.d/br_netfilter.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
```

> **Sans cette étape, les règles FORWARD ci-dessous n'ont aucun effet** — le trafic entre CT1 et CT3 transite directement via le bridge sans passer par le kernel en tant que trafic routé.

Deux règles iptables sur le host Proxmox restreignent l'accès à CT3 (PostgreSQL, port 5432) au seul CT2 (Kanboard) :

```bash
# Autoriser CT2 → CT3:5432
iptables -A FORWARD -s 10.10.10.20 -d 10.10.10.30 -p tcp --dport 5432 -j ACCEPT

# Bloquer tout le reste vers CT3:5432
iptables -A FORWARD -d 10.10.10.30 -p tcp --dport 5432 -j DROP
```

> **Important :** l'ordre des règles est crucial — la règle ACCEPT doit être insérée **avant** la règle DROP, sinon CT2 sera également bloqué.

**Vérification :**

```bash
# Depuis CT2 (doit réussir)
pct enter 102
apt install -y postgresql-client
psql -h 10.10.10.30 -U userexemple -d dbexemple -c "SELECT 1;"
# → résultat OK

# Depuis CT1 (doit échouer / timeout)
pct enter 101
apt install -y postgresql-client
psql -h 10.10.10.30 -U userexemple -d dbexemple -c "SELECT 1;"
# → connexion refusée / timeout
```

Pour afficher les règles en place :

```bash
iptables -L FORWARD -n -v
iptables -t nat -L -n -v
```

**Persistance :** les règles sont ajoutées dans `/etc/network/interfaces` dans le bloc `vmbr1` via des lignes `post-up` (cf. section 4.2).

---

## 5. Fichiers de configuration

| Fichier | Emplacement | Description |
|---------|-------------|-------------|
| `/etc/network/interfaces` | Host Proxmox | Config réseau (vmbr0 + vmbr1, NAT) |
| `/etc/fstab` | Host Proxmox | Montage persistant du disque de sauvegarde (`sdb1` → `/mnt/backup`) | (cf. section 7.2)
| `docker-compose.yml` | CT3 : `/opt/postgres/` | Déploiement PostgreSQL 16 |
| `.env` | CT3 : `/opt/postgres/` | Credentials PostgreSQL (non versionné) |
| `docker-compose.yml` | CT2 : `/opt/kanboard/` | Déploiement Kanboard |
| `.env` | CT2 : `/opt/kanboard/` | URL de connexion à la DB (non versionné) |
| `docker-compose.yml` | CT1 : `/opt/nginx/` | Déploiement Nginx reverse proxy |
| `nginx.conf` | CT1 : `/opt/nginx/` | Configuration reverse proxy |

---

## 6. Tests de validation

Checklist de validation post-installation. Si l'infrastructure est correctement configurée, chaque test doit produire le résultat indiqué.

| # | Test | Commande | Résultat attendu |
|---|------|----------|------------------|
| 1 | Ping host → CT1 | `ping -c 2 10.10.10.10` | 0% packet loss |
| 2 | Ping host → CT2 | `ping -c 2 10.10.10.20` | 0% packet loss |
| 3 | Ping host → CT3 | `ping -c 2 10.10.10.30` | 0% packet loss |
| 4 | Ping CT1 → CT2 | `ping -c 2 10.10.10.20` (depuis CT1) | 0% packet loss |
| 5 | Accès internet CT | `ping -c 2 8.8.8.8` (depuis chaque CT) | 0% packet loss |
| 6 | Docker sur CT3 | `docker compose ps` (dans /opt/postgres) | kanboard-db Up |
| 7 | Docker sur CT2 | `docker compose ps` (dans /opt/kanboard) | kanboard-app Up (healthy) |
| 8 | Docker sur CT1 | `docker compose ps` (dans /opt/nginx) | nginx-proxy Up |
| 9 | Bout en bout (CT1) | `curl -I http://localhost` | HTTP 302 → /login |
| 10 | Bout en bout (host) | `curl -I http://10.10.10.10` | HTTP 302 → /login |
| 11 | Accès externe (EPFL) | `http://10.95.48.44` dans navigateur | Page login Kanboard |
| 12 | Isolation DB (CT2 → CT3) | `psql -h 10.10.10.30` depuis CT2 | Connexion réussie |
| 13 | Isolation DB (CT1 → CT3) | `psql -h 10.10.10.30` depuis CT1 | Connexion refusée / timeout |

---

## 7. Snapshots et restauration

### Snapshots LXC

Les snapshots Proxmox permettent de capturer l'état complet d'un conteneur (filesystem, config) à un instant donné. Ils utilisent le thin provisioning LVM et ne stockent que les blocs modifiés après le snapshot (copy-on-write).

**Création d'un snapshot :**

```bash
pct snapshot 101 snap-avant-modif --description "État stable avant modifications"
pct snapshot 102 snap-avant-modif --description "État stable avant modifications"
pct snapshot 103 snap-avant-modif --description "État stable avant modifications"
```

**Lister les snapshots d'un conteneur :**

```bash
pct listsnapshot <ID>
```

**Restauration (rollback) :**

```bash
# Arrêter le conteneur avant le rollback
pct stop <ID>

# Restaurer le snapshot
pct rollback <ID> snap-avant-modif

# Redémarrer
pct start <ID>
```

> **Note :** le rollback supprime toutes les modifications effectuées après le snapshot. Les données ajoutées entre-temps seront perdues.

### Backups sur disque externe

La VM hôte dispose de deux disques :

| Disque | Taille | Usage |
|--------|--------|-------|
| `sda` | 40 Go | OS Proxmox + thin pool LVM (conteneurs CT1/CT2/CT3) |
| `sdb` | 150 Go | Vierge à l'installation |

Le disque `sda` héberge l'intégralité de l'infrastructure (OS, swap, thin pool). Le thin pool LVM de ~10 Go contient les 3 conteneurs (28 Go alloués, ~50% utilisés réellement grâce au thin provisioning). L'espace libre sur `sda` est limité (~5 Go).

Le disque `sdb` a été configuré comme **stockage dédié aux sauvegardes**, ce qui permet de séparer les données de production des backups — bonne pratique en cas de défaillance du disque principal.

**Partitionnement et formatage (sur le host) :**

```bash
# Créer une partition primaire sur tout le disque
fdisk /dev/sdb
# → n (new), p (primary), 1, Entrée, Entrée, w (write)

# Formater en ext4
mkfs.ext4 /dev/sdb1

# Créer le point de montage et monter
mkdir -p /mnt/backup
mount /dev/sdb1 /mnt/backup

# Vérifier
df -h /mnt/backup
# Doit afficher ~148G disponible

# Rendre le montage persistant au reboot
echo '/dev/sdb1 /mnt/backup ext4 defaults 0 2' >> /etc/fstab
```

**Ajout dans Proxmox (interface web) :**

1. Aller dans **Datacenter** → **Stockage** → **Ajouter** → **Répertoire**
2. Remplir les champs :

| Champ | Valeur |
|-------|--------|
| **ID** | `backup-disk` |
| **Répertoire** | `/mnt/backup` |
| **Contenu** | Fichier de sauvegarde VZDump |
| **Noeuds** | Tout (par défaut) |
| **Activer** | coché |
| **Partagé** | décoché |
| **Dérnières à consérver** | `3` (conserve les 3 dernières par CT) |

**Lancer un backup des 3 conteneurs :**

```bash
vzdump 101 102 103 --storage backup-disk --mode stop --compress zstd
```

- `--mode stop` : arrête brièvement le conteneur pour garantir la cohérence des données
- `--compress zstd` : compression rapide et efficace

Les fichiers de backup sont stockés dans `/mnt/backup/dump/` et peuvent être utilisés pour restaurer un conteneur complet via l'interface Proxmox ou via `pct restore`.

---
