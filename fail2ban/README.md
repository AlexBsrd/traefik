# fail2ban pour Traefik

Configuration fail2ban qui bannit automatiquement les IPs qui scannent
des paths connus de bots (WordPress, `.env`, `.git`, phpMyAdmin, etc.) dans
les access logs Traefik.

## Comment ça marche

- **Filter** (`filter.d/traefik-scrapers.conf`) : regex sur les logs JSON de
  Traefik (`/var/lib/docker/volumes/traefik-logs/_data/access.log`). Capture
  `ClientHost` et matche une liste de signatures de scan.
- **Action** (`action.d/docker-user.conf`) : crée une sous-chaîne iptables
  dédiée `f2b-<name>` et la hook en tête de `DOCKER-USER`. Les bans sont
  insérés dans cette sous-chaîne uniquement, pour cohabiter proprement avec
  `ufw-docker` (qui écrit aussi dans `DOCKER-USER`).
- **Jail** (`jail.d/traefik-scrapers.conf`) : 2 tentatives en 10 min → ban
  24h. Les bots scannent plusieurs paths d'un coup, 2 suffit.

## Installation

```bash
sudo ln -s ~/dev/traefik/fail2ban/filter.d/traefik-scrapers.conf /etc/fail2ban/filter.d/
sudo ln -s ~/dev/traefik/fail2ban/action.d/docker-user.conf      /etc/fail2ban/action.d/
sudo ln -s ~/dev/traefik/fail2ban/jail.d/traefik-scrapers.conf   /etc/fail2ban/jail.d/
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
```

## Vérification

```bash
# jail démarré et logs attrapés
sudo fail2ban-client status traefik-scrapers

# test de la regex sur le log actuel
sudo fail2ban-regex /var/lib/docker/volumes/traefik-logs/_data/access.log \
  /etc/fail2ban/filter.d/traefik-scrapers.conf

# chaîne iptables en place
sudo iptables -n -L DOCKER-USER | head
sudo iptables -n -L f2b-traefik-scrapers
```

## Gestion des bans

```bash
# unban une IP (faux positif)
sudo fail2ban-client set traefik-scrapers unbanip 1.2.3.4

# liste des IPs bannies
sudo fail2ban-client status traefik-scrapers
```

## Important : après avoir recréé Traefik

`ufw-docker allow traefik 80/tcp` insère un ACCEPT en tête de `DOCKER-USER`
et pousse le jump vers `f2b-traefik-scrapers` plus bas. Pour remettre le
jump en position 1 :

```bash
sudo systemctl restart fail2ban
```

(À faire en plus des commandes `ufw-docker allow traefik 80|443/tcp` déjà
documentées dans `~/dev/CLAUDE.md`.)

## Ajuster les patterns

Éditer `filter.d/traefik-scrapers.conf` puis :

```bash
sudo fail2ban-client reload traefik-scrapers
```

## Désinstallation

```bash
sudo fail2ban-client stop traefik-scrapers
sudo rm /etc/fail2ban/filter.d/traefik-scrapers.conf
sudo rm /etc/fail2ban/action.d/docker-user.conf
sudo rm /etc/fail2ban/jail.d/traefik-scrapers.conf
sudo systemctl restart fail2ban
```
