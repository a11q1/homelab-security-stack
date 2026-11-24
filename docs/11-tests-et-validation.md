# 11 — Tests et validation

[← Secrets](10-secrets-et-rotation.md) · [README](../README.md) · [Runbooks →](12-runbooks.md)

## Ce qu'on cherche à prouver

Pas « ça a l'air de marcher ». Quatre choses :

1. **Aucune régression** : les 65 vhosts répondent toujours.
2. **Le WAF bloque** : XSS et SQLi retournent 403.
3. **Le WAF ne bloque pas trop** : une page normale retourne 200.
4. **Les admins sont protégés** : les consoles retournent 302 (Access).

Et deux choses qu'on ne prouve pas encore :

5. La DMZ ne peut pas joindre le LAN (règle vérifiée, test négatif à rejouer).
6. Le hop WAF → Caddy est authentifié (non : `verify=no`).

## Niveaux de test

| Niveau | Quoi | Outil | Quand |
|---|---|---|---|
| L0 | syntaxe des configs | `caddy validate`, `iptables-restore --test`, `cloudflared ingress validate` | avant reload |
| L1 | vhosts via le WAF | `curl -H "Host: …"` | après chaque ajout |
| L2 | vhosts en live | `curl https://…` | après bascule DNS |
| L3 | négatif | payloads, timeouts, ports fermés | après activation |
| L4 | régression | campagne complète | avant de dormir |

## L0 — Valider avant de recharger

```bash
# Caddy
caddy validate --config /etc/caddy/Caddyfile

# Cloudflare Tunnel (le fichier d'ingress)
cloudflared tunnel ingress validate --config /etc/cloudflared/config.yml

# iptables : recharger dans un namespace de test n'est pas trivial ;
# on utilise --test sur un restore complet
iptables-save > /tmp/iptables.avant
# … modifications …
iptables-restore --test /tmp/iptables.nouveau && echo "syntaxe OK"

# Compose
docker compose -f /opt/edge/docker-compose.yml config >/dev/null && echo "compose OK"
```

Une config invalide rechargée à chaud peut couper le service. C'est la première
barrière, pas la dernière.

## L1/L2 — La campagne vhosts

Le script complet :
[`scripts/test-vhosts.sh.example`](../scripts/test-vhosts.sh.example).

```bash
./test-vhosts.sh --internal-only   # via le WAF uniquement
./test-vhosts.sh                   # via le WAF + le live
```

Sortie réelle, tronquée :

```text
HOST                                              BW   BW_SIZE   LIVE LIVE_SIZE OK
sitecreator.shop                                 200     18422    200     18422 OK
www.sitecreator.shop                             200     18422    200     18422 OK
cogtopia.events                                  200      9120    200      9120 OK
help.cogtopia.events                             200      4210    200      4210 OK
support.sitecreator.shop                         200     12044    200     12044 OK
beta.sitecreator.shop                            200      2044    200      2044 OK
cours.example.com                                200      8801    200      8801 OK
hackaton.example.com                             200      6712    200      6712 OK
admin.hackaton.example.com                       302         0    302         0 OK
diagrams.example.com                             302         0    302         0 OK
checkmk.example.com                              302         0    302         0 OK
…
Échecs: 0 / 65
```

### Ce que « 0 / 65 » veut dire

- 65 hostnames testés, tous ont répondu autre chose que timeout / 000.
- Les origines déjà mortes avant la refonte sont exclues du décompte
  (`automaton`, `preview-client`, `watch`) : elles restent dans le jeu de
  test pour suivre la régression, mais un 502 est attendu.
- Les redirections (`play`, `map`, `legacy-help`) et la carte fermée ne sont pas
  dans la campagne : elles ne servent pas de contenu.
- **Ce n'est pas un test fonctionnel.** Un 200 ne dit pas que le formulaire
  envoie un mail, ni que le paiement passe. C'est un test de disponibilité HTTP.

### Pourquoi 68 noms dans le WAF et 65 testés

68 `SERVER_NAME` déclarés. La campagne en couvre 64, plus `automaton` (mort,
conservé pour la régression). Les 4 noms non testés sont des redirections ou
une carte fermée :

```text
play.sitecreator.shop      → 301
map.sitecreator.shop       → 301
legacy-help.sitecreator.shop → 301
map.cogtopia.events        → 503 volontaire
```

## L3 — Négatif

### WAF

```bash
# XSS
curl -s -o /dev/null -w '%{http_code}\n' \
  'https://sitecreator.shop/?q=%3Cscript%3Ealert(1)%3C%2Fscript%3E'
# 403

# SQLi
curl -s -o /dev/null -w '%{http_code}\n' \
  "https://sitecreator.shop/?id=1%27%20OR%20%271%27%3D%271"
# 403

# Contrôle positif : ne pas bloquer le légitime
curl -s -o /dev/null -w '%{http_code}\n' 'https://sitecreator.shop/'
# 200
```

Un WAF qui renvoie 403 sur la page d'accueil est un WAF mal réglé. Le contrôle
positif fait partie du test, pas seulement les payloads.

### Access

```bash
for h in proxmox.example.com checkmk.example.com diagrams.example.com \
         crafty.cogtopia.events admin.hackaton.example.com; do
  code=$(curl -s -o /dev/null -w '%{http_code}' -I "https://$h")
  printf '%-34s %s\n' "$h" "$code"
done
# attendu : 302 partout
```

### TLS interne

```bash
openssl verify -CAfile /etc/caddy/pki/root.crt \
  -untrusted /etc/caddy/pki/intermediate.crt /etc/caddy/pki/caddy.crt
# caddy.crt: OK

openssl s_client -connect 10.20.30.156:443 -servername caddy.dmz.internal \
  -CAfile /etc/caddy/pki/root.crt </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -dates
```

### Ports admin de l'hôte

```bash
pct exec 142 -- timeout 3 bash -c '</dev/tcp/22.22.22.1/8006' \
  && echo OUVERT || echo FERMÉ
# attendu : FERMÉ

pct exec 142 -- timeout 3 bash -c '</dev/tcp/22.22.22.1/22' \
  && echo OUVERT || echo FERMÉ
# attendu : FERMÉ
```

### Isolation DMZ

```bash
# Depuis Caddy : le LAN doit être injoignable
pct exec 156 -- curl -s -o /dev/null -w '%{http_code}\n' --max-time 3 \
  http://192.168.1.30:8006
# attendu : 000
```

Ce dernier test est un **test de règle**, pas un test applicatif : il prouve
que le paquet ne sort pas, pas que la règle est la bonne. Les compteurs
`iptables -vnL` complètent.

## Tableau de résultats

| Test | Attendu | Mesuré le 24/11 | Statut |
|---|---|---|---|
| Campagne vhosts | 65 réponses | 65 / 65 | **OK** |
| XSS `?q=<script>` | 403 | 403 | **OK** |
| SQLi `?id=1' OR '1'='1` | 403 | 403 | **OK** |
| Page normale | 200 | 200 | **OK** |
| `proxmox` (com/net) | 302 | 302 | **OK** |
| `checkmk` | 302 | 302 | **OK** |
| `diagrams` | 302 | 302 | **OK** |
| `crafty` | 302 | 302 | **OK** |
| `freellm` | 302 | 302 | **OK** |
| `admin.sitecreator.shop` | 200 (piège) | 200 | **OK** |
| `openssl verify` | OK | OK | **OK** |
| DMZ → LAN | timeout | non rejoué | **à faire** |
| Hôte `:8006` depuis vmbr2 | fermé | non rejoué | **à faire** |
| WAF → Caddy authentifié | vérifié | `verify=no` | **non** |

Les trois dernières lignes sont volontairement dans le tableau. Un rapport de
test sans ligne rouge est un rapport incomplet.

## Procédure de rollback

### Critères de déclenchement

- Plus de 5 % des vhosts en 5xx pendant 5 minutes.
- Erreur TLS en série sur le hop WAF → Caddy.
- Perte d'accès admin (Proxmox ou SSH).
- Flux DMZ → LAN ou hôte observé comme ouvert.
- WAF qui bloque un vhost légitime sans exclusion rapide possible.

### Ordre

```text
1. Ingress tunnel  → revenir aux JSON sauvegardés (origines directes)
2. DNS             → ne pas toucher si le tunnel suffit
3. WAF             → docker compose stop (le trafic ne passe plus par lui)
4. Watchdog        → restaurer la version .bak-20261124-refonte
5. Firewall        → retirer REFONTE / REFONTE-IN
6. Vérifier        → campagne vhosts + accès admin
```

```bash
# 1. Ingress
cp /root/backup/tunnels/<tunnel>.json /root/.cloudflared/config.yml
systemctl restart cloudflared-xxx
curl -s http://127.0.0.1:20241/ready

# 3. WAF
cd /opt/edge && docker compose stop

# 4. Watchdog
cp /root/agency_tools/tunnel_watchdog.py.bak-20261124-refonte \
   /root/agency_tools/tunnel_watchdog.py
systemctl restart tunnel-watchdog.timer   # ou le cron

# 5. Firewall : retirer les chaînes, les règles historiques restent
iptables -D FORWARD -j REFONTE 2>/dev/null || true
iptables -F REFONTE && iptables -X REFONTE
iptables -D INPUT -j REFONTE-IN 2>/dev/null || true
iptables -F REFONTE-IN && iptables -X REFONTE-IN

# 6. Vérifier
./test-vhosts.sh
curl -sI https://proxmox.example.com | head -1
```

**Temps cible** : moins de 15 minutes. **Temps réel** : non mesuré à froid.
C'est une des limites du dispositif : un rollback non chronométré n'est pas
un rollback prouvé.

### Ce qui rend le rollback possible

| Élément | Emplacement |
|---|---|
| Configs de tunnels | backup local, hors repo |
| `iptables-save` avant | `/root/backup/iptables-20261124.txt` |
| `Caddyfile` avant | backup daté |
| `bw.env` avant | backup daté, `600` |
| Watchdog avant | `.bak-20261124-refonte` |
| Ancien DNS | export de zone |

Sans ces fichiers, le rollback est une reconstruction. C'est pour ça que le
backup a été fait **avant** l'audit, pas après.

## Journal de test (modèle)

```text
Date :
Opérateur :
Commit / version :
Environnement :

Tests exécutés :
  [ ] L0 config
  [ ] L1 vhosts WAF
  [ ] L2 vhosts live
  [ ] L3 XSS / SQLi / positif
  [ ] L3 Access
  [ ] L3 TLS
  [ ] L3 ports admin
  [ ] L3 isolation DMZ

Résultats :
Échecs :
Décision : GO / NO-GO / rollback
Durée :
```

## Limites de cette validation

- Pas de test de charge, pas de mesure de latence p95.
- Pas de test d'intrusion complet (pas de scan, pas d'exploitation).
- Pas de test d'authentification Access (on teste le 302, pas le contournement).
- Pas de test de restauration de sauvegarde.
- Pas de test après reboot de l'hôte — alors que la persistance du firewall
  n'est pas câblée.
- Les tests sont manuels, pas en CI.

## Suites

- [12 — Runbooks](12-runbooks.md)
- [13 — Leçons et limites](13-lecons-et-limites.md)
