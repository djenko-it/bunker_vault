# BunkerWeb avec Authentik et Vaultwarden

Ce dépôt contient une configuration Docker Compose pour déployer une infrastructure complète incluant :

- **BunkerWeb** : Reverse proxy sécurisé basé sur NGINX
- **Authentik** : Solution d'identité et d'accès (SSO, MFA)
- **Vaultwarden** : Gestionnaire de mots de passe auto-hébergé compatible Bitwarden

## Architecture

L'infrastructure est composée des éléments suivants :

- **BunkerWeb** : Sert de point d'entrée pour toutes les applications, avec une configuration renforcée
- **Authentik** : Gère l'authentification centralisée
- **Vaultwarden** : Gestionnaire de mots de passe avec intégration Authentik pour l'accès admin

## Fonctionnalités

- Configuration entièrement paramétrée via variables d'environnement
- Gestion sécurisée des secrets avec Docker Secrets
- Protection géographique (par pays)
- Protection contre les bots et scanners
- Authentification centralisée pour les applications

## Prérequis

- Docker et Docker Compose v2 ou supérieur
- Domaines configurés et pointant vers votre serveur
- Serveur Linux avec au moins 2Go de RAM

## Structure des fichiers

```
.
├── docker-compose.yml     # Configuration principale
├── .env                   # Variables d'environnement (à partir du template)
├── secrets/               # Dossier contenant les secrets
│   ├── postgres_password.txt
│   ├── vaultwarden_admin_token.txt
│   └── bw_ui_admin_password.txt
├── media/                 # Dossier pour les médias d'Authentik
├── custom-templates/      # Templates personnalisés pour Authentik
├── certs/                 # Certificats (utilisés par Authentik)
└── vw-data/               # Données Vaultwarden
```

## Installation

1. **Cloner le dépôt**
   ```bash
   git clone https://github.com/VOTRE-UTILISATEUR/VOTRE-REPO.git
   cd VOTRE-REPO
   ```

2. **Créer la structure des dossiers et les secrets**
   ```bash
   # Exécuter le script de création des secrets
   chmod +x create-secrets.sh
   ./create-secrets.sh
   ```

3. **Configurer le fichier .env**
   ```bash
   # Le script create-secrets.sh a déjà créé le fichier .env à partir du template
   # Éditer le fichier avec vos propres valeurs
   nano .env
   ```
   
   Au minimum, modifiez les variables suivantes :
   - `BASE_DOMAIN` : votre domaine principal
   - `AUTH_DOMAIN`, `ADMIN_DOMAIN`, `VAULT_DOMAIN` : sous-domaines pour vos applications
   - `SERVER_NAMES` : liste de tous vos domaines (séparés par des espaces)
   - `WHITELIST_COUNTRY` : codes des pays autorisés (ISO 3166-1 alpha-2)
   - `AUTHENTIK_COOKIE_DOMAIN` : domaine pour les cookies Authentik

4. **Démarrer les services**
   ```bash
   docker compose up -d
   ```

## URLs des services

Une fois déployé, les services seront disponibles aux adresses suivantes (adaptez selon vos domaines) :

- **Authentik** : https://${AUTH_DOMAIN} (par défaut : https://bunk.djenkoit.fr)
- **BunkerWeb Admin** : https://${ADMIN_DOMAIN} (par défaut : https://admin.djenkoit.fr)
- **Vaultwarden** : https://${VAULT_DOMAIN} (par défaut : https://vault.djenkoit.fr)
- **Interface d'administration Vaultwarden** (protégée par Authentik) : https://${VAULT_DOMAIN}/admin

## Personnalisation

### Ajouter une nouvelle application

Pour ajouter une nouvelle application à votre infrastructure :

1. Définissez les variables pour votre application dans `.env` :
   ```
   APP_NEW_DOMAIN=app-new.votredomaine.com
   ```

2. Ajoutez votre domaine à la liste `SERVER_NAMES` dans `.env` :
   ```
   SERVER_NAMES=bunk.votredomaine.com admin.votredomaine.com vault.votredomaine.com app-new.votredomaine.com
   ```

3. Créez un nouveau service dans `docker-compose.yml` ou utilisez un service existant.

4. Ajoutez la configuration de proxy pour votre application dans la section `bw-scheduler` :
   ```yaml
   - ${APP_NEW_DOMAIN}_REVERSE_PROXY_URL=/
   - ${APP_NEW_DOMAIN}_REVERSE_PROXY_HOST=http://your-app-service:port
   ```

5. Si vous souhaitez protéger l'application avec Authentik, ajoutez :
   ```yaml
   - ${APP_NEW_DOMAIN}_REVERSE_PROXY_AUTH_REQUEST=/outpost.goauthentik.io/auth/nginx
   - ${APP_NEW_DOMAIN}_REVERSE_PROXY_AUTH_REQUEST_SIGNIN_URL=https://${AUTH_DOMAIN}/outpost.goauthentik.io/start?rd=$$scheme%3A%2F%2F$$host$$request_uri
   - ${APP_NEW_DOMAIN}_REVERSE_PROXY_AUTH_REQUEST_SET=$$auth_cookie $$upstream_http_set_cookie;$$authentik_username $$upstream_http_x_authentik_username;$$authentik_groups $$upstream_http_x_authentik_groups;$$authentik_email $$upstream_http_x_authentik_email;$$authentik_name $$upstream_http_x_authentik_name;$$authentik_uid $$upstream_http_x_authentik_uid
   - ${APP_NEW_DOMAIN}_REVERSE_PROXY_HEADERS_CLIENT=Set-Cookie $$auth_cookie
   - ${APP_NEW_DOMAIN}_REVERSE_PROXY_HEADERS=X-authentik-username $$authentik_username;X-authentik-groups $$authentik_groups;X-authentik-email $$authentik_email;X-authentik-name $$authentik_name;X-authentik-uid $$authentik_uid
   ```

6. Redémarrez les services :
   ```bash
   docker compose up -d
   ```

## Maintenance

### Mettre à jour les conteneurs
```bash
# Mettre à jour les versions dans .env si nécessaire
nano .env

# Mettre à jour les conteneurs
docker compose pull
docker compose up -d
```

### Sauvegarder les données
Il est recommandé de sauvegarder régulièrement :
- Le dossier `vw-data/` (données Vaultwarden)
- Le volume `database` (base de données PostgreSQL)
- Le dossier `media/` (fichiers média Authentik)
- Le dossier `secrets/` (pour les mots de passe)
- Le fichier `.env` (pour la configuration)

## Dépannage

### Logs
```bash
# Tous les services
docker compose logs

# Service spécifique
docker compose logs server
docker compose logs bunkerweb
```

### Problèmes courants

1. **Erreurs de certificats SSL**
   - Vérifiez que vos domaines pointent correctement vers votre serveur
   - Vérifiez que les ports 80 et 443 sont accessibles
   - Vérifiez les logs BunkerWeb : `docker compose logs bunkerweb`

2. **Problèmes d'authentification Authentik**
   - Vérifiez que le service Authentik est opérationnel : `docker compose logs server`
   - Vérifiez la configuration du proxy dans BunkerWeb

3. **Accès refusé à BunkerWeb UI**
   - Vérifiez les identifiants dans `secrets/bw_ui_admin_password.txt`
   - Vérifiez les logs BunkerWeb : `docker compose logs bw-ui`

## Sécurité

Cette configuration inclut plusieurs mesures de sécurité :
- Utilisation de Docker Secrets pour les mots de passe
- Limitation des accès par pays (configurable via `WHITELIST_COUNTRY`)
- Protection contre les bots (blacklisting de User-Agents via `BLACKLIST_RDNS`)
- Limitation de débit pour les API
- Authentification centralisée via Authentik
