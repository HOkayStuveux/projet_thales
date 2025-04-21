# README - Déploiement Sécurisé Docker/Ansible

## Objectif

Mettre en place une infrastructure d’automatisation sécurisée avec :
1. Un serveur SSH sous Debian 12 hébergeant Nginx
2. Un contrôleur Ansible sous AlmaLinux 9
3. Un playbook Ansible pour déployer Nginx

---

## 1. Démarche et choix d’architecture

### Approche initiale

Au début, j’ai créé et lancé chaque conteneur séparément en ligne de commande Docker.  
**Limites rencontrées :**
- Difficulté à gérer les réseaux et la communication inter-conteneurs.
- Démarrage manuel et séquentiel fastidieux.
- Configuration dispersée, difficile à maintenir et à versionner.

### Passage à Docker Compose

J’ai migré vers Docker Compose pour :
- **Centraliser la configuration** dans un fichier YAML unique.
- **Automatiser la création d’un réseau interne** (`ansible-net`) pour isoler les communications.
- **Définir des dépendances** (`depends_on` et `healthcheck`) pour garantir l’ordre de démarrage.
- **Simplifier le déploiement** et la reproductibilité de l’environnement.

---

## 2. Serveur SSH (Debian 12)

**Implémentation Docker :**
- Dockerfile basé sur `debian:12`
- Installation d’OpenSSH, Python3 et sudo
- Création d’un utilisateur non-root `ansible-user` avec sudo sans mot de passe
- Génération des clés d’hôte avec `ssh-keygen -A`
- Script d’entrée [`docker-entrypoint.sh`](#docker-entrypointsh) pour créer `/run/sshd` et démarrer le service SSH
- Ports exposés : 8022 (SSH), 8080 (Nginx)

**Sécurité :**
- Utilisateur non-root pour les connexions SSH et les tâches Ansible
- Désactivation de l’authentification par mot de passe dans `sshd_config`
- Montage des clés publiques en lecture seule via volume Docker
- Réseau interne Docker (`internal: true`) pour limiter l’exposition

**Défis rencontrés :**
- Erreur `no hostkeys available` : résolue par la génération automatique des clés d’hôte dans le Dockerfile
- Erreur `Missing privilege separation directory: /run/sshd` : corrigée en créant ce dossier dans l’entrypoint
- Problèmes de permissions sur les clés SSH : résolus par un script de génération et des commandes `chmod`
- Problèmes DNS dans le conteneur lors de l’installation de paquets : résolus en forçant le DNS dans la configuration Docker Desktop

---

## 3. Contrôleur Ansible (AlmaLinux 9)

**Implémentation Docker :**
- Dockerfile basé sur `almalinux:9`
- Installation d’Ansible via pip
- Utilisation d’un utilisateur non-root (UID 1000)
- Montage du dossier contenant les clés SSH en lecture seule
- Commande principale : `tail -f /dev/null` pour garder le conteneur actif
- Healthcheck avec `ansible --version`
- Réseau dédié `ansible-net` pour l’isolation

**Sécurité :**
- Exécution en utilisateur non-root
- Volumes montés en lecture seule pour les fichiers sensibles
- Séparation réseau stricte (bridge interne Docker)
- Pas de ports exposés côté hôte

**Défis :**
- Conteneur qui s’arrêtait immédiatement (exit 0) : corrigé par l’ajout d’une commande persistante
- Problèmes de résolution DNS entre conteneurs : réglés en utilisant les hostnames Docker et un réseau dédié

---

## 4. Gestion des clés SSH

- Génération de la paire de clés avec [`ssh-keygen.sh`](#ssh-keygensh) :
  - Clé privée montée en lecture seule dans le conteneur Ansible (`./ssh-keys:/ansible/ssh-keys:ro`)
  - Clé publique copiée dans `ssh-server/authorized_keys`
- Permissions strictes appliquées (`chmod 600` pour la clé privée, `chmod 644` pour la clé publique)
- Vérification automatique de la présence des clés lors du démarrage

---

## 5. Playbook Ansible pour Nginx

**Structure :**
- Playbook YAML (`deploy-nginx.yml`) qui :
  - Installe Nginx sur le serveur SSH cible
  - Démarre et active le service Nginx
  - Vérifie l’accessibilité via le module `uri`
- Inventory Ansible pointant sur le hostname du service SSH Docker

**Sécurité :**
- Désactivation du host key checking dans `ansible.cfg`
- Utilisation exclusive de clés SSH pour l’authentification
- Aucune donnée sensible stockée en clair dans les playbooks ou inventaires

**Défis :**
- Connexion SSH refusée : corrigée par la bonne gestion des clés et permissions dans `authorized_keys`
- Module `apt` qui échouait sur Debian sans Python : résolu par l’installation de Python3 dans le Dockerfile SSH

---

## 6. Fichiers et scripts principaux

- [`Docker-compose.yaml`](#docker-composeyaml) : Orchestration complète de l’infrastructure
- [`ssh-keygen.sh`](#ssh-keygensh) : Génération et distribution des clés SSH
- [`docker-entrypoint.sh`](#docker-entrypointsh) : Démarrage sécurisé du serveur SSH
- `sshd_config`, `sudoers`, `deploy-nginx.yml`, `ansible.cfg`, `inventory.ini` : configuration fine des services et des accès

---

## 7. Bilan technique

### Réussites
- **Isolation réseau** via Docker Compose et réseau interne
- **Sécurisation des accès** (utilisateur non-root, clés, restrictions)
- **Déploiement idempotent** de Nginx via Ansible (tests validés)
- **Respect des bonnes pratiques Docker** (volumes en ro, healthchecks, pas de mode privilégié)
- **Documentation claire** et reproductibilité garantie

### Limites et axes d’amélioration
- Gestion des secrets via Ansible Vault non implémentée
- Rotation et gestion avancée des clés SSH non automatisée
- Monitoring et logs centralisés non mis en place
- Problèmes DNS récurrents sous Docker Desktop Windows (résolus manuellement)
- Pas d’intégration CI/CD complète (hors scope)

---

## 8. Structure des fichiers

.
├── Docker-compose.yaml
├── ssh-server/
│ ├── Dockerfile
│ ├── sshd_config
│ ├── authorized_keys
│ ├── docker-entrypoint.sh
├── ansible/
│ ├── Dockerfile
│ ├── ansible.cfg
│ ├── inventory.ini
│ ├── deploy-nginx.yml
├── ssh-keys/
│ ├── id_rsa
│ └── id_rsa.pub
├── ssh-keygen.sh

text

---

## 9. Conclusion

Ce projet démontre une approche sécurisée et reproductible du déploiement d’applications avec Docker et Ansible, en mettant l’accent sur la séparation des privilèges, la gestion des accès et la clarté des configurations.  
L’expérience a montré que Docker Compose est indispensable pour gérer efficacement une infrastructure multi-conteneurs, tant pour la sécurité que pour la maintenabilité.  
Les difficultés rencontrées (notamment liées à Docker Desktop et à la gestion des volumes/permissions sous Windows) ont permis de renforcer la robustesse des scripts et des procédures.

Pour toute question ou suggestion, n’hésitez pas à me contacter.
