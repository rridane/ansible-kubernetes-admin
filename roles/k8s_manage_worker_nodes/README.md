
---

# README pour `k8s_manage_worker_nodes`

```markdown
Role Name
=========

`k8s_manage_worker_nodes` — Ce rôle permet de joindre un nœud comme **worker** dans un cluster Kubernetes, en utilisant une commande `kubeadm join` générée au préalable (cf. rôle `k8s_get_join_command`).

Requirements
------------

- Le cluster doit être déjà initialisé (rôle `k8s_bootstrap_control_plane`).
- Une commande `k8s_join_command_worker` valide doit être fournie (souvent via `hostvars[k8s_primary_cp_host]`).
- Paquets Kubernetes installés (`kubeadm`, `kubelet`).
- Accès root (`become: yes`).
- Python 3 et Ansible ≥ 2.15.

Role Variables
--------------

Variables principales (définies dans `defaults/main.yml`) :

| Variable | Default | Description |
|----------|---------|-------------|
| `k8s_join_command_worker` | (obligatoire) | Commande complète `kubeadm join ...` pour un worker. |
| `k8s_joined_check_path` | `/etc/kubernetes/kubelet.conf` | Fichier dont la présence indique que le nœud est déjà joint. |
| `k8s_manage_kubelet` | `true` | Si vrai, assure que le service `kubelet` est activé et démarré. |
| `k8s_kubelet_service_name` | `kubelet` | Nom du service systemd à gérer. |
| `k8s_join_debug` | `false` | Si vrai, affiche la commande join en debug. |

Dependencies
------------

Aucune dépendance externe.  
Ce rôle est généralement utilisé après :
- `k8s_bootstrap_control_plane`
- `k8s_get_join_command`

Example Playbook
----------------

```yaml
- name: Join workers to the cluster
  hosts: workers
  become: yes
  vars:
    k8s_join_command_worker: "{{ hostvars[k8s_primary_cp_host].k8s_join_command_worker }}"
  roles:
    - role: rridane.kubernetes.k8s_manage_worker_nodes
```

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
