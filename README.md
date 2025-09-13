# Unbound + AdGuard Home — DNS récursif, DNSSEC, sans forwarders (Ansible)

Ce dépôt déploie une **stack DNS auto‑hébergée et indépendante** :
- **Unbound** — résolveur récursif qui part de la **racine (root hints)**, avec **DNSSEC**.
- **AdGuard Home** — filtrage, réécritures locales, interface web conviviale.
- **Mini exporter Prometheus** pour Unbound (endpoint HTTP `/metrics`).

**Flux** : les clients du LAN interrogent **AdGuard**, qui interroge **Unbound**.  
Unbound résout **directement** depuis la racine, **sans forwarders** (ni Google, ni Cloudflare).

---

## 🧭 Architecture (exemple)

```
Clients LAN  ─────►  AdGuard (192.0.2.6:53)  ─────►  Unbound (192.0.2.7:53)
                                 ▲
                                 │  (UI AdGuard, filtres, rewrites)
```
> Les IPs d’exemple utilisent le préfixe **192.0.2.0/24** réservé à la doc. Remplacez‑les par vos IPs locales en pratique.

---

## ⚙️ Installation (pas à pas)

### 0) Pré‑requis
- Une machine d’admin avec **Ansible ≥ 2.12**.
- Une VM/LXC **Debian 12** (ou Ubuntu) pour **Unbound**, accès **SSH par clé**.
- Une VM/host pour **AdGuard Home** (déjà installée ou à installer).

### 1) Récupérer le dépôt et préparer vos fichiers **locaux**
```bash
git clone https://github.com/<vous>/ansible-dns-stack-unbound-adguard.git
cd ansible-dns-stack-unbound-adguard

# Copier les exemples vers vos fichiers non suivis par Git
cp inventory.example.ini inventory.ini
cp group_vars/all.example.yml group_vars/all.yml
```

Éditez **`inventory.ini`** et **`group_vars/all.yml`** avec **vos** IPs/domaines (ex. `192.168.1.6/7`, `example.lan`).

### 2) Vérifier la connexion Ansible
```bash
ansible -i inventory.ini unbound -m ping
```

### 3) Déployer
```bash
# Dry-run (vérifie ce qui serait fait)
ansible-playbook -i inventory.ini site.yml --check --diff

# Lancer le déploiement
ansible-playbook -i inventory.ini site.yml --diff
```

### 4) Configurer AdGuard → **Upstream DNS**
Dans l’UI d’AdGuard : **Settings → DNS settings → Upstream DNS servers** → ajoutez **`192.0.2.7:53`** (UDP+TCP).

### 5) Tests rapides
```bash
# Sur Unbound (cible)
dig @127.0.0.1 cloudflare.com +trace | head -n 30   # racine -> .com -> NS du domaine
dig @127.0.0.1 example.com +dnssec | head -n 15     # NOERROR + flag AD
dig @127.0.0.1 dnssec-failed.org +dnssec | head -n 15  # SERVFAIL attendu

# Sur AdGuard (host AdGuard)
dig @127.0.0.1 example.com +dnssec | head -n 15
dig @127.0.0.1 dnssec-failed.org +dnssec | head -n 15

# Sur un client Windows (via AdGuard seulement)
Resolve-DnsName example.com -Server 192.0.2.6 -DnsSecOk
Resolve-DnsName dnssec-failed.org -Server 192.0.2.6 -DnsSecOk   # doit échouer (SERVFAIL)
```

---

## 📚 Référence des paramètres (variables Ansible)

Toutes les variables se trouvent dans **`group_vars/all.yml`**.  
Les valeurs ci‑dessous correspondent aux **défauts** fournis dans `all.example.yml`. Adaptez‑les à votre contexte.

### Réseau & écoute
| Variable | Type | Défaut | Description / possibilités |
|---|---|---:|---|
| `unbound_listen_ipv4` | list\<string> | `["192.0.2.7", "127.0.0.1"]` | Adresses IPv4 sur lesquelles Unbound écoute. Mettez l’IP LAN d’Unbound + `127.0.0.1`. |
| `unbound_listen_ipv6` | list\<string> | `[]` | Adresses IPv6 à écouter. Laissez vide si vous n’avez pas d’IPv6 routé. |
| `unbound_port` | int | `53` | Port d’écoute d’Unbound. Peut être changé si port 53 déjà pris. |
| `unbound_do_ip6` | bool | `false` | Active la résolution IPv6 sortante. Passez à `true` si votre réseau a IPv6. |
| `unbound_do_udp` | bool | `true` | Autorise UDP. Conservez `true`. |
| `unbound_do_tcp` | bool | `true` | Autorise TCP fallback. Conservez `true`. |

### ACL (qui peut interroger Unbound)
| Variable | Type | Défaut | Description / possibilités |
|---|---|---:|---|
| `unbound_acl_allow` | list\<string> | `["192.0.2.6/32","127.0.0.1/32"]` | Plages autorisées (CIDR). Par défaut : **AdGuard** + loopback. Exemple pour tout un LAN : `["192.168.1.0/24","127.0.0.1/32"]`. **Évitez** d’ouvrir à tout le monde. |

### DNSSEC & racine
| Variable | Type | Défaut | Description / possibilités |
|---|---|---:|---|
| `unbound_auto_trust_anchor_file` | string | `/var/lib/unbound/root.key` | Fichier **d’ancre DNSSEC** (géré par Unbound). |
| `unbound_root_hints` | string | `/var/lib/unbound/root.hints` | Fichier **root hints** (liste des serveurs racine). |
| `unbound_manage_root_hints` | bool | `true` | Si `true`, le rôle installe un script + cron **hebdo** pour maintenir `root.hints` à jour. Si `false`, rien n’est géré. |

*Notes* :  
- Le rôle tente d’abord un téléchargement HTTP depuis InterNIC. Si le DNS n’est pas dispo au tout début, il **bascule** sur un `dig @9.9.9.9 . NS` pour générer un `root.hints` fonctionnel.  
- L’**ancre DNSSEC** est initialisée si absente (commande `unbound-anchor` via les paquets).

### Caching & performance
| Variable | Type | Défaut | Description / possibilités |
|---|---|---:|---|
| `unbound_cache_min_ttl` | int | `300` | TTL minimal en cache (secondes). |
| `unbound_cache_max_ttl` | int | `14400` | TTL maximal en cache (secondes). |
| `unbound_rrset_cache_size` | string | `"256m"` | Taille du cache RRset. |
| `unbound_msg_cache_size` | string | `"128m"` | Taille du cache des messages. |
| `unbound_so_rcvbuf` | string | `"1m"` | Socket receive buffer. Si journal indique *not granted*, voir **sysctl** ci‑dessous. |
| `unbound_edns_buffer_size` | int | `1232` | Taille EDNS (utile pour éviter fragmentation). 1232 est un bon compromis. |
| `unbound_serve_expired` | bool | `true` | Servez des réponses **expirées** en cas d’indisponibilité amont. |
| `unbound_serve_expired_ttl` | int | `3600` | TTL servi pour une réponse expirée. |

**Astuce** : pour supprimer l’alerte `so-rcvbuf … not granted` :  
- soit **baisser** `unbound_so_rcvbuf` (ex. `"425984"`),  
- soit **augmenter** `net.core.rmem_max` via `sysctl`.

### Journalisation & verbosité
| Variable | Type | Défaut | Description / possibilités |
|---|---|---:|---|
| `unbound_logfile` | string | `/var/log/unbound.log` | Fichier de logs. |
| `unbound_verbosity` | int | `2` | 0..5 (plus haut = plus verbeux). |
| `unbound_log_queries` | bool | `true` | Log des requêtes (attention au volume). |
| `unbound_log_replies` | bool | `true` | Log des réponses. |
| `unbound_log_servfail` | bool | `true` | Log des SERVFAIL. |

### Durscissement (hardening)
| Variable | Type | Défaut | Description / possibilités |
|---|---|---:|---|
| `unbound_qname_minimisation` | bool | `true` | QNAME minimisation (privacy, moins de fuites). |
| `unbound_harden_glue` | bool | `true` | Valide le glue (cohérence des NS). |
| `unbound_harden_dnssec_stripped` | bool | `true` | Rejette si DNSSEC est retiré. |
| `unbound_harden_below_nxdomain` | bool | `true` | Renforce les NXDOMAIN. |
| `unbound_aggressive_nsec` | bool | `true` | Utilise NSEC/NSEC3 pour prouver l’inexistence. |
| `unbound_harden_referral_path` | bool | `true` | Renforce le chemin de délégation. |

### Redirections locales (DNS internes)
| Variable | Type | Défaut | Description / possibilités |
|---|---|---:|---|
| `unbound_local_redirects` | liste d’objets | `[]` | Entrées **A** locales servies par Unbound. Format : `{ zone: "host.example.lan.", a: "192.168.1.10" }`. |
| `unbound_tld_refuse` | list\<string> | `["ru","cn","su","kp","ir"]` | TLD rejetés au niveau d’Unbound (ex. blocage géopolitique). Mettre `[]` pour désactiver. |

> **Note** : le rôle génère des blocs `local-zone: "<fqdn>." redirect` + `local-data: "<fqdn>. IN A <ip>"`.  
> Vous pouvez dupliquer la structure pour autant d’entrées que nécessaire. (Support AAAA/CNAME à étendre si besoin.)

### Exporter Prometheus
| Variable | Type | Défaut | Description / possibilités |
|---|---|---:|---|
| `unbound_exporter_enabled` | bool | `true` | Déploie un petit exporter Python. |
| `unbound_exporter_listen` | string | `"127.0.0.1:9167"` | Adresse:port d’écoute de l’exporter. Utilisez `0.0.0.0:9167` si Prometheus scrute depuis l’extérieur. |
| `unbound_control_sock` | string | `"/run/unbound.ctl"` | Socket **unbound-control** utilisé pour collecter les stats. |

**Pré‑requis exporter** : le **remote‑control** d’Unbound doit être **activé**.  
Si besoin, créez `/etc/unbound/unbound.conf.d/remote-control.conf` :
```conf
remote-control:
  control-enable: yes
  control-interface: /run/unbound.ctl
```
Puis : `systemctl restart unbound`.

---

## 🔒 Sécurité & bonnes pratiques

- **Écoute** : limitez `unbound_listen_ipv4` à l’IP d’Unbound + loopback.  
- **ACL** : autorisez **uniquement** AdGuard (et/ou vos serveurs autorisés).  
- **DNSSEC** : laissez activé. Le test `dnssec-failed.org` doit renvoyer **SERVFAIL**.  
- **Logs** : gardez `log-queries`/`log-replies` pour l’audit, mais surveillez l’espace disque.  
- **Exporter** : laissez `127.0.0.1:9167` si Prometheus scrute localement, sinon ouvrez l’adresse avec un firewall.

---

## 🧯 Dépannage express

- **SSH Permission denied (publickey)** → `ssh-copy-id root@<ip-unbound>` ; inventaire : `ansible_user=root ansible_become=false`.
- **APT lock / permissions** → `dpkg --configure -a && rm -f /var/lib/dpkg/lock-frontend` puis relancer le playbook.
- **`so-rcvbuf … not granted`** → voir note **sysctl** dans *Caching & performance*.
- **`dig +trace` montre des erreurs IPv6** → inoffensif si pas d’IPv6 ; mettez `unbound_do_ip6: false` et/ou utilisez `dig -4`.
- **Windows “BAD_PACKET” vers Unbound** → souvent **REFUSED** par ACL (comportement attendu si clients interdits).

---

## 📈 Prometheus / Grafana (aperçu)

L’exporter expose `/metrics` (format Prometheus). Exemple de job :
```yaml
scrape_configs:
  - job_name: 'unbound'
    static_configs:
      - targets: ['192.0.2.7:9167']   # ou 127.0.0.1 si Prometheus est sur le même hôte
```
**Queries utiles** :  
- Requêtes/s : `rate(unbound_num_queries[5m])`  
- Hit cache : `rate(unbound_cachehits[5m]) / (rate(unbound_cachehits[5m]) + rate(unbound_cachemiss[5m]))`  
- SERVFAIL/s : `rate(unbound_servfail[5m])`  
- Latence moyenne : `unbound_time_avg`

---

## 📄 Licence

MIT — voir `LICENSE`.
