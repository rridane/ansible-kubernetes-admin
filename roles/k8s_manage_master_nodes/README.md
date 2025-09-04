# Ansible Role: k8s_manage_master_nodes

`k8s_manage_master_nodes` — This role allows joining a node as an **additional control-plane** in a Kubernetes cluster, using a `kubeadm join` command generated beforehand (see role `k8s_get_join_command`).

## Requirements

- The cluster must already be initialized (role `k8s_bootstrap_control_plane`).
- A valid `k8s_join_command_control_plane` command must be provided (often via `hostvars[k8s_primary_cp_host]`).
- Kubernetes packages installed (`kubeadm`, `kubelet`).
- Root access (`become: yes`).
- Python 3 and Ansible ≥ 2.15.

## Role Variables

Main variables (defined in `defaults/main.yml`):

| Variable | Default | Description |
|----------|---------|-------------|
| `k8s_join_command_control_plane` | (required) | Full command `kubeadm join ... --control-plane --certificate-key=...`. |
| `k8s_joined_check_path` | `/etc/kubernetes/kubelet.conf` | File whose presence indicates that the node is already joined. |
| `k8s_manage_kubelet` | `true` | If true, ensures that the `kubelet` service is enabled and started. |
| `k8s_kubelet_service_name` | `kubelet` | Name of the systemd service to manage. |
| `k8s_join_debug` | `false` | If true, prints the join command in debug. |

## Dependencies

No external dependencies.  
This role is generally used after:
- `k8s_bootstrap_control_plane`
- `k8s_get_join_command`

## Example Playbook

```yaml
- name: Join secondary control-planes
  hosts: masters:!kmaster1
  become: yes
  vars:
    k8s_join_command_control_plane: "{{ hostvars[k8s_primary_cp_host].k8s_join_command_control_plane }}"
  roles:
    - role: rridane.kubernetes.k8s_manage_master_nodes
```

## License

BSD
