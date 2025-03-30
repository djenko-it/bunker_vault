# BunkerWeb + Authentik + Vaultwarden Stack

Déploiement sécurisé avec :
- 🛡️ **BunkerWeb** : Reverse proxy/WAF
- 🔐 **Authentik** : Gestion des identités (SSO)
- 🔒 **Vaultwarden** : Gestion des mots de passe

## 📋 Prérequis
- Docker 23.0+
- Docker Compose 2.20+
- Domaines valides (configurés dans `SERVER_NAME`)
- Fichier `.env` configuré

## ⚙️ Configuration (.env)
Créez un fichier `.env` à la racine avec ces variables :

```ini
# Authentik - PostgreSQL
PG_PASS=votre_mot_de_passe_fort
PG_USER=authentik
PG_DB=authentik

# Authentik - Image (optionnel)
AUTHENTIK_IMAGE=ghcr.io/goauthentik/server
AUTHENTIK_TAG=2025.2.3

# BunkerWeb UI (à changer!)
ADMIN_USERNAME=admin
ADMIN_PASSWORD=changemeAdmin11!

# Vaultwarden
ADMIN_TOKEN=TokenSecurise123
