Unbound + AdGuard Home — DNS récursif, DNSSEC, sans forwarders (Ansible)
========================================================================

Ce dépôt déploie une stack DNS auto‑hébergée :
- Unbound (résolveur récursif, DNSSEC, part du root — sans Google/Cloudflare)
- AdGuard Home (filtrage/rewrite, interface web)
- un mini exporter Prometheus pour Unbound

Les clients LAN parlent à AdGuard, qui interroge Unbound. Unbound résout directement depuis la racine (root hints), DNSSEC activé.


Architecture (exemple)
----------------------
Clients LAN  ->  AdGuard (192.0.2.6:53)  ->  Unbound (192.0.2.7:53)
                              ^
                              | (UI, filtres, rewrites)

Les IPs d’exemple utilisent 192.0.2.0/24 (préfixe réservé à la documentation).


Caractéristiques
----------------
- Unbound récursif (pas de forwarders), DNSSEC strict
- root.hints téléchargés et mis à jour chaque semaine (cron)
- ACL Unbound restreinte : seul AdGuard et loopback autorisés
- Redirects locaux (ex. portail.example.lan -> 192.168.1.10)
- Exporter Prometheus /metrics pour graphiques Grafana


Prérequis
---------
- Ansible >= 2.12 (machine d’admin)
- Cible Unbound : Debian/Ubuntu (LXC/VM ok), accès SSH par clé
- (Optionnel) Prometheus / Grafana


Vie privée / données sensibles
------------------------------
Le dépôt public ne doit contenir aucune donnée perso.
Des fichiers *.example* sont fournis. Vos fichiers réels (inventory.ini, group_vars/all.yml, etc.) sont ignorés via .gitignore.

Utilisation typique :
  git clone https://github.com/<vous>/ansible-dns-stack-unbound-adguard.git
  cd ansible-dns-stack-unbound-adguard
  cp inventory.example.ini inventory.ini
  cp group_vars/all.example.yml group_vars/all.yml
  # Éditez inventory.ini et group_vars/all.yml avec VOS IP/domaines (192.168.x.x, etc.)


Déploiement rapide
------------------
# Vérifier l’accès SSH
ansible -i inventory.ini unbound -m ping

# Dry-run
ansible-playbook -i inventory.ini site.yml --check --diff

# Déployer
ansible-playbook -i inventory.ini site.yml --diff

Par défaut, l’inventaire cible Unbound en root sans sudo. Adaptez ansible_user / ansible_become si besoin.


Variables clés (extrait de group_vars/all.example.yml)
------------------------------------------------------
unbound_listen_ipv4: ["192.0.2.7", "127.0.0.1"]
unbound_listen_ipv6: []
unbound_port: 53

# ACL : n’autoriser qu’AdGuard + loopback
unbound_acl_allow:
  - "192.0.2.6/32"
  - "127.0.0.1/32"

# DNSSEC / root hints
unbound_auto_trust_anchor_file: "/var/lib/unbound/root.key"
unbound_root_hints: "/var/lib/unbound/root.hints"
unbound_manage_root_hints: true

# TLD à refuser (exemple)
unbound_tld_refuse: ["ru","cn","su","kp","ir"]

# Redirects locaux (exemple)
unbound_local_redirects:
  - { zone: "portail.example.lan.", a: "192.168.1.10" }
  - { zone: "bunkerweb.example.lan.", a: "192.168.1.11" }

# Exporter Prometheus
unbound_exporter_enabled: true
unbound_exporter_listen: "127.0.0.1:9167"   # mettre l’IP LAN si Prometheus est externe
unbound_control_sock: "/run/unbound.ctl"


Tests de bout en bout
---------------------
1) Sur Unbound (résolution directe, sans forwarders)
   dig @127.0.0.1 cloudflare.com +trace | head -n 30     # voit la racine puis .com puis NS du domaine
   dig @127.0.0.1 example.com +dnssec | head -n 15       # NOERROR + flag AD
   dig @127.0.0.1 dnssec-failed.org +dnssec | head -n 15 # SERVFAIL

2) Sur AdGuard
   dig @127.0.0.1 example.com +dnssec | head -n 15
   dig @127.0.0.1 dnssec-failed.org +dnssec | head -n 15
   # Preuve que l’upstream est Unbound (redirects locaux servis par Unbound)
   dig @127.0.0.1 portail.example.lan A +short

   Dans l’UI AdGuard → Settings → Upstream DNS servers : mettez 192.0.2.7:53

3) Sur un client Windows (via AdGuard, pas Unbound)
   PowerShell :
     Resolve-DnsName example.com -Server 192.0.2.6 -DnsSecOk
     Resolve-DnsName dnssec-failed.org -Server 192.0.2.6 -DnsSecOk   # doit échouer (SERVFAIL)

     # Direct Unbound (doit être refusé par l’ACL)
     Resolve-DnsName example.com -Server 192.0.2.7 -DnsSecOk


Sécurité
--------
- Unbound écoute uniquement sur l’IP du serveur + loopback
- ACLs restrictives : seul AdGuard est autorisé
- DNSSEC activé (anchor auto, validation stricte)
- TLD bloqués côté Unbound

Alerte “so-rcvbuf … not granted” bénigne.
Pour la supprimer : baisser so-rcvbuf (ex. 425984) ou augmenter net.core.rmem_max via sysctl.


Maintenance
-----------
- root.hints : téléchargés et mis à jour chaque semaine par cron
  (/usr/local/sbin/update-unbound-root-hints)
- Logs Unbound : /var/log/unbound.log


Prometheus / Grafana
--------------------
L’exporter expose /metrics (par défaut 127.0.0.1:9167).

Exemple Prometheus :
scrape_configs:
  - job_name: 'unbound'
    static_configs:
      - targets: ['192.0.2.7:9167']   # ou 127.0.0.1 si Prometheus local

Queries utiles (Grafana) :
- Requêtes/s : rate(unbound_num_queries[5m])
- Taux de hit cache : rate(unbound_cachehits[5m]) / (rate(unbound_cachehits[5m]) + rate(unbound_cachemiss[5m]))
- SERVFAIL/s : rate(unbound_servfail[5m])
- Latence moyenne : unbound_time_avg


Troubleshooting
---------------
- SSH “Permission denied (publickey)”
  ssh-copy-id root@<ip-unbound>
  inventaire : ansible_user=root ansible_become=false

- Ansible “Missing sudo password”
  Lancez sans become (si user=root) ou utilisez -K (si user non-root avec sudo)

- APT lock / permissions
  dpkg --configure -a && rm -f /var/lib/dpkg/lock-frontend

- Windows “BAD_PACKET” vers Unbound
  C’est souvent un REFUSED côté ACL interprété bizarrement par PowerShell → normal si vous bloquez l’accès direct.


Arborescence (résumé)
---------------------
ansible-dns-stack-unbound-adguard/
├─ inventory.example.ini
├─ site.yml
├─ group_vars/
│  └─ all.example.yml
├─ roles/
│  ├─ unbound/
│  │  ├─ tasks/main.yml
│  │  ├─ handlers/main.yml
│  │  └─ templates/
│  │     ├─ unbound.conf.j2
│  │     └─ update-unbound-root-hints.sh.j2
│  └─ unbound_exporter/
│     ├─ defaults/main.yml
│     ├─ tasks/main.yml
│     └─ templates/
│        ├─ exporter.py.j2
│        └─ unbound_simple_exporter.service.j2
├─ .gitignore
└─ LICENSE


Licence
-------
MIT — voir LICENSE.
