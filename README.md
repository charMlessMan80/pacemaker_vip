# pacemaker_vip

Playbook Ansible pour créer un cluster Pacemaker à 2 nœuds Oracle Linux 9 (VMs Hyper-V)
portant une ressource VIP unique, avec le rôle `ha_cluster` de
`fedora.linux_system_roles` 1.88.9.

> État : socle uniquement (L1). Le preflight (L2) et le playbook cluster + VIP (L3) ne
> sont pas encore livrés.
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

## Documentation

- [`CLAUDE.md`](CLAUDE.md) : règles du projet et leurs raisons.
- [`docs/pacemaker-facts.md`](docs/pacemaker-facts.md) : faits sourcés, versions
  épinglées, décisions, risques, points ouverts, commandes à lancer sur cible.
