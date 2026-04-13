# nginx-config

Configuration nginx pour le VPS `lasserre-consulting.fr`. Gère le reverse proxy HTTPS pour tous les projets hébergés, ainsi qu'un environnement nginx local Docker pour le développement.

## Architecture de reverse proxy

```
Internet
    │
    ▼
┌─────────────────────────────────────────┐
│  nginx (port 443 SSL / port 80 redirect) │
│  lasserre-consulting.fr                  │
└──────────────┬──────────────────────────┘
               │
   ┌───────────┼──────────────────────────┐
   │           │                          │
   ▼           ▼                          ▼
/qualidoc/  /mission-tracker/       /carnetroute/
   │              │                       │
   ├─ static      ├─ static               ├─ static
   │  /var/www/   │  /var/www/            │  /var/www/
   │  qualidoc/   │  mission-tracker/     │  carnetroute/
   │              │                       │
   └─ /api/ ──►  └─ /api/ ──►           └─ /api/ ──►
      :8082          :8081                   :8080
   (backend       (backend                (backend
    Docker)        Docker)                 Docker)
               │
   ┌───────────┘
   ▼
/ (racine)
└─ static /var/www/lasserre-consulting-site/
```

### Routes de production

| Path | Type | Destination | Notes |
|---|---|---|---|
| `/` | Static | `/var/www/lasserre-consulting-site/` | Site vitrine |
| `/qualidoc/` | Static | `/var/www/qualidoc/` | SPA Angular |
| `/qualidoc/api/` | Proxy | `localhost:8082/api/` | Backend Docker |
| `/mission-tracker/` | Static | `/var/www/mission-tracker/` | SPA Angular |
| `/mission-tracker/api/` | Proxy | `localhost:8081/api/` | Backend Docker + SSE |
| `/carnetroute/` | Static | `/var/www/carnetroute/` | SPA Angular |
| `/carnetroute/api/` | Proxy | `localhost:8080/api/` | Backend Docker |
| `/ws/` | WebSocket | `localhost:8080/ws/` | WebSocket carnetroute |

## Structure du repo

```
nginx-config/
├── sites-available/
│   ├── lasserre-consulting     # Config principale HTTPS + tous les projets
│   ├── default                 # Config par défaut nginx
│   └── jenkins                 # Reverse proxy Jenkins
├── local/
│   ├── nginx.conf              # Config nginx pour dev local (Docker)
│   └── docker-compose.yml      # Lance nginx Docker sur port 80
├── nginx.conf                  # Config nginx globale (worker_processes, etc.)
├── fastcgi.conf                # Params FastCGI
├── fastcgi_params
├── proxy_params                # Headers proxy communs
├── mime.types
├── snippets/
│   ├── fastcgi-php.conf
│   └── snakeoil.conf
└── Jenkinsfile                 # Pipeline validation + reload nginx
```

## Configuration production

### SSL (Let's Encrypt)

```nginx
ssl_certificate /etc/letsencrypt/live/lasserre-consulting.fr/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/lasserre-consulting.fr/privkey.pem;
include /etc/letsencrypt/options-ssl-nginx.conf;
ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
```

Renouvellement automatique via certbot. Redirection HTTP → HTTPS systématique.

### Security headers

Appliqués sur le serveur nginx interne des frontends (via `frontend/nginx.conf` de chaque projet) :

| Header | Valeur |
|---|---|
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `X-XSS-Protection` | `1; mode=block` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` |

### SSE (Server-Sent Events)

Le bloc `/mission-tracker/api/` désactive le buffering pour le streaming SSE :

```nginx
proxy_buffering off;
proxy_cache off;
proxy_read_timeout 3600s;
```

### SPA routing

Chaque frontend utilise `try_files $uri $uri/ /projet/index.html` pour que le routeur Angular gère les URLs directes.

## Configuration locale (`local/`)

Reproduit l'architecture de production en local via Docker nginx. Utile pour tester le routing multi-projets ou la config nginx avant déploiement.

### Lancer la config locale

```bash
cd local/
docker compose up -d
```

nginx écoute sur `http://localhost` (port 80) et proxifie vers les services dev sur la machine hôte via `host.docker.internal`.

### Ports de développement

| Projet | Frontend dev | Backend dev | URL locale |
|---|---|---|---|
| qualidoc | `4200` (ng serve) | `8083` (gradlew bootRun) | `http://localhost/qualidoc/` |
| mission-tracker | `4201` (ng serve) | `8081` (gradlew run / docker) | `http://localhost/mission-tracker/` |
| carnetroute | `4202` (ng serve) | `8080` (docker) | `http://localhost/carnetroute/` |

> **Note** : `ng serve` doit être lancé avec `--host 0.0.0.0` pour être accessible depuis le container Docker.

### Différences avec la prod

En local, les frontends sont servis par le dev server (`ng serve`) et non par nginx statique. Les fichiers sont proxifiés en temps réel, ce qui active le hot reload.

## Pipeline Jenkins

Le `Jenkinsfile` exécute :

1. **Checkout** — `git pull origin main` sur le VPS
2. **Test config** — `sudo nginx -t` (valide la syntaxe)
3. **Reload** — `sudo systemctl reload nginx` (rechargement sans interruption)

Le pipeline échoue si `nginx -t` retourne une erreur, empêchant le déploiement d'une config invalide.

## Ajouter un nouveau site

### 1. Ajouter les blocs dans `sites-available/lasserre-consulting`

```nginx
# ===== MON PROJET =====
location = /mon-projet {
    return 301 https://$host/mon-projet/;
}

location /mon-projet/ {
    root /var/www;
    try_files $uri $uri/ /mon-projet/index.html;
}

location /mon-projet/api/ {
    proxy_pass http://localhost:PORT/api/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

### 2. Ajouter le port dans `local/nginx.conf`

```nginx
location /mon-projet/api/ {
    proxy_pass http://host.docker.internal:PORT/api/;
    # ...headers...
}

location /mon-projet/ {
    proxy_pass http://host.docker.internal:FRONTEND_PORT;
    # ...headers...
}
```

### 3. Créer le répertoire web sur le VPS

```bash
sudo mkdir -p /var/www/mon-projet
sudo chown www-data:www-data /var/www/mon-projet
```

### 4. Tester et déployer

```bash
sudo nginx -t          # vérifier la syntaxe
sudo systemctl reload nginx  # appliquer
```

Ou pusher sur `main` et laisser Jenkins faire le déploiement automatiquement.

## Commandes utiles

```bash
# Tester la configuration
sudo nginx -t

# Recharger sans interruption de service
sudo systemctl reload nginx

# Redémarrer complètement
sudo systemctl restart nginx

# Statut
sudo systemctl status nginx

# Logs d'accès en temps réel
sudo tail -f /var/log/nginx/access.log

# Logs d'erreur
sudo tail -f /var/log/nginx/error.log

# Renouveler les certificats SSL (dry run)
sudo certbot renew --dry-run

# Renouveler les certificats SSL
sudo certbot renew
```

## Troubleshooting

| Symptôme | Cause probable | Solution |
|---|---|---|
| `502 Bad Gateway` | Service upstream non démarré | Vérifier que le container Docker ou le process backend est actif |
| `404` sur URL SPA | `try_files` ou `alias` mal configuré | Vérifier que le fallback vers `index.html` est correct |
| `ERR_TOO_MANY_REDIRECTS` | Boucle de redirect | Vérifier `proxy_pass` avec/sans trailing slash |
| Certificat expiré | Renouvellement auto raté | `sudo certbot renew` |
| `403 Forbidden` | Permissions `/var/www/` | `sudo chown -R www-data:www-data /var/www/projet/` |
| HMR WebSocket KO en local | nginx ne proxifie pas le WebSocket | Ajouter `proxy_http_version 1.1` + `Upgrade` headers |
| Config invalide Jenkins | Erreur syntaxe nginx | Corriger l'erreur affichée par `nginx -t` |
