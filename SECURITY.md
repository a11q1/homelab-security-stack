# Politique de sécurité

## Périmètre

Ce dépôt contient des **exemples de configuration** et de la documentation.
Il ne contient aucune infrastructure live, aucun secret et aucun accès.

Les fichiers `examples/` sont des modèles : ils ne sont pas déployés tels quels.
Les adresses RFC1918, les noms de conteneurs et les domaines d'exemple sont
volontairement conservés pour la lisibilité.

## Signaler une vulnérabilité

**Ne pas ouvrir d'issue publique.** Utilisez l'onglet *Security* → *Report a
vulnerability* du dépôt GitHub (GitHub Security Advisories, canal privé).

Merci d'inclure :

- le fichier ou la section concernée ;
- ce qui peut mal tourner concrètement ;
- une reproduction minimale si possible ;
- la version ou le commit.

Accusé de réception sous **72 h**. Ce projet est maintenu par une personne,
en marge d'un BTS SIO : pas de SLA contractuel, mais un traitement sérieux.

## Ce qui est hors périmètre

- Les services tiers (Cloudflare, BunkerWeb, Caddy, Proxmox) : signalez aux
  éditeurs concernés.
- Les vulnérabilités applicatives des sites servis par ce lab.
- Les problèmes qui supposent un accès physique à l'hyperviseur.

## Principes du dépôt

1. **Aucun secret versionné.** `.gitleaks.toml` tourne en pre-commit et en CI.
2. **Placeholders explicites.** `${VAR}`, `change-me`, `example.com`.
3. **Configs lisibles avant d'être complètes.** Une règle commentée vaut mieux
   qu'une règle magique.
4. **Default deny.** Tout ce qui n'est pas listé est refusé, et le refus est
   documenté.
5. **Réversibilité.** Chaque changement a une procédure de rollback dans
   [docs/12-runbooks.md](docs/12-runbooks.md).

## Secrets : ce qui ne doit jamais apparaître ici

| Élément | Exemple | Remplacement |
|---|---|---|
| Token de tunnel Cloudflare | chaîne base64 de 40+ caractères | `TUNNEL_TOKEN` |
| Token API BunkerWeb | `API_TOKEN=` | `BW_API_TOKEN` |
| Mot de passe base BunkerWeb | `DATABASE_URI` | `BW_DB_PASSWORD` |
| Clé privée PKI | bloc PEM `BEGIN EC PRIVATE KEY` | jamais commité |
| Identité age / SOPS | `AGE-SECRET-KEY-1…` | coffre hors ligne |
| ID de tunnel / zone / compte | UUID | `TUNNEL_ID`, `ZONE_ID` |
| Email personnel | adresse réelle | contact via GitHub |
| IP publique | adresse du foyer | RFC1918 ou doc |

## Divulgation responsable

Si vous trouvez un secret oublié dans l'historique : ne le copiez pas, ne
l'exploitez pas, signalez-le. La procédure de réponse (révocation → rotation →
purge d'historique) est décrite dans
[docs/10-secrets-et-rotation.md](docs/10-secrets-et-rotation.md#incident--un-secret-a-fuité).
