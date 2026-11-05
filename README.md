# pacemaker_vip

Playbook Ansible pour créer un cluster Pacemaker à 2 nœuds Oracle Linux 9 (VMs Hyper-V)
portant une ressource VIP unique, avec le rôle `ha_cluster` de
`fedora.linux_system_roles` 1.88.9.

> État : socle (L1), preflight (L2) et preflight étendu (L3b) livrés. Le playbook
> cluster + VIP (L3c) n'est pas encore livré.
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
  - G1 : `ansible.cfg` du dépôt lu ; aucun réglage ni variable `ANSIBLE_*` venant de
    l'environnement, sauf `ANSIBLE_SSH_EXTRA_ARGS`, `ANSIBLE_SSH_CONTROL_PATH_DIR`,
    `ANSIBLE_LOCAL_TEMP` et `ANSIBLE_LOG_PATH` ; vérification des clés d'hôte SSH
    jamais désactivée (`StrictHostKeyChecking=no/off/false/accept-new`,
    `UserKnownHostsFile=/dev/null`, `host_key_checking = False`).
  - C1 : verbosité 0 (ni `-v`, ni `ANSIBLE_VERBOSITY`, ni `verbosity` dans le cfg).
  - C2 : ni débogage, ni `keep_remote_files`, ni `display_args_to_stdout`, callback
    `default` seul.
  - C3 : journal Ansible absent, ou protégé (fichier sans droit groupe/autres, à
    l'utilisateur courant ; répertoires parents non inscriptibles par d'autres).
  - C4 : pas de `--diff`, même avec `--check` : le rôle ha_cluster y afficherait les
    clés du cluster.
  - G2 : ansible-core 2.14.x.
  - G3 : netaddr >= 0.10.1 importable.
  - G4 : collections installées aux versions exactes de `collections/requirements.yml`.
  - G5 : `pacemaker_nodes` contient exactement 2 hôtes.
  - G6 : pas de `--limit`.
  - G7 à G10 : nom de cluster, VIP, interface et mot de passe Vault valides et
    identiques sur les 2 nœuds.
  - G11 : `pacemaker_node_ip` par hôte, IPv4 du réseau de la VIP, distincte par nœud.
  - G12 : aucune variable d'inventaire ne désactive la vérification des clés SSH.
- **Play 2 (`pacemaker_nodes`, `become: true`, `any_errors_fatal`,
  `ansible_python_interpreter: /usr/bin/python3.9`)** : nœuds, en lecture seule.
  - N1 : Oracle Linux 9.
  - N2 : interface présente, IPv4 statique gérée par NetworkManager.
  - N3 : VIP dans le réseau de l'interface, même préfixe.
  - N4 : VIP différente de l'adresse principale des nœuds.
  - N5 : VIP non utilisée par une machine tierce (`arping -D`).
  - N6 : paquets du rôle disponibles dans les dépôts activés (sans paquet de fencing).
  - N7 : nom d'inventaire = `uname -n` du nœud ; `pacemaker_node_ip` = IPv4 principale
    de l'interface de la VIP.

Chaque échec dit quoi corriger et où. Les gardes n'affichent jamais le mot de passe ni
une clé ; toute verbosité est refusée (C1). Les gardes portent le tag `always` : `--tags` et `--skip-tags` ne les sautent
pas, sauf `--skip-tags always`.

Journal Ansible : autorisé seulement protégé, par exemple
`install -d -m 0700 ~/ansible-logs` puis `umask 077` avant de lancer avec
`ANSIBLE_LOG_PATH=~/ansible-logs/ansible.log`. Ansible crée le fichier avec les droits
de l'umask : un journal déjà créé en 0644 doit être passé en 0600.

Le preflight ne modifie pas la configuration des nœuds, mais `dnf repoquery` (N6) peut
rafraîchir le cache des métadonnées dnf.

Inventaire réel : copier `inventories/example/`, nommer chaque hôte exactement comme
`uname -n` sur le nœud (adresse SSH dans `ansible_host` si besoin), renseigner
`pacemaker_node_ip` par hôte et `group_vars/pacemaker_nodes.yml`, et définir
`vault_pacemaker_hacluster_password` dans un fichier chiffré par `ansible-vault` chargé pour le groupe `pacemaker_nodes`. Le fichier de mot de passe
Vault reste hors du dépôt.

## Documentation

- [`CLAUDE.md`](CLAUDE.md) : règles du projet et leurs raisons.
- [`docs/pacemaker-facts.md`](docs/pacemaker-facts.md) : faits sourcés, versions
  épinglées, décisions, risques, points ouverts, commandes à lancer sur cible.
