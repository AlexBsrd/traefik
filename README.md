# Traefik

Reverse proxy pour tous les sites hébergés sur ce Raspberry (*.bsrd.fr et alentours). Gère le TLS via Let's Encrypt automatiquement.

## Démarrage

```bash
docker network create proxy   # une seule fois
docker compose up -d
```

## Config

- **Statique** (flags `command:` dans `docker-compose.yml`) — entryPoints, providers, certResolver. Modifier nécessite un redémarrage du container.
- **Dynamique** (`dynamic/*.yml`) — routers/services/middlewares déclarés en fichier. Le dossier est watché, les modifs sont prises en compte à chaud.
- **Labels Docker** — la plupart des sites se déclarent eux-mêmes via des labels sur leur service (voir n'importe quel autre projet dans `~/dev/`).

## Volumes

- `traefik-letsencrypt` (named) — contient `acme.json`, les certificats Let's Encrypt. Ne pas supprimer.
- `./dynamic` — monté en lecture seule dans le container.

## Réseau

Tous les services exposés doivent être sur le réseau Docker externe `proxy`.
