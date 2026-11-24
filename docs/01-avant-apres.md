# 01 — Avant / après

[← Contexte](00-contexte.md) · [README](../README.md) · [Architecture cible →](02-architecture-cible.md)

## La bascule en un tableau

| Sujet | Avant | Après |
|---|---|---|
| **Ingress HTTP** | connecteurs dispersés sur 4 CT (2 morts), backends joints en direct | connecteurs → WAF → Caddy → backends |
| **Tunnels** | 7 IDs distincts, 3 connecteurs sans config sauvegardée | 7 IDs : 2 pour le web clients/communautés, 1 Minecraft, 1 inutilisé |
| **Chiffrement** | HTTP clair de bout en bout après Cloudflare | TLS interne PKI sur le hop WAF → Caddy |
| **WAF** | aucun | BunkerWeb 1.6.18, CRS, 68 vhosts |
| **Origines admin** | `admin.*`, `checkmk`, `diagrams`, CTF accessibles | Access (302) ou honeypot assumé |
| **Segmentation** | `FORWARD ACCEPT` sauf vmbr4 | allowlist `REFONTE` + isolation DMZ |
| **Hôte Proxmox** | `:22/:8006/:3128` depuis tous les CT | LAN / WG / Tailscale / CT131 uniquement |
| **SSRF** | `/node/<port>/` proxifiait 8191, 55432, 56379 | **404** |
| **Conflit IP** | `.66` sur CT138 et CT141 | CT141 → `.67` |
| **Watchdog** | réappliquait les ingress vers CT147 | pointe vers le WAF, élague les morts |
| **Supervision** | 5/14 backends joignables depuis la DMZ | allowlist explicite par (IP, port) |

## Les 8 problèmes de départ

### 1. Sept tunnels, aucune vue d'ensemble

Six configurations sauvegardées, un septième tunnel actif dans le DNS sans
fichier de config. Trois connecteurs (`CT116`, `CT111`, `CT133`) dont le tunnel
n'existe dans aucun backup. Deux sont morts (503, zéro connexion) et personne
ne le sait parce que le DNS pointe encore dessus.

**Ce que ça coûte** : impossible de savoir quel hostname arrive où sans recouper
DNS, `cloudflared tunnel list` et les fichiers JSON à la main.

### 2. Origines en HTTP clair

Cloudflare termine le TLS public, puis le connecteur parle en `http://` au
backend. Sur le réseau interne, n'importe quel conteneur peut lire ou modifier
le trafic entre le connecteur et l'origine.

### 3. Pas de WAF

Aucune inspection L7. Un scanner qui trouve `?q=<script>` obtient un 200.
Les CMS maison, les formulaires de contact et les vhosts clients sont exposés
tels quels.

### 4. Le SSRF `/node/<port>/`

Dans le `default_server` de nginx (CT147 et CT132) :

```nginx
location ~ ^/node/([0-9]+)/(.*)$ {
    proxy_pass http://127.0.0.1:$1/$2;
    autoindex on;
}
```

Résultat : `https://beta.sitecreator.shop/node/8191/` atteint FlareSolverr,
`/node/55432/` atteint Postgres, `/node/56379/` atteint Redis. Un proxy ouvert
vers tous les ports locaux, depuis Internet.

### 5. Conflit d'IP `10.10.10.66`

CT138 (honeypot) et CT141 (CTF) déclarent la même adresse sur `vmbr1`, avec
deux MAC différentes. L'ARP répond alternativement. Les flux vers cette IP sont
non déterministes — et le diagnostic devient un cauchemar.

### 6. Un watchdog qui annule la refonte

`tunnel_watchdog.py`, cron toutes les 5 minutes, maintient les ingress vers
`http://10.20.30.147`. Toute tentative de faire passer le trafic par le WAF est
réécrite en moins de cinq minutes. **Le watchdog est une source de vérité
cachée** : il faut le patcher avant de basculer, pas après.

### 7. Un réseau plat

`FORWARD policy ACCEPT` entre vmbr1, vmbr2, vmbr3 et vmbr5. Seule la DMZ
(vmbr4) est partiellement filtrée. Un conteneur compromis — le CTF, le client
torrent, un bot Discord — atteint tout le reste, y compris `:8006` de
l'hyperviseur.

### 8. Des admins publics sans SSO

`proxmox.*.net`, `checkmk`, `diagrams`, `admin.hackaton`, `hackaton-admin` :
aucun `access.required` dans l'ingress. Le `.com` de Proxmox était protégé,
le `.net` non. Deux noms pour un service, deux politiques.

## Les 9 changements

| # | Changement | Effet |
|---|---|---|
| 1 | **CT155 BunkerWeb** dual-homed (vmbr5 ingress, vmbr4 egress) | un seul point d'inspection L7 |
| 2 | **CT156 Caddy** + PKI interne | routage Host + TLS interne |
| 3 | **Nouveau bridge vmbr5** `10.30.30.0/24` | transit edge séparé des backends |
| 4 | **Ingress repointés** vers `10.30.30.155:8080` | les tunnels ne touchent plus CT147 |
| 5 | **Chaîne `REFONTE`** en tête de `FORWARD` | allowlist edge + isolation DMZ |
| 6 | **Chaîne `REFONTE-IN`** en tête de `INPUT` | `:22/:8006/:3128` restreints |
| 7 | **Cloudflare Access** sur 5 hostnames admin | 302 sur le périmètre testé |
| 8 | **SSRF supprimé**, conflit IP résolu, CT116 routé puis arrêté | trois bugs qui pourrissaient le diagnostic |
| 9 | **Watchdog patché** | plus de réécriture silencieuse |

## Le durcissement après la bascule

La refonte réseau était finie vers 19:00. Trois chantiers ont suivi, parce
qu'une bascule qui marche n'est pas une bascule qu'on peut laisser tourner.

| Sujet | Avant | Après |
|---|---|---|
| **Persistance** | script firewall à rejouer à la main | `refonte-firewall.service` (`--strict`), activé, après `pve-firewall.service` |
| **Secrets** | `.env` en clair, tokens dans les unités | SOPS + age, `--token-file` en `600` |
| **Tokens cloudflared** | visibles dans `ps` (CT132, CT116, CT111) | sortis des lignes de commande ; CT111 et CT116 arrêtés |
| **SSH inter-CT** | ouvert entre tous les conteneurs | allowlist admin (CT131, LAN, WG, Tailscale), le reste `DROP` |
| **`REAL_IP_FROM`** | 7 IP dont honeypot et beta | `22.22.22.20 10.20.30.40` uniquement |

Détail : [08 — Firewall](08-firewall-segmentation.md#persistance),
[10 — Secrets](10-secrets-et-rotation.md#le-déploiement-sops--age).

## Ce qui n'a pas changé (et pourquoi)

| Élément | Raison |
|---|---|
| Cloudflare comme ingress | pas d'IP publique fixe, pas de port WAN ouvert (C2) |
| nginx mutualisé CT147 | 44 vhosts clients, migration trop risquée en une nuit |
| HTTP entre Caddy et les backends | intra-DMZ, accepté et documenté |
| Deux domaines publics | `sitecreator.shop` et `cogtopia.events` ont des usages distincts |
| playit.gg pour Minecraft | hors HTTP, protocole UDP, aucun WAF pertinent |
| Le honeypot `admin.sitecreator.shop` | piège assumé, il doit rester joignable |

## Chronologie de la nuit

Heures locales, approximatives. Elles disent surtout l'ordre des dépendances.

| Heure | Étape | Pourquoi cet ordre |
|---|---|---|
| 18:12 | Backup tunnels, DNS, interfaces | rollback avant tout |
| 18:15–18:35 | Audit lecture seule | constater, pas supposer |
| 18:19 | Démarrage CT155 / CT156 | construire hors chemin |
| 18:35 | `vmbr5` + adressage | l'edge a besoin d'un transit dédié |
| 18:50 | Patch du watchdog | sinon il annule la bascule |
| 18:52 | Repointage des ingress | le DNS reste l'interrupteur |
| 18:54 | Chaînes `REFONTE` / `REFONTE-IN` | fermer après avoir ouvert |
| 18:56 | Première campagne 65 vhosts | valider, pas espérer |
| 19:00 | Correctifs SSRF / IP / CT116 | les bugs révélés par les tests |
| 19:08 | `refonte-firewall.service --strict` | survivre au reboot |
| 19:09 | Chiffrement SOPS + age | sortir les secrets du disque en clair |
| 19:10 | Tokens cloudflared en `--token-file` | plus de token dans `ps` |
| 19:11 | Tests finaux (15 vhosts, SSH, hôte) | dernière preuve avant de figer |
| 20:00 | État figé + documentation | tant que c'est frais |

## Le vrai coût

Ce n'est pas le WAF ni la PKI. C'est **l'ordre** :

1. le watchdog avant les ingress ;
2. les `ACCEPT` avant les `DROP` dans une chaîne en tête de `FORWARD` ;
3. le DNS après que le backend répond ;
4. le test après chaque tunnel, pas à la fin.

Une règle au mauvais endroit ne casse pas une fois : elle casse jusqu'au
prochain redémarrage, silencieusement.

## Suites

- [02 — Architecture cible](02-architecture-cible.md)
- [08 — Firewall et ordre des règles](08-firewall-segmentation.md)
- [11 — Tests et validation](11-tests-et-validation.md)
