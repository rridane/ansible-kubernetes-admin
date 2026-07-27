# Ansible Role: rke2_server

Installe et configure un **serveur RKE2**. Deux **modes**, déterminés par
`rke2_primary` (nom d'inventaire du nœud d'amorçage) :

- **BOOTSTRAP** — `inventory_hostname == rke2_primary` : amorce le cluster **sur sa
  propre IP**. `config.yaml` **sans `server:` ni `token:`** (cette absence est le
  signal « premier serveur » côté RKE2 ; il génère le token).
- **JOIN** — les autres serveurs : `server: {{ rke2_server_url }}` + `token`
  (`rke2_join_token`, fourni par le rôle `rke2_fetch_token`).

## Gestion de la configuration

La conf RKE2 est un **dict pass-through** `rke2_config`, rendu **tel quel** (1:1) dans
`/etc/rancher/rke2/config.yaml` via `to_nice_yaml`. Le rôle n'est **pas opinioné** : il
n'injecte que le dynamique (`server:`/`token:` en mode join). Tout le reste (cni,
disable-kube-proxy, cidr, tls-san, node-ip, disable…) vit dans `rke2_config`, chez le
**consommateur** (group_vars de l'inventaire). C'est le modèle du vendeur lui-même
(Rancher `machineGlobalConfig` = map libre).

## Variables

| Variable | Défaut | Description |
|---|---|---|
| `rke2_primary` | (requis) | nom d'inventaire du serveur d'amorçage → mode |
| `rke2_server_url` | `""` | **JOIN** : endpoint `server:` (ex. `https://10.30.1.5:9345`) |
| `rke2_join_token` | `""` | **JOIN** : token, via `rke2_fetch_token` |
| `rke2_version` / `rke2_channel` | `""` / `stable` | version épinglée ou canal |
| `rke2_config` | `{}` | conf RKE2 pass-through (1:1 config.yaml) |
| `rke2_bootstrap_timeout` | `600` | attente du kubeconfig au bootstrap (s) |

## Exemple (côté consommateur)

```yaml
# group_vars/all/rke2_vars.yml (inventaire rke2-ovh)
rke2_config:
  cni: cilium
  disable-kube-proxy: true
  cluster-cidr: 10.42.0.0/16
  service-cidr: 10.43.0.0/16
  tls-san:
    - 10.30.1.5
    - api.k8s.interne
  node-ip: "{{ vrack_ip }}"
  disable:
    - rke2-ingress-nginx
```

## Ordre d'orchestration (playbook)

```yaml
- hosts: rke2_primary
  roles: [rke2_cilium, rke2_server, rke2_kubevip]   # Cilium AVANT le start
- hosts: rke2_servers:!rke2_primary
  serial: 1
  vars: { rke2_server_url: "https://10.30.1.5:9345" }
  roles: [rke2_fetch_token, rke2_server]            # JOIN
- hosts: rke2_agents
  vars: { rke2_server_url: "https://10.30.1.5:9345" }
  roles: [rke2_fetch_token, rke2_agent]
```

## Dépendances

Aucune. Préparer les nœuds en amont avec la collection `rridane.base_systems`
(netplan, time, sysctl, swap…).
