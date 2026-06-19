# 42---B2BR

*This activity has been created as part of the 42 curriculum by chlepain.*

# Born2beRoot

**Configuration d'un serveur Linux sécurisé sous machine virtuelle.**

---

## 📌 Description

Ce projet est réalisé dans le cadre du **Tronc Commun de l'École 42**. Il introduit les fondamentaux de l'administration système Linux : virtualisation, partitionnement chiffré, gestion des utilisateurs, sécurisation des accès et surveillance du serveur.

- **Objectif :** Configurer un serveur Debian minimal dans une machine virtuelle en respectant des règles de sécurité strictes — chiffrement des partitions, politique de mots de passe robuste, pare-feu, accès SSH sécurisé et monitoring automatique.
- **Système d'exploitation :** Debian GNU/Linux 13 (Trixie)
- **Hyperviseur :** VirtualBox 7.x

---

## 🖥️ Description du projet

### Qu'est-ce qu'une machine virtuelle ?

Une machine virtuelle est un mini PC virtuel ayant un rôle précis : être un serveur, être un poste de travail avec interface graphique, être une machine de test, être un routeur virtuel, …
Elle est créée depuis une machine hôte via un hyperviseur (VirtualBox par exemple). Le rôle de celui-ci est de prendre une part du matériel réel du PC (RAM, processeurs, disque, réseau,…) et de la distribuer à la VM. Celle-ci pense avoir son propre matériel mais en réalité ce n’est qu'une partie du matériel hôte.
Ensuite, place à la configuration de cette machine comme si c'était une machine physique. Nous devons y installer un système d'exploitation, un système de sécurité, un système de gestion des utilisateurs, paramétrer la gestion des ressources et espaces mémoires, etc.


### Choix de l'OS : Debian

Pour ce premier projet, j'ai choisi **Debian** plutôt que Rocky Linux principalement pour sa simplicité de configuration mais aussi parce que le projet Born 2 Be Root ne nécessite pas un niveau de sécurité complexe.

Voici une brève description de ses avantages & inconvénients :

**Avantages de Debian :**
- Distribution communautaire, stable et éprouvée
- Gestionnaire de paquets `apt` simple et bien documenté
- AppArmor intégré par défaut
- Recommandé pour les débutants en administration système

**Inconvénients de Debian :**
- Versions des paquets parfois plus anciennes que d'autres distributions
- Moins utilisé en environnement entreprise que RHEL/Rocky

**Pourquoi pas Rocky Linux ?**
Rocky est un clone de RHEL orienté entreprise — plus complexe à configurer (SELinux, firewalld) et moins adapté à une première approche de l'administration système.

---

### Comparatifs techniques

#### Debian vs Rocky Linux

| Critère | Debian | Rocky Linux |
|---------|--------|-------------|
| Type | Communautaire | Entreprise (clone RHEL) |
| Gestionnaire de paquets | `apt` | `dnf` |
| Sécurité MAC | AppArmor | SELinux |
| Pare-feu | UFW | firewalld |
| Difficulté | Débutant-friendly | Plus complexe |
| Usage | Serveurs web, postes | Environnements entreprise |

#### AppArmor vs SELinux

| Critère | AppArmor | SELinux |
|---------|----------|---------|
| Approche | Basée sur les chemins de fichiers | Basée sur les étiquettes (labels) |
| Complexité | Simple, profils lisibles | Très complexe, courbe d'apprentissage élevée |
| Configuration | Profils dans `/etc/apparmor.d/` | Politiques complexes |
| Modes | Enforce / Complain | Enforcing / Permissive / Disabled |
| Distribution | Debian, Ubuntu | RHEL, Rocky, Fedora |

Les deux sont des systèmes **MAC (Mandatory Access Control)** qui confinent les programmes — même root est limité par leurs règles. AppArmor est plus accessible pour débuter.

#### UFW vs firewalld

| Critère | UFW | firewalld |
|---------|-----|-----------|
| Signification | Uncomplicated Firewall | — |
| Approche | Règles simples par port | Zones de sécurité (trusted, public, dmz...) |
| Complexité | Très simple | Plus complexe mais plus flexible |
| Distribution | Debian, Ubuntu | Rocky, RHEL, Fedora |
| Commande exemple | `ufw allow 4242` | `firewall-cmd --add-port=4242/tcp` |

Les deux sont des **interfaces** pour gérer le pare-feu du noyau Linux (netfilter). UFW suffit largement pour un serveur simple.

#### VirtualBox vs UTM

| Critère | VirtualBox | UTM |
|---------|-----------|-----|
| Plateforme | Windows, Linux, macOS (Intel) | macOS (Apple Silicon principalement) |
| Type | Hyperviseur de type 2 | Hyperviseur basé sur QEMU |
| Interface | Graphique complète | Graphique simple |
| Usage | Standard, très documenté | Nécessaire sur Mac M1/M2/M3 |
| Gratuité | Oui | Oui |

VirtualBox est le choix standard pour ce projet sur machines Intel/AMD.

---

## 🛠️ Choix de conception & configuration réalisée

### Partitionnement

Le disque virtuel de 12 Go a été partitionné avec la méthode guidée **LVM chiffré** :

- **sda1** (500 Mo) → partition primaire non chiffrée, montée sur `/boot`. Elle doit rester lisible au démarrage pour que GRUB puisse charger le noyau Linux avant que LUKS ne demande la passphrase.
- **sda2** (11,5 Go) → partition étendue, conteneur des partitions logiques.
- **sda5** → partition logique chiffrée avec **LUKS** (Linux Unified Key Setup), déclarée comme PV et versée dans le VG.

L'ordre de démarrage est le suivant :
```
BIOS → GRUB (/boot) → noyau Linux → LUKS (passphrase) → LVM → système démarré
```

<img width="527" height="199" alt="image" src="https://github.com/user-attachments/assets/ff57d1c4-0f18-4d43-ab08-cf4007264764" />



Le **VG (Volume Group)** regroupe l'espace des PV en un pool commun, ensuite découpé en **LV (Logical Volumes)** — les partitions finales redimensionnables à la volée :

| LV | Point de montage | Taille |
|----|-----------------|--------|
| LV root | `/` | 7,7 Go |
| LV home | `/home` | 3,4 Go |
| LV swap | `[SWAP]` | 620 Mo |

### Sécurité système

- **AppArmor** actif au démarrage — confine chaque programme dans un profil de règles précis (fichiers accessibles, réseau autorisé). Même root est limité par ces règles.
- **UFW** configuré — seul le port **4242** est ouvert. Tout le reste est bloqué.
- Chiffrement **LUKS** sur toutes les partitions sauf `/boot` — les données sont illisibles sans la passphrase, même si le disque est volé.

### Accès SSH

- Port **4242** au lieu du port 22 par défaut — réduit les attaques automatisées par force brute.
- Connexion root via SSH **interdite** (`PermitRootLogin no`) — double verrou de sécurité : un attaquant doit d'abord trouver un login utilisateur valide, puis son mot de passe, avant de pouvoir utiliser sudo.

### Politique de mots de passe

| Règle | Valeur |
|-------|--------|
| Expiration | 30 jours |
| Délai minimum entre changements | 2 jours |
| Avertissement avant expiration | 7 jours |
| Longueur minimum | 10 caractères |
| Complexité | 1 majuscule + 1 minuscule + 1 chiffre |
| Répétition max | 3 caractères identiques consécutifs |
| Différence avec ancien mdp | 7 caractères minimum |

Configurée via `/etc/login.defs` (expiration) et `libpam-pwquality` (complexité).

### Configuration sudo

- Maximum **3 tentatives** en cas de mauvais mot de passe
- Message personnalisé en cas d'erreur
- Logs complets dans `/var/log/sudo/sudo.log` (inputs ET outputs)
- Mode **TTY** obligatoire — sudo ne peut être utilisé que depuis un vrai terminal interactif, pas depuis un script automatique

### Gestion des utilisateurs

- Utilisateur `chlepain42` dans les groupes **sudo** et **user42**
- Principe du **moindre privilège** : connexion quotidienne en utilisateur normal, droits root uniquement via sudo pour les actions qui le nécessitent

### Monitoring automatique

- Script `monitoring.sh` en bash affichant 12 informations système sur tous les terminaux via `wall`
- Exécution automatique toutes les **10 minutes** via cron (`*/10 * * * *`)
- Mise en pause ou arrêt possible sans modification du script : `systemctl stop cron`

## 🎯 Compétences techniques validées

- **Virtualisation :** Création et configuration d'une VM sous VirtualBox
- **Administration Linux :** Gestion des utilisateurs, groupes, permissions et services
- **Sécurité système :** Chiffrement LUKS, AppArmor, UFW, politique de mots de passe, sudo
- **Scripting Bash :** Collecte d'informations système, variables, pipes, awk, grep
- **Planification de tâches :** Configuration de cron

---

## 📖 Ressources

### Documentation de référence
- [Documentation Debian](https://www.debian.org/doc/) — référence officielle Debian
- [Man pages Linux](https://www.man7.org/linux/man-pages/) — référence des commandes
- [Wiki AppArmor](https://gitlab.com/apparmor/apparmor/-/wikis/home) — documentation AppArmor
- [UFW documentation](https://help.ubuntu.com/community/UFW) — guide UFW
- [PAM documentation](https://www.linux-pam.org/Linux-PAM-html/) — modules d'authentification

### Utilisation de l'IA
L'IA (Claude) a été utilisée dans ce projet comme outil pédagogique pour :
- **Comprendre les concepts** : virtualisation, LVM, LUKS, partitionnement, protocoles réseau (SSH, TCP/IP, ports)
- **Accompagnement pas à pas** : guidance lors de l'installation et de la configuration
- **Révision** : explications approfondies de chaque notion avant toute manipulation
- **Débogage** : analyse des erreurs de configuration et corrections

---

## 💻 Instructions

### Prérequis
- VirtualBox installé sur la machine hôte
- Fichier `B2BR.vdi` disponible sur support de stockage externe

### Lancer la VM
1. Ouvrir VirtualBox
2. Charger la machine B2BR depuis le fichier `.vbox`
3. Démarrer la VM
4. Saisir la passphrase LUKS au démarrage
5. Se connecter avec le compte utilisateur

### Tester SSH
```bash
ssh chlepain42@<IP_VM> -p 4242
```

### Vérifier le monitoring
```bash
bash /usr/local/bin/monitoring.sh
```

### Interrompre le monitoring sans modifier le script
```bash
systemctl stop cron    # stoppe cron temporairement
systemctl start cron   # relance cron
```
