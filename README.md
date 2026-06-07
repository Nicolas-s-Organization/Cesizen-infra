# Cesizen-infra

Infrastructure et déploiement du projet **CESIZen** sur VM Azure. Deux environnements (**staging** et **prod**) cohabitent sur la même VM derrière **Traefik** (HTTPS Let's Encrypt). Les mises à jour sont **pull-based via Watchtower** (pas de SSH depuis CI).

## Architecture

```
Internet ─HTTPS:443─> Traefik ┬─ Host(cesizen-staging.duckdns.org) ─┬─ /api/* → api-staging:3000
                              │                                    └─       → web-staging:8080
                              │
                              ├─ Host(cesizen-prod.duckdns.org)    ─┬─ /api/* → api-prod:3000
                              │                                    └─       → web-prod:8080
                              │
                              └─ Host(cesizen-pgadmin.duckdns.org) ─→ pgadmin (staging uniquement)

Réseaux Docker :
  web                    : public  → Traefik + tous les services HTTP
  cesizen-staging        : internal → api-staging ↔ postgres-staging (pas d'Internet)
  cesizen-prod           : internal → api-prod    ↔ postgres-prod    (pas d'Internet)

Watchtower scrute GHCR toutes les 60s :
  cesizen-api:develop / cesizen-app-web:develop   → staging
  cesizen-api:latest  / cesizen-app-web:latest    → prod
```

## Structure du repo

```
Cesizen-infra/
├── traefik/
│   ├── docker-compose.yml         # stack infra : Traefik + Watchtower
│   ├── .env.example
│   └── letsencrypt/               # certs persistés (gitignored)
├── compose/
│   ├── docker-compose.staging.yml # stack STAGING (db + api + web + pgAdmin)
│   ├── docker-compose.prod.yml    # stack PROD    (db + api + web)
│   ├── .env.staging.example
│   └── .env.prod.example
├── .gitignore
└── README.md
```

## Pré-requis sur la VM

- Ubuntu 22.04 LTS ou +, Docker Engine + plugin Compose
- Ports **80** et **443** ouverts (UFW *et* NSG Azure)
- DuckDNS : 2 labels au minimum (`cesizen-staging`, `cesizen-prod`) + 1 optionnel (`cesizen-pgadmin`), tous pointant vers l'IP publique de la VM
- PAT GitHub avec scope `read:packages` (pour pull les images GHCR privées)

## Bootstrap (premier déploiement sur la VM)

```bash
# ── 1. Cloner ce repo sur la VM ────────────────────────────────────────────
git clone https://github.com/Nicolas-s-Organization/Cesizen-infra.git
cd Cesizen-infra

# ── 2. Créer le réseau Docker partagé "web" (une seule fois) ───────────────
sudo docker network create web

# ── 3. Login GHCR (crée /root/.docker/config.json pour Watchtower) ─────────
echo $GHCR_PAT | sudo docker login ghcr.io -u <ton-user-github> --password-stdin

# ── 4. Lancer la stack INFRA (Traefik + Watchtower) ────────────────────────
cp traefik/.env.example traefik/.env
nano traefik/.env                                       # remplir LETSENCRYPT_EMAIL
sudo docker compose -f traefik/docker-compose.yml --env-file traefik/.env up -d

# ── 5. Lancer la stack STAGING ─────────────────────────────────────────────
cp compose/.env.staging.example compose/.env.staging
nano compose/.env.staging                               # remplir POSTGRES_*, ACCESS/REFRESH_TOKEN_SECRET, etc.
sudo docker compose -f compose/docker-compose.staging.yml --env-file compose/.env.staging up -d

# ── 6. Lancer la stack PROD ────────────────────────────────────────────────
cp compose/.env.prod.example compose/.env.prod
nano compose/.env.prod                                  # secrets DIFFÉRENTS du staging
sudo docker compose -f compose/docker-compose.prod.yml --env-file compose/.env.prod up -d

# ── 7. Suivre les logs Traefik pour valider l'émission des certificats ─────
sudo docker compose -f traefik/docker-compose.yml logs -f traefik
```

Une fois Traefik a obtenu les certs (~30-60 s pour chaque domaine), ouvre dans un navigateur :
- `https://cesizen-staging.duckdns.org/login` → SPA staging
- `https://cesizen-prod.duckdns.org/login` → SPA prod
- `https://cesizen-pgadmin.duckdns.org/` → pgAdmin (si activé)

## Mises à jour

**Automatiques** via Watchtower (scrute GHCR toutes les 60 s) :
- Tout push CI sur `develop` → image `:develop` mise à jour sur GHCR → Watchtower pull et restart `api-staging` / `web-staging`.
- Création d'un tag git `v*` (`v1.2.3`) sur main → CI produit `:latest` → Watchtower pull et restart `api-prod` / `web-prod`.

**Manuelles** sur la VM :
```bash
sudo docker compose -f compose/docker-compose.<env>.yml pull       # force le pull
sudo docker compose -f compose/docker-compose.<env>.yml up -d      # restart si l'image a changé
```

## Troubleshooting

**Traefik n'émet pas le certificat** :
- Vérifier que les ports 80/443 sont accessibles depuis Internet (`curl -v http://<DOMAIN>`).
- Logs : `sudo docker compose -f traefik/docker-compose.yml logs traefik`.
- Le challenge TLS-ALPN-01 nécessite que rien d'autre n'écoute sur le port 443.

**Watchtower ne pull pas** :
- Vérifier `/root/.docker/config.json` contient l'auth GHCR : `sudo cat /root/.docker/config.json`.
- Logs : `sudo docker compose -f traefik/docker-compose.yml logs watchtower`.

**L'API ne démarre pas / erreurs Prisma migrate** :
- Vérifier que la DB est `healthy` : `sudo docker compose -f compose/docker-compose.<env>.yml ps`.
- Les migrations sont appliquées automatiquement au démarrage (entrypoint API).
