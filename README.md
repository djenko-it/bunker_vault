# BunkerWeb avec Authentik et Vaultwarden

[![GitHub](https://img.shields.io/badge/GitHub-djenko--it%2Fbunker__vault-blue?logo=github)](https://github.com/djenko-it/bunker_vault)

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

## Installation

1. **Cloner le dépôt**
   ```bash
   git clone https://github.com/djenko-it/bunker_vault.git
   cd bunker_vault
   ```

2. **Créer uniquement le dossier secrets (les autres seront créés automatiquement)**
   ```bash
   # Créer le dossier secrets
   mkdir -p secrets
   ```

3. **Créer les fichiers de secrets**
   ```bash
   # Générer les mots de passe pour PostgreSQL
   openssl rand -base64 24 | tr -d '\n/+=' > secrets/postgres_password.txt
   
   # Générer un token admin pour Vaultwarden
   openssl rand -base64 24 | tr -d '\n/+=' > secrets/vaultwarden_admin_token.txt
   
   # Générer un mot de passe admin pour BunkerWeb UI
   openssl rand -base64 24 | tr -d '\n/+=' > secrets/bw_ui_admin_password.txt
   
   # Limiter les permissions des fichiers de secrets
   chmod 600 secrets/*
   ```

4. **Configurer le fichier .env**
   ```bash
   # Créer un fichier .env vide
   touch .env
   
   # Configuration des domaines (remplacez example.com par votre domaine)
   echo "BASE_DOMAIN=example.com" >> .env
   echo "AUTH_DOMAIN=bunk.example.com" >> .env
   echo "ADMIN_DOMAIN=admin.example.com" >> .env
   echo "VAULT_DOMAIN=vault.example.com" >> .env
   echo "SERVER_NAMES=bunk.example.com admin.example.com vault.example.com" >> .env
   echo "AUTHENTIK_COOKIE_DOMAIN=example.com" >> .env
   echo "BW_UI_ADMIN_USER=admin" >> .env

   
   # Générer une clé secrète pour Authentik
   echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 36 | tr -d '\n')" >> .env
   
   # Configuration email (optionnel)
   # echo "AUTHENTIK_EMAIL__HOST=smtp.gmail.com" >> .env
   # echo "AUTHENTIK_EMAIL__PORT=587" >> .env
   # echo "AUTHENTIK_EMAIL__USE_TLS=true" >> .env
   # echo "AUTHENTIK_EMAIL__USERNAME=your-email@gmail.com" >> .env
   # echo "AUTHENTIK_EMAIL__PASSWORD=your-app-password" >> .env
   # echo "AUTHENTIK_EMAIL__FROM=your-email@gmail.com" >> .env
   ```
   


5. **Démarrer les services**
   ```bash
   docker compose up -d
   ```

## URLs des services

Une fois déployé, les services seront disponibles aux adresses suivantes (adaptez selon vos domaines) :

- **Authentik** : https://${AUTH_DOMAIN} (par défaut : https://bunk.example.com)
- **BunkerWeb Admin** : https://${ADMIN_DOMAIN} (par défaut : https://admin.example.com)
- **Vaultwarden** : https://${VAULT_DOMAIN} (par défaut : https://vault.example.com)
- **Interface d'administration Vaultwarden** (protégée par Authentik) : https://${VAULT_DOMAIN}/admin

## Première connexion

Lors de la première connexion à Authentik (https://${AUTH_DOMAIN}), vous devrez créer un compte administrateur.
Conservez précieusement ces identifiants, ils vous permettront d'administrer toute votre infrastructure.

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
