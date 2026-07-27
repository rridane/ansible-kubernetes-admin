# Ansible Role: rke2_cilium

Dépose le **`HelmChartConfig` `rke2-cilium`** dans le dossier d'auto-deploy RKE2
`/var/lib/rancher/rke2/server/manifests/`. Il **règle** le CNI Cilium que RKE2 déploie
lui-même quand `cni: cilium` (dans `rke2_server`) — on n'installe pas Cilium à la main,
on **surcharge la chart packagée**.

**À poser AVANT le premier démarrage du primaire** : Cilium est le CNI, il doit monter
tuné dès le départ (sinon réseau du cluster cassé). À jouer sur le primaire (ressource
cluster-wide, appliquée une fois).

## Gestion de la configuration

Une seule variable, **`cilium_values`** (dict pass-through → `valuesContent`). Défaut =
socle RKE2 kube-proxy-free ; le consommateur étend/surcharge :

```yaml
cilium_values:
  kubeProxyReplacement: true
  k8sServiceHost: "127.0.0.1"     # LB API local RKE2 (pas la VIP) — évite le deadlock
  k8sServicePort: 6443
  l2announcements:
    enabled: true
  k8sClientRateLimit: { qps: 10, burst: 20 }
```

> Le `k8sServiceHost: 127.0.0.1` est une connaissance de mécanisme RKE2 (chaque nœud a
> un LB API local) : Cilium joint l'API en localhost, jamais via la VIP → pas de
> deadlock au bootstrap.

## Variables

| Variable | Défaut | Description |
|---|---|---|
| `cilium_values` | socle RKE2 kube-proxy-free | valeurs Helm Cilium (pass-through) |
| `cilium_manifest_path` | `…/server/manifests/rke2-cilium-config.yaml` | chemin du HelmChartConfig |

## Hors périmètre

Les CRD `CiliumLoadBalancerIPPool` + `CiliumL2AnnouncementPolicy` (VIP ingress) sont des
ressources **runtime** (post-Cilium) → couche addons/GitOps, pas ce rôle de bootstrap.
