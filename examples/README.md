# Exemples de configuration

Tous ces fichiers sont des **modèles**. Ils sont volontairement génériques :
remplacez les IP, domaines, chemins et noms avant de les utiliser.

| Fichier | Rôle | À adapter |
|---|---|---|
| `docker-compose.bunkerweb.yml` | stack WAF complète | IP de bind, secrets, image |
| `bw.env.example` | config BunkerWeb | `SERVER_NAME`, IP connecteurs |
| `Caddyfile.example` | routage Host + TLS interne | domaines, backends, chemins PKI |
| `gen-pki.sh.example` | root / intermediate / leaf | chemins, SAN, durées |
| `refonte-firewall.sh.example` | chaînes `REFONTE` / `REFONTE-IN` | IP, bridges, backends |
| `cloudflared-ingress.example.json` | ingress d'un tunnel | tunnel ID, hostnames, origines |

## Valider avant de déployer

```bash
# Compose
docker compose -f docker-compose.bunkerweb.yml config >/dev/null

# Caddy
caddy validate --config Caddyfile.example

# Tunnel
cloudflared tunnel ingress validate --config cloudflared-ingress.example.json

# Firewall (syntaxe seulement)
bash -n refonte-firewall.sh.example

# PKI (dans un répertoire de test)
bash -n gen-pki.sh.example
# ./gen-pki.sh.example /tmp/pki-test && openssl verify ...

# JSON
jq empty cloudflared-ingress.example.json
```

## Ce qui n'est pas dans ces fichiers

- Aucune clé privée, aucun certificat.
- Aucun token de tunnel, token API ou mot de passe.
- Aucun ID de tunnel, de zone ou de compte Cloudflare.
- Aucune IP publique, aucun email personnel.

Les valeurs sensibles sont des variables d'environnement (`${BW_API_TOKEN}`,
`${BW_DB_PASSWORD}`) ou des placeholders (`TUNNEL_ID_PLACEHOLDER`).

## Ordre de mise en place conseillé

1. PKI : générer la root et l'intermédiaire, vérifier la chaîne.
2. Caddy : un vhost témoin, test direct par IP.
3. BunkerWeb : `MULTISITE=yes`, un seul nom, test depuis un conteneur.
4. Firewall : allowlist, en mode normal d'abord.
5. Tunnel : un seul hostname, puis campagne complète.
6. Access : après que le hostname répond, jamais avant.
