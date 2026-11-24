# 10 — Secrets et rotation

[← Accès admin](09-acces-admin-cloudflare.md) · [README](../README.md) · [Tests →](11-tests-et-validation.md)

## Le principe

Un secret qui a touché git est un secret compromis. Même après suppression :
l'historique, les forks, les caches CI et les clones le gardent.

Trois règles :

1. **Rien de secret dans le dépôt.** Le repo public ne contient que des
   placeholders.
2. **Un secret, un endroit.** Pas de copie dans un wiki, un ticket, un message.
3. **Rotation planifiée.** Un secret qu'on ne sait pas remplacer est un secret
   qu'on ne peut pas révoquer.

## Inventaire

| # | Secret | Où il vit | Portée | Rotation |
|---|---|---|---|---|
| S1 | Token de tunnel Cloudflare | `--token-file` (600) + copie SOPS | créer un connecteur sur le tunnel | 90 j ou incident |
| S2 | Token API BunkerWeb | `refonte-secrets.enc.yaml` (SOPS) | config du WAF | 90 j |
| S3 | Mot de passe MariaDB BunkerWeb | `refonte-secrets.enc.yaml` (SOPS) | base du WAF | 180 j |
| S4 | Clé root PKI | CT156 + `pki.tgz.age` | signer une intermediate | 10 ans (ou compromission) |
| S5 | Clé intermediate PKI | CT156 + `pki.tgz.age` | signer les feuilles | 5 ans |
| S6 | Clé leaf Caddy | CT156 + `pki.tgz.age` | TLS interne | 825 j |
| S7 | Token API Cloudflare | `refonte-secrets.enc.yaml` (SOPS) | DNS, tunnels, Access | 90 j |
| S8 | Service token Access | app appelante | webhook machine | 180 j |
| S9 | `.env` applicatifs | chaque CT | apps | à la demande |
| S10 | Clé DKIM | zone DNS / backup | signature mail | 1 an |

## L'état réel après la refonte

Cette fois, l'hygiène des secrets a suivi la bascule. Ce qui est fait, et ce
qui ne l'est pas :

| Point | État |
|---|---|
| Clé age | `/root/.config/sops/age/keys.txt` sur CT131, `600` |
| Secrets edge | `refonte-secrets.enc.yaml`, SOPS + age, hors git |
| Sauvegarde PKI | `pki.tgz.age`, archive chiffrée |
| Tokens cloudflared | `--token-file` en `600` (CT132) ; CT116 et CT111 arrêtés |
| `bw.env` | `640` sur CT155, généré depuis les secrets, hors repo |
| `.env` applicatifs | encore dispersés dans les CT, non chiffrés |
| Rotation | manuelle, pas planifiée |
| Clé age hors ligne | **pas encore** — à sauvegarder |

Le point qui reste le plus fragile n'est pas le chiffrement, c'est la **clé
age** : si CT131 brûle et que la clé n'est pas sauvegardée ailleurs, les
secrets chiffrés sont perdus. Une sauvegarde chiffrée sans sauvegarde de clé
n'est pas une sauvegarde.

## Le déploiement SOPS + age

### Ce qui est chiffré

| Fichier | Contenu |
|---|---|
| `refonte-secrets.enc.yaml` | mot de passe et token API BunkerWeb, tokens des 3 tunnels cloudflared, token API Cloudflare |
| `pki.tgz.age` | archive de `/etc/caddy/pki` : root, intermediate, leaf |

### Où vivent les fichiers

```text
/root/refonte-2026-11-24/secrets/        # hors du dépôt public
├── .sops.yaml
├── refonte-secrets.enc.yaml             # 600
└── pki.tgz.age                          # 640

/root/.config/sops/age/keys.txt          # CT131, 600
```

Rien de tout ça n'est dans ce dépôt. Le dépôt ne contient que des placeholders.

### Configuration

```yaml
# .sops.yaml
creation_rules:
  - path_regex: .*\.yaml$
    age: age1REMPLACER_PAR_VOTRE_CLE_PUBLIQUE
```

Une seule règle suffit tant qu'il n'y a qu'un destinataire. Le jour où la CI
doit déchiffrer, ajouter sa clé publique à la liste.

### Usage

```bash
# Éditer un fichier chiffré (déchiffre à la volée, rechiffre à la sortie)
SOPS_AGE_KEY_FILE=/root/.config/sops/age/keys.txt \
  sops /root/refonte-2026-11-24/secrets/refonte-secrets.enc.yaml

# Déchiffrer dans un fichier temporaire, jamais dans le repo
SOPS_AGE_KEY_FILE=/root/.config/sops/age/keys.txt \
  sops -d /root/refonte-2026-11-24/secrets/refonte-secrets.enc.yaml > /run/edge.yaml
chmod 600 /run/edge.yaml

# Vérifier que le fichier est bien chiffré (aucune valeur en clair)
grep -c 'ENC\[AES256_GCM' /root/refonte-2026-11-24/secrets/refonte-secrets.enc.yaml

# Vérifier qu'aucun clair ne traîne dans le repo
git grep -nE '(TOKEN|PASSWORD|SECRET)=' -- ':!*.example' || true
```

### Ce qu'on met dedans

```yaml
# structure — les valeurs sont des blocs ENC[...] une fois chiffrées
bunkerweb:
    db_password: ENC[AES256_GCM,data:...,type:str]
    api_token: ENC[AES256_GCM,data:...,type:str]
cloudflared:
    entreprises_token: ENC[AES256_GCM,data:...,type:str]
    wildcard_token: ENC[AES256_GCM,data:...,type:str]
    console_token: ENC[AES256_GCM,data:...,type:str]
cloudflare:
    api_token: ENC[AES256_GCM,data:...,type:str]
```

Les noms de clés restent lisibles : on sait ce qu'on rotate, sans voir la valeur.

### Ce qui n'est pas encore couvert

- Les `.env` des applications (bots, apps Node, Python) : dispersion inchangée.
- La clé DKIM dans l'export DNS : à traiter comme un secret, pas comme une config.
- La rotation automatique : SOPS chiffre, il ne planifie rien.
- La distribution de la clé age aux autres machines : manuelle.
- La révocation : pas de CRL pour la PKI, pas de rotation forcée des tokens.

## Rotation

| Secret | Fréquence | Procédure courte |
|---|---|---|
| Token tunnel | 90 j | nouveau token, `--token-file` 600, restart, révoquer l'ancien |
| Token API WAF | 90 j | régénérer dans l'UI, mettre à jour SOPS, régénérer l'env, redémarrer le scheduler |
| Mot de passe DB WAF | 180 j | `ALTER USER`, mettre à jour SOPS, régénérer l'env, redémarrer |
| Leaf Caddy | 825 j | [RB-03](12-runbooks.md#rb-03--rotation-du-certificat-caddy) |
| Intermediate | 5 ans | re-signer les feuilles, double trust pendant la transition |
| Root | 10 ans | nouvelle hiérarchie, distribution, retrait de l'ancienne |
| Token API Cloudflare | 90 j | nouveau token scopé, tester, révoquer |
| Service token Access | 180 j | nouveau token, déployer, supprimer l'ancien |

**Règle d'or** : toujours créer le nouveau avant de révoquer l'ancien. Une
rotation qui commence par une révocation est une panne.

### Rotation d'un token de tunnel sans coupure

```bash
# 1. nouveau token côté Cloudflare (même tunnel)
# 2. sur le connecteur : sauvegarder l'unité
cp /etc/systemd/system/cloudflared-xxx.service /root/cloudflared-xxx.service.bak

# 3. écrire le token dans un fichier 600
install -m 600 /dev/null /root/.cloudflared/xxx.token
printf '%s' "$NEW_TOKEN" > /root/.cloudflared/xxx.token

# 4. vérifier que l'unité lit le fichier, pas la ligne de commande
grep ExecStart /etc/systemd/system/cloudflared-xxx.service
# doit contenir : --token-file /root/.cloudflared/xxx.token

systemctl daemon-reload
systemctl restart cloudflared-xxx
journalctl -u cloudflared-xxx -n 30 --no-pager
curl -s http://127.0.0.1:20241/ready   # 4 connexions attendues

# 5. révoquer l'ancien token
```

`--token-file` lit le token depuis un fichier `600` : il n'apparaît ni dans
`ps`, ni dans `journalctl`. C'est le correctif au constat « token visible dans
`ps` » sur CT132, CT116 et CT111. CT116 et CT111 ont ensuite été arrêtés, leurs
tunnels n'étant plus utilisés.

## gitleaks

### Local

```bash
# Avant commit
gitleaks protect --staged --config .gitleaks.toml

# Sur tout l'historique
gitleaks detect --source . --config .gitleaks.toml --redact

# Sur un dépôt distant déjà poussé
gitleaks detect --source . --log-opts="--all"
```

### Pre-commit

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks
```

### CI

```yaml
# .github/workflows/secret-scan.yml
name: secret-scan
on: [push, pull_request]
jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Checklist avant publication

- [ ] `gitleaks detect --source . --redact` ne retourne rien.
- [ ] `git grep -iE 'token|password|secret|apikey'` : uniquement des placeholders.
- [ ] Aucun fichier `.env`, `.key`, `.pem`, `.crt` réel.
- [ ] Aucun UUID de tunnel, zone ou compte.
- [ ] Aucune adresse email personnelle.
- [ ] Aucune IP publique.
- [ ] Les captures d'écran sont relues (barre d'URL, tokens visibles).
- [ ] L'historique git est propre, pas seulement le dernier commit.

## Incident : un secret a fuité

L'ordre compte. Ne pas inverser.

```text
1. Révoquer      → le secret ne vaut plus rien
2. Remplacer     → nouveau secret déployé
3. Vérifier      → le service fonctionne avec le nouveau
4. Purger        → réécrire l'historique (git filter-repo), force push
5. Prévenir      → si données personnelles ou clients concernés
6. Documenter    → post-mortem court, sans blâme
```

Purger l'historique **avant** de révoquer ne sert à rien : pendant la fenêtre,
le secret est encore valide. Et un fork peut l'avoir déjà.

### Cas concret de la refonte

Un token de tunnel était visible dans la ligne de commande d'un conteneur
(`ps aux`). Ce n'est pas une fuite publique, mais c'est une exposition locale :
tout processus du conteneur pouvait le lire.

Correctif appliqué : passage à `--token-file` en `600` sur CT132, CT116 et
CT111, puis arrêt de CT116 et CT111 dont les tunnels n'étaient plus utilisés.
Les tokens sont aussi dans le fichier SOPS. **Reste à faire** : rotation
effective des tokens qui ont été exposés — les sortir de `ps` ne les invalide
pas.

## Ce qui ne doit jamais aller dans git

| Interdit | Pourquoi |
|---|---|
| Token de tunnel | permet de créer un connecteur |
| Token API Cloudflare | DNS, tunnels, Access |
| Clé privée PKI | signe des certificats |
| `.env` applicatif | souvent la clé de tout le reste |
| Export DNS complet | contient DKIM, SPF, parfois des secrets |
| `cloudflared/*.json` | credentials du tunnel |
| Sauvegarde `iptables-save` | révèle le plan d'adressage et les trous |
| Capture d'écran de l'UI WAF | tokens, IP, hostnames |

## Suites

- [11 — Tests et validation](11-tests-et-validation.md)
- [SECURITY.md](../SECURITY.md)
- [13 — Leçons et limites](13-lecons-et-limites.md)
