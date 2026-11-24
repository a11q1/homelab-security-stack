# 08 — Firewall et segmentation

[← PKI](07-pki-interne-caddy.md) · [README](../README.md) · [Accès admin →](09-acces-admin-cloudflare.md)

## Pourquoi sur l'hôte, et pas dans les conteneurs

Le trafic entre deux LXC sur des bridges différents passe par l'hôte Proxmox.
C'est lui qui route. C'est donc lui qui voit tous les flux inter-zones, et lui
qui peut les refuser.

Le filtrage Proxmox par conteneur (`firewall=1` + `.fw`) n'est **pas effectif**
pour le trafic bridgé dans cette configuration : les paquets ne remontent pas
systématiquement dans le netfilter du conteneur, et `br_netfilter` n'est pas
armé. Constaté à l'audit : les règles `.fw` de CT155/CT156 existent, mais c'est
la chaîne `FORWARD` de l'hôte qui fait le travail.

Conséquence assumée : **la source de vérité est l'hôte**, pas les `.fw`.

## Les deux chaînes

```text
FORWARD
  └── REFONTE                    (position 1)
        1. ESTABLISHED,RELATED
        2. DNS / NTP edge
        3. connecteurs → WAF:8080
        4. WAF → Caddy:443
        5. Caddy → backends nominatifs
        6. SSH inter-CT (admin only)
        7. DROP WAF:8080 pour les autres
        8. DROP Caddy:443 pour les autres
        9. DROP intra-DMZ
       10. RETURN
  └── règles historiques (vmbr4, vmbr3, wg0, DNAT…)

INPUT
  └── REFONTE-IN                 (position 1)
        1. ESTABLISHED,RELATED
        2. lo
        3. LAN / WG / Tailscale / CT131 → 22, 8006, 3128
        4. Checkmk → 6556, 9100
        5. CT147 / CT132 → 8811, 9099
        6. DROP des ports admin depuis vmbr1/2/3/5
        7. RETURN
  └── règles historiques
```

`RETURN` à la fin est volontaire : la chaîne ne remplace pas l'existant, elle
**ajoute une couche en tête**. Les règles `vmbr4`, les DNAT du CTF et WireGuard
continuent de fonctionner. C'est ce qui permet une bascule sans tout réécrire.

## Le script

Version complète et paramétrée :
[`examples/refonte-firewall.sh.example`](../examples/refonte-firewall.sh.example).

Extrait commenté :

```bash
CHAIN=REFONTE
iptables -N "$CHAIN" 2>/dev/null || true
iptables -F "$CHAIN"                 # idempotent : on repart de zéro

# 1. Ne jamais casser les réponses
iptables -A "$CHAIN" -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 2. L'edge a besoin de DNS et NTP
for ip in 10.20.30.155 10.20.30.156; do
  iptables -A "$CHAIN" -s "$ip" -p udp --dport 53 -j ACCEPT
  iptables -A "$CHAIN" -s "$ip" -p tcp --dport 53 -j ACCEPT
  iptables -A "$CHAIN" -s "$ip" -p udp --dport 123 -j ACCEPT
done

# 3. Seuls les connecteurs parlent au WAF
for src in 22.22.22.20 10.20.30.40 10.10.10.116 22.22.22.116; do
  iptables -A "$CHAIN" -s "$src" -d 10.30.30.155 -p tcp --dport 8080 -j ACCEPT
done

# 4. Seul le WAF parle à Caddy
iptables -A "$CHAIN" -s 10.20.30.155 -d 10.20.30.156 -p tcp --dport 443 -j ACCEPT

# 5. Caddy → backends, un couple (IP, port) par ligne
iptables -A "$CHAIN" -s 10.20.30.156 -d 10.20.30.147 -p tcp --dport 80 -j ACCEPT
iptables -A "$CHAIN" -s 10.20.30.156 -d 10.20.30.126 -p tcp --dport 3000 -j ACCEPT
# …

# 7bis. Fermeture : tout ce qui n'est pas nommé au-dessus
iptables -A "$CHAIN" -d 10.20.30.156 -p tcp --dport 443 -j DROP
iptables -A "$CHAIN" -d 10.30.30.155 -p tcp --dport 8080 -j DROP
iptables -A "$CHAIN" -i vmbr4 -o vmbr4 -j DROP

iptables -A "$CHAIN" -j RETURN
```

Trois propriétés :

- **Idempotent** : `-F` puis reconstruction. Rejouable cent fois.
- **En tête** : `-D FORWARD -j REFONTE` puis `-I FORWARD 1 -j REFONTE`, donc
  jamais deux copies.
- **Non destructif** : ne touche pas aux règles historiques.

## Ce que ça bloque concrètement

| Flux | Avant | Après |
|---|---|---|
| CT lambda → WAF:8080 | ouvert | **refusé** (sauf connecteurs) |
| CT lambda → Caddy:443 | ouvert | **refusé** (sauf WAF) |
| DMZ → DMZ non listé | ouvert | **refusé** |
| DMZ → LAN | refusé (déjà) | refusé |
| CT → hôte:8006 | ouvert | **refusé** depuis vmbr1/2/3/5 |
| CT → hôte:22 | ouvert | **refusé** depuis vmbr1/2/3/5 |
| SSH inter-CT | ouvert entre tous les conteneurs | **allowlist admin**, le reste `DROP` |
| wg0 → partout | ouvert | **toujours ouvert** (limite) |
| vmbr1 ↔ vmbr2 | ouvert | **toujours ouvert** (limite) |

## Le mode strict

```bash
./refonte-firewall.sh --strict
```

Ajoute quatre choses :

1. Les connecteurs gardent des accès directs nécessaires : DNS, 443 sortant,
   7844, Proxmox `:8006`, honeypot, webhook Stripe.
2. `DROP` de tout le trafic sortant de Caddy non listé.
3. `DROP` de tout le trafic sortant du WAF non listé.
4. `DROP` de tout le trafic entrant depuis `vmbr5` non listé.

C'est le mode à utiliser une fois que la matrice est stable. En cas de doute,
rester en mode normal : le strict transforme une règle manquante en panne.

## L'ordre, encore

Le 13 novembre, une règle `-I` insérée **sous** les `DROP` a coupé les réponses
vers `22.22.22.0/24`. Le pare-feu était « correct » sur le papier, mais l'ordre
rendait la règle inatteignable.

Deux garde-fous :

```bash
# 1. Compter les paquets par règle
iptables -vnL REFONTE --line-numbers

# 2. Vérifier qu'un ACCEPT est bien avant le DROP générique
iptables -L REFONTE -n --line-numbers | grep -E 'ACCEPT|DROP'
```

Une règle `DROP` avec un compteur élevé juste après un `ACCEPT` à zéro signifie
que l'`ACCEPT` ne matche pas ce qu'on croit.

## Isolation intra-DMZ : la conséquence à connaître

```bash
iptables -A REFONTE -i vmbr4 -o vmbr4 -j DROP
```

Cette règle empêche un conteneur DMZ compromis de rebondir vers ses voisins.
Mais elle casse aussi les proxys internes : un vhost nginx de CT147 qui faisait
`proxy_pass http://10.20.30.126:3000` ne passe plus, puisque CT147 n'est pas
dans la liste des ACCEPT.

C'est voulu : **le routage des vhosts doit passer par Caddy**, qui est
explicitement autorisé. Les vhosts `support`, portfolio et `preview-client` ont été
rebranchés dans le Caddyfile pour cette raison. Tout nouveau vhost qui pointe
directement d'un CT DMZ vers un autre CT DMZ doit être ajouté à la main, sinon
il tombera en timeout.

## Le cas des `.fw` par conteneur

CT155 et CT156 ont `firewall=1` et un fichier `.fw` avec `policy_in: DROP`.
C'est une ceinture supplémentaire, pas la source de vérité :

```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT

[RULES]
IN ACCEPT -source 22.22.22.20 -p tcp -dport 22
IN ACCEPT -source 192.168.1.0/24 -p tcp -dport 22
IN ACCEPT -source 22.22.22.150 -p tcp -dport 6556
IN ACCEPT -source 10.20.30.155 -p tcp -dport 443
IN ACCEPT -p icmp
```

Ces règles sont utiles si un jour `br_netfilter` est armé et que Proxmox filtre
réellement. Aujourd'hui, elles documentent l'intention.

## Persistance

Le script est idempotent, mais un reboot ne le rejoue pas tout seul. Une unité
systemd s'en charge :

```ini
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
systemctl status refonte-firewall.service --no-pager
```

Deux détails qui comptent :

- **`--strict`** : au boot, on veut la version fermée, pas la version de test.
- **`After=pve-firewall.service`** : le script replace `PVEFW-FORWARD` juste
  après `REFONTE`, pour que les règles par conteneur des CT155/156 restent
  évaluées avant les `ACCEPT` historiques.

Vérifier après un reboot planifié :

```bash
iptables -L FORWARD -n --line-numbers | head -4
# REFONTE en 1, PVEFW-FORWARD en 2
systemctl is-enabled refonte-firewall.service
iptables -vnL REFONTE --line-numbers
```

## Tester

```bash
# La chaîne est-elle en tête ?
iptables -L FORWARD -n --line-numbers | head -3

# Les compteurs bougent-ils ?
iptables -vnL REFONTE --line-numbers

# Un CT quelconque peut-il joindre le WAF ?
pct exec 142 -- curl -s -o /dev/null -w '%{http_code}\n' --max-time 3 http://10.30.30.155:8080/
# attendu : refusé (000)

# Le WAF peut-il joindre Caddy ?
pct exec 155 -- curl -sk -o /dev/null -w '%{http_code}\n' https://10.20.30.156/health
# attendu : 200

# La DMZ peut-elle joindre le LAN ?
pct exec 156 -- curl -s -o /dev/null -w '%{http_code}\n' --max-time 3 http://192.168.1.30:8006
# attendu : 000

# Les ports admin de l'hôte depuis vmbr2
pct exec 142 -- timeout 3 bash -c '</dev/tcp/22.22.22.1/8006' && echo OUVERT || echo FERMÉ
# attendu : FERMÉ
```

## Ce que ce firewall ne fait pas

- Il ne filtre pas le trafic **à l'intérieur** d'un même bridge (sauf DMZ).
- Il ne voit pas le trafic chiffré (TLS) : c'est le rôle du WAF, pas du L4.
- Il ne protège pas contre une compromission de l'hôte lui-même.
- Il dépend d'un script : pas d'IaC, pas de tests automatiques.
- Il n'est pas versionné en IaC : le script est la source de vérité.

## Suites

- [09 — Accès admin](09-acces-admin-cloudflare.md)
- [11 — Tests](11-tests-et-validation.md)
- [12 — Runbooks](12-runbooks.md)
