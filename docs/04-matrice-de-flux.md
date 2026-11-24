# 04 — Matrice de flux

[← Plan d'adressage](03-plan-adressage.md) · [README](../README.md) · [Threat model →](05-threat-model.md)

## Conventions

| Colonne | Sens |
|---|---|
| **Décision** | `ALLOW` explicite, `DENY` explicite, `HÉRITÉ` = passe par une règle antérieure non revue |
| **Appliqué par** | où la décision est prise : `REFONTE`, `REFONTE-IN`, règle `vmbr4` historique, Caddy, BunkerWeb, Access |
| **Zone** | `LAN` = vmbr0, `INT` = vmbr1, `PROD` = vmbr2, `CTF` = vmbr3, `DMZ` = vmbr4, `EDGE` = vmbr5 |

Une ligne `HÉRITÉ` n'est pas une erreur : c'est une dette assumée. Elle est
listée pour ne pas donner l'illusion d'un default deny complet.

## Matrice

### Entrée publique

| # | Source | Destination | Port | Décision | Appliqué par |
|---|---|---|---|---|---|
| 1 | Internet | Cloudflare edge | 443/tcp | ALLOW | Cloudflare |
| 2 | Cloudflare | connecteurs CT131 / CT132 | 7844/tcp sortant | ALLOW | cloudflared |
| 3 | CT131 `22.22.22.20` | WAF `10.30.30.155` | 8080/tcp | ALLOW | `REFONTE` |
| 4 | CT132 `10.20.30.40` | WAF `10.30.30.155` | 8080/tcp | ALLOW | `REFONTE` |
| 5 | CT116 `10.10.10.116` / `22.22.22.116` | WAF `10.30.30.155` | 8080/tcp | ALLOW (compat) | `REFONTE` |
| 6 | autre source | WAF `10.30.30.155` | 8080/tcp | **DENY** | `REFONTE` (DROP générique) |
| 7 | navigateur admin | Cloudflare Access | 443/tcp | ALLOW après SSO | Access |

### Edge

| # | Source | Destination | Port | Décision | Appliqué par |
|---|---|---|---|---|---|
| 8 | BunkerWeb `10.20.30.155` | Caddy `10.20.30.156` | 443/tcp | ALLOW | `REFONTE` |
| 9 | autre source | Caddy `10.20.30.156` | 443/tcp | **DENY** | `REFONTE` |
| 10 | BunkerWeb / Caddy | DNS `10.10.10.1` ou `1.1.1.1` | 53 | ALLOW | `REFONTE` |
| 11 | BunkerWeb / Caddy | NTP | 123/udp | ALLOW | `REFONTE` |
| 12 | edge | reste du homelab | * | **DENY** (strict) | `REFONTE` |

### Caddy → backends

Chaque destination est nominative. Pas de `22.22.22.0/24` en bloc.

| # | Destination | Port | Vhost concerné | Appliqué par |
|---|---|---|---|---|
| 13 | `10.20.30.147` CT147 nginx | 80 | `sitecreator.shop`, `*.sitecreator.shop`, `cogtopia.events`, `beta` | `REFONTE` |
| 14 | `10.20.30.140` CT140 | 80 | `lab.example.com` | `REFONTE` |
| 15 | `10.20.30.129` CT129 | 80 | portfolio | `REFONTE` |
| 16 | `10.20.30.10` CT101 | 80 | `preview-client` | `REFONTE` |
| 17 | `10.20.30.126` CT126 | 3000 | `support` | `REFONTE` |
| 18 | `22.22.22.142` CT142 | 8080 | `cours` | `REFONTE` |
| 19 | `22.22.22.144` CT144 | 8080 | `diagrams` | `REFONTE` |
| 20 | `22.22.22.150` CT150 | 80 | `checkmk` | `REFONTE` |
| 21 | `22.22.22.108` CT108 | 3001 | `freellm` | `REFONTE` |
| 22 | `22.22.22.20` CT131 | 8900 | app agent | `REFONTE` |
| 23 | `10.66.66.2` CT141 | 8081, 3001-3004, 9000 | hackaton, player*, admin | `REFONTE` |
| 24 | toute autre destination | * | **DENY** | `REFONTE` |

**Hors Caddyfile** : `watch` et `automaton` ne sont plus routés par Caddy. Ils
restent des exceptions héritées, décrites plus bas.

## Les exceptions hors WAF

Tout le web public ne passe pas par le WAF. C'est écrit, pas caché.

| Hostname | Chemin | Pourquoi | État |
|---|---|---|---|
| `admin.sitecreator.shop` | tunnel → CT132 `:80` | honeypot assumé | 200 volontaire |
| `crafty.cogtopia.events` | tunnel CT137 → `:8443` | panneau de jeu, Access devant | 302 Access |
| `map.cogtopia.events` | tunnel CT137 → `:8100` | carte BlueMap | 503 volontaire |
| `watch.sitecreator.shop` | CT147 → CT137 `:8970` | legacy, 401 Basic Auth | à réparer ou retirer |
| `automaton.example.com` | ingress direct → CT145 `:8798` | backend mort | 502 |
| `play.example.com` | playit.gg, hors Cloudflare | Minecraft UDP/TCP | hors périmètre |


### Administration de l'hôte

| # | Source | Destination | Port | Décision | Appliqué par |
|---|---|---|---|---|---|
| 27 | LAN `192.168.1.0/24` | hôte `192.168.1.30` | 22, 8006, 3128 | ALLOW | `REFONTE-IN` |
| 28 | WireGuard `10.8.0.0/24` | hôte | 22, 8006, 3128 | ALLOW | `REFONTE-IN` |
| 29 | Tailscale `100.64.0.0/10` | hôte | 22, 8006, 3128 | ALLOW | `REFONTE-IN` |
| 30 | CT131 `22.22.22.20` | hôte | 22 | ALLOW | `REFONTE-IN` |
| 31 | CT150 Checkmk | hôte | 6556, 9100 | ALLOW | `REFONTE-IN` |
| 32 | CT147 / CT132 | hôte | 8811, 9099 | ALLOW | `REFONTE-IN` |
| 33 | vmbr1 / vmbr2 / vmbr3 / vmbr5 | hôte | 22, 8006, 3128, 6556, 9100, 8811, 9099 | **DENY** | `REFONTE-IN` |
| 34 | tout conteneur | hôte | autre port | HÉRITÉ | `INPUT` historique |

### Est / sud

| # | Source | Destination | Port | Décision | Appliqué par |
|---|---|---|---|---|---|
| 35 | DMZ | LAN `192.168.1.0/24` | * | **DENY** | règle `vmbr4` historique |
| 36 | DMZ | INT / PROD / CTF | * | **DENY** sauf lignes 13–23 | règles `vmbr4` |
| 37 | DMZ | Internet | 443, 53 | ALLOW (mises à jour) | `FORWARD -i vmbr4 -o vmbr0` |
| 38 | DMZ | DMZ | * | **DENY** sauf ACCEPT nominatifs | `REFONTE` : `-i vmbr4 -o vmbr4 -j DROP` |
| 39 | CTF `10.66.66.0/24` | partout | * | **HÉRITÉ** | `FORWARD ACCEPT` |
| 40 | INT ↔ PROD | partout | * | **HÉRITÉ** | `FORWARD ACCEPT` |
| 41 | tout CT | Internet | 443 | ALLOW (NAT) | `MASQUERADE` |
| 42 | CT138 honeypot | Discord | 443 | ALLOW | sortie NAT |
| 43 | CT137 playit / frpc | Internet | UDP/TCP jeu | ALLOW | **contournement documenté** |
| 44 | CT131, LAN, WG, Tailscale | n'importe quel CT | 22/tcp | ALLOW | `REFONTE` (7ter) |
| 45 | tout autre CT | tout autre CT | 22/tcp | **DENY** | `REFONTE` (7ter) |

### Supervision

| # | Source | Destination | Port | Décision | Appliqué par |
|---|---|---|---|---|---|
| 46 | CT150 Checkmk | tous les CT | 6556, 9100 | HÉRITÉ | pas de règle dédiée |
| 47 | Checkmk | hôte | 6556, 9100 | ALLOW | `REFONTE-IN` |

## Ce que la refonte ne filtre pas encore

C'est la partie la plus importante du document. Le WAF et la DMZ sont contenus.
Le reste du réseau ne l'est pas complètement.

| Flux | État | Risque |
|---|---|---|
| `vmbr1` ↔ `vmbr2` ↔ `vmbr3` | **ouvert** (`FORWARD ACCEPT`) | un conteneur compromis latéralise |
| Docker dans chaque CT | règles `docker0` / `br-*` | les conteneurs sortent librement |
| Tailscale | `ts-forward` accept-all par défaut | un appareil du tailnet voit tout |
| WireGuard | `-i wg0 -j ACCEPT` / `-o wg0 -j ACCEPT` | un client VPN compromis = accès total |
| Supervision | agents `6556/9100` joignables depuis tous les bridges | énumération d'inventaire |
| CTF | 17 ports DNAT sur `vmbr0` | surface d'attaque volontaire, mais large |

Ces lignes sont dans [13 — Leçons et limites](13-lecons-et-limites.md). Elles
ne sont pas cachées parce qu'elles sont gênantes.

## L'ordre des règles, et pourquoi ça casse

`REFONTE` est insérée en **position 1** de `FORWARD`. Elle se termine par
`RETURN`. Donc :

```text
FORWARD
  └─ REFONTE
       1. ESTABLISHED,RELATED        ACCEPT
       2. DNS / NTP edge             ACCEPT
       3. connecteurs → WAF:8080     ACCEPT
       4. WAF → Caddy:443            ACCEPT
       5. Caddy → backends           ACCEPT
       6. SSH inter-CT admin         ACCEPT
       7. SSH inter-CT autres        DROP
       8. DROP WAF:8080 (autres)     DROP
       9. DROP Caddy:443 (autres)    DROP
      10. -i vmbr4 -o vmbr4          DROP   ← l'isolation DMZ
      11. RETURN                     →
  └─ règles historiques (vmbr4, vmbr3, wg0…)
```

Deux conséquences :

- **Les ACCEPT nominatifs doivent être avant les DROP génériques.** Sinon Caddy
  ne joint plus ses backends, et le WAF ne joint plus Caddy.
- **Le `DROP -i vmbr4 -o vmbr4` casse tout flux intra-DMZ non listé.** Les
  vhosts qui passaient par nginx CT147 pour rejoindre un autre CT de la DMZ
  (`support`, portfolio, `preview-client`) doivent désormais être routés par Caddy.
  C'est le cas dans le Caddyfile, mais c'est un piège pour le prochain vhost.

C'est exactement l'incident du 13 novembre : une règle `-I` insérée sous les
`DROP` a coupé les réponses vers `22.22.22.0/24`. L'ordre compte plus que la
règle.

## Tester un flux

```bash
# Depuis le WAF : Caddy doit répondre en TLS
pct exec 155 -- curl -sk -o /dev/null -w '%{http_code}\n' https://10.20.30.156/health

# Depuis un conteneur lambda : le WAF doit être joignable sur 8080
pct exec 142 -- curl -s -o /dev/null -w '%{http_code}\n' --max-time 3 http://10.30.30.155:8080/

# Depuis la DMZ : le LAN doit être injoignable
pct exec 156 -- curl -s -o /dev/null -w '%{http_code}\n' --max-time 3 http://192.168.1.30:8006
# attendu : 000 (timeout)

# Depuis un conteneur : les ports admin de l'hôte doivent être fermés
pct exec 142 -- timeout 3 bash -c '</dev/tcp/22.22.22.1/8006' && echo OUVERT || echo FERMÉ
# attendu : FERMÉ sur vmbr2

# Compteurs plutôt que confiance
iptables -vnL REFONTE --line-numbers
iptables -vnL REFONTE-IN --line-numbers
```

Un flux autorisé sans compteur qui bouge est un flux mort. Un flux refusé avec
un compteur qui monte est un flux qui essaie encore.

## Avant / après en chiffres

| Indicateur | Avant | Après |
|---|---|---|
| Règles `FORWARD` | 91 (doublons) | 91 historiques + ~35 dans `REFONTE` |
| Zones avec allowlist edge | 0 | `vmbr4` isolée, `vmbr5` en allowlist, `INPUT` admin restreint |
| Backends Caddy joignables depuis la DMZ | 5 / 14 | 14 / 14 sur les backends routés par Caddy |
| Ports admin de l'hôte depuis un CT | 5 ouverts | 0 (hors LAN/WG/Tailscale/CT131) |
| Hostnames admin sans SSO | 5+ | 0 sur le périmètre testé |
| Persistance firewall | script à rejouer à la main | `refonte-firewall.service --strict` activé au boot |
| Secrets edge | `.env` en clair | SOPS + age, tokens en `--token-file` |

## Suites

- [05 — Threat model](05-threat-model.md)
- [08 — Firewall](08-firewall-segmentation.md)
- [11 — Tests](11-tests-et-validation.md)
