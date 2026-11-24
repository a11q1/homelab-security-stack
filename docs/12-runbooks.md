# 12 — Runbooks

[← Tests](11-tests-et-validation.md) · [README](../README.md) · [Leçons →](13-lecons-et-limites.md)

Huit procédures. Chacune a un objectif, des commandes, une vérification et un
retour arrière. Les chemins et IP correspondent au lab décrit dans ce dépôt ;
adaptez-les avant de copier.

---

## RB-01 — Ajouter un vhost

**Objectif** : rendre `nouveau.example.com` accessible via le WAF.
**Durée** : 10 min. **Risque** : moyen (ordre des étapes).

### Prérequis

- le backend répond en HTTP sur une IP joignable depuis Caddy ;
- le nom est décidé ;
- vous savez si le hostname doit être public ou derrière Access.

### Étapes

```text
1. BunkerWeb   → ajouter le nom dans SERVER_NAME
2. Caddy       → ajouter un matcher host + reverse_proxy
3. Firewall    → si le backend est hors DMZ, ajouter le couple (IP, port)
4. Tunnel      → ajouter l'ingress vers http://10.30.30.155:8080
5. DNS         → CNAME vers le tunnel (souvent déjà couvert par le wildcard)
6. Test        → WAF puis live
```

```bash
# 1. BunkerWeb
cp /opt/edge/bw.env /opt/edge/bw.env.bak-$(date +%F)
# ajouter le nom à la fin de SERVER_NAME, sur une seule ligne
docker compose -f /opt/edge/docker-compose.yml restart bw-scheduler

# 2. Caddy
cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.bak-$(date +%F)
cat >> /etc/caddy/Caddyfile <<'EOF'

@nouveau host nouveau.example.com
handle @nouveau {
    reverse_proxy 10.20.30.147:80
}
EOF
# attention : le bloc doit être AVANT le `respond 404` final
caddy validate --config /etc/caddy/Caddyfile
# `admin off` : reload impossible, restart obligatoire (bref blip)
systemctl restart caddy

# 3. Firewall (seulement si hors DMZ)
# éditer /usr/local/sbin/refonte-firewall.sh, ajouter la ligne AVANT les DROP
# puis rejouer le script (idempotent)
/usr/local/sbin/refonte-firewall.sh

# 4. Tunnel
cloudflared tunnel ingress validate --config /etc/cloudflared/config.yml
systemctl restart cloudflared-xxx
curl -s http://127.0.0.1:20241/ready

# 5. DNS (si hors wildcard)
# CNAME nouveau.example.com → <tunnel-id>.cfargotunnel.com, proxied

# 6. Test
curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: nouveau.example.com' \
  http://10.30.30.155:8080/
curl -s -o /dev/null -w '%{http_code}\n' https://nouveau.example.com/
```

### Vérification

- `200` via le WAF et en live ;
- le vhost apparaît dans les logs BunkerWeb avec la bonne IP client ;
- `iptables -vnL REFONTE` montre le compteur de la règle backend qui monte.

### Retour arrière

Retirer le bloc Caddy, restaurer `bw.env`, recharger, retirer l'ingress.
Le DNS peut rester : sans ingress, il renvoie une erreur Cloudflare, pas une
page blanche.

### Pièges

- Ajouter le DNS **avant** BunkerWeb : le Host tombe sur le vhost par défaut.
- Oublier le firewall pour un backend `vmbr2` : Caddy timeout, 502.
- Mettre le bloc Caddy après `respond 404` : il n'est jamais atteint.

---

## RB-02 — Ajouter un site client

**Objectif** : publier `<client>.sitecreator.shop`.
**Durée** : 20 min. **Risque** : faible.

### Étapes

```bash
# 1. Fichiers du site (hors de ce dépôt)
install -d -m 755 /var/www/clients/<client>
# déposer le build statique ou le PHP

# 2. Vhost nginx sur CT147
cat > /etc/nginx/sites-available/<client>.conf <<'EOF'
server {
    listen 80;
    server_name <client>.sitecreator.shop;
    root /var/www/clients/<client>;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
    }
}
EOF
ln -s /etc/nginx/sites-available/<client>.conf /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx

# 3. WAF + Caddy : RB-01
#    (le wildcard DNS *.sitecreator.shop couvre déjà le nom)

# 4. Test
curl -s -o /dev/null -w '%{http_code}\n' https://<client>.sitecreator.shop/
```

### Vérification

- `200`, contenu du site ;
- pas de `403` WAF sur les pages et formulaires ;
- HTTPS valide (certificat Cloudflare).

### Retour arrière

Retirer le symlink nginx, recharger. Le nom repasse en 404 ou sur le vhost par
défaut. Le DNS wildcard n'a pas bougé.

### Pièges

- Un client avec un formulaire complexe peut déclencher CRS : tester l'envoi,
  pas seulement l'affichage.
- Les uploads > 1 Mo : vérifier `client_max_body_size` côté nginx.
- Ne jamais mettre de données réelles dans un site de démonstration.

---

## RB-03 — Rotation du certificat Caddy

**Objectif** : remplacer la feuille sans coupure.
**Durée** : 10 min. **Risque** : faible si la chaîne est validée avant reload.

```bash
cd /etc/caddy/pki
STAMP=$(date +%F)

# 1. Sauvegarder
cp caddy.crt caddy.crt.bak-$STAMP
cp caddy.key caddy.key.bak-$STAMP
cp chain.crt chain.crt.bak-$STAMP

# 2. Nouvelle clé + CSR
openssl ecparam -name prime256v1 -genkey -noout -out caddy.key.new
openssl req -new -key caddy.key.new -out caddy.csr \
  -subj "/CN=caddy.dmz.internal/O=Example Lab/OU=Edge"

# Le script de génération supprime caddy.ext : le recréer avant de signer.
cat > caddy.ext <<'EXT'
basicConstraints=critical,CA:FALSE
keyUsage=critical,digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=DNS:caddy.dmz.internal,IP:10.20.30.156
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
EXT

# 3. Signer avec l'intermédiaire
openssl x509 -req -in caddy.csr -CA intermediate.crt -CAkey intermediate.key \
  -CAcreateserial -out caddy.crt.new -days 825 -sha256 -extfile caddy.ext

# 4. Chaîne
cat caddy.crt.new intermediate.crt > chain.crt.new

# 5. Vérifier AVANT de remplacer
openssl verify -CAfile root.crt -untrusted intermediate.crt caddy.crt.new
# attendu : OK

# 6. Basculer
mv caddy.key.new caddy.key
mv caddy.crt.new caddy.crt
mv chain.crt.new chain.crt
chmod 600 caddy.key && chmod 644 caddy.crt chain.crt

# 7. Recharger
caddy validate --config /etc/caddy/Caddyfile
systemctl restart caddy
```

### Vérification

```bash
echo | openssl s_client -connect 10.20.30.156:443 \
  -servername caddy.dmz.internal -CAfile /etc/caddy/pki/root.crt 2>/dev/null \
  | openssl x509 -noout -dates -subject

pct exec 155 -- curl -sk -o /dev/null -w '%{http_code}\n' https://10.20.30.156/health
```

### Retour arrière

```bash
cd /etc/caddy/pki
mv caddy.crt.bak-$STAMP caddy.crt
mv caddy.key.bak-$STAMP caddy.key
mv chain.crt.bak-$STAMP chain.crt
systemctl restart caddy
```

### Pièges

- Oublier `intermediate.crt` dans `chain.crt` : le client ne peut pas construire
  la chaîne.
- Générer une clé en `644` : `umask 077` avant, toujours.
- Recharger sans `openssl verify` : l'erreur arrive en production.
- Oublier de réencrypter la sauvegarde : la prochaine restauration repart avec
  l'ancien certificat.

### Après la rotation

```bash
# Réencrypter la sauvegarde PKI avec la nouvelle clé
tar -C /etc/caddy -czf - pki \
  | age -r "$AGE_RECIPIENT" -o /root/refonte-2026-11-24/secrets/pki.tgz.age
chmod 640 /root/refonte-2026-11-24/secrets/pki.tgz.age
```

---

## RB-04 — Panne ou faux positif WAF

**Symptômes** : 403 sur un vhost légitime, 502 généralisé, UI WAF inaccessible,
CPU du conteneur à 100 %.

### A. Faux positif (un site légitime est bloqué)

```bash
# 1. Identifier la requête et la règle
docker logs bunkerweb --since 15m 2>&1 | grep -i 'modsecurity\|deny' | tail -40

# 2. Repasser temporairement en détection (global)
sed -i 's/^SECURITY_MODE=block/SECURITY_MODE=detection/' /opt/edge/bw.env
docker compose -f /opt/edge/docker-compose.yml restart bw-scheduler

# 3. Vérifier que le site remarche
curl -s -o /dev/null -w '%{http_code}\n' https://sitecreator.shop/

# 4. Exclusion ciblée par règle et par vhost, puis retour en block
#    (ne jamais désactiver CRS globalement)
sed -i 's/^SECURITY_MODE=detection/SECURITY_MODE=block/' /opt/edge/bw.env
docker compose -f /opt/edge/docker-compose.yml restart bw-scheduler

# 5. Rejouer la requête + la campagne
./test-vhosts.sh
```

### B. WAF down (502 partout)

```bash
# 1. Constater
docker ps -a --filter name=bunkerweb
docker logs bunkerweb --tail 100

# 2. Redémarrer la stack
cd /opt/edge && docker compose up -d
curl -s -o /dev/null -w '%{http_code}\n' http://10.30.30.155:8080/

# 3. Si ça ne repart pas : sortir le WAF du chemin
#    Repointer l'ingress vers Caddy (TLS interne, verify off côté cloudflared)
#    ou vers l'ancienne origine CT147 pour un rollback complet (RB-05).
```

### Vérification

- `200` sur 3 vhosts témoins (`sitecreator.shop`, `cogtopia.events`, `support`) ;
- pas de 403 sur la page d'accueil ;
- logs qui reprennent.

### Retour arrière

`SECURITY_MODE=detection` est le filet. Il laisse passer mais logue. À utiliser
le temps de corriger, pas comme état permanent.

---

## RB-05 — Rollback tunnel

**Objectif** : revenir au chemin d'avant la refonte pour un tunnel.
**Durée** : 5 min. **Risque** : faible (le DNS ne bouge pas).

```bash
# 1. Sauvegarder l'état courant
cp /etc/cloudflared/config.yml /root/config.yml.refonte-$(date +%F)

# 2. Restaurer la config d'avant
cp /root/backup/tunnels/<tunnel>.json /etc/cloudflared/config.yml
chmod 600 /etc/cloudflared/config.yml

# 3. Valider
cloudflared tunnel ingress validate --config /etc/cloudflared/config.yml
cloudflared tunnel ingress rules --config /etc/cloudflared/config.yml

# 4. Redémarrer
systemctl cat cloudflared-xxx | grep -E 'ExecStart|token-file'
# le token doit venir d'un fichier 600, pas d'un argument en clair
systemctl restart cloudflared-xxx
sleep 5

# 5. Vérifier les connexions
curl -s http://127.0.0.1:20241/ready
# attendu : 4 connexions "ready"

# 6. Vérifier un vhost
curl -s -o /dev/null -w '%{http_code}\n' https://sitecreator.shop/
```

### Vérification

- 4 connexions `ready` (ou le nombre attendu) ;
- un vhost témoin répond ;
- les logs ne montrent pas de `connection refused`.

### Retour arrière du rollback

Remettre `/root/config.yml.refonte-<date>` et redémarrer. C'est pour ça qu'on
sauvegarde l'état courant avant de restaurer l'ancien.

### Pièges

- Redémarrer le tunnel sans `ingress validate` : une erreur de syntaxe laisse
  le connecteur en boucle.
- Oublier que le watchdog peut réécrire la config dans les 5 minutes. Le
  désactiver pendant l'opération.
- Restaurer un ancien JSON qui contient un token en clair : le déplacer dans un
  fichier `600` et passer l'unité en `--token-file`. Ne jamais remettre un
  token dans la ligne de commande.
- Redémarrer CT116 ou CT111 « pour voir » : ils sont arrêtés volontairement,
  leurs tunnels ne servent plus.

---

## RB-06 — Ajouter une politique Access

**Objectif** : protéger `nouveau-admin.example.com`.
**Durée** : 5 min. **Risque** : faible, mais un mauvais scope peut verrouiller.

```text
1. Cloudflare Zero Trust → Access → Applications → Add an application
2. Type : Self-hosted
3. Application domain : nouveau-admin.example.com
4. Session duration : 8–24 h selon le service
5. Policy :
     name    : admin-only
     action  : Allow
     include : Emails → admin@example.com
6. Save
```

### Vérification

```bash
curl -sI https://nouveau-admin.example.com | head -3
# attendu : 302 vers <team>.cloudflareaccess.com

# Avec un navigateur en navigation privée : le portail s'affiche,
# l'email non autorisé est refusé.
```

### Retour arrière

Supprimer l'application Access. Le hostname redevient public **immédiatement** :
vérifier qu'il n'est pas sensible avant de faire ça.

### Pièges

- Mettre l'application sur `*.example.com` : protège tout, y compris les sites
  publics. Scope exact, toujours.
- Oublier le `www` : `www.admin.example.com` reste ouvert.
- Créer un `Bypass` « pour tester » et l'oublier.

---

## RB-07 — Redémarrage de l'hôte

**Objectif** : retrouver l'état refonte après un reboot.
**Durée** : 15 min. **Risque** : moyen (le reboot lui-même n'a pas été exercé).

```bash
# 1. Le service firewall a-t-il rejoué la configuration ?
systemctl status refonte-firewall.service --no-pager
systemctl is-enabled refonte-firewall.service

# 2. Ordre des chaînes : REFONTE en 1, PVEFW-FORWARD en 2
iptables -L FORWARD -n --line-numbers | head -4

# 3. Les conteneurs edge sont-ils remontés ?
pct status 155 && pct status 156
pct exec 155 -- docker ps
pct exec 156 -- systemctl status caddy --no-pager

# 4. Le WAF répond-il ?
pct exec 155 -- curl -s -o /dev/null -w '%{http_code}\n' http://10.30.30.155:8080/

# 5. CT116 et CT111 doivent rester arrêtés
pct status 116
pct status 111
# ne pas les démarrer sans décision : leurs tunnels ne sont plus utilisés

# 6. Campagne
./test-vhosts.sh
```

### Vérification

- `refonte-firewall.service` est `active (exited)` et `enabled` ;
- `REFONTE` est en position 1, `PVEFW-FORWARD` en 2 ;
- 65/65 ;
- 302 sur les admins ;
- `iptables -vnL REFONTE` avec des compteurs qui bougent.

### La persistance

Le service est déployé :

```ini
# /etc/systemd/system/refonte-firewall.service
[Unit]
Description=Refonte 2026-11-24 - firewall edge/DMZ/admin
After=network-online.target pve-firewall.service
Wants=network-online.target pve-firewall.service

[Service]
Type=oneshot
ExecStart=/root/refonte-firewall.sh --strict
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable --now refonte-firewall.service
```

Deux points à surveiller :

- si `iptables -L FORWARD` ne montre pas `REFONTE` en premier, le service a
  échoué : `journalctl -u refonte-firewall.service -n 50` ;
- `--strict` est volontaire au boot. Un mode permissif ne doit pas revenir par
  accident après un redémarrage.

Tester avec un `reboot` planifié, pas en pleine journée. Tant que ce test n'a
pas été fait, la persistance est **configurée**, pas **prouvée**.

---

## RB-08 — Ajouter un backend hors DMZ

**Objectif** : router un vhost vers un service sur `vmbr2`, `vmbr1` ou `vmbr3`.
**Durée** : 15 min. **Risque** : moyen.

```bash
# 1. Vérifier que Caddy peut joindre le backend
pct exec 156 -- curl -s -o /dev/null -w '%{http_code}\n' --max-time 3 \
  http://22.22.22.142:8080/
# si timeout : la règle firewall manque

# 2. Ajouter la règle dans le script, AVANT les DROP
#    iptables -A REFONTE -s 10.20.30.156 -d 22.22.22.142 -p tcp --dport 8080 -j ACCEPT

# 3. Rejouer le script (idempotent)
/usr/local/sbin/refonte-firewall.sh

# 4. Ajouter le bloc Caddy
cat >> /etc/caddy/Caddyfile <<'EOF'

@nouveau host nouveau.example.com
handle @nouveau {
    reverse_proxy 22.22.22.142:8080
}
EOF
caddy validate --config /etc/caddy/Caddyfile && systemctl restart caddy

# 5. Ajouter le nom dans BunkerWeb (RB-01), tester
curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: nouveau.example.com' \
  http://10.30.30.155:8080/
```

### Vérification

- `200` via le WAF ;
- `iptables -vnL REFONTE` : le compteur de la nouvelle règle monte ;
- le trafic inverse (backend → Caddy) n'est pas nécessaire : Caddy initie.

### Retour arrière

Retirer le bloc Caddy et la ligne firewall, rejouer le script. Le backend
redevient injoignable depuis la DMZ, ce qui est l'état par défaut.

### Pièges

- Mettre la règle après le `DROP` : elle ne sert à rien.
- Oublier que `-i vmbr4 -o vmbr4 DROP` ne concerne pas ce flux (Caddy →
  vmbr2), mais que le `DROP` générique de fin, si vous en ajoutez un, le ferait.
- Ouvrir tout le bridge « pour que ça marche » : c'est exactement ce qu'on
  essaie d'éviter.

---

## Aide-mémoire

| Besoin | Runbook |
|---|---|
| Nouveau sous-domaine | RB-01 |
| Nouveau client | RB-02 |
| Certificat expiré | RB-03 |
| 403 / 502 WAF | RB-04 |
| Retour à l'ancien tunnel | RB-05 |
| Protéger une console | RB-06 |
| Après un reboot | RB-07 |
| Backend hors DMZ | RB-08 |

## Suites

- [13 — Leçons et limites](13-lecons-et-limites.md)
- [11 — Tests](11-tests-et-validation.md)
