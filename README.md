# Ansible Collection: rridane.kubernetes_admin

Cette collection regroupe un ensemble de rôles Ansible pour **administrer un cluster Kubernetes** avec `kubeadm`.  
L’objectif est de simplifier les opérations d’installation et de gestion des nœuds du cluster via des rôles spécialisés.

---

## 📦 Rôles disponibles

### 🔹 `k8s_bootstrap_control_plane`
Initialise le **control-plane primaire** via `kubeadm init`.
- Génère et applique un `kubeadm.yaml`.
- Crée un `kubeconfig` pour root (et optionnellement pour un utilisateur).
- Prépare le cluster à recevoir des workers et d’autres control-planes.

👉 [Voir la documentation du rôle](roles/k8s_bootstrap_control_plane/README.md)

---

### 🔹 `k8s_get_join_command`
Génère les commandes `kubeadm join` à exécuter sur les autres nœuds :
- **worker** (simple `join`)
- **control-plane additionnel** (`join --control-plane --certificate-key=...`).

👉 [Voir la documentation du rôle](roles/k8s_get_join_command/README.md)

---

### 🔹 `k8s_manage_master_nodes`
Permet de joindre un nœud comme **control-plane additionnel** à partir de la commande générée par `k8s_get_join_command`.
- Exécute le `kubeadm join ... --control-plane`.
- Assure le démarrage du service `kubelet`.

👉 [Voir la documentation du rôle](roles/k8s_manage_master_nodes/README.md)

---

### 🔹 `k8s_manage_worker_nodes`
Permet de joindre un nœud comme **worker** au cluster.
- Exécute le `kubeadm join ...`.
- Assure le démarrage du service `kubelet`.

👉 [Voir la documentation du rôle](roles/k8s_manage_worker_nodes/README.md)

---

## 🚀 Exemple d’utilisation

Voici un enchaînement typique utilisant la collection :

```yaml
- name: Bootstrap du cluster (control-plane primaire)
  hosts: masters[0]
  become: yes
  roles:
    - role: rridane.kubernetes_admin.k8s_bootstrap_control_plane

- name: Générer les commandes join
  hosts: masters[0]
  become: yes
  roles:
    - role: rridane.kubernetes_admin.k8s_get_join_command

- name: Joindre les autres control-planes
  hosts: masters[1:]
  become: yes
  roles:
    - role: rridane.kubernetes_admin.k8s_manage_master_nodes

- name: Joindre les workers
  hosts: workers
  become: yes
  roles:
    - role: rridane.kubernetes_admin.k8s_manage_worker_nodes
```

## Compatibilité

OS : Debian 12+, Ubuntu 20.04/22.04

Ansible : ≥ 2.15

Kubernetes : testé avec kubeadm ≥ 1.26

Nécessite become: true pour la majorité des opérations.

## 🛠 Dépendances

Les paquets Kubernetes (kubeadm, kubelet, kubectl) doivent être installés au préalable (ex : via un rôle type kube_packages).

Un CNI (Calico, Cilium, Flannel…) doit être déployé après l’initialisation pour que les Pods puissent communiquer.

## 📌 Note importante

Cette collection est en cours de construction 🏗️.
👉 Avant la première release officielle, merci de consulter directement la documentation de chaque rôle (README.md dans chaque dossier) pour les variables et usages détaillés.

## 📜 License

Apache-2

👤 Auteur
Maintenu par rridane