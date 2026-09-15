# ansible-play-homelab

Example play project consuming the
[tititwister.homelab](https://github.com/TitiTwister/ansible-homelab-collection)
collection. It deploys a full homelab stack on a fictional `homelab` site:
base configuration + node_exporter everywhere, BIND DNS, a step-ca PKI,
HAProxy, Prometheus/Alertmanager/Pushgateway, Loki + Promtail, Grafana, a
PostgreSQL HA cluster (Patroni/etcd), a Kubernetes cluster (kubeadm) with
Forgejo, and a dual OpenVPN/WireGuard DMZ gateway with an offline root CA
cold vault.

All names are placeholders (`homelab` site, `homelab.example.com` domain,
`192.168.1.x` addresses, `203.0.113.x` DMZ addresses, `changeme` secrets) —
nothing here is real infrastructure. Use it as a template for your own site.

## Layout

```
inventories/homelab/              # one directory per site
├── homelab.yml                   # site group: children per function
├── homelab_dns.yml               # hosts per function group
├── homelab_ca.yml
├── homelab_prom.yml
├── homelab_loki.yml
├── homelab_grafana.yml
├── homelab_haproxy.yml
├── homelab_postgresql.yml
├── homelab_k8s_{masters,workers,controllers}.yml
├── homelab_dmz.yml               # standalone hosts: ovpn, wireguard, cold vault
├── group_vars/
│   ├── all/                      # fleet-wide: system, firewall, promtail,
│   │                             # haproxy stats, vault_dns, vault_haproxy
│   ├── homelab/                  # site-wide: vault_forgejo
│   └── <group>/                  # per function: vars_*.yml + vault_*.yml
└── host_vars/
    ├── homelab-ovpn/             # openvpn + wireguard + pki vars
    └── homelab-cold-vault/       # offline root CA pki vars
playbooks/site.yml                # plays mapping function groups to roles
playbooks/cold_vault.yml          # one-shot offline Root CA generation
files/grafana/dashboards/         # controller-side dashboard JSONs
files/step_ca/, files/openvpn/, files/wireguard/, files/k8s/
                                   # generated at runtime (gitignored or absent)
```

Roles are referenced by FQCN (`tititwister.homelab.prometheus`) and pinned to a
collection release in `requirements.yml`.

## Setup

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/ansible-galaxy collection install -r requirements.yml
```

## Run

```bash
.venv/bin/ansible-playbook playbooks/cold_vault.yml   # once: offline Root CA
.venv/bin/ansible-playbook playbooks/site.yml
.venv/bin/ansible-playbook playbooks/site.yml --tags role:prometheus
```

## Adapting to your site

1. Rename or copy `inventories/homelab/` to your site name and update
   `ansible.cfg` (`inventory =`).
2. Replace hosts, IPs and group names; keep the `<site>_<function>`
   grouping convention.
3. Adjust `group_vars` — the values that re-inject site specifics over
   the collection's neutral defaults live there (`prometheus_group`,
   `grafana_prometheus_group`, `grafana_dashboards_src`, ...).
4. Replace the example secrets in the `vault_*.yml` files with real
   values and encrypt them:

   ```bash
   .venv/bin/ansible-vault encrypt inventories/homelab/group_vars/*/vault_*.yml \
     inventories/homelab/host_vars/*/vault_*.yml
   ```

5. Trim what you do not deploy: plays and their groups are independent —
   remove plays from `site.yml` and the matching inventory groups and
   `group_vars` directories.

## License

[Beerware](https://github.com/TitiTwister/ansible-homelab-collection/blob/main/LICENSE)
