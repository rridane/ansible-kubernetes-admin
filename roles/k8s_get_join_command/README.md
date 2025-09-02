Role Name
=========

`k8s_get_join_command` — Ce rôle génère les commandes **`kubeadm join`** pour ajouter des nœuds au cluster Kubernetes :
- une commande pour les **workers**
- une commande pour les **control-planes additionnels** (avec `--control-plane` et `--certificate-key`).

Requirements
------------

- Le cluster doit avoir déjà été initialisé avec `kubeadm init` (cf. rôle `k8s_bootstrap_control_plane`).
- Le rôle doit être exécuté sur l’hôte primaire du control-plane (`k8s_primary_cp_host`).
- Paquets Kubernetes installés (`kubeadm`, `kubectl`).
- Python 3 et Ansible ≥ 2.15.

Role Variables
--------------

Les variables principales (définies dans `defaults/main.yml`) :

| Variable | Default | Description |
|----------|---------|-------------|
| `k8s_primary_cp_host` | (obligatoire) | Nom inventory de l’hôte primaire du control-plane. |
| `k8s_kubeadm_bin` | `kubeadm` | Binaire kubeadm. |
| `k8s_join_token_ttl` | `24h0m0s` | Durée de validité du token `kubeadm`. Vide = TTL par défaut de kubeadm. |
| `k8s_upload_certs` | `true` | Active l’upload des certificats pour join des control-planes. |
| `k8s_extra_join_args` | `[]` | Arguments supplémentaires à ajouter à la commande `join`. |
| `k8s_admin_conf_path_check` | `/etc/kubernetes/admin.conf` | Fichier dont la présence confirme que le cluster est initialisé. |
| `k8s_join_debug` | `false` | Si vrai, affiche les commandes générées en debug. |

Sorties (facts définis sur le primaire, accessibles via `hostvars`) :

| Fact | Description |
|------|-------------|
| `k8s_join_command_worker` | Commande join pour un worker. |
| `k8s_join_command_control_plane` | Commande join pour un control-plane additionnel. |
| `k8s_certificate_key` | Clé de certificats générée par `upload-certs`. |

Dependencies
------------

Aucune dépendance externe.  
En pratique, ce rôle est souvent utilisé après :
- `k8s_bootstrap_control_plane` (init du cluster).
- avant un rôle type `k8s_join_nodes` qui exécute réellement les commandes sur les autres hôtes.

Example Playbook
----------------

Exemple pour générer les commandes sur le primaire :

```yaml
- name: Get join commands
  hosts: masters
  become: yes
  vars:
    k8s_primary_cp_host: kmaster1
    k8s_join_debug: true
  roles:
    - role: rridane.kubernetes.k8s_get_join_command
```

EXEMPLE
-------
```yaml
- name: Join workers
  hosts: workers
  become: yes
  tasks:
    - name: Join this worker to cluster
      ansible.builtin.command: "{{ hostvars[k8s_primary_cp_host].k8s_join_command_worker }}"

- name: Join secondary masters
  hosts: masters:!kmaster1
  become: yes
  tasks:
    - name: Join this master to cluster
      ansible.builtin.command: "{{ hostvars[k8s_primary_cp_host].k8s_join_command_control_plane }}"
```


License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
