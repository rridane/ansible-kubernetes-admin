Role Name
=========

`k8s_bootstrap_control_plane` — Ce rôle initialise le **control-plane Kubernetes** via `kubeadm init`.  
Il génère le fichier `kubeadm.yaml`, exécute l’initialisation, et installe un `kubeconfig` utilisable pour l’administrateur et/ou pour un utilisateur spécifié.

Requirements
------------

- Un hôte Linux avec accès root (ou `become: yes`).
- Paquets Kubernetes déjà installés (`kubeadm`, `kubelet`, `kubectl`).
- Un CNI (ex: Calico, Cilium, Flannel) doit être déployé **après** l’exécution du rôle pour que les Pods communiquent.
- Optionnel : un groupe `[masters]` défini dans l’inventaire.
- Python 3 et Ansible ≥ 2.15.

Role Variables
--------------

Les variables principales (définies dans `defaults/main.yml`) :

| Variable | Default | Description |
|----------|---------|-------------|
| `k8s_primary_cp_host` | (doit être défini dans l’inventaire) | Nom de l’hôte primaire du control-plane. |
| `k8s_kubeadm_config_path` | `/tmp/kubeadm.yaml` | Chemin temporaire où rendre le fichier de config kubeadm. |
| `k8s_kubeadm_bin` | `kubeadm` | Binaire kubeadm à utiliser. |
| `k8s_kubeadm_init_extra_args` | `[]` | Liste d’arguments supplémentaires à passer à `kubeadm init`. |
| `k8s_admin_conf_path_check` | `/etc/kubernetes/admin.conf` | Fichier dont la présence indique que kubeadm est déjà initialisé. |
| `k8s_kubeconfig_src` | `/etc/kubernetes/admin.conf` | Fichier kubeconfig généré par kubeadm. |
| `k8s_kubeconfig_dest` | `/root/.kube/config` | Destination du kubeconfig pour root. |
| `k8s_kubeconfig_owner` | `root` | Propriétaire du kubeconfig root. |
| `k8s_kubeconfig_group` | `root` | Groupe du kubeconfig root. |
| `k8s_kubeconfig_dir_mode` | `0700` | Mode du répertoire `~/.kube`. |
| `k8s_kubeconfig_file_mode` | `0600` | Mode du fichier kubeconfig. |
| `k8s_copy_kubeconfig_for_user_enabled` | `false` | Active la copie d’un kubeconfig pour un utilisateur spécifique. |
| `k8s_copy_kubeconfig_for_user` | dict | Détail (owner, group, chemin) du kubeconfig utilisateur. |
| `k8s_cluster_name` | `kubernetes` | Nom du cluster. |
| `k8s_kubernetes_version` | `v1.26.0` | Version cible de Kubernetes. |
| `k8s_control_plane_endpoint` | `""` | Endpoint DNS:port du control-plane (obligatoire en HA). |
| `k8s_certificates_dir` | `/etc/kubernetes/pki` | Répertoire des certificats. |
| `k8s_image_repository` | `registry.k8s.io` | Registry des images Kubernetes. |
| `k8s_advertise_address` | `{{ ansible_host }}` | Adresse IP du kube-apiserver. |
| `k8s_bind_port` | `6443` | Port du kube-apiserver. |
| `k8s_apiserver_certSANs` | `[]` | SANs supplémentaires pour le certificat API server. |
| `k8s_apiserver_extra_args` | `{ authorization-mode: "Node,RBAC" }` | Arguments additionnels pour l’API server. |
| `k8s_controller_manager_extra_args` | `{}` | Arguments additionnels pour le controller-manager. |
| `k8s_scheduler_extra_args` | `{}` | Arguments additionnels pour le scheduler. |
| `k8s_dns_domain` | `cluster.local` | Domaine interne du cluster. |
| `k8s_pod_subnet` | `10.244.0.0/16` | CIDR des Pods. |
| `k8s_service_subnet` | `10.96.0.0/12` | CIDR des Services. |
| `k8s_etcd_local_enabled` | `true` | Active etcd local. |
| `k8s_etcd_data_dir` | `/mnt/data/etcd` | Répertoire des données etcd. |

Dependencies
------------

Aucune dépendance externe.  
Tu peux toutefois enchaîner avec :
- Un rôle CNI (Calico, Cilium, Flannel, etc.).
- Un rôle `k8s_cleanup_control_plane` si tu veux gérer le `state: absent`.

Example Playbook
----------------

Exemple avec un cluster single-master :

```yaml
- name: Bootstrap Kubernetes control-plane
  hosts: masters
  become: yes
  vars:
    k8s_primary_cp_host: kmaster1
    k8s_cluster_name: demo-cluster
    k8s_kubernetes_version: v1.30.4
    k8s_control_plane_endpoint: "api.demo.local:6443"
    k8s_apiserver_certSANs:
      - "127.0.0.1"
      - "localhost"
      - "api.demo.local"
  roles:
    - role: rridane.kubernetes.k8s_bootstrap_control_plane
```

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
