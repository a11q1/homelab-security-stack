# 00 — Contexte, périmètre, méthode

[← README](../README.md) · [Avant / après →](01-avant-apres.md)

## En une page

Un homelab Proxmox qui fait tourner en continu : le site de l'agence et ses
44 vhosts clients, deux domaines communautaires, un serveur Minecraft, de la
supervision, des outils admin, des bots et quelques labs. **30 LXC + 1 VM**,
un seul opérateur, budget zéro.

Le 24 novembre 2026, une nuit a été bloquée pour reprendre le réseau et la
sécurité de l'edge. Pas pour ajouter un service : pour **réduire la surface
d'attaque sans couper ce qui tourne**.

Ce dépôt est le compte rendu technique. Il assume les compromis et documente
ce qui n'est pas fini.

## Chiffres de départ

Relevés en lecture seule avant toute modification (`iptables-save`, `ss -tlnp`,
`nginx -T`, configs de tunnels, tests TCP) :

| Constat | Valeur |
|---|---|
| Conteneurs déclarés | 35 (32 running, 3 stopped) + 1 VM arrêtée |
| Bridges | 6 + WireGuard + Tailscale |
| Zones réellement filtrées | **1 sur 6** |
| Règles `FORWARD` | **91** (dont 3× doublons) pour ~40 règles logiques |
| DNAT `PREROUTING` | **42** (2 à 3 copies de chacune) |
| Tunnels Cloudflare distincts | **7** (6 configs sauvegardées) |
| Ingress HTTP | **~45** |
| Backends Caddy joignables depuis la DMZ | **5 sur 14** |
| Hostnames publics sans Access | **20+** |
| Connecteurs cloudflared en panne | **2** (503, 0 connexion) |

## Contraintes

Ces contraintes expliquent la plupart des choix, y compris les moins élégants.

| # | Contrainte | Conséquence |
|---|---|---|
| C1 | **Aucune coupure constatée** | Bascule tunnel par tunnel, DNS comme interrupteur |
| C2 | **Pas d'IP publique fixe** | Cloudflare Tunnel obligatoire, pas de port WAN ouvert |
| C3 | **Pas de pfSense / OPNsense** | Segmentation en iptables sur l'hôte Proxmox |
| C4 | **Un seul opérateur, une seule nuit** | Scripts idempotents, rollback écrit avant |
| C5 | **24/7** : jeu, sites clients, bots | Fenêtre de maintenance réelle : 2 h |
| C6 | **Budget 0 €** | Open source uniquement, pas d'équipement |
| C7 | **Pas de redémarrage de l'hôte** | Tout doit se recharger à chaud |

## Méthode

1. **Audit en lecture seule** (18:15–18:35). Aucune modification, aucun restart.
   Les constats sont datés et prouvés par des commandes.
2. **Sauvegardes** : configs de tunnels, DNS, `/etc/network/interfaces`,
   `iptables-save`, Caddyfile, compose. Vérification que le backup est complet —
   il ne l'était pas (ni `vmbr5` ni `wg0`).
3. **Construction hors chemin** : CT155 (WAF) et CT156 (Caddy) montés sans
   recevoir de trafic. Tests directs par IP.
4. **Bascule progressive** : un tunnel, un vhost, puis la campagne complète.
   Le DNS reste l'interrupteur de rollback.
5. **Validation** : 65 vhosts, payloads XSS/SQLi, codes Access, vérification PKI.
6. **Documentation à chaud**, tant que les commandes et les erreurs sont fraîches.

## Ce qui est hors périmètre

- Réécriture applicative, audit de code des sites clients.
- Migration cloud, achat de matériel, haute disponibilité.
- Poste de travail, sécurité physique, sauvegardes hors site.
- Minecraft (playit.gg), qui contourne volontairement l'edge HTTP.
- VLAN 802.1Q : les bridges Proxmox restent des sous-réseaux L3 séparés.

## Le contexte humain

C'est un lab, pas une banque. L'objectif est double : que ça tienne debout, et
que ce soit défendable en entretien BTS SIO SISR / alternance infra-sécurité.
D'où le choix de publier les erreurs autant que les réussites.

## Suites

- [01 — Avant / après](01-avant-apres.md)
- [05 — Threat model](05-threat-model.md)
- [13 — Leçons et limites](13-lecons-et-limites.md)
