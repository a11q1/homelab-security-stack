# 13 — Leçons et limites

[← Runbooks](12-runbooks.md) · [README](../README.md)

## Ce qui a marché

1. **Construire hors chemin.** CT155 et CT156 montés avant de recevoir du
   trafic. Les tests directs par IP ont évité de découvrir les erreurs en prod.
2. **Le DNS comme interrupteur.** Repointer un CNAME est un rollback en secondes.
   Aucun besoin de redémarrer quoi que ce soit.
3. **Les chaînes en tête de FORWARD.** `REFONTE` + `RETURN` ajoute une couche
   sans réécrire l'existant. C'est ce qui a rendu la bascule possible en une nuit.
4. **Le WAF en détection d'abord.** Les faux positifs 401 ont été vus avant de
   bloquer des admins légitimes.
5. **Le test des 65 vhosts.** Un chiffre, pas une impression. C'est ce qui a
   permis de dire « c'est bon » et de le prouver.

## Ce qui a été douloureux

### L'ordre des règles

Une règle correcte au mauvais endroit est une règle morte. Le `DROP` intra-DMZ
doit venir **après** les `ACCEPT` nominatifs, sinon Caddy ne joint plus ses
backends. Et un `ACCEPT` après un `DROP` ne sert à rien.

### Le watchdog

`tunnel_watchdog.py` réappliquait l'ancien chemin toutes les 5 minutes. Il a
fallu le patcher **avant** de basculer, sinon chaque test était invalidé en
silence. Leçon : chercher les boucles de réconciliation avant de changer une
config.

### Les 401

Le plugin « bad behavior » compte les 401 comme suspects. Checkmk et les vhosts
Basic Auth se sont fait bloquer. Deux réglages à désactiver :
`USE_BAD_BEHAVIOR=no` et `REVERSE_PROXY_INTERCEPT_ERRORS=no`.

### Le multisite explicite

68 noms à la main. Un oubli et le Host tombe sur le vhost par défaut. C'est
fragile, mais c'est le prix du multisite BunkerWeb sans wildcard.

### Les origines mortes

Trois vhosts pointent vers des services qui ne répondent plus. Ils étaient
morts avant la refonte, mais le WAF les rend visibles : 502 au lieu d'un
timeout silencieux. C'est mieux, à condition de les réparer ou de les retirer.

## Points ouverts

État au 24 novembre 2026, 20:00.

| # | Point | Impact | Priorité |
|---|---|---|---|
| 1 | `automaton` : backend `:8798` non écouté (dashboard sur `:8799`, bind localhost) | 502 | P1 |
| 2 | `preview-client` : origine `10.20.30.10:80` ne répond pas | 502 derrière Access | P1 |
| 3 | `grafana` : DNS orphelin, service mort | nom qui traîne | P2 |
| 4 | `watch` : routé via CT147 → CT137, 401 Basic Auth | vhost à moitié mort | P2 |
| 5 | Tunnel CT116 (autre compte Cloudflare) inutilisé, token visible dans `ps` | secret exposé localement | **P1** |
| 6 | Firewall par CT Proxmox non effectif pour le trafic bridgé | fausse ceinture | P2 |
| 7 | TOTP Proxmox non activé | Access est le seul facteur | **P1** |
| 8 | `REVERSE_PROXY_SSL_VERIFY=no` | hop chiffré, non authentifié | P2 |
| 9 | Persistance du firewall au boot non câblée | reboot = retour aux règles historiques | **P1** |
| 10 | `REAL_IP_FROM` contient des IP non-connecteurs | spoofing possible | P2 |
| 11 | Latéralisation `vmbr1 ↔ vmbr2 ↔ vmbr3` | mouvement latéral | **P1** |
| 12 | SOPS/age non déployé | secrets en clair sur disque | P2 |
| 13 | Rotation du token de tunnel non faite | dette de sécurité | P1 |
| 14 | Rollback non chronométré | procédure non prouvée | P2 |

Les points 5, 7, 9, 11 et 13 sont ceux qui me dérangent le plus. Ils sont
traitables en une journée chacun, mais ils ne l'ont pas été cette nuit.

## Ce que je ferais différemment

### 1. Écrire les tests avant la bascule

Les 65 vhosts ont été testés après. Un script lancé avant aurait donné une
ligne de base : quels vhosts étaient déjà morts, lesquels répondaient 200.
Sans ça, « 0 échec » demande une explication.

### 2. Versionner la config edge dès le début

Le `bw.env`, le `Caddyfile` et le script firewall ont été écrits sur le serveur,
puis copiés dans un dépôt. L'inverse (dépôt → déploiement) aurait évité les
écarts entre la doc et la réalité.

### 3. Un seul connecteur, pas deux

CT131 et CT132 portent encore le web public. C'est mieux qu'avant (6 hôtes),
mais ce n'est pas « un seul point d'entrée ». La raison est structurelle :
certains services écoutent sur `127.0.0.1` et ne peuvent être joints que par un
connecteur local. Les rebinder est un projet à part.

### 4. SOPS dès le premier jour

Chiffrer les secrets avant de les poser sur disque. Pas après.

### 5. Mesurer le rollback

Un rollback de 40 minutes n'est pas un rollback. Il faut le chronométrer à
froid, pas le découvrir à 2 h du matin.

## Ce que je ne referais pas

- **Empiler des exceptions iptables** sans matrice. C'est comme ça qu'on arrive
  à 91 règles `FORWARD` pour 40 règles logiques.
- **Faire confiance à un backup non vérifié.** Celui des interfaces ne
  contenait ni `vmbr5` ni `wg0` : une restauration aurait coupé le WAF et le VPN.
- **Laisser un token dans une ligne de commande.** `ps` est lisible par tous
  les processus du conteneur.
- **Confondre « le WAF est là » et « le WAF est dans le chemin ».** Avant la
  bascule, aucun tunnel ne pointait vers lui.

## Les limites structurelles

| Limite | Pourquoi elle existe | Ce qu'il faudrait |
|---|---|---|
| Un seul hyperviseur | budget, lab | un second nœud, Proxmox HA |
| Un seul WAF | simplicité | deux instances + keepalived |
| Un seul Caddy | simplicité | idem |
| HTTP intra-DMZ | temps, compatibilité | TLS partout, mTLS |
| Pas d'IDS réseau | pas installé | Suricata en miroir sur vmbr4 |
| Pas de VLAN | bridges L3 | 802.1Q + VLAN filtering Proxmox |
| Pas de SIEM | volume faible | Loki + Grafana, rétention 30 j |
| Sauvegardes non testées | hors périmètre | restauration à blanc trimestrielle |

## La suite, par ordre de valeur

### 30 jours

1. Rotationner le token du tunnel orphelin et supprimer le tunnel CT116.
2. Activer TOTP sur Proxmox.
3. Câbler `refonte-firewall.service` au boot et tester un reboot planifié.
4. Réparer ou retirer `automaton`, `preview-client`, `grafana`, `watch`.
5. Resserrer `REAL_IP_FROM` aux seuls connecteurs.

### 60 jours

6. Activer `REVERSE_PROXY_SSL_VERIFY=yes` après validation de la chaîne.
7. Déployer SOPS + age pour les secrets edge.
8. Écrire la matrice `vmbr1 ↔ vmbr2 ↔ vmbr3` et la passer en allowlist.
9. Chronométrer le rollback complet.

### 90 jours

10. Passer d'une PKI de lab à une PKI avec révocation (CRL) et clé root hors ligne.
11. Ajouter un IDS réseau sur la DMZ.
12. Automatiser la campagne de tests (CI + cron).
13. Envisager un second hyperviseur si la charge le justifie.

## Conclusion

La refonte a fait ce qu'elle devait faire : **un chemin d'inspection, une DMZ
contenue, des admins derrière un SSO, et des preuves**. Elle n'a pas rendu le
lab invulnérable, et ce document ne prétend pas le contraire.

Ce qui reste ouvert est écrit, priorisé et daté. C'est la différence entre une
dette technique et une surprise.

## Suites

- [README](../README.md)
- [00 — Contexte](00-contexte.md)
- [04 — Matrice de flux](04-matrice-de-flux.md)
