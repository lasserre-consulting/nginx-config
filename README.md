# nginx-config

Configuration nginx pour le VPS `lasserre-consulting.fr`. Gère le reverse proxy HTTPS pour tous les projets hébergés, ainsi qu'un environnement nginx local Docker pour le développement.

## Convention des ports — à respecter sur VPS et en local

> **Important** : toujours respecter cette convention lors de l'ajout ou la modification d'un service, que ce soit sur le VPS ou en développement local.

### Règles générales

- Tout port Docker doit être bindé sur `127.0.0.1` (jamais `0.0.0.0`) — nginx est le seul point d'entrée public
- Les bases de données n'exposent **pas** de port hôte sauf besoin explicite (accès client DB en dev)
- Chaque nouveau projet réserve un bloc de ports selon les ranges ci-dessous

### Ranges

| Range | Usage |
|---|---|
| `3000–3099` | Apps Node / Next.js |
| `4200–4299` | Frontends Angular (containers, accès local uniquement) |
| `5432–5439` | PostgreSQL |
| `8080–8089` | Backends Java (Spring Boot) |
| `8090–8099` | Infrastructure (Jenkins, outils internes) |
| `9092`, `2181` | Kafka / Zookeeper (carnetroute) |

### Ports alloués

| Port | Service | Projet | Binding |
|---|---|---|---|
| `22` | SSH | système | public |
| `80` | nginx HTTP | système | public |
| `443` | nginx HTTPS | système | public |
| `3000` | knido-app | knido | `127.0.0.1` |
| `3001` | entrevia-app | entrevia | `127.0.0.1` |
| `4200` | lasserre-consulting-site (frontend) | lasserre-consulting-site | `127.0.0.1` |
| `4201` | mt-frontend | mission-tracker | `127.0.0.1` |
| `4202` | carnetroute-frontend | carnetroute | `127.0.0.1` |
| `4203` | qualidoc-frontend | qualidoc | `127.0.0.1` |
| `5432` | knido-postgres | knido | interne Docker |
| `5433` | mt-postgres | mission-tracker | `127.0.0.1` |
| `5434` | qualidoc-postgres | qualidoc | interne Docker |
| `5435` | entrevia-postgres | entrevia | interne Docker |
| `8080` | carnetroute-backend | carnetroute | `127.0.0.1` |
| `8081` | mt-backend | mission-tracker | `127.0.0.1` |
| `8082` | qualidoc-backend | qualidoc | `127.0.0.1` |
| `8083` | carnetroute-kafka-ui | carnetroute | `127.0.0.1` |
| `8090` | Jenkins | infrastructure | `127.0.0.1` |
| `9092` | Kafka | carnetroute | `127.0.0.1` |
| `2181` | Zookeeper | carnetroute | `127.0.0.1` |

> Prochain projet Node/Next.js : `3002`. Prochain backend Java : `8084`. Prochaine DB Postgres : `5436`.

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
| `/entrevia/` | Proxy | `localhost:3001/entrevia/` | Next.js fullstack |

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
| qualidoc | `4203` (container) | `8082` (docker) | `http://localhost/qualidoc/` |
| mission-tracker | `4201` (container) | `8081` (docker) | `http://localhost/mission-tracker/` |
| carnetroute | `4202` (container) | `8080` (docker) | `http://localhost/carnetroute/` |
| knido | — | `3000` (docker) | `http://localhost/` |
| entrevia | — | `3001` (docker) | `http://localhost/entrevia/` |

## Pipeline Jenkins

Le `Jenkinsfile` exécute :

1. **Checkout** — `git pull origin main` sur le VPS
2. **Test config** — `sudo nginx -t` (valide la syntaxe)
3. **Reload** — `sudo systemctl reload nginx` (rechargement sans interruption)

Le pipeline échoue si `nginx -t` retourne une erreur, empêchant le déploiement d'une config invalide.

## Ajouter un nouveau projet

### 1. Choisir les ports selon la convention

Consulter le tableau "Ports alloués" ci-dessus et prendre les prochains disponibles dans chaque range.

### 2. Ajouter les blocs dans `sites-available/lasserre-consulting`

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

### 3. Dans le `docker-compose.prod.yml` du projet

Toujours binder sur `127.0.0.1` :

```yaml
ports:
  - "127.0.0.1:PORT:PORT_INTERNE"
```

### 4. Créer le répertoire web sur le VPS

```bash
sudo mkdir -p /var/www/mon-projet
sudo chown www-data:www-data /var/www/mon-projet
```

### 5. Tester et déployer

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Ou pusher sur `main` et laisser Jenkins faire le déploiement automatiquement.

### 6. Mettre à jour ce README

Ajouter le nouveau service dans le tableau "Ports alloués" et mettre à jour la ligne "Prochain port disponible".

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
| Port déjà utilisé | Conflit de port | Consulter le tableau "Ports alloués" dans ce README |
