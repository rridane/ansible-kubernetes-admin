# Ansible Role: k8s_upgrade_control_plane

Monte **un control-plane kubeadm** d'une mineure (ou d'un patch), selon la procédure
officielle (`kubeadm upgrade apply` sur le premier, `kubeadm upgrade node` sur les autres).
À jouer en `serial: 1`, **primaire d'abord**. Le kubelet n'est pas touché : il suit nœud
par nœud avec `k8s_upgrade_node`.

Prérequis : kubeadm à la version cible sur le nœud (rôle `k8s_upgrade_packages`).

## Deux modes : `prepare` puis `apply`

`k8s_upgrade_mode: prepare` joue tout ce qui ne touche pas au control plane : garde-fous,
pré-pull des images et `kubeadm upgrade plan` (primaire). On relit le plan, on règle ce qu'il
signale (registre, proxy, préflight), et on rejoue autant qu'il faut. `apply` (défaut)
déroule ensuite l'upgrade. Les sauvegardes ne sont prises qu'en `apply`, juste avant l'upgrade.

## Déroulé

1. **Découverte** (lecture seule, jouée aussi en `--check`) : version de l'apiserver local,
   version du cluster (`kubeadm-config`), versions de tous les apiservers, readiness des
   control-planes, version de kubeadm. Un nœud déjà à la cible est **sauté** (idempotent).
2. **Garde-fous**
   - tous les control-planes `Ready` ;
   - primaire : une mineure à la fois, pas de redescente, tous les apiservers ont fini le
     saut précédent (pas plus d'une mineure d'écart en HA) ;
   - secondaire : le primaire a déjà appliqué la cible ;
   - kubeadm à la cible ; etcd `endpoint health --cluster` OK.
3. **Sauvegardes** (mode `apply`) sous `<k8s_upgrade_backup_dir>/v<version>/` : copie de
   `/etc/kubernetes`, et sur le primaire un **snapshot etcd** (`etcdctl snapshot save` dans
   le pod etcd, le data dir étant monté au même chemin sur l'hôte, puis déplacé).
4. **Upgrade** : `kubeadm config images pull`, puis sur le primaire `kubeadm upgrade plan`
   (affiché) et `kubeadm upgrade apply --yes`, sur les autres `kubeadm upgrade node`.
5. **Contrôles** : `/readyz` de l'apiserver local, static pods du nœud à la cible et
   `Ready`, etcd sain, et sur le primaire `kubeadm-config` à la cible.

kubeadm ne sait pas redescendre : le snapshot etcd et la copie de `/etc/kubernetes` sont le
seul retour arrière.

## Variables

| Variable | Défaut | Description |
|---|---|---|
| `k8s_upgrade_version` | (requis) | version cible `x.y.z` |
| `k8s_upgrade_mode` | `apply` | `prepare` (jusqu'au plan) ou `apply` |
| `k8s_primary_cp_host` | (requis) | control-plane qui joue `apply` |
| `k8s_upgrade_node_name` | `{{ ansible_hostname \| lower }}` | nom du nœud Kubernetes |
| `k8s_upgrade_kubeconfig` | `/etc/kubernetes/admin.conf` | kubeconfig admin local |
| `k8s_upgrade_manifests_dir` | `/etc/kubernetes/manifests` | static pods |
| `k8s_upgrade_etcd_pki_dir` | `/etc/kubernetes/pki/etcd` | PKI etcd |
| `k8s_upgrade_backup_dir` | `/root/k8s-upgrade-backups` | racine des sauvegardes |
| `k8s_upgrade_backup_config` | `true` | copie de `/etc/kubernetes` |
| `k8s_upgrade_etcd_snapshot` | `true` | snapshot etcd (primaire) |
| `k8s_upgrade_prepull_images` | `true` | pré-téléchargement des images |
| `k8s_upgrade_show_plan` | `true` | affiche `kubeadm upgrade plan` |
| `k8s_upgrade_apply_extra_args` | `[]` | ex. `--ignore-preflight-errors=KubeletVersion` |
| `k8s_upgrade_node_extra_args` | `[]` | args de `kubeadm upgrade node` |
| `k8s_upgrade_health_retries` / `_delay` | `60` / `5` | attente des contrôles |

## Exemple

```yaml
- hosts: master_primary
  become: true
  serial: 1
  roles:
    - role: rridane.kubernetes_admin.k8s_upgrade_packages
      vars: { k8s_upgrade_packages: [kubeadm] }
    - role: rridane.kubernetes_admin.k8s_upgrade_control_plane

- hosts: masters:!master_primary
  become: true
  serial: 1
  roles:
    - role: rridane.kubernetes_admin.k8s_upgrade_packages
      vars: { k8s_upgrade_packages: [kubeadm] }
    - role: rridane.kubernetes_admin.k8s_upgrade_control_plane
```

## Licence

MIT
