# Ansible Role: k8s_upgrade_node

Monte le **kubelet d'un nœud** kubeadm (control-plane ou worker) à la version cible, une
fois **tout le control plane** à cette version (`k8s_upgrade_control_plane`). À jouer en
`serial: 1` (control-planes, puis workers). Les `kubectl` sont délégués à
`k8s_primary_cp_host`.

## Déroulé

1. **Découverte** (lecture seule, jouée aussi en `--check`) : version du kubelet (API et
   binaire local), readiness, nœud déjà cordonné ou non, versions des apiservers. Un nœud
   déjà à la cible est **sauté** — pas de drain inutile.
2. **Garde-fous**
   - tous les apiservers ≥ cible : un kubelet ne doit **jamais** être plus récent que
     l'apiserver ;
   - pas de redescente, et une mineure à la fois (`k8s_upgrade_allow_kubelet_skip`).
3. **Drain** (`--ignore-daemonsets --delete-emptydir-data`), par tentatives de
   `k8s_upgrade_drain_attempt_timeout` (120 s). Les PDB sont respectés ; les pods sans
   contrôleur bloquent (→ `k8s_upgrade_drain_extra_args: [--force]`). Voir « Blocages ».
4. **Paquets** : `k8s_upgrade_packages` (kubeadm, kubelet, kubectl), puis sur un **worker**
   `kubeadm upgrade node` (config kubelet). Sur un control-plane c'est déjà fait.
5. **Ajustements**
   - retire `--container-runtime=remote` de `kubeadm-flags.env` (flag supprimé en 1.27) ;
   - aligne `sandbox_image` de containerd sur la pause attendue par kubeadm, redémarre
     containerd et pré-tire l'image.
6. **Kubelet** : `daemon-reload` + restart, attente `Ready` à la cible (5 min par attente).
7. **Uncordon** — seulement si le nœud était en service avant l'upgrade. Avant le drain, le
   rôle pose l'annotation `k8s-upgrade/cordoned` (retirée à l'uncordon) : après un arrêt
   (`abort`, échec), la relance reconnaît **son** cordon et remet le nœud en service ; un
   cordon posé par un tiers reste respecté.

## Blocages : le rôle attend l'opérateur

Drain bloqué **ou** kubelet qui ne revient pas : le rôle ne part pas en échec au bout d'un
timeout. Il affiche un **diagnostic** puis se met en **pause, sans limite de temps** :

| Blocage | Diagnostic affiché |
|---|---|
| Drain | fin de la sortie `kubectl drain`, pods encore sur le nœud (propriétaire vide = sans contrôleur), PDB à `disruptionsAllowed: 0` |
| Kubelet | `systemctl status kubelet`, 30 dernières lignes de `journalctl -u kubelet` |

On débloque (scale, PDB, pod nu supprimé, config kubelet corrigée…), puis **[Entrée]** pour
une nouvelle tentative, ou **`abort`** pour arrêter. Au-delà de `*_max_attempts`, le rôle
échoue. Dans tous les cas le nœud reste **cordonné** (et le kubelet intact si c'est le drain
qui bloque) : relancer reprend là où on en était ; `kubectl uncordon` pour renoncer.

Sans opérateur (CI) : `k8s_upgrade_interactive: false` → tentatives automatiques, puis échec.
En `--check`, ni drain ni pause.

## Variables

| Variable | Défaut | Description |
|---|---|---|
| `k8s_upgrade_version` | (requis) | version cible `x.y.z` |
| `k8s_primary_cp_host` | (requis) | hôte des `kubectl` délégués |
| `k8s_upgrade_node_name` | `{{ ansible_hostname \| lower }}` | nom du nœud |
| `k8s_upgrade_kubeconfig` | `/etc/kubernetes/admin.conf` | kubeconfig sur le primaire |
| `k8s_upgrade_node_packages` | `[kubeadm, kubelet, kubectl]` | paquets montés |
| `k8s_upgrade_drain` | `true` | drain avant upgrade |
| `k8s_upgrade_interactive` | `true` | pause opérateur sur blocage (sinon tentatives auto) |
| `k8s_upgrade_drain_args` | `[--ignore-daemonsets, --delete-emptydir-data]` | args du drain |
| `k8s_upgrade_drain_attempt_timeout` | `120s` | durée d'une tentative de drain |
| `k8s_upgrade_drain_max_attempts` | `5` | tentatives de drain |
| `k8s_upgrade_drain_extra_args` | `[]` | ex. `--force` |
| `k8s_upgrade_uncordon` | `true` | remise en service |
| `k8s_upgrade_cordon_annotation` | `k8s-upgrade/cordoned` | marque « cordonné par l'upgrade » |
| `k8s_upgrade_allow_kubelet_skip` | `false` | saut > 1 mineure |
| `k8s_upgrade_sandbox_image_fix` | `true` | aligne `sandbox_image` |
| `k8s_upgrade_containerd_config` | `/etc/containerd/config.toml` | conf containerd |
| `k8s_upgrade_containerd_socket` | `unix:///run/containerd/containerd.sock` | socket CRI |
| `k8s_upgrade_remove_container_runtime_flag` | `true` | retire `--container-runtime=remote` |
| `k8s_upgrade_kubeadm_flags_env` | `/var/lib/kubelet/kubeadm-flags.env` | args kubelet |
| `k8s_upgrade_kubeadm_unset_env` | `[http_proxy, https_proxy, no_proxy, HTTP_PROXY, HTTPS_PROXY, NO_PROXY]` | variables retirées de l'environnement de kubeadm |
| `k8s_upgrade_ready_retries` / `_delay` | `60` / `5` | une attente Ready (5 min) |
| `k8s_upgrade_ready_max_attempts` | `3` | attentes Ready |

Les variables de `k8s_upgrade_packages` (`k8s_upgrade_apt_keyring`,
`k8s_upgrade_environment`…) s'appliquent aussi. La clé du dépôt doit être posée sur chaque
nœud au préalable (README de `k8s_upgrade_packages`, « Pose de la clé »).

## Exemple

```yaml
- hosts: masters
  become: true
  serial: 1
  roles:
    - rridane.kubernetes_admin.k8s_upgrade_node

- hosts: workers
  become: true
  serial: 1
  roles:
    - rridane.kubernetes_admin.k8s_upgrade_node
```

## Licence

MIT
