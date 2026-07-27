# Ansible Role: rke2_fetch_token

Récupère le **token de cluster RKE2** en le lisant **en direct sur le primaire**
(`slurp` délégué à `rke2_primary`) et l'expose en fact **`rke2_join_token`**. Rien
n'est écrit sur disque ni loggé (`no_log`).

C'est l'analogue RKE2 de `k8s_get_join_command` : un rôle qui *produit* le secret de
jointure, consommé ensuite par les rôles qui *joignent* (`rke2_server` en mode join,
`rke2_agent`).

## Pourquoi un rôle à part

- **Réutilisable** : mêmes 3 tâches pour le join des serveurs, des agents, et pour
  **ajouter un nœud plus tard** (on cible le nouveau nœud seul, le token est lu en
  direct sur le primaire — pas besoin de rejouer l'amorçage).
- **Pas de secret persisté** : ni dans git, ni sur la machine de contrôle.

## Variables

| Variable | Défaut | Description |
|---|---|---|
| `rke2_primary` | (requis) | nom d'inventaire du serveur d'amorçage |
| `rke2_token_path` | `/var/lib/rancher/rke2/server/node-token` | token côté primaire (pin CA) |

## Exemple

```yaml
- hosts: rke2_servers:!rke2_primary
  roles:
    - rke2_fetch_token      # définit rke2_join_token
    - rke2_server           # le consomme (mode join)
```
