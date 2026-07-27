# Ansible Role: rke2_agent

Installe et configure un **agent RKE2** (worker). Toujours en **mode JOIN** : rejoint le
cluster via `rke2_server_url` avec le token `rke2_join_token` fourni par le rôle
`rke2_fetch_token`.

## Gestion de la configuration

Comme `rke2_server`, la conf est un **dict pass-through** `rke2_config`, rendu tel quel
dans `/etc/rancher/rke2/config.yaml` (node-ip, node-label, node-taint…). Le rôle injecte
`server:`/`token:` ; le reste vient de `rke2_config`.

## Variables

| Variable | Défaut | Description |
|---|---|---|
| `rke2_server_url` | (requis) | endpoint `server:` (ex. `https://10.30.1.5:9345`) |
| `rke2_join_token` | (requis) | token, via `rke2_fetch_token` |
| `rke2_version` / `rke2_channel` | `""` / `stable` | version épinglée ou canal |
| `rke2_config` | `{}` | conf RKE2 agent pass-through (1:1 config.yaml) |

## Exemple (playbook)

```yaml
- hosts: rke2_agents
  vars:
    rke2_server_url: "https://10.30.1.5:9345"
  roles:
    - rke2_fetch_token   # définit rke2_join_token
    - rke2_agent
```

## Dépendances

Aucune. Préparer les nœuds en amont avec la collection `rridane.base_systems`.
