# pacemaker_vip

Playbook Ansible pour créer un cluster Pacemaker à 2 nœuds Oracle Linux 9 (VMs Hyper-V)
portant une ressource VIP unique, avec le rôle `ha_cluster` de
`fedora.linux_system_roles` 1.88.9.

> État : socle (L1), preflight (L2), preflight étendu (L3b), playbook cluster + VIP
> (L3c) et corrections de revue (L4b) livrés. Le playbook cluster est **validé sur cible
> (acc, 2026-11-10)** : premier déploiement sur nœuds vierges, second passage à
> l'identique, redémarrage propre du nœud porteur (D11). **Non testé** : arrêt brutal
> d'un nœud, changements de configuration.
>
> **Aucun fencing n'est configuré** (décision D2). Un split-brain peut donc rendre la
> VIP active sur les deux nœuds, et cette configuration n'est pas supportée par
> Red Hat. Voir [`docs/pacemaker-facts.md`](docs/pacemaker-facts.md).

## Prérequis du contrôleur

- Oracle Linux 9, `ansible-core` 2.14.18 (RPM `ol9_appstream`), Python 3.9.
- `python3-netaddr` >= 0.10.1, requis par les filtres `ansible.utils.ipaddr`
  (garde G3 ; constaté présent sur le contrôleur, C-01).
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
`-i <inventaire hors dépôt>`. L'inventaire réel n'est jamais versionné : `.gitignore`
ignore `inventories/*` sauf `inventories/example/`, modèle non déployable (hôtes
`.invalid`, variables vides).

## Preflight (`preflight.yml`)

Gardes bloquantes, en lecture seule, à lancer avant toute modification des nœuds. Le
playbook cluster (`cluster.yml`) l'importe en tête. Depuis la **racine du dépôt** :

```sh
ansible-playbook -i <inventaire hors dépôt> preflight.yml --ask-vault-pass
```

- **Play 1 (`localhost`)** : contrôleur, inventaire et variables. Ce play s'exécute même
  si `-i` est oublié : il échoue alors sur l'absence du groupe `pacemaker_nodes`.
  - G13 (en premier) : ni `--tags` ni `--skip-tags` (G13a) ; aucune variable du
    projet en `-e` (G13b : `pacemaker_*`, `vault_pacemaker_*`, `ha_cluster*`,
    `__ha_cluster*`, `ansible_python_interpreter` ; noms seuls dans le message) ;
    pas de `--check` (G13c). Limite : une variable du projet placée dans
    `group_vars/all` ou `host_vars/localhost` est refusée comme un `-e`.
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
  - En fin de play : marqueur `pacemaker_preflight_passed`, exigé ensuite par
    `cluster.yml`.

Chaque échec dit quoi corriger et où. Les gardes n'affichent jamais le mot de passe ni
une clé ; toute verbosité est refusée (C1). Les gardes portent le tag `always` :
`--tags` et `--skip-tags` ne les sautent pas (et G13a les refuse), sauf
`--skip-tags always`. `--skip-tags always`, `--start-at-task` et `--step` sautent tout
le preflight : `cluster.yml` s'arrête alors sur le marqueur absent (voir plus bas).

Journal Ansible : autorisé seulement protégé, par exemple
`install -d -m 0700 ~/ansible-logs` puis `umask 077` avant de lancer avec
`ANSIBLE_LOG_PATH=~/ansible-logs/ansible.log`. Ansible crée le fichier avec les droits
de l'umask : un journal déjà créé en 0644 doit être passé en 0600.

Le preflight ne modifie pas la configuration des nœuds, mais `dnf repoquery` (N6) peut
rafraîchir le cache des métadonnées dnf.

Inventaire réel, **hors du dépôt** : copier `inventories/example/` ailleurs (par
exemple `<répertoire hors dépôt>/pacemaker_vip/`), nommer chaque hôte exactement comme
`uname -n` sur le nœud (adresse SSH dans `ansible_host` si besoin), renseigner
`pacemaker_node_ip` par hôte et `group_vars/pacemaker_nodes.yml`, et définir
`vault_pacemaker_hacluster_password` dans un fichier chiffré par `ansible-vault`
chargé pour le groupe `pacemaker_nodes`. Le fichier de mot de passe Vault reste hors
du dépôt. Aucune variable du projet dans `group_vars/all` (G13b). Le dépôt est
**public** : n'y écrire aucun nom d'hôte, adresse, domaine ni chemin réel, même dans
un message de commit ; ces valeurs ne vivent que dans l'inventaire hors dépôt (D26).

## Playbook cluster (`cluster.yml`)

Crée le cluster Pacemaker à 2 nœuds et la ressource VIP `vip` (IPaddr2) avec le rôle
`fedora.linux_system_roles.ha_cluster` 1.88.9.

1. `import_playbook: preflight.yml` : toutes les gardes ci-dessus ; si l'une échoue,
   rien ne touche les nœuds.
2. Play `pacemaker_nodes` (`become`, `any_errors_fatal`, `/usr/bin/python3.9`) :
   - **Garde sans tag** (première tâche) : marqueur `pacemaker_preflight_passed`
     présent sur le nœud (preflight allé à son terme dans ce run), pas de
     `--skip-tags`, pas de `--check`. Elle s'exécute même sous `--skip-tags always`.
   - **C5** : chaque variable `ha_cluster*` a exactement la valeur et le type déclarés
     dans `cluster.yml` (dictionnaire `pacemaker_ha_cluster_declared`, source unique).
     Un `-e` qui en écrase une, ou une variable `ha_cluster*` non déclarée venue de
     l'inventaire ou de `-e`, arrête le play avant toute modification.
   - **D21** : sur un nœud vierge (`/var/lib/pcsd` absent), installe pcs et démarre
     pcsd une fois, pour que systemd crée `/var/lib/pcsd` (0700 root:root). Sans
     effet si le répertoire existe. Sans cela, le rôle échoue sur un nœud vierge.
   - Rôle `ha_cluster` : knet IPv4, `wait_for_all: "0"`, `stonith-enabled=false`,
     `resource-stickiness=1`, aucun paquet de fencing, adresses corosync et pcs =
     `pacemaker_node_ip`, nom de nœud = nom d'inventaire.
   - **V1-V6**, en lecture seule, chacune relancée au plus 30 fois toutes les 2 s,
     après un contrôle du marqueur (cas `--start-at-task`) :
     - V1 : `vip` Started sur exactement un nœud (crm_mon), et `inet <ip>/<préfixe>`
       de `pacemaker_vip_cidr` posé sur ce seul nœud ;
     - V2 : `stonith-enabled=false` dans les options effectives ;
     - V3 : `corosync-quorumtool -s` rc 0, Flags `2Node Quorate`, sans `WaitForAll` ;
     - V4 : `ring0_addr` de chaque nœud = son `pacemaker_node_ip`, `ip_version` ipv4,
       lien knet connecté entre ces adresses ;
     - V5 : `resource-stickiness=1` dans `rsc_defaults`, non redéfini sur `vip` ;
     - V6 : nom de cluster chargé par corosync = `pacemaker_cluster_name`.

Aucune vérification n'affiche de clé ni le mot de passe. `ha_cluster.pcs_address`
n'est pas vérifiée : elle n'est lisible que dans `/var/lib/pcsd/known-hosts`, avec
les jetons pcsd (accepté).

## Exploitation

> **Validé en repro conteneur (2 nœuds OL9 systemd, rôle réellement exécuté), puis
> validé sur cible (acc, 2026-11-10)** pour le premier déploiement, le second passage
> et le redémarrage propre d'un nœud (D11). **Non testé** : arrêt brutal d'un nœud
> (VM coupée), changements de configuration. Les durées indiquées viennent de la
> repro, sauf mention « sur cible ».

### Premier déploiement

Prérequis : `preflight.yml` vert sur la cible avec le même inventaire et le même
environnement ; inventaire réel **hors du dépôt** (le dépôt est public) ; harnais
limité aux 4 variables admises par G1, journal éventuel protégé (C3). Depuis la
**racine du dépôt** :

```sh
ansible-playbook -i <chemin de l'inventaire hors dépôt> cluster.yml --ask-vault-pass
```

Jamais `-v`, `--check`, `--diff`, `--limit`, `--tags`, `--skip-tags`,
`--start-at-task`, `--step`, ni `-e` sur une variable du projet (CLAUDE.md §9.1 ;
gardes C1, G13c, C4, G6, G13a, G13b, garde sans tag, marqueur). Un premier
déploiement ne se simule pas en `--check` (R-28). Le rôle n'a aucun tag : sans G13a,
`--tags <x>` terminerait en rc=0 après le seul preflight (R-35). Pour reprendre un run
interrompu, relancer la commande complète, jamais `--start-at-task` : le preflight
et C5 seraient sautés ; D21 et le rôle, conditionnés au preflight complet du même
run (sinon une reprise avec `--limit` redéploierait un cluster à 1 nœud), seraient
sautés eux aussi, et le run échouerait sur le marqueur en tête des vérifications.

Résultat attendu : `failed=0` partout et V1-V6 `ok`. V1-V6 garantissent, au moment de
la vérification, ce qui est listé ci-dessus. Elles ne garantissent pas la bascule
(voir plus bas), ni l'absence de VIP en double après une coupure réseau entre nœuds
(aucun fencing, D2).

Le rôle signale toujours `changed` : un second passage sans changement de variables ne
doit pas toucher corosync.conf ni la VIP. Validé sur cible (acc, 2026-11-10) : la tâche
« Stop cluster daemons to reload configuration » est sautée sur les 2 nœuds,
corosync.conf et la CIB sont identiques (sha256), et la ligne `crm_mon` de la VIP est
inchangée. En repro, l'historique d'opérations de la VIP est en plus inchangé (ni
arrêt ni déplacement).

### Changement de configuration

Modifier `cluster.yml` (variables du rôle) ou l'inventaire (`pacemaker_cluster_name`,
`pacemaker_vip_cidr`, `pacemaker_vip_nic`, `pacemaker_node_ip`), puis relancer la même
commande, en **fenêtre de maintenance** (jamais par `-e`, D24, G13b).
- Tout changement de corosync.conf (nom de cluster, transport, quorum, adresses des
  nœuds) ou des clés arrête `pacemaker` et `corosync` **sur les deux nœuds à la
  fois**, puis les redémarre : la VIP est coupée (F-08 ; constaté en repro pour
  `wait_for_all`). **Non testé** en repro pour le nom de cluster, les adresses des
  nœuds et la VIP : seul `wait_for_all` l'a été. Le nom de cluster est aussi une
  propriété CIB (`cluster-name`), épargnée par la purge du rôle (F-07) : sa mise à
  jour après un renommage est à constater.
- Un changement limité à la CIB (paramètres de la VIP, stickiness) ne redémarre pas
  corosync : l'arrêt n'a lieu que si corosync.conf, une authkey ou la configuration
  SBD change (`fedora.linux_system_roles` 1.88.9
  `roles/ha_cluster/tasks/shell_pcs/cluster-start-and-reload.yml:24-42`). À VÉRIFIER (non testé) : si la VIP est redémarrée quand ses paramètres
  changent.

Ne pas configurer le cluster à la main (`pcs …`) : le rôle purge au passage suivant
tout ce qui n'est pas déclaré (CLAUDE.md §8).

### Redémarrage décalé d'un nœud (D11)

Un nœud à la fois :
1. Repérer le porteur de la VIP et vérifier que l'autre nœud est en ligne :
   `crm_mon -1` (les deux nœuds `Online`).
2. Redémarrer le nœud. S'il porte la VIP, elle passe sur l'autre (1 à 2 s en
   repro, avec un arrêt propre, F-58). Validé sur cible (acc, 2026-11-10) avec
   `systemctl reboot` du porteur : plus longue interruption vue du contrôleur 0,4 s
   (ping toutes les 0,2 s). **Non testé** : arrêt brutal du porteur (VM coupée), où
   la bascule attend que corosync détecte la perte du nœud.
3. Attendre son retour : `crm_mon -1` (deux nœuds `Online`) et
   `corosync-quorumtool -s` (`Flags: 2Node Quorate`).
4. La VIP **ne revient pas** sur le nœud redémarré (stickiness 1 ; validé sur cible,
   acc, 2026-11-10). Vérifier
   `crm_mon -1 --include=failcounts` ; en cas d'échec ancien, `pcs resource cleanup vip`.
5. Seulement ensuite, l'autre nœud.

Si les deux nœuds sont arrêtés, le premier qui démarre devient quorate seul et démarre
la VIP (23 à 25 s en repro, F-59), même si l'autre est vivant mais injoignable (D8, risque
accepté).

Sur le nœud qui porte la VIP, ne pas réactiver le profil NetworkManager de l'interface
(`nmcli con up …`) : la VIP est retirée jusqu'au moniteur suivant (10 s) (repro L3c,
R-34).

## Documentation

- [`CLAUDE.md`](CLAUDE.md) : règles du projet et leurs raisons.
- [`docs/pacemaker-facts.md`](docs/pacemaker-facts.md) : faits sourcés, versions
  épinglées, décisions, risques, points ouverts, commandes à lancer sur cible.
