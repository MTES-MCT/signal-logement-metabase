# Metabase - Signal Logement

Instance Metabase de Signal Logement déployée sur Scalingo et protégée par `oauth2-proxy`.

## Déploiement sur Scalingo

### Configuration du remote Scalingo

Configurer le remote Git Scalingo :

```bash
scalingo --app signal-logement-metabase git-setup
```

Vérifier les remotes :

```bash
git remote -v
```

Le dépôt GitHub et l'application Scalingo sont deux remotes distincts :

```text
origin    git@github.com:MTES-MCT/signal-logement-metabase.git
scalingo  git@ssh.osc-fr1.scalingo.com:signal-logement-metabase.git
```

## Mise à jour de Metabase

La version de Metabase est récupérée par le buildpack Metabase lors du déploiement.

Le buildpack Metabase est disponible ici :
https://github.com/metabase/metabase-buildpack

### Workflow Git

Le dépôt GitHub constitue le dépôt de référence :

```text
GitHub
MTES-MCT/signal-logement-metabase
        │ ▲
        │ │
        │ │ git pull / git push
        │ │ 
        ▼
     Dépôt local
        │
        │ git push scalingo main
        ▼
      Scalingo
signal-logement-metabase
```

---

## Protection par oauth2-proxy (GitHub)

L'instance Metabase est placée derrière `oauth2-proxy`.

L'authentification est réalisée avec GitHub. Les utilisateurs doivent s'authentifier avant de pouvoir accéder à Metabase.

Documentation oauth2-proxy :
https://oauth2-proxy.github.io/oauth2-proxy/

Buildpack utilisé :
https://github.com/betagouv/oauth2-proxy-buildpack

### Fonctionnement

Le buildpack `oauth2-proxy-buildpack` est déclaré dans `.buildpacks`.

Le `Procfile` lance `oauth2-proxy` devant Metabase.

L'architecture est la suivante :

```text
Utilisateur
    │
    ▼
Scalingo
    │
    ▼
oauth2-proxy
    │
    ├── Authentification GitHub
    │
    ▼
Metabase
127.0.0.1:3000
```

`oauth2-proxy` écoute sur le port public fourni par Scalingo et transmet les requêtes authentifiées vers Metabase.

Metabase n'est donc pas directement exposé publiquement.

## Créer l'application OAuth GitHub

Créer une OAuth App GitHub depuis les paramètres de l'organisation ou du compte utilisé pour l'authentification.

Configurer comme callback :

```text
https://signal-logement-metabase.osc-fr1.scalingo.io/oauth2/callback
```

GitHub fournit ensuite :

- un `Client ID` ;
- un `Client Secret`.

Ces valeurs doivent être configurées uniquement dans les variables d'environnement Scalingo et ne doivent jamais être commitées dans le dépôt.

### Variables d'environnement à définir sur Scalingo

```text
# --- Github ---

OAUTH2_PROXY_PROVIDER=github
OAUTH2_PROXY_SCOPE="read:org user:email"

# --- Identifiants de l'app OAuth Github ---
OAUTH2_PROXY_CLIENT_ID=<client_id>
OAUTH2_PROXY_CLIENT_SECRET=<client_secret>
OAUTH2_PROXY_COOKIE_SECRET=<cookie_secret> # ex: openssl rand -base64 32 | head -c 32

# --- Réseau / cookies ---
OAUTH2_PROXY_UPSTREAMS=http://127.0.0.1:8080/   # Metabase en interne
OAUTH2_PROXY_REDIRECT_URL=https://signal-logement-metabase.osc-fr1.scalingo.io/oauth2/callback
OAUTH2_PROXY_COOKIE_DOMAINS=signal-logement-metabase.osc-fr1.scalingo.io
OAUTH2_PROXY_COOKIE_SECURE=true
OAUTH2_PROXY_REVERSE_PROXY=true

# --- Restriction d'accès par utilisateur appartenant au dépot  MTES-MCT/histologe ---
OAUTH2_PROXY_GITHUB_REPO=MTES-MCT/histologe
```



Pour publier des modifications :

```bash
git push origin main
```

### Exclure des routes de l'authentification (embeds Metabase, optionnel)

Si vous utilisez les dashboards embarqués Metabase, excluez les routes publiques :

```bash
OAUTH2_PROXY_SKIP_AUTH_ROUTES=^/public/.*,^/api/public/.*,^/app/.*,^/api/session/properties$,^/embed/.*,^/api/embed/.*
```

### Déploiement

### Forcer une version lors du déploiement

Pour les hotfix en particulier, il est parfois intéressant de forcer une version spécifique de Metabase.

Pour cela, il faut éditer la variable d'environnement suivante directement dans Scalingo : `METABASE_VERSION`.

### Commande

#### Deploy depuis la cli Scalingo

```bash
scalingo --app signal-logement-metabase deploy https://github.com/MTES-MCT/signal-logement-metabase/archive/refs/heads/main.tar.gz
```

#### Deploy depuis la cli Scalingo

Cette version comporte l'inconvénient de devoir forcer un commit vide vers Scalingo, et devoir rappatrier les commit des autres contributeurs.

Par ailleurs, le déploiement est aléatoire côté Scalingo à cause des commit vides.

```bash
git checkout main
git pull origin main
git commit --allow-empty -m "empty commit"
git push scalingo main
```
