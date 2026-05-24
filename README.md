# Procédure GoPhish — Déploiement et campagne de phishing interne

> 🎣 Guide complet pour déployer GoPhish sur Linux et mener une campagne 
> de phishing interne : installation, configuration SMTP, création d'un 
> e-mail piégé type Microsoft 365, landing page et analyse des résultats.

## Contexte

Cette procédure a été réalisée dans le cadre d'une mission de sensibilisation 
à la sécurité informatique en entreprise. L'objectif était de tester la 
vigilance des collaborateurs face aux attaques de phishing et d'identifier 
les axes d'amélioration.

**Stack utilisée :** GoPhish · Ubuntu 24.04 · SMTP Office 365 · MariaDB

## Introduction

GoPhish est un framework open-source de simulation de phishing conçu pour aider les entreprises et les professionnels de la cybersécurité à tester et renforcer la vigilance de leurs employés face aux attaques de phishing.

---

## Installation

### Prérequis

- Une machine Linux (cette procédure utilise Ubuntu 24.04.2 LTS)
- Accès root ou sudo

### Mise à jour et dépendances

```bash
sudo apt -y update && sudo apt upgrade && sudo apt full-upgrade && sudo apt autoclean && sudo apt clean
sudo apt -y install wget unzip
```

### Ouverture des ports

```bash
sudo apt install -y iptables
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 3333 -j ACCEPT
sudo mkdir -p /etc/iptables
sudo sh -c "iptables-save > /etc/iptables/rules.v4"
sudo iptables-restore < /etc/iptables/rules.v4
sudo iptables -L -v -n
```

| Port | Usage |
|---|---|
| 80 | Trafic HTTP (landing page) |
| 443 | Trafic HTTPS |
| 3333 | Interface d'administration GoPhish |

### Télécharger et installer GoPhish

```bash
# Version française (non officielle)
cd /tmp
wget https://github.com/PassAndSecure/Template_Gophish/releases/download/gophish-v0.12.1-linux-64bit-fr/gophish-v0.12.1-linux-64bit-fr.zip
sudo unzip gophish-v0.12.1-linux-64bit-fr.zip -d /opt
sudo mv /opt/gophish-v0.12.1-linux-64bit-fr /opt/gophish

# Version anglaise officielle
cd /tmp
wget https://github.com/gophish/gophish/releases/download/v0.12.1/gophish-v0.12.1-linux-64bit.zip
sudo unzip gophish-v0.12.1-linux-64bit.zip -d /opt
sudo mv /opt/gophish-v0.12.1-linux-64bit /opt/gophish
```

Rendez le binaire exécutable et lancez-le une première fois pour récupérer le mot de passe administrateur :

```bash
sudo chmod +x /opt/gophish/gophish
cd /opt/gophish
sudo ./gophish
```

> Le mot de passe administrateur s'affiche dans les logs au premier lancement.

Accédez à l'interface d'administration via : `https://<IP_de_votre_machine>:3333`
Identifiant : `admin` / Mot de passe : *(récupéré dans les logs)*

---

## Configuration de GoPhish

### Fichier de configuration

```bash
cd /opt/gophish
sudo cp config.json config.json.backup
sudo nano config.json
```

Contenu du fichier `config.json` :

```json
{
  "admin_server": {
    "listen_url": "0.0.0.0:3333",
    "use_tls": true,
    "cert_path": "gophish_admin.crt",
    "key_path": "gophish_admin.key",
    "trusted_origins": []
  },
  "phish_server": {
    "listen_url": "0.0.0.0:80",
    "use_tls": false,
    "cert_path": "gophish_admin.crt",
    "key_path": "gophish_admin.key"
  },
  "db_name": "sqlite3",
  "db_path": "gophish.db",
  "migrations_prefix": "db/db_",
  "contact_address": "",
  "logging": {
    "filename": "",
    "level": ""
  }
}
```

### Créer un service systemd

Pour que GoPhish démarre automatiquement et fonctionne même après fermeture de session :

```bash
sudo nano /etc/systemd/system/gophish.service
```

```ini
[Unit]
Description=Gophish Phishing Framework
After=network.target

[Service]
ExecStart=/opt/gophish/gophish
WorkingDirectory=/opt/gophish
User=root
Group=root
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart gophish
sudo systemctl enable gophish
sudo systemctl status gophish
```

### Base de données MariaDB (optionnel)

GoPhish utilise SQLite par défaut. Pour une gestion plus robuste avec MariaDB :

```bash
sudo apt install -y mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo mysql_secure_installation
```

> ⚠️ Évitez les caractères suivants dans vos mots de passe : `%[];<>`

```sql
CREATE DATABASE gophish CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'gophish_user'@'localhost' IDENTIFIED BY 'votre_mdp_fort';
GRANT ALL PRIVILEGES ON gophish.* TO 'gophish_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Ajustez le mode SQL de MariaDB :

```bash
echo -e "\n[mysqld]\nsql_mode=ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION" | sudo tee -a /etc/mysql/mariadb.conf.d/50-server.cnf
sudo systemctl restart mariadb
```

Modifiez `config.json` pour utiliser MariaDB :

```json
"db_name": "mysql",
"db_path": "gophish_user:votre_mdp_fort@(localhost:3306)/gophish?charset=utf8&parseTime=True&loc=UTC"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart gophish
journalctl -u gophish.service | grep "password"
```

---

## Création de la campagne de phishing

### 1. Créer les utilisateurs et groupes

Dans l'interface GoPhish, allez dans **Utilisateurs & Groupes > Nouveau Groupe**.

Vous pouvez importer les utilisateurs via un fichier CSV. Pour extraire les utilisateurs depuis Active Directory (PowerShell) :

```powershell
Get-ADUser -Filter * -SearchBase "OU=VotreOU,DC=votre-domaine,DC=local" -Properties mail,givenName,sn,title |
Select-Object @{n='First Name';e={$_.givenName}},@{n='Last Name';e={$_.sn}},@{n='Email';e={$_.mail}},@{n='Position';e={$_.Title}} |
Export-CSV -Path "C:\UsersGophish.csv" -Delimiter "," -NoTypeInformation
```

### 2. Créer le profil d'envoi

Allez dans **Profils d'Envoi > Nouveau Profil**.

Exemple avec Gmail (nécessite un [mot de passe d'application](https://myaccount.google.com/apppasswords)) :

| Champ | Valeur |
|---|---|
| Nom | Google |
| Type d'interface | SMTP |
| Expéditeur SMTP | `votre-email@gmail.com` |
| Hôte | `smtp.gmail.com:465` |
| Nom d'utilisateur | `votre-email@gmail.com` |
| Mot de passe | *(mot de passe d'application)* |
| Ignorer les erreurs de certificat | ✅ |

> Envoyez un e-mail de test pour valider la configuration avant de lancer la campagne.

### 3. Créer l'e-mail piégé

Allez dans **Modèles d'Email > Nouveau Modèle**.

Exemple de configuration pour une simulation d'alerte Microsoft 365 :

| Champ | Valeur |
|---|---|
| Nom | AlerteSecuriteMicrosoft |
| Sujet | `No-reply alerte de sécurité Microsoft` |
| Ajouter une image de suivi | ✅ |

> Utilisez la variable `{{.Email}}` dans le corps de l'e-mail pour personnaliser chaque envoi avec l'adresse du destinataire.

### 4. Créer la landing page

Allez dans **Pages de Destination > Nouvelle Page**.

| Champ | Valeur |
|---|---|
| Nom | m365-connexion |
| Capturer les données soumises | ✅ |
| Capturer les mots de passe | ❌ |
| Redirection vers | `https://www.office.com` |

> ⚠️ **Avertissement** : les identifiants capturés sont stockés en clair dans la base de données. Ne capturez jamais de vrais mots de passe en production.

### 5. Lancer la campagne

Allez dans **Campagnes > Nouvelle Campagne** :

| Champ | Valeur |
|---|---|
| Modèle d'Email | AlerteSecuriteMicrosoft |
| Page de Destination | m365-connexion |
| URL | `http://<IP_de_votre_machine>` |
| Profil d'Envoi | *(votre profil SMTP)* |
| Groupes | *(sélectionnez vos groupes cibles)* |

> **Conseil** : définissez une date de fin pour que GoPhish répartisse les envois dans le temps — cela évite que tous les destinataires reçoivent l'e-mail simultanément.

---

## Résultats

GoPhish vous permet de suivre en temps réel :

- **Email Sent** : nombre d'e-mails envoyés
- **Email Opened** : nombre d'e-mails ouverts
- **Clicked Link** : nombre de clics sur le lien piégé
- **Submitted Data** : nombre de formulaires soumis
- **Email Reported** : nombre d'e-mails signalés comme suspects

---

## Ressources

- [GoPhish — Documentation officielle](https://docs.getgophish.com/)
- [IT-Connect — Créer une campagne de phishing avec GoPhish](https://www.it-connect.fr/comment-creer-une-campagne-de-phishing-avec-gophish/)
- [Templates GoPhish FR](https://github.com/PassAndSecure/Template_Gophish)
