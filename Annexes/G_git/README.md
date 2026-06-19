# Mini SOC — M1Tech Solutions

Hackathon M1 Infrastructures Réseaux & Cybersécurité — Déploiement et exploitation d'un mini SOC (Security Operations Center) pour une PME fictive de 80 collaborateurs.

**Auteur :** Adéyèmi MOUNIROU
**Formation :** Master 1 Cloud, Sécurité et Infrastructure — Ynov Campus Paris

---

## Présentation du projet

Ce projet met en place une infrastructure de supervision et de détection d'incidents de sécurité pour M1Tech Solutions, une PME fictive confrontée à une absence d'outils centralisés de monitoring et de corrélation de journaux.

L'infrastructure repose sur **4 conteneurs Docker** orchestrés via Docker Compose :

| Service | Image | Rôle |
|---|---|---|
| `m1tech-web` | `nginx:latest` | Serveur web (site institutionnel) |
| `m1tech-db` | `mariadb:11` | Base de données (non exposée, réseau interne uniquement) |
| `m1tech-uptimekuma` | `louislam/uptime-kuma:1` | Supervision de disponibilité |
| `m1tech-wazuh-manager` | `wazuh/wazuh-manager:4.9.0` | Détection et corrélation d'événements de sécurité |

Le déploiement comprend également un **agent Wazuh installé directement sur la VM hôte**, qui surveille les journaux système (`auth.log`) et transmet les événements au manager.

### Fonctionnalités démontrées

- Sécurisation de l'accès SSH (port non standard, désactivation du login root) et pare-feu UFW
- Segmentation réseau Docker (`frontend_net` exposé / `backend_net` interne)
- Supervision de disponibilité avec détection de panne et remontée d'alerte (Uptime Kuma)
- Détection de 3 événements de sécurité réels : bruteforce SSH, scan Nmap, création d'utilisateur suspect
- Analyse d'incident complète sur l'événement de bruteforce SSH (chronologie, impacts, actions correctives)

### Incidents rencontrés et résolus

Deux incidents techniques significatifs ont été diagnostiqués et résolus pendant le déploiement (détaillés dans le rapport principal, Parties 3.3 et 6.1) :

1. **Saturation du disque (LVM)** sur la première VM, ayant nécessité la recréation d'une seconde machine virtuelle correctement dimensionnée.
2. **Agent Wazuh ne surveillant pas `/var/log/auth.log`** par défaut, empêchant la détection des tentatives SSH jusqu'à correction manuelle de sa configuration.

---

## Structure du dépôt

```
Adeyemi_MOUNIROU_Hackaton_M1/
├── Rapport/                          # Rapport principal (PDF/DOCX)
├── Annexes/
│   ├── A_schemas/                    # Schémas réseau, logique, cartographie des flux
│   ├── B_docker/                     # docker-compose.yml, .env.example, configuration Nginx
│   ├── C_scripts/                    # Scripts et configuration Wazuh corrigée
│   ├── D_commandes/                  # commandes.txt — journal complet des commandes
│   ├── E_journaux/                   # Extraits de logs (auth.log, access.log Nginx)
│   ├── F_captures/                   # Captures d'écran organisées par thème
│   └── G_git/                        # .gitignore
└── README.md
```

---

## Procédure de reproduction du projet

### Prérequis

- Une machine virtuelle Ubuntu Server 22.04 LTS (ou supérieur), avec **au moins 45-50 Go de disque** et 4 Go de RAM minimum
  ⚠️ *Un disque sous-dimensionné a causé un incident bloquant lors de ce projet (voir rapport, Partie 3.3) — prévoir une marge confortable, en particulier si le provisionnement utilise LVM.*
- Deux interfaces réseau configurées (une en NAT pour l'accès Internet, une en réseau privé hôte/host-only pour simuler le réseau interne de l'entreprise)
- Accès SSH à la VM
- Docker Engine et le plugin Docker Compose installés (voir étape 2)

### 1. Cloner le dépôt

```bash
git clone git@github.com:Ade-yemi12/Adeyemi_MOUNIROU_Hackaton_M1.git
cd Adeyemi_MOUNIROU_Hackaton_M1
```

### 2. Installer Docker (si non présent)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
# Se déconnecter / reconnecter pour appliquer le changement de groupe
```

### 3. Augmenter la limite mmap système (requis par Wazuh)

```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### 4. Configurer les variables d'environnement

```bash
cd Annexes/B_docker
cp .env.example .env
nano .env
# Renseigner des mots de passe réels pour MARIADB_ROOT_PASSWORD et MARIADB_USER_PASSWORD
```

### 5. Lancer la stack

```bash
docker compose up -d
docker ps
# Vérifier que les 4 conteneurs sont "Up"
```

### 6. Configurer Uptime Kuma

Ouvrir `http://<IP_VM>:3001`, créer le compte administrateur, puis ajouter au minimum :
- Une sonde HTTP(s) sur `http://<IP_VM>:8080` (Nginx)
- Une sonde TCP Port sur `<IP_VM>:55000` (API Wazuh)

### 7. Installer l'agent Wazuh sur la VM hôte

```bash
sudo apt-get install -y gnupg apt-transport-https
curl -so wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.9.0-1_amd64.deb
sudo WAZUH_MANAGER='127.0.0.1' dpkg -i ./wazuh-agent.deb
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### 8. ⚠️ Étape critique : activer la surveillance de `/var/log/auth.log`

La configuration par défaut de l'agent Wazuh **ne surveille pas** `/var/log/auth.log`, ce qui empêche toute détection des événements SSH. Une configuration corrigée est fournie dans ce dépôt (`Annexes/C_scripts/ossec_agent_config_auth-log.conf`) à titre de référence. Pour l'appliquer :

```bash
sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak

grep -n "</ossec_config>" /var/ossec/etc/ossec.conf
# Repérer le numéro de la dernière ligne de fermeture (section "Log analysis")

sudo sed -i 'NUMERO_LIGNEa\
  <localfile>\
    <log_format>syslog</log_format>\
    <location>/var/log/auth.log</location>\
  </localfile>' /var/ossec/etc/ossec.conf

sudo systemctl restart wazuh-agent
```

(Remplacer `NUMERO_LIGNE` par le numéro identifié juste avant, moins 1.)

### 9. Vérifier le fonctionnement

```bash
docker exec -it m1tech-wazuh-manager /var/ossec/bin/agent_control -l
# L'agent doit apparaître avec le statut "Active"
```

### 10. Reproduire les événements de sécurité (optionnel, à but de test)

```bash
# Bruteforce SSH (depuis un poste externe)
ssh -p 2222 utilisateur_inexistant@<IP_VM>

# Scan Nmap (depuis un poste externe)
nmap -sV <IP_VM>

# Création d'utilisateur suspect (sur la VM)
sudo useradd -m -s /bin/bash utilisateur_suspect
```

Les alertes correspondantes sont consultables via :
```bash
docker exec -it m1tech-wazuh-manager tail -50 /var/ossec/logs/alerts/alerts.log
```

---

## Documentation complémentaire

- **Rapport principal** : analyse complète du besoin métier, architecture, sécurisation, supervision, détection et analyse d'incident — voir `Rapport/`
- **Journal des commandes** : `Annexes/D_commandes/commandes.txt` — historique commenté de toutes les commandes exécutées, incluant le diagnostic des deux incidents rencontrés
- **Schémas d'architecture** : `Annexes/A_schemas/`
