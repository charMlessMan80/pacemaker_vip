# pacemaker_vip

Playbook Ansible pour créer un cluster Pacemaker à 2 nœuds Oracle Linux 9 (VMs Hyper-V)
portant une ressource VIP unique, avec le rôle `ha_cluster` de
`fedora.linux_system_roles` 1.88.9.

> État : socle (L1) et preflight (L2) livrés. Le playbook cluster + VIP (L3) n'est pas
> encore livré.
>
> **Aucun fencing n'est configuré** (décision D2). Un split-brain peut donc rendre la
> VIP active sur les deux nœuds, et cette configuration n'est pas supportée par
> Red Hat. Voir [`docs/pacemaker-facts.md`](docs/pacemaker-facts.md).

## Prérequis du contrôleur

- Oracle Linux 9, `ansible-core` 2.14.18 (RPM `ol9_appstream`), Python 3.9.
- `python3-netaddr`, requis par les filtres `ansible.utils.ipaddr`.
  À VÉRIFIER : présence sur le contrôleur (`rpm -q python3-netaddr`).
- Accès à Ansible Galaxy, ou à une copie des tarballs dont les sha256 figurent dans
  le fichier de faits.

## Installation des collections

Depuis la **racine du dépôt** :

```sh
ansible-galaxy collection install -r collections/requirements.yml -p collections
ansible-galaxy collection list -p collections
```

Les versions sont exactes (`==`). Elles sont lues uniquement depuis `./collections`
(voir `ansible.cfg`).

## Piège : `ansible.cfg` n'est lu que depuis la racine du dépôt

Ansible ne lit `./ansible.cfg` que dans le répertoire courant, et seulement si :
- `ANSIBLE_CONFIG` n'est pas positionné ;
- ce répertoire n'est pas accessible en écriture à tous.

Sinon, `~/.ansible.cfg` ou `/etc/ansible/ansible.cfg` s'appliquent **en silence**
(ansible-core 2.14.18, `lib/ansible/config/manager.py:212-248`). À contrôler avant
toute exécution :

```sh
ansible-config dump --only-changed   # doit afficher CONFIG_FILE() = <racine>/ansible.cfg
```

## Inventaire

Aucun inventaire par défaut (`inventory = /dev/null`). Il faut toujours passer
`-i inventories/<env>`. `inventories/example/` est un modèle non déployable : hôtes
`.invalid`, variables vides.

## Preflight (`preflight.yml`)

Gardes bloquantes, en lecture seule, à lancer avant toute modification des nœuds. Le
playbook cluster (L3) l'importera en tête. Depuis la **racine du dépôt** :

```sh
ansible-playbook -i inventories/<env> preflight.yml --ask-vault-pass
```

- **Play 1 (`localhost`)** : contrôleur, inventaire et variables. Ce play s'exécute même
  si `-i` est oublié : il échoue alors sur l'absence du groupe `pacemaker_nodes`.
  - G1 : `ansible.cfg` du dépôt lu, et ses réglages non écrasés par `ANSIBLE_*`.
  - G2 : ansible-core 2.14.x.
  - G3 : netaddr >= 0.10.1 importable.
  - G4 : collections installées aux versions exactes de `collections/requirements.yml`.
  - G5 : `pacemaker_nodes` contient exactement 2 hôtes.
  - G6 : pas de `--limit`.
  - G7 à G10 : nom de cluster, VIP, interface et mot de passe Vault valides et
    identiques sur les 2 nœuds.
- **Play 2 (`pacemaker_nodes`, `become: true`, `any_errors_fatal`)** : nœuds, en lecture
  seule.
  - N1 : Oracle Linux 9.
  - N2 : interface présente, IPv4 statique gérée par NetworkManager.
  - N3 : VIP dans le réseau de l'interface, même préfixe.
  - N4 : VIP différente de l'adresse principale des nœuds.
  - N5 : VIP non utilisée par une machine tierce (`arping -D`).
  - N6 : paquets du rôle disponibles dans les dépôts activés.

Chaque échec dit quoi corriger et où. Le mot de passe n'est jamais affiché, y compris en
`-vvv`. Les gardes portent le tag `always` : `--tags` et `--skip-tags` ne les sautent
pas, sauf `--skip-tags always`.

Le preflight ne modifie pas la configuration des nœuds, mais `dnf repoquery` (N6) peut
rafraîchir le cache des métadonnées dnf.

Inventaire réel : copier `inventories/example/`, renseigner `group_vars/pacemaker_nodes.yml`,
et définir `vault_pacemaker_hacluster_password` dans un fichier chiffré par
`ansible-vault` chargé pour le groupe `pacemaker_nodes`. Le fichier de mot de passe
Vault reste hors du dépôt.

## Documentation

- [`CLAUDE.md`](CLAUDE.md) : règles du projet et leurs raisons.
- [`docs/pacemaker-facts.md`](docs/pacemaker-facts.md) : faits sourcés, versions
  épinglées, décisions, risques, points ouverts, commandes à lancer sur cible.
