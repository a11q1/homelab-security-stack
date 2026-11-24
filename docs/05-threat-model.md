# 05 — Threat model

[← Matrice de flux](04-matrice-de-flux.md) · [README](../README.md) · [WAF →](06-waf-bunkerweb.md)

## Méthode

Pas de STRIDE complet en 40 pages. Un modèle utile pour un lab :

1. lister ce qui a de la valeur ;
2. lister qui attaque, avec quels moyens ;
3. décrire des scénarios concrets ;
4. pour chacun : ce qui protège, ce qui reste ouvert ;
5. classer ce qui reste.

Le modèle est revu après chaque incident ou ajout majeur. Il est daté :
**24 novembre 2026**.

## Les actifs

| # | Actif | Pourquoi c'est sensible | Criticité |
|---|---|---|---|
| A1 | **44 sites clients** (`sitecreator.shop`) | réputation, données de contact, accès aux boîtes mail | haute |
| A2 | **Hyperviseur Proxmox** | compromission = tous les CT, toutes les sauvegardes | critique |
| A3 | **Secrets d'infra** | tokens de tunnel, clés PKI, tokens API, `.env` applicatifs | critique |
| A4 | **Données communautaires** (Cogtopia) | comptes joueurs, économie, tickets | moyenne |
| A5 | **Disponibilité** | jeu 24/7, sites clients, bots | haute |
| A6 | **Sauvegardes** | si chiffrées par un rançongiciel, plus de rollback | haute |

## Les adversaires

| # | Profil | Moyens | Objectif |
|---|---|---|---|
| T1 | **Scanner opportuniste** | Shodan, nuclei, wordlists | trouver une porte ouverte, deface, crypto-miner |
| T2 | **Attaquant ciblé** | OSINT, phishing, exploitation d'app | accès, persistance, données |
| T3 | **Site client compromis** | upload PHP, plugin vulnérable, mot de passe faible | rebond vers l'infra |
| T4 | **Erreur interne** | mauvaise règle, mauvais bind, secret commité | outage, fuite |
| T5 | **Compte Cloudflare compromis** | token API, session | détourner le trafic |

T4 est le plus probable. T3 est le plus sous-estimé : un site client est du code
qu'on n'a pas écrit, exécuté sur le même réseau que l'infra.

## Scénarios

### S1 — Compromission d'un site client

**Chemin** : un CMS maison sur CT147 → upload ou inclusion → exécution PHP →
lecture de `wp-config` / `.env` → reconnaissance réseau → tentative vers
`22.22.22.0/24`, `:8006`, la DMZ.

**Protections** :

- DMZ → LAN / INT / PROD en `DROP` (hérité de `vmbr4`, conservé).
- `REFONTE` limite la DMZ aux backends nominatifs et à Internet:443/53.
- Les ports admin de l'hôte sont fermés depuis `vmbr4`.
- Le WAF bloque les payloads HTTP connus **avant** CT147.

**Limites** :

- Le code PHP tourne avec les droits de son pool ; le WAF ne voit pas une
  requête légitime qui exploite une faille logique.
- CT147 → autres CT de la DMZ est bloqué sauf ACCEPT nominatifs, mais CT147
  garde un accès sortant Internet (mises à jour, mail).
- Pas de détection d'exécution (pas d'EDR, pas d'auditd centralisé).

### S2 — Vol d'un secret

**Chemins réels constatés** :

1. Un token de tunnel visible dans `ps` sur un connecteur (ligne de commande
   `--token`). N'importe quel processus du conteneur peut le lire.
2. Une clé DKIM en clair dans un export DNS sauvegardé.
3. Des `.env` applicatifs dans des répertoires de projet, sans chiffrement.
4. Un token API BunkerWeb dans un `env_file` en clair sur disque.

**Protections** :

- Aucun secret dans ce dépôt, `.gitleaks` en preflight.
- **SOPS + age** : tokens Cloudflare, token API BunkerWeb et mot de passe de la
  base sont chiffrés dans `refonte-secrets.enc.yaml`, hors git.
- **`--token-file`** en `600` pour les trois connecteurs (CT132, CT116, CT111).
  Plus de token dans la ligne de commande.
- CT111 et CT116 sont arrêtés : leurs tunnels n'étaient plus utilisés.
- La PKI est sauvegardée dans `pki.tgz.age` (archive chiffrée).
- Access devant les consoles qui exposeraient des secrets (Proxmox, checkmk).

**Limites** :

- La clé age est sur CT131, en `600` : elle doit être sauvegardée hors ligne.
- Pas de rotation automatique ni de coffre central.
- Les `.env` applicatifs des autres CT ne sont pas tous passés sous SOPS.
- Les tokens ont été sortis de `ps`, pas nécessairement rotationnés.

### S3 — Exposition d'une console admin

**Avant** : `proxmox.*.net`, `checkmk`, `diagrams`, `admin.hackaton`,
`hackaton-admin` répondaient sans SSO. Le `.com` de Proxmox était protégé,
le `.net` non — deux noms, deux politiques, un trou.

**Protections** :

- Cloudflare Access sur les 5 hostnames admin ajoutés, plus correction de
  `proxmox.example.net` qui n'avait aucune politique.
- Access est évalué **avant** le tunnel : la requête n'atteint jamais l'origine.
- Le WAF reste derrière, en défense en profondeur.
- `admin.sitecreator.shop` est un honeypot assumé : 200, pas d'auth, logs.

**Limites** :

- Access dépend de Cloudflare : une erreur de policy ouvre tout.
- Une session Access volée (cookie) vaut authentification.
- Proxmox n'a pas encore de TOTP applicatif : Access est le seul second facteur.
- Les hostnames `.net` qui dupliquent les `.com` restent une dette.

### S4 — SSRF

**Avant** : `https://beta.sitecreator.shop/node/8191/` atteignait FlareSolverr,
`/node/55432/` Postgres, `/node/56379/` Redis. Un proxy ouvert vers tous les
ports locaux, avec `autoindex on`.

**Protections** :

- La route `/node/<port>/` a été supprimée : **404**.
- Le WAF bloque les tentatives d'accès à des chemins internes.
- La DMZ n'a plus d'accès libre vers l'INT/PROD.

**Limites** :

- Un SSRF applicatif légitime (webhook, import d'URL) reste possible dans une
  app mal codée ; le WAF ne comprend pas la sémantique métier.
- Aucun scan automatisé de régression SSRF dans la CI.

### S5 — Mouvement latéral

**Avant** : `FORWARD policy ACCEPT` entre vmbr1, vmbr2, vmbr3 et vmbr5.
Un CTF compromis atteignait tous les autres conteneurs et l'hôte.

**Protections** :

- `REFONTE` en tête de `FORWARD` : l'edge et les connecteurs sont limités à
  leurs flux nominatifs, la DMZ est isolée.
- `REFONTE-IN` ferme les ports admin de l'hôte depuis vmbr1/2/3/5.
- Le conflit d'IP `.66` est résolu : les flux sont de nouveau déterministes.

**Limites** :

- `vmbr1 ↔ vmbr2 ↔ vmbr3` reste en `ACCEPT`. **La latéralisation est toujours
  possible entre ces trois zones.**
- Docker ajoute ses propres règles `FORWARD` par conteneur.
- WireGuard et Tailscale sont en accept-all.
- Pas de microsegmentation par application.

### S6 — Détournement de tunnel

**Chemin** : vol d'un token `cloudflared` → un attaquant crée un connecteur
avec le même tunnel → il reçoit une partie du trafic (load balancing entre
connecteurs) ou peut modifier les ingress si le token le permet.

**Protections** :

- Tokens hors git, permissions 600.
- Un seul connecteur par tunnel pour les flux critiques (pas de course).
- Access sur les hostnames sensibles : même avec le tunnel, il faut le SSO.
- Surveillance des connexions `cloudflared` (nombre de connecteurs, `ready`).

**Limites** :

- Pas de rotation planifiée des tokens.
- Le tunnel orphelin de CT116 (autre compte) est arrêté, et son token a été
  sorti de la ligne de commande. La rotation reste à confirmer.
- Cloudflare reste un fournisseur de confiance : un compte compromis = trafic
  détourné, quelles que soient les règles locales.

### S7 — Déni de service applicatif

**Chemin** : requêtes massives sur un vhost, formulaire coûteux, upload géant.

**Protections** :

- `LIMIT_REQ_RATE=20r/s` par client réel.
- `MAX_CLIENT_SIZE=50m` (aligné sur les besoins draw.io / rapports).
- Cloudflare absorbe une partie du volume et son WAF managé filtre le bruit.

**Limites** :

- 20 r/s est une valeur globale, pas par vhost ni par route.
- Pas de quota par IP au niveau applicatif.
- Un attaquant distribué passe sous le seuil.

## Matrice de risque résiduelle

| Risque | Avant | Après | Priorité |
|---|---|---|---|
| Hyperviseur exposé sans SSO | critique | **traité** (Access) | — |
| SSRF `/node/` | critique | **traité** (404) | — |
| Conflit IP | élevé | **traité** | — |
| Watchdog qui annule la refonte | élevé | **traité** | — |
| Admin sans SSO | élevé | **traité** sur le périmètre testé | — |
| Token de tunnel dans `ps` | élevé | **traité** (`--token-file`, CT111/116 arrêtés) | — |
| Secrets en clair sur disque | élevé | **partiel** (SOPS + age sur l'edge) | P2 |
| Latéralisation INT/PROD/CTF | critique | **ouvert** | P1 |
| Rotation des tokens | moyen | **ouvert** | P2 |
| Péri-CT Proxmox non effectif | moyen | **ouvert** | P2 |
| TOTP Proxmox absent | moyen | **ouvert** | P1 |
| `verify upstream` désactivé | moyen | **ouvert** | P2 |
| Sauvegardes non testées | élevé | **hors périmètre** | P1 |

## Ce que ce modèle ne couvre pas

- L'ingénierie sociale et le phishing.
- La sécurité physique et le vol de matériel.
- Les vulnérabilités zero-day des composants.
- La compromission du compte Cloudflare (partiellement traitée par Access).
- La sécurité du poste de travail de l'admin.
- Les sauvegardes : chiffrement, intégrité, restauration testée.
- Le code applicatif des 44 sites clients.

Un threat model honnête dit aussi ce qu'il ignore.

## Suites

- [06 — WAF BunkerWeb](06-waf-bunkerweb.md)
- [09 — Accès admin](09-acces-admin-cloudflare.md)
- [13 — Leçons et limites](13-lecons-et-limites.md)
