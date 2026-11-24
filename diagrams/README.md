# Schémas Mermaid

Sources des schémas du dépôt. Ils sont aussi intégrés dans les documents
(`docs/`) quand c'est utile.

| Fichier | Contenu |
|---|---|
| `01-avant.mmd` | l'état d'avant : tunnels dispersés, réseau plat, SSRF |
| `02-cible.mmd` | l'architecture après : zones, edge, DMZ, backends |
| `03-flux-requete.mmd` | le chemin d'une requête HTTP, étape par étape |
| `04-pki.mmd` | la hiérarchie root / intermediate / leaf |
| `05-zones-firewall.mmd` | les zones et les décisions ALLOW / DENY |

## Rendu

- GitHub affiche Mermaid dans les blocs ` ```mermaid ` des fichiers Markdown,
  pas directement les `.mmd`.
- Pour obtenir un SVG : `npx -y @mermaid-js/mermaid-cli -i 01-avant.mmd -o 01-avant.svg`
- Ou coller le contenu sur <https://mermaid.live>.

## Conventions

- Nœuds entre guillemets dès qu'il y a un espace, un `:` ou un `/`.
- Pas de HTML complexe : `<br/>` pour les retours à la ligne, `&lt;` pour `<`.
- Les flèches pointillées (`-.->`) marquent un flux refusé, absent ou différé.
- Les couleurs sont indicatives : le texte du lien porte la décision.
