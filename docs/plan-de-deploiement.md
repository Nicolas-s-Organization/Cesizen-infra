# Plan de déploiement — CESIZen

Document de référence décrivant **comment, où et avec quoi** l'application CESIZen est déployée. Couvre les environnements, l'infrastructure cible, le versioning, le pipeline CI/CD, la stratégie de release et de rollback, ainsi que les responsabilités associées.

---

## 1. Contexte projet

CESIZen est une application web de prévention en santé mentale (gestion du stress, suivi d'émotions, articles de sensibilisation). Le projet est porté en simulation pour le Ministère de la Santé et de la Prévention.

**Architecture applicative :**
- API Node.js / Express 5 / Prisma 7 / Postgres 16
- SPA React 19 buildée par Vite, servie en statique par nginx
- Reverse-proxy unifié via Traefik v3.7 avec terminaison TLS (Let's Encrypt)

Le périmètre du présent plan correspond au **bloc 3 CDA — Déployer et sécuriser** : déploiement reproductible, séparation des environnements, automatisation maximale, traçabilité des releases.

---

## 2. Environnements

Le projet est déployé sur **2 environnements distincts** hébergés sur la même VM Azure mais isolés au niveau Docker (réseaux, bases, secrets, images).

| Environnement | Domaine | Image GHCR | Branche / Tag source | Déploiement |
|---|---|---|---|---|
| **Staging** | `cesizen-staging.duckdns.org` | `:develop` | merge sur `develop` | automatique (Watchtower, poll 60 s) |
| **Production** | `cesizen-production.duckdns.org` | `:latest`, `:vX.Y.Z` | tag git `v*` depuis `main` | automatique (Watchtower, poll 60 s) |

### Caractéristiques

- **Base de données isolée par environnement** : chaque stack a son propre volume Postgres et son propre réseau Docker `internal`. Aucune connexion possible entre les deux.
- **Secrets isolés** : `traefik/.env`, `compose/.env.staging`, `compose/.env.prod` ; le contenu de prod n'est jamais en clair dans le repo (gitignored).
- **Pas d'environnement local mutualisé** : chaque développeur dispose de son propre `docker-compose.yml` à la racine de `Cesizen-api/` pour le développement (Postgres + API, port 3001 vers 3000).

---

## 3. Infrastructure cible

### VM Azure

| Caractéristique | Valeur |
|---|---|
| Nom | `devops-cesizen` |
| Resource group | `cesi` |
| Région | UAE North |
| OS | Ubuntu 24.04 LTS |
| IP publique | `74.162.35.99` |
| FQDN Azure | `cesizen.uaenorth.cloudapp.azure.com` |
| Accès SSH | par clé uniquement, alias `ssh devops-cesizen` |

### Composants déployés sur la VM

```
┌─ traefik/docker-compose.yml ─────────────────────────────────┐
│   • traefik v3.7   (reverse proxy + TLS Let's Encrypt)       │
│   • watchtower 1.7 (poll GHCR, deploy auto, notif Discord)   │
└──────────────────────────────────────────────────────────────┘

┌─ compose/docker-compose.staging.yml ─────────────────────────┐
│   • postgres 16-alpine  (réseau internal — isolé Internet)   │
│   • cesizen-api-staging (image :develop, port 3000 interne)  │
│   • cesizen-web-staging (image :develop, port 8080 interne)  │
└──────────────────────────────────────────────────────────────┘

┌─ compose/docker-compose.prod.yml ────────────────────────────┐
│   • postgres 16-alpine  (volume + réseau internal séparés)   │
│   • cesizen-api-prod    (image :latest, port 3000 interne)   │
│   • cesizen-web-prod    (image :latest, port 8080 interne)   │
└──────────────────────────────────────────────────────────────┘
```

### Services externes

| Service | Rôle | Coût |
|---|---|---|
| GitHub | Hébergement code, CI (Actions), registry images (GHCR) | Gratuit (org publique) |
| Let's Encrypt | Autorité de certification TLS | Gratuit |
| DuckDNS | DNS dynamique (2 sous-domaines) | Gratuit |
| Discord | Webhook de notification de déploiement | Gratuit |
| Azure | Hébergement VM | Crédits étudiants |

---

## 4. Stratégie de versioning

### Branches Git

| Branche / pattern | Rôle | Cible de PR | Protection |
|---|---|---|---|
| `main` | Production stable | — | Push direct interdit, PR + CI verte obligatoire |
| `develop` | Intégration staging | `main` (release) | Push direct interdit, PR + CI verte obligatoire |
| `feat/<slug>` | Nouvelle fonctionnalité | `develop` | Libre |
| `fix/<slug>` | Correction de bug | `develop` (ou `main` pour hotfix) | Libre |
| `chore/<slug>` | Outillage / dette technique | `develop` | Libre |

### Versions semver sur les images

Chaque release prod crée un **tag git** au format `vMAJOR.MINOR.PATCH`. La CI publie alors plusieurs tags d'image sur GHCR :

- `:latest` — pointe sur la dernière prod
- `:0.3.0` — version exacte
- `:0.3` — version mineure
- `:0` — version majeure
- `:sha-abc123` — SHA court immuable du commit

Cela permet, le cas échéant, de :
- Re-déployer une version précise (`:0.3.0`)
- Pin une version mineure dans un compose (`:0.3` → toutes les patchs)
- Garantir la traçabilité immuable (`:sha-...`)

---

## 5. Pipeline CI

### Architecture des workflows

| Repo | Workflow | Déclencheurs |
|---|---|---|
| `Cesizen-api` | `.github/workflows/ci-back.yml` | push `main`, push `develop`, push tag `v*`, PR vers `main`/`develop` |
| `Cesizen-app-web` | `.github/workflows/ci-front.yml` | push `main`, push `develop`, push tag `v*`, PR vers `main`/`develop` |
| `Cesizen-infra` | — | pas de CI (compose statique versionné) |

### Jobs CI Backend (parallèles)

| Job | Outil | Rôle | Bloque le merge ? |
|---|---|---|---|
| `test` | Vitest | 55 tests unitaires (Prisma mocké) | ✅ |
| `typecheck` | `tsc --noEmit` | Vérification TypeScript strict | ✅ |
| `audit` | `npm audit --audit-level=critical` | Vulnérabilités des dépendances | ✅ |
| `gitleaks` | `gitleaks` (image docker) | Scan secrets dans tout l'historique git | ✅ |
| `build-push` | `docker/buildx`, Trivy, GHCR | Build + scan CRITICAL + push si OK | ✅ |

### Jobs CI Frontend (parallèles)

| Job | Outil | Rôle |
|---|---|---|
| `lint` | ESLint | Analyse statique |
| `build` | `tsc -b` + `vite build` | Build production (vérifie types et bundle) |
| `e2e` | Selenium WebDriver (Chrome headless) | Tests E2E sur le parcours `/login` |
| `build-push` | `docker/buildx`, Trivy, GHCR | Build + scan + push (avec `--build-arg VITE_API_URL=/api`) |

### Sécurité shift-left dans le job `build-push`

Étape par étape pour démontrer le « scan avant push » :

```
1. Build image LOCALEMENT (load: true, push: false)
2. Trivy SARIF (HIGH+CRITICAL, exit 0) → upload onglet Security GitHub
3. Trivy gate strict (CRITICAL --ignore-unfixed, exit 1)
   → si KO : pipeline rouge, image NON poussée
4. Push GHCR conditionnel :
   - push: false sur PR
   - push: true sur push develop/main/tag v*
```

**Conséquence** : aucune image vulnérable ne peut atteindre la registry, et aucune PR avec image vulnérable ne peut être mergée.

---

## 6. Pipeline CD

### Stratégie : déploiement **pull-based**

Aucune connexion entrante depuis GitHub Actions vers la VM. La VM va elle-même chercher les nouvelles images sur GHCR.

```
GitHub Actions ──push image──▶ GHCR ◄──pull──── Watchtower (sur la VM)
                                                       │
                                                       ▼
                                              docker compose
                                              recreate container
                                                       │
                                                       ▼
                                              Discord notification
```

**Justification** : le risque connu de l'action Appleboy (SSH-from-CI) est éliminé. Si la CI est compromise, l'attaquant ne possède pas de clé d'accès à la VM.

### Composants côté VM

| Composant | Rôle | Configuration |
|---|---|---|
| Watchtower 1.7.1 | Poll GHCR (60 s), pull images, recreate containers | Labels `com.centurylinklabs.watchtower.enable=true` sur les services concernés |
| Authentification GHCR | PAT scope `read:packages` dans `/root/.docker/config.json` | Monté en read-only dans Watchtower |
| Notifications | Webhook Discord via shoutrrr | `WATCHTOWER_NOTIFICATIONS=shoutrrr`, `WATCHTOWER_NOTIFICATION_REPORT=true` |

### Déclenchement automatique

- **Staging** : tout merge sur `develop` → CI → image `:develop` → Watchtower → `cesizen-{api,web}-staging` redémarrés
- **Production** : création d'une release GitHub (tag `vX.Y.Z` depuis `main`) → CI → images `:latest, :X.Y.Z, …` → Watchtower → `cesizen-{api,web}-prod` redémarrés

---

## 7. Stratégie de release production

### Pré-requis avant release

- [ ] Le code à publier est sur `develop`
- [ ] La staging tourne sans erreur depuis au moins 24 h
- [ ] Les éventuels bugs détectés en staging ont été corrigés
- [ ] La grille de versioning (semver) a été respectée (impact : MAJOR pour breaking change, MINOR pour nouvelle fonctionnalité, PATCH pour correction)

### Procédure

```
1. Côté code (api ET web) :
   - Ouvrir PR develop → main, titre : "release: vX.Y.Z"
   - CI doit être verte
   - Merger
2. Sur GitHub UI, créer une Release :
   - Target : main
   - Tag : vX.Y.Z (nouveau)
   - Title : "vX.Y.Z — <synthèse>"
   - Description : changelog des PRs incluses
   - Publish release → crée le tag git
3. La CI redémarre sur le tag, build et push GHCR
4. Watchtower détecte les nouvelles images :latest (60 s max)
5. Containers prod redémarrent
6. Notification Discord d'arrivée des updates
7. Smoke test manuel : curl https://cesizen-production.duckdns.org/api/health
```

### Schedule

Pas de rythme de release imposé. Une release par lot fonctionnel cohérent ou correctif critique.

---

## 8. Stratégie de rollback

### En cas d'incident détecté en prod après release

#### Option A — Rollback rapide via image GHCR antérieure

Si la précédente version d'image fonctionnait :

```bash
ssh devops-cesizen
cd ~/Cesizen-infra
# éditer compose/.env.prod pour figer une version précise
nano compose/.env.prod
  # API_TAG=v0.2.0   (au lieu de :latest)
  # WEB_TAG=v0.2.0
docker compose -f compose/docker-compose.prod.yml --env-file compose/.env.prod up -d
# Watchtower respectera ce tag et ne mettra plus à jour automatiquement vers :latest
```

Une fois le hotfix prêt :
- créer un tag `v0.3.1` (PATCH)
- remettre `API_TAG=latest` / `WEB_TAG=latest` dans `compose/.env.prod`
- `docker compose ... up -d`

#### Option B — Hotfix express via branche `fix/*`

Pour un correctif rapide sans rollback :

```
1. branche fix/<incident-slug> depuis main
2. corriger + PR vers main
3. CI verte → merge
4. créer release vX.Y.(Z+1)
5. CD auto via Watchtower
```

#### Option C — Restauration de la base de données

En cas de corruption / migration ratée :

```bash
docker compose -f compose/docker-compose.prod.yml --env-file compose/.env.prod stop api
# restauration du volume Postgres depuis backup (cf. annexe sauvegardes)
docker compose -f compose/docker-compose.prod.yml --env-file compose/.env.prod start api
```

### Garanties

- **Images immuables** : chaque tag GHCR est immuable, donc rejouer un déploiement antérieur garantit le même artefact que celui qui tournait
- **Données séparées du code** : le volume Postgres est isolé, un rollback applicatif ne touche pas aux données

---

## 9. Tests dans le pipeline

| Niveau | Outil | Couverture | Quand |
|---|---|---|---|
| Unit API | Vitest + Prisma mocké | 55 tests, services métier (category, emotion, trackerItem, user) | À chaque PR + push |
| Type-check | tsc strict | API + Web | À chaque PR + push |
| Lint front | ESLint | React/TS | À chaque PR + push |
| E2E front | Selenium WebDriver | Validation Zod login, toggle visibilité mot de passe | À chaque PR + push |
| Audit deps | `npm audit --audit-level=critical` | API + Web | À chaque PR + push |
| Secrets | gitleaks (sur tout l'historique) | API | À chaque PR + push |
| Image vulns | Trivy CRITICAL `--ignore-unfixed` | Image API + Image Web | Avant push GHCR |
| Smoke test | `curl /api/health`, `/health` | Prod après release | Manuel post-release |

---

## 10. Monitoring & supervision

| Type | Outil | Cible |
|---|---|---|
| Déploiements | Watchtower → Discord (shoutrrr) | Canal `#cesizen-monitoring` |
| Healthcheck container | Docker `HEALTHCHECK` natif | Container API (curl `/health` interne) |
| Logs container | `docker logs` | À la demande, sur la VM |
| Logs Traefik | access log activé | `docker logs traefik` |
| Certificats TLS | Watchtower renouvelle Traefik, LE envoie email d'expiration 20 j avant | `nicolas.descarp@outlook.fr` |
| Vulnérabilités images | Onglet Security GitHub (SARIF Trivy) | Mise à jour à chaque CI |

**Non couvert (assumé)** : agrégation centralisée des logs (Loki/Grafana, ELK). Volontairement écarté pour rester proportionné à la taille du projet et de la VM (4 Go RAM). À envisager en V2.

---

## 11. Rôles & responsabilités

Projet porté en solo (un seul développeur joue tous les rôles), mais le plan suivant décrit la matrice qui s'appliquerait en équipe :

| Rôle | Périmètre | Cas CESIZen |
|---|---|---|
| Développeur | Ouvre branches, code, écrit tests, ouvre PR | Nicolas Descarpentries |
| Reviewer | Valide la PR, exige tests verts, demande modifs | Nicolas Descarpentries (auto-review) |
| Mainteneur (release manager) | Crée les tags, publie les releases | Nicolas Descarpentries |
| Ops / on-call | Reçoit les notifs Discord, investigue incidents | Nicolas Descarpentries |

→ Dans un projet en équipe, le **reviewer doit être différent de l'auteur** (branch protection le force déjà côté GitHub si la règle « 1 review approuvée » est activée).

---

## 12. Synthèse — points forts du plan

- **2 environnements isolés** : staging et prod, mêmes mécanismes mais cycles de release différents
- **CI/CD entièrement automatisé** : aucune action manuelle entre le merge et le container en production
- **Sécurité shift-left** : Trivy bloque avant push, gitleaks scanne tout l'historique, branch protection bloque tout contournement
- **CD pull-based** : aucune surface d'attaque CI → infra
- **Traçabilité** : chaque image est tagguée par SHA immuable, chaque release a une note publique, chaque déploiement génère une notif Discord
- **Reproductibilité** : images Docker reproductibles, secrets externalisés, infrastructure documentée dans `Cesizen-infra/` versionné

---

## Annexes — fichiers liés

- [Architecture applicative](./architecture-applicative.md) — flux d'une requête
- [Architecture infrastructure](./architecture-infrastructure.md) — sécurité réseau & isolation
- [Cycle CI/CD](./cycle-cicd.md) — détail des étapes pipeline
- [Cycle de vie d'un ticket](./ticket-lifecycle.md) — process maintenance / anomalie
