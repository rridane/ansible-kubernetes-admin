# Ansible Role: k8s_upgrade_packages

Amène les paquets `kubeadm` / `kubelet` / `kubectl` d'un nœud à une **version cible
exacte**. Brique commune de l'upgrade kubeadm : `k8s_upgrade_control_plane` s'en sert
pour `kubeadm` seul, `k8s_upgrade_node` pour `kubelet`/`kubectl` (nœud drainé).

Ce qui revient à **chaque mineure** est automatisé (série du dépôt, version exacte, hold).
Ce qui est **ponctuel** — la clé du dépôt — se pose à la main, une fois (voir plus bas).

## Ce que fait le rôle

1. **Anciennes sources** : échoue si `apt.kubernetes.io` / `packages.cloud.google.com`
   est encore référencé ailleurs que dans `k8s_upgrade_apt_list` (dépôt supprimé en
   mars 2024 : `apt-get update` casserait). Le nettoyage reste manuel.
2. **Clé** : vérifie que `k8s_upgrade_apt_keyring` existe et **n'est pas expirée**, et
   affiche le nombre de jours restants. Sinon il s'arrête **avant toute modification**, en
   donnant la commande de pose. Il ne télécharge jamais la clé.
3. **Restes** : signale les anciens keyrings (`kubernetes-archive-keyring.gpg`…) qui ne
   servent plus — sans les supprimer : le rôle ne les a pas créés.
4. **Dépôt** : réécrit `kubernetes.list` **en place** sur la série cible (pkgs.k8s.io = un
   dépôt par mineure : `/core:/stable:/v1.28/deb/`), `apt-get update`, vérifie que la
   version existe.
5. **Paquets** : installe `<paquet>=<version>-<révision>` (redescente autorisée), puis
   `hold`. Seuls les paquets listés bougent.

Aucun autre fichier n'est créé : rien à nettoyer derrière le rôle.

## Pose de la clé (une fois par cluster, à la main)

Tous les dépôts pkgs.k8s.io sont signés par la **même** clé OBS
(`DE15 B144 86CD 377B 9E87 6E1A 2346 54DA 9A29 6436`) : une fois posée, elle vaut pour toutes
les séries. Sur chaque nœud :

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/<SÉRIE>/deb/Release.key \
  | sudo gpg --dearmor --yes -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
gpg --show-keys /etc/apt/keyrings/kubernetes-apt-keyring.gpg   # vérifier [expire : …]
```

`<SÉRIE>` = une série **maintenue** (ex. la plus récente). Piège : le `Release.key` des
séries EOL (v1.27…) est une copie figée à leur dernière publication, avec l'ancienne date
d'expiration → `EXPKEYSIG 234654DA9A296436`. La copie d'une série maintenue porte la même
clé, prolongée. À refaire à l'expiration (le rôle affiche les jours restants).

Derrière un proxy : exporter `https_proxy` avant le `curl`.

## Variables

| Variable | Défaut | Description |
|---|---|---|
| `k8s_upgrade_version` | (requis) | version cible `x.y.z` (ex. `1.28.15`) |
| `k8s_upgrade_package_revision` | `1.1` | révision pkgs.k8s.io |
| `k8s_upgrade_packages` | `[kubeadm]` | paquets à amener à la cible |
| `k8s_upgrade_apt_base_url` | `https://pkgs.k8s.io/core:/stable` | base des dépôts |
| `k8s_upgrade_apt_keyring` | `/etc/apt/keyrings/kubernetes-apt-keyring.gpg` | keyring `signed-by` (vérifié, jamais posé) |
| `k8s_upgrade_apt_list` | `/etc/apt/sources.list.d/kubernetes.list` | source réécrite en place |
| `k8s_upgrade_allow_downgrade` | `true` | autorise la redescente |
| `k8s_upgrade_hold` | `true` | `apt-mark hold` après installation |
| `k8s_upgrade_environment` | `{}` | env des tâches réseau (proxy) |

## Exemple

```yaml
- hosts: masters
  become: true
  serial: 1
  vars:
    k8s_upgrade_version: "1.28.15"
  roles:
    - role: rridane.kubernetes_admin.k8s_upgrade_packages
      vars:
        k8s_upgrade_packages: [kubeadm]
```

## Licence

MIT
