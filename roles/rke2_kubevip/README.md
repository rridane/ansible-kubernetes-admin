# Ansible Role: rke2_kubevip

Dépose le manifest **kube-vip** (DaemonSet + RBAC) dans le dossier d'auto-deploy RKE2
`/var/lib/rancher/rke2/server/manifests/`. RKE2 l'applique au démarrage du cluster →
**VIP de l'API** en **ARP L2** avec **leader election** (un seul annonceur à la fois,
coordination via un `Lease` Kubernetes — pas de conflit ARP).

À jouer **sur le primaire** : le DaemonSet couvre ensuite tous les control-plane
(kube-vip tourne sur chacun, la VIP bascule via le Lease).

`cp_enable: true` / `svc_enable: false` : on ne gère **que** la VIP de l'API. Les
services `LoadBalancer` (VIP ingress) relèvent de Cilium (LB-IPAM + L2).

## Variables

| Variable | Défaut | Description |
|---|---|---|
| `kubevip_vip` | (requis) | adresse de la VIP de l'API |
| `kubevip_interface` | `""` | interface ; vide => auto-détection |
| `kubevip_image` | `ghcr.io/kube-vip/kube-vip:v0.8.7` | image épinglée (à bumper) |
| `kubevip_extra_env` | `{}` | env kube-vip en plus (nom => valeur) |
| `kubevip_manifest_path` | `…/server/manifests/kube-vip.yaml` | chemin du manifest |

## Orchestration

```yaml
- hosts: rke2_primary
  vars:
    kubevip_vip: "10.30.1.5"
  roles:
    - rke2_cilium       # HelmChartConfig
    - rke2_kubevip      # DaemonSet kube-vip
    - rke2_server       # bootstrap → RKE2 applique les deux en montant
```

## Dépendances

Aucune. Le kubeconfig n'est pas requis (le DaemonSet utilise un ServiceAccount).
