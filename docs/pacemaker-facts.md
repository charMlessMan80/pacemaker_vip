# pacemaker_vip — tableau de bord des faits

Fichier mis à jour **en dernier** dans chaque série (CLAUDE.md §13). Il ne contient pas
ce que git sait déjà.

**Abréviations de sources**

| Abréviation | Source |
|---|---|
| **LSR** | `fedora.linux_system_roles` 1.88.9, `roles/ha_cluster/` (tarball Galaxy, sha256 ci-dessous) |
| **LSR-fw** | même collection, `roles/firewall/` |
| **AC** | ansible-core 2.14.18, `lib/ansible/` (sdist PyPI `ansible_core-2.14.18.tar.gz`, sha256 `a953acb0db9e851c7044da9876b98924b7c2941148cab0c45f094309f17f8fd7`) |
| **AU** | `ansible.utils` 4.1.0 |
| **pcs** | ClusterLabs/pcs tag `v0.11.11` |
| **corosync** | corosync/corosync tag `v3.1.10` |
| **RA** | ClusterLabs/resource-agents tag `v4.10.0` (amont ; le RPM OL9 contient des correctifs rétroportés) |
| **pcs-src** | archive GitHub `pcs-0.11.11.tar.gz`, sha256 `4d3577571aab01a8c2b44c1ad18a0a61363a4a7957ad461c4a73e4afd2d6d6b2` |
| **corosync-src** | archive GitHub `corosync-3.1.10.tar.gz`, sha256 `03c55b984380bdb68ecd8d0f9cb1f9f0c2a7e6e7b7f437becc33e9a804018340` |
| **iputils** | archive GitHub `iputils-20210202.tar.gz` (sha256 `3f557ecfd2ace873801231d2c1f42de73ced9fbc1ef3a438d847688b5fb0e8ab`), `arping.c` ; page de manuel `arping.8` extraite du RPM OL9 `iputils-20210202-15.0.1.el9_7` (sha256 `ff498fa2625d945dde3b50f88e367d794c1f6b628da30d8e0885aac5df9f703e`) |

**Types de preuve**

| Mention | Signification |
|---|---|
| repoquery | `dnf repoquery` exécuté le 2026-11-05 sur `https://yum.oracle.com/repo/OracleLinux/OL9/{baseos/latest,appstream,addons}/x86_64/` |
| repro locale | conteneur `docker.io/library/oraclelinux:9` (Oracle Linux 9.8) avec le RPM `ansible-core-2.14.18-3.el9_8.1` et `python3-3.9.25-7.0.1.el9_8.2`. **Ce n'est ni le contrôleur ni un nœud.** |
| repro conteneur (L2) | contrôleur = repro locale + `python3-netaddr-0.10.1-3.el9` + `podman-remote` ; 2 nœuds `oraclelinux:9` avec systemd, NetworkManager et sudo (pv-n1 192.0.2.11/24, pv-n2 192.0.2.12/24, réseau podman 192.0.2.0/24, RFC 5737), connexion `containers.podman.podman` 1.21.2. Dépôts : yum.oracle.com le 2026-11-05. **Ni le contrôleur, ni les nœuds, ni Hyper-V.** |

Statuts : **PROUVÉ** (lu dans un artefact cité) ou **À VÉRIFIER : où**.

---

## 1. Faits sourcés

### Outillage et collections

| ID | Fait | Source | Statut |
|---|---|---|---|
| F-01 | `fedora.linux_system_roles` **1.88.9** est la dernière version dont `requires_ansible` accepte 2.14.18 (`">=2.14.0"`). La 1.89.0 exige `">=2.15.0"`, les versions 1.122.0 à 2.5.0 exigent `">=2.16.0"`. | `meta/runtime.yml:2` des tarballs 1.88.9 et 1.89.0 ; API Galaxy `/versions/<v>/` | PROUVÉ |
| F-02 | ansible-core 2.14.18 exige Python >= 3.9. Le RPM OL9 `ansible-core-2.14.18-3.el9_8.1` (ol9_appstream) utilise `python(abi) = 3.9` et embarque Jinja2 3.1.2. | API PyPI `requires_python` ; `AC/cli/__init__.py:16` ; repoquery `--requires` / `--provides` (`bundled(python-jinja2) = 3.1.2`) | PROUVÉ |
| F-03 | Les dépendances de LSR 1.88.9 sont `ansible.posix: "*"`, `community.general: "*"` et `containers.podman: "*"`. `ansible-galaxy` ne lit pas `requires_ansible`. Le contrôle n'a lieu qu'au chargement d'un plugin, en `warning` par défaut. Repro L0 : community.general 14.0.0 (`>=2.19.0`) et ansible.posix 2.2.2 (`>=2.16.0`) installés sans avertissement. | `MANIFEST.json` 1.88.9 ; aucune occurrence dans `AC/galaxy/` ; `AC/plugins/loader.py:1411-1417` ; `AC/config/base.yml:223-233` ; repro locale L0 | PROUVÉ |
| F-19 | Le filtre `ansible.utils.ipaddr` (AU 4.1.0, `>=2.14.0` ; la 5.0.0 exige `>=2.15.0`) requiert `netaddr>=0.10.1`. **Sans netaddr**, il lève une erreur explicite, mais le message cite le nom d'hôte alors que le filtre tourne sur le contrôleur. **Avec netaddr**, une entrée invalide renvoie `False` sans erreur, `192.0.2.10` sans préfixe donne `prefix=32`, et `192.0.2.0/24` donne `address=''`. Le RPM `python3-netaddr-0.10.1-3.el9` est dans ol9_appstream, qui contient aussi `0.8.0-5.el9`. ansible-core 2.14 n'a aucun filtre ni test IP natif. Le filtre nu `ipaddr` est redirigé vers `ansible.netcommon.ipaddr`. | `AU/meta/runtime.yml:2` ; `AU/requirements.txt:5` ; `AU/plugins/filter/ipaddr.py:32-38,296-299` ; `AU/plugins/plugin_utils/base/ipaddr_utils.py:632-636` ; repro locale L0 et L1 ; repoquery ; `AC/plugins/filter/` (core, encryption, mathstuff, urls, urlsplit) ; `AC/config/ansible_builtin_runtime.yml:9438-9439` | PROUVÉ |

### Rôle ha_cluster sur OL9

| ID | Fait | Source | Statut |
|---|---|---|---|
| F-04 | Sur OL9, `distribution=OracleLinux` et `os_family=RedHat`. Le rôle ne charge que `vars/RedHat.yml` : il n'existe ni `OracleLinux*.yml` ni `RedHat_9.yml` applicable. Conséquences : `__ha_cluster_repos: []`, donc `ha_cluster_enable_repos` (défaut `true`) exécute `dnf repolist` puis une boucle vide, un **saut silencieux**. Le README du rôle ne liste pas Oracle Linux (« RHEL 8.3+, Fedora 31+ »). | `AC/module_utils/facts/system/distribution.py:58,512-513` ; `LSR/tasks/set_vars.yml:19-32` ; `LSR/vars/RedHat.yml:11` ; `LSR/defaults/main.yml:7` ; `LSR/tasks/enable-package-repositories.yml:3-26` ; `LSR/tasks/enable-repositories/RedHat.yml:3-16` ; `LSR/README.md:9-11` ; repro locale L0 (`repos=[]`, `pcs_provider=pcs-0.10`) | PROUVÉ |
| F-05 | Paquets installés : `pcs`, `corosync-qnetd` et `openssl` **avant** la vérification des variables ; puis `corosync`, `libknet1-plugins-all`, `resource-agents`, `pacemaker` ; plus `fence-agents-all` et `fence-virt` (x86_64). `/etc/cluster/fence_xvm.key` est distribué sur x86_64. | `LSR/vars/RedHat.yml:13-24,48` ; `LSR/tasks/main.yml:11-17,58-67` ; `LSR/defaults/main.yml:17` | PROUVÉ |
| F-06 | La liste des nœuds de corosync.conf vient de `ansible_play_hosts`. Avec un play partiel (`--limit`, nœud injoignable), le rôle génère un corosync.conf à 1 nœud. | `LSR/tasks/shell_pcs/check-and-prepare-role-variables.yml:9` ; `LSR/tasks/shell_pcs/pcs-cluster-setup-pcs-0.10.yml:14-15` | PROUVÉ (code) |
| F-07 | Le CIB est déclaratif. À chaque passage, le rôle purge ressources, contraintes, attributs de nœuds et propriétés `crm_config`, sauf `cluster-infrastructure`, `cluster-name`, `dc-version`, `have-watchdog`, `last-lrm-refresh` et `stonith-watchdog-timeout`. Il reconstruit à partir des variables, puis applique `crm_diff` et `cibadmin --patch`. | `LSR/tasks/shell_pcs/create-and-push-cib.yml:51-70,257-288` ; `LSR/README.md:15-16` | PROUVÉ |
| F-08 | Si corosync.conf, une authkey ou la config SBD change, le rôle arrête `pacemaker` puis `corosync` sur **tous** les nœuds à la fois, puis les redémarre. corosync.conf est régénéré à chaque passage (`pcs cluster setup --overwrite`). | `LSR/tasks/shell_pcs/cluster-start-and-reload.yml:9-11,24-42` ; `LSR/tasks/shell_pcs/cluster-setup-corosync.yml:26-34` ; `LSR/tasks/shell_pcs/pcs-cluster-setup-pcs-0.10.yml:4-8` | PROUVÉ (code) |
| F-09 | Le rôle signale `changed` à chaque passage : pcsd est arrêté puis redémarré, et de nombreuses tâches ont `changed_when: not ansible_check_mode`. | `LSR/tasks/shell_pcs/configure-shell.yml:6-9,88-92` ; `LSR/tasks/shell_pcs/create-and-push-cib.yml` (passim) | PROUVÉ (code) |
| F-10 | Variables destructrices : `ha_cluster_cluster_present: false` (`pcs cluster destroy` et suppression de `fence_xvm.key`) ; `ha_cluster_regenerate_keys: true` (nouvelles clés, donc redémarrage complet, et suppression du certificat pcsd). | `LSR/tasks/main.yml:98-107` ; `LSR/tasks/shell_pcs/cluster-destroy-pcs-0.10.yml:9-15` ; `LSR/tasks/presharedkey.yml:55-58` ; `LSR/tasks/shell_pcs/configure-shell.yml:11-20` | PROUVÉ (code) |
| F-11 | Le rôle ne pose jamais `stonith-enabled` (aucune occurrence dans `tasks/`, `vars/`, `defaults/`). Pacemaker utilise alors son défaut `true`, et « If true, at least one fence device must be configured before resources are allowed to run ». Avec `false` : « resource recovery on online nodes starts without any further protection ». | grep LSR ; https://clusterlabs.org/projects/pacemaker/doc/2.1/Pacemaker_Explained/html/cluster-options.html | PROUVÉ |
| F-12 | Le rôle n'émet une section `quorum` que si `ha_cluster_quorum.options` est déclaré. pcs pose `two_node: 1` automatiquement pour 2 nœuds, sans auto_tie_breaker ni qdevice. `two_node: 1` active implicitement `wait_for_all`. Le cluster ne devient quorate la première fois que lorsque tous les nœuds se sont vus en même temps, au premier démarrage comme après un arrêt complet. | `LSR/tasks/shell_pcs/pcs-cluster-setup-pcs-0.10.yml:56-72` ; `pcs/lib/corosync/config_facade.py:75,869-880` ; `corosync/man/votequorum.5:94-105,136-137,151-156` | PROUVÉ (code et doc amont) |
| F-13 | Variable `ha_cluster_hacluster_password` (défaut `""`). Le rôle échoue (« must be specified ») si elle est vide et que `cluster_present` est vrai. `openssl passwd` est en `no_log` et le paramètre `password` du module `user` est `no_log=True`. **`pcs host auth` reçoit le mot de passe en `stdin` sans `no_log`.** Repro L1 : sans `no_target_syslog`, la cible journalise `ansible-ansible.legacy.command: Invoked with stdin=<secret> …`. | `LSR/defaults/main.yml:19` ; `LSR/tasks/shell_pcs/check-and-prepare-role-variables.yml:43-51` ; `LSR/tasks/main.yml:22-41` ; `AC/modules/user.py:3108` ; `LSR/tasks/shell_pcs/cluster-auth-pcs-0.10.yml:15` ; `AC/modules/command.py:239` ; `AC/module_utils/basic.py:526-527,1370-1401` ; repro locale L1 | PROUVÉ |
| F-14 | IPaddr2 : `ip` est obligatoire ; `nic` et `cidr_netmask` sont facultatifs et sinon déduits de la table de routage (`ip route list match … scope link`, préfixe le plus long). **Si `nic` ET `cidr_netmask` sont fournis, aucune vérification de route n'est faite.** Prérequis de `nic` : une IP statique non gérée par le cluster sur l'interface. | `RA/heartbeat/IPaddr2:163-206,534-555` ; `RA/heartbeat/findif.sh:213-244` | PROUVÉ (amont) ; À VÉRIFIER : RPM OL9 (§6) |
| F-15 | `ha_cluster_manage_firewall` (défaut `false`) : si `true`, le rôle firewall **installe, démasque, active et démarre firewalld**. `ha_cluster_manage_selinux` (défaut `false`) : si `true`, il est sauté en silence quand firewalld ne tourne pas. Le rôle ne modifie jamais le mode SELinux. | `LSR/defaults/main.yml:66,70` ; `LSR/tasks/firewall.yml:3-14` ; `LSR-fw/tasks/firewalld.yml:31-36` ; `LSR-fw/tasks/main.yml:22-32` ; `LSR/tasks/selinux.yml:6-10` | PROUVÉ |
| F-16 | Service firewalld `high-availability` (`firewalld-1.3.4-20.0.1.el9_8`) : TCP 2224, 3121, 5403, 9929, 21064 ; UDP 5404, 5405-5412, 9929. Le rôle ajoute 1229/tcp (fence-virt). D'après Red Hat, **seuls TCP 2224 (pcsd) et UDP 5404-5412 (corosync)** sont utiles pour notre cas : 3121 sert à pacemaker_remote, 5403 à qnetd, 21064 à DLM, 9929 à booth. | repro locale L0 (`/usr/lib/firewalld/services/high-availability.xml:5-12`) ; `LSR/tasks/firewall.yml:9-10` ; Red Hat RHEL 9 « Configuring and managing high availability clusters », table « Ports to Enable », lue via https://web.archive.org/web/20260128233618/https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/configuring_and_managing_high_availability_clusters/index | PROUVÉ ; port knet effectif À VÉRIFIER (§6) |

### Paquets, fencing, support

| ID | Fait | Source | Statut |
|---|---|---|---|
| F-17 | **ol9_addons** : `pacemaker-2.1.10-3.0.1.el9_8`, `pcs-0.11.11-2.el9_8.1`, `corosync-3.1.10-1.el9_8.2`, `corosync-qnetd/qdevice-3.0.4-1.el9`, `libknet1-plugins-all-1.28-1.el9`, `resource-agents-4.10.0-111.el9_8.19`, `sbd-1.5.2-1.el9`. **ol9_appstream** : `fence-agents-all-4.10.0-110.el9_8.8`, `fence-virt-4.10.0-110.el9_8.8`, `python3-netaddr-0.10.1-3.el9`. **ol9_baseos_latest** : `openssl`. Doc Oracle : `dnf config-manager --enable ol9_appstream ol9_baseos_latest ol9_addons`. **Correction L2** : `pacemaker` (>= 2.1.9) et `corosync` (>= 3.1.5) sont **aussi** dans ol9_appstream, et `openssl` aussi dans ol9_appstream. `pcs`, `corosync-qnetd`, `libknet1-plugins-all` et `resource-agents` ne sont **que** dans ol9_addons. Sans ol9_addons, il manque ces 4 paquets. L'image `oraclelinux:9` n'active pas ol9_addons. | repoquery ; https://docs.oracle.com/en/operating-systems/oracle-linux/9/availability/availability-EnablingAccesstothePacemakerandCorosyncPackages.html ; repro conteneur L2 (`dnf repoquery --available --qf '%{name}-%{version}-%{release} %{repoid}'`, avec et sans ol9_addons) | PROUVÉ ; contenu du miroir interne À VÉRIFIER (§6) |
| F-18 | Aucun agent de fencing Hyper-V ou SCVMM, ni dans les dépôts OL9 (plus de 50 paquets `fence-agents-*`) ni en amont. | repoquery ; https://github.com/ClusterLabs/fence-agents/tree/main/agents | PROUVÉ |
| F-24 | Hyper-V. VHD Set : « shared Virtual Disk model for guest clusters ». Pour les réservations persistantes, seule `Test-VHD -SupportPersistentReservations` en parle. Le support Linux d'un VHDX partagé avec réservations persistantes n'a pas été trouvé. Aucun watchdog Hyper-V trouvé. softdog : « It is not recommended to use such a configuration within a Production capacity » (Red Hat) ; « As a fall-back, you can use the softdog module » (man sbd). | https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/manage/create-vhdset-file ; https://learn.microsoft.com/en-us/powershell/module/hyper-v/test-vhd ; https://access.redhat.com/articles/2800691 ; https://raw.githubusercontent.com/ClusterLabs/sbd/main/man/sbd.8.pod.in | PROUVÉ (pour référence ; D2 écarte le fencing) |
| F-25 | « Red Hat does not support clusters when fencing is disabled, as it is not suitable for a production environment. » Oracle désactive le fencing dans son exemple OL9 : « For simplicity, it is disabled in this example », avec `pcs property set stonith-enabled=false`. | Red Hat RHEL 9 HA (copie Wayback, URL F-16) ; https://docs.oracle.com/en/operating-systems/oracle-linux/9/availability/OL9-AVAILABILITY.pdf | PROUVÉ |

### Configuration Ansible (L1)

| ID | Fait | Source | Statut |
|---|---|---|---|
| F-20 | Ordre de recherche de `ansible.cfg` : `ANSIBLE_CONFIG`, puis `./ansible.cfg` (ignoré si le CWD est world-writable, avec un avertissement), puis `~/.ansible.cfg`, puis `/etc/ansible/ansible.cfg`. Le premier fichier trouvé est utilisé, sans fusion. Repro L1 : depuis la racine, `CONFIG_FILE() = /repo/ansible.cfg` avec nos 5 réglages ; depuis `/tmp`, `CONFIG_FILE() = /etc/ansible/ansible.cfg` et aucun réglage. La variable magique `ansible_config_file` existe en 2.14.18. | `AC/config/manager.py:212-258` ; `AC/vars/manager.py:456` ; repro locale L1 | PROUVÉ |
| F-21 | `inventory =` vide est résolu en chemin relatif au `.cfg` : la racine du dépôt devient l'inventaire. Sans la ligne, c'est `/etc/ansible/hosts`. Avec `/dev/null`, l'inventaire est vide. `-i` remplace la valeur du cfg. | `AC/config/base.yml:729-740` ; repro locale L1 (3 cas, plus `--graph` avec `-i`) | PROUVÉ |
| F-22 | `collections_path` (singulier) est valable depuis 2.10. `collections_paths` est encore lue en 2.14.18, sans dépréciation émise (simple TODO). Les chemins relatifs du `.cfg` sont résolus depuis le répertoire du fichier. `collections_scan_sys_path` vaut `true` par défaut, et `sys.path` est alors ajouté **après** les chemins configurés. | `AC/config/base.yml:194-222` ; `AC/config/manager.py:77-78` ; `AC/utils/path.py:49-52` ; `AC/utils/collection_loader/_collection_finder.py:94-98` ; repro L1 (`COLLECTIONS_PATHS(/repo/ansible.cfg) = ['/repo/collections']`) | PROUVÉ |
| F-23 | `no_target_syslog` est transmis comme `_ansible_no_log` au module, ce qui positionne `AnsibleModule.no_log`. Cela saute `_log_invocation` et `log()`. C'est son seul usage côté module, qui ne masque pas les résultats. Repro L1 avec rsyslog local : la ligne « Invoked with stdin=SECRET… » est présente sans l'option, et absente avec l'option, que ce soit via le cfg ou via `ANSIBLE_NO_TARGET_SYSLOG`. **Limite** : l'option n'agit pas sur l'affichage de `invocation` côté contrôleur en `-vvv`. | `AC/config/base.yml:922-934` ; `AC/plugins/action/__init__.py:969-970` ; `AC/module_utils/common/parameters.py:93` ; `AC/module_utils/basic.py:526-527,1311` ; `AC/plugins/callback/__init__.py:234-235` ; repro locale L1 (rsyslog, sans journald) | PROUVÉ ; comportement avec journald sur les nœuds À VÉRIFIER (§6) |

### Preflight (L2)

| ID | Fait | Source | Statut |
|---|---|---|---|
| F-26 | `--limit` réduit **aussi** `ansible_play_hosts_all` : `get_hosts()` est appelé avec `ignore_restrictions=True`, mais sans `ignore_limits`. Comparer `ansible_play_hosts` à `ansible_play_hosts_all` ne détecte donc pas `--limit`. `ansible_limit` est défini dès que `--limit` est passé. `groups` n'est pas réduit. `ANSIBLE_LIMIT` n'existe pas en 2.14.18. Sous `--limit` sans `localhost`, un play `hosts: localhost` est sauté (« skipping: no hosts matched »). | `AC/plugins/strategy/__init__.py:274` ; `AC/vars/manager.py:487,496-501` ; `AC/inventory/manager.py:388-389,404-407` ; `AC/utils/vars.py:211-230` ; `AC/constants.py:110` ; `AC/cli/arguments/option_helpers.py:289` ; repro locale L2 (`--limit h1` donne `ansible_play_hosts_all = [h1]`, `groups` = 2 hôtes ; `ANSIBLE_LIMIT=h1,h2` laisse `ansible_limit` non défini) | PROUVÉ |
| F-27 | Lue via `hostvars`, une variable dont le gabarit pointe vers une variable absente vaut le **texte brut** `"{{ vault_pacemaker_hacluster_password }}"`. Dans ce cas, `is defined` vaut `true` et `default()` ne s'applique pas. Sur l'hôte lui-même, `is defined` vaut `false` et `default()` s'applique. Une variable `null` est `defined` (`is none` vrai), et `null \| string` donne `"None"`. | `AC/vars/hostvars.py:138-141` (`fail_on_undefined=False`) ; repro locale L2 (`/work/exp/exp.yml`) | PROUVÉ |
| F-28 | Le nom de cluster passé à `pcs cluster setup --corosync_conf`, que le rôle appelle sans `--force`, doit respecter trois règles. Il est non vide. Il correspond à `^[a-zA-Z0-9_-]{0,32}$` (la règle vient de GFS2, mais elle est bloquante sans `--force`). Il ne contient ni `{`, ni `}`, ni `\n`, ni `\r`. corosync 3.1.10 ne documente aucune contrainte de caractères : le nom sert à dériver un identifiant. | `pcs-src pcs/lib/corosync/config_validators.py:57-91,276-293` ; `pcs/lib/validate.py:596-626` ; `pcs/lib/commands/cluster.py:506,611-618` ; `pcs/cluster.py:1800-1804` ; `LSR/tasks/shell_pcs/pcs-cluster-setup-pcs-0.10.yml:3-13` (sans `--force`) ; `corosync-src man/corosync.conf.5:338-340`, `exec/totemconfig.c:946-973` | PROUVÉ (code) |
| F-29 | `arping -D` (iputils-20210202, OL9) émet ses sondes avec la source 0.0.0.0. Il renvoie 0 sans réponse et 1 si l'adresse répond. **Piège** : sur une interface sans ARP (`lo`, NOARP), il renvoie 0 sans rien envoyer (« Interface "lo" is not ARPable »). Interface inexistante : rc 2. Il faut être root : `getcap /usr/bin/arping` est vide, et sous `nobody` on obtient « arping: socket: Operation not permitted », rc 2. | `iputils arping.c:300-301,562-570,822-824,1006-1009` ; `arping.8` (RPM OL9) ; repro conteneur L2 (adresse libre : rc 0 « Received 0 response(s) » ; adresse prise : rc 1 ; `-I lo` : rc 0 ; `-I nope0` : rc 2) | PROUVÉ |
| F-30 | Une IPv4 posée hors NetworkManager (`ip addr add`, ou configuration de podman) apparaît en `GENERAL.STATE = 100 (connected (externally))`, avec un profil généré **`ipv4.method = manual`**. La méthode seule ne prouve donc pas qu'une IP est statique et gérée. Un profil créé par `nmcli` et activé donne `100 (connected)`. | repro conteneur L2 (`nmcli --get-values GENERAL.CONNECTION,GENERAL.STATE device show`, `ipv4.method`) | PROUVÉ (repro) ; comportement sur les VM Hyper-V À VÉRIFIER (§6) |
| F-31 | `include_tasks` avec `apply: {tags: [always]}`, dans un rôle appelé avec `tags: always`, exécute les tâches incluses sous `--tags foo` et sous `--skip-tags foo`. | repro locale L2 (rôle jetable, puis `preflight.yml` en nominal sous `--tags foo` et `--skip-tags foo` : rc=0, toutes les gardes exécutées) | PROUVÉ (repro) |
| F-32 | La clé de fait d'une interface remplace `-` par `_` (`pv-dum0` → `ansible_facts['pv_dum0']`). La liste `ansible_facts['interfaces']` garde le nom réel. | `AC/module_utils/facts/namespace.py:41,49-51` ; `AC/module_utils/facts/network/linux.py:283-288` (`:` → `_`) ; repro conteneur L2 | PROUVÉ |
| F-33 | Avec `any_errors_fatal: true`, un nœud injoignable pendant la collecte des faits arrête le play pour tous les hôtes (« NO MORE HOSTS LEFT »), avant toute tâche du rôle. | `AC/plugins/strategy/linear.py:350-368` ; repro conteneur L2 (pv-n2 arrêté) | PROUVÉ |
| F-34 | Quand le play 1 (`localhost`) échoue, le play 2 ne s'exécute pas. | repro locale L2 (`-i inventories/example` : échec en G7, play nœuds absent du journal) | PROUVÉ (repro) |

## 2. Versions épinglées

Fichier `collections/requirements.yml`. Le sha256 publié par l'API Galaxy a été comparé au sha256 du tarball téléchargé le 2026-11-05 : tous identiques.

| Collection | Version | requires_ansible | Première version exclue | sha256 du tarball |
|---|---|---|---|---|
| fedora.linux_system_roles | 1.88.9 | `>=2.14.0` | 1.89.0 (`>=2.15.0`) | `3c93089ceaf80587cea4362edad3381990d78c07b6b56ff53c9bedf24c169de4` |
| ansible.utils | 4.1.0 | `>=2.14.0` | 5.0.0 (`>=2.15.0`) | `4be2f5ae5a17fa4f1d973672a7d914f1d443d075222d00cf7d90db58acfa70b4` |
| ansible.posix | 1.6.1 | `>=2.14.0` | 1.6.2 (`>=2.15.0`) | `8f7f7313d32acd24af2b3d1ff4f00d99011cecf0a2d01007c09494f060c34b23` |
| community.general | 9.5.13 | `>=2.13.0` | 10.0.0 (`>=2.15.0`) | `9b7a0fb976f853efe8b48030843a1e1984f1c4c0c04059e6963a5019bb36e646` |
| containers.podman | 1.21.2 | `>=2.8` | aucune | `86fefb89cc86b213e7d3906068b525311ea91881369a3866616b5c109cd8d431` |

Les 4 dépendances n'ont aucune dépendance transitive (`metadata.dependencies = {}`, API Galaxy). Repro L1 : `ansible-galaxy collection install -r collections/requirements.yml -p collections` installe exactement ces 5 versions et rien d'autre.

`requirements.yml` ne permet pas d'imposer un sha256 en 2.14. Le contrôle d'intégrité des tarballs reste manuel. Le preflight (G4) compare les **versions** installées (`MANIFEST.json`) à `requirements.yml`, pas les sha256 (P-13).

## 3. Décisions

| ID | Date | Décision | Motif |
|---|---|---|---|
| D1 | 2026-11-05 | Rôle `ha_cluster` de `fedora.linux_system_roles` 1.88.9 épinglé, sur ansible-core 2.14.18 (contrôleur OL9, Python 3.9.25). Pas de venv, pas de montée de version d'ansible-core. | Contraintes du contrôleur existant (F-01, F-02). |
| D2 | 2026-11-05 | **Aucun fencing.** Risque accepté : split-brain possible, donc VIP active sur les deux nœuds ; configuration non supportée par Red Hat (F-25). `stonith-enabled: false` sera déclaré dans les variables du rôle (F-07, F-11). | **[OUVERT — à fournir par l'utilisateur]** |
| D3 | 2026-11-05 | VIP fournie en une variable au format CIDR, interface en variable. Nœuds et VIP dans le même /24. | Prémisse utilisateur. |
| D4 | 2026-11-05 | firewalld désactivé sur les nœuds ; filtrage géré par l'infra réseau. | Prémisse utilisateur. Conséquence : `ha_cluster_manage_firewall: false` obligatoire (F-15). |
| D5 | 2026-11-05 | Mot de passe hacluster via Ansible Vault. | Prémisse utilisateur. |
| D6 | 2026-11-05 | Projet indépendant : aucune convention héritée d'un autre dépôt. | Décision utilisateur. |
| D7 | 2026-11-05 | Contrôleur distinct de la machine de développement ; ni contrôleur ni nœuds joignables depuis celle-ci. | Prémisse utilisateur. |

## 4. Risques d'échec silencieux et gardes

| ID | Risque | Garde prévue | Livrable |
|---|---|---|---|
| R-01 | Dépendances Galaxy incompatibles installées sans avertissement (F-03). | Épinglage `==` de toutes les collections, plus `collections_on_ansible_version_mismatch = error` (repro L1 : `ERROR! Collection community.general does not support Ansible version 2.14.18`, rc=1). | **L1 (fait)** |
| R-02 | `ansible.cfg` du projet non lu (CWD, `ANSIBLE_CONFIG`, répertoire world-writable) : la configuration du contrôleur s'applique (F-20). | G1 : `ansible_config_file` = cfg du dépôt, et réglages effectifs non écrasés par `ANSIBLE_*`. | **L2 (fait)** |
| R-03 | Inventaire oublié : avec `/dev/null`, le play ne trouve aucun hôte et se termine sans rien faire. Une garde placée sur `hosts: pacemaker_nodes` **ne s'exécuterait pas**. | G5 sur `localhost` : `groups['pacemaker_nodes']` existe et contient exactement 2 hôtes. | **L2 (fait)** |
| R-04 | Play partiel (`--limit`, nœud injoignable) : corosync.conf à 1 nœud, puis redémarrage (F-06). | G6 : `ansible_limit` non défini sur les 2 plays, et `ansible_play_hosts_all` = `groups['pacemaker_nodes']` sur le play nœuds. La comparaison prévue au L0 (`ansible_play_hosts == ansible_play_hosts_all`) **ne détecte pas `--limit`** (F-26). Plus `any_errors_fatal` (F-33). | **L2 (fait)** ; L3 : `import_playbook: preflight.yml` |
| R-05 | `enable_repos` sans effet sur OL9 (F-04). | N6 : paquets disponibles dans les dépôts activés (**L2, fait**) ; `ha_cluster_enable_repos: false` explicite (L3). | L2 / L3 |
| R-06 | `stonith-enabled` absent : la VIP ne démarre jamais, alors que le rôle termine en succès (F-11). | Propriété déclarée (D2), plus un contrôle après exécution que la VIP est « Started ». | L3 |
| R-07 | Ressources ou propriétés manuelles purgées à chaque passage (F-07). | Tout déclarer ; comparer le CIB avant et après. | L3 |
| R-08 | IPaddr2 avec `nic` et `cidr_netmask` fournis : aucune vérification de sous-réseau. Sans `nic` : choix automatique de l'interface (F-14). | N2 (interface, IPv4 statique) et N3 (même réseau, même préfixe), sur les faits des nœuds. | **L2 (fait)** |
| R-09 | `ipaddr` renvoie `False` ou `/32` sans erreur (F-19). | G8 : `/` obligatoire, octets et préfixe (1 à 30) vérifiés par expression régulière, recoupés par `ansible.utils.ipv4`, ni réseau ni broadcast ; N3 : préfixe égal à celui de l'interface. | **L2 (fait)** |
| R-10 | `manage_firewall: true` réactive firewalld ; `manage_selinux` est sauté si firewalld est arrêté (F-15). | Ces variables restent explicitement à `false` ; assertion sur leur valeur. | L3 |
| R-11 | Mot de passe hacluster journalisé sur la cible (F-13). | `no_target_syslog = True` (repro L1). | **L1 (fait)** |
| R-12 | Mot de passe affiché côté contrôleur en `-vvv` (F-23, limite). | Contre-mesure à établir (P-07). | L3 |
| R-13 | Une modification de corosync.conf ou des clés arrête les 2 nœuds à la fois, et le rôle termine en succès (F-08). | Passage `--check --diff` d'abord, fenêtre de maintenance (CLAUDE.md §9). | L3 (procédure) |
| R-14 | Nom de cluster par défaut `my-cluster` (`LSR/defaults/main.yml:28`). | G7 : `pacemaker_cluster_name` non vide, différent de `my-cluster`, conforme à pcs (F-28). | **L2 (fait)** |
| R-15 | Test d'idempotence naïf faussé : `changed` vaut toujours vrai (F-09). | Test portant sur l'état. | L3 |
| R-16 | `collections_scan_sys_path` actif : une collection hors `./collections` serait trouvée dans `sys.path` (F-22). | `collections_scan_sys_path = False`. | **L1 (fait)** |
| R-17 | Variables de l'exemple chargées depuis un mauvais emplacement, sans erreur. | `group_vars/` à côté de `hosts.yml` ; chargement démontré par `ansible-inventory --host` (repro L1). Les variables nulles font échouer le preflight en G7 (repro L2). | **L1 / L2 (fait)** |
| R-18 | Garde sur une variable de gabarit via `hostvars` : `is defined` vrai alors que la source Vault manque (F-27). | G10 teste `vault_pacemaker_hacluster_password`, puis l'égalité `pacemaker_hacluster_password == vault_…`. Les valeurs contenant `{{`/`{%` sont refusées (G9, G10). | **L2 (fait)** |
| R-19 | `arping -D` renvoie 0 sans test sur une interface sans ARP (F-29). | N5 exige aussi « Sent N probes » (N >= 1) et « Received 0 response ». | **L2 (fait)** |
| R-20 | Preflight sauté par `--skip-tags always`. | Aucune garde technique possible : la consigne d'exploitation est de ne jamais l'utiliser (README). | ouvert |
| R-21 | N6 lance `dnf repoquery`, qui peut rafraîchir le cache des métadonnées dnf du nœud (écriture dans `/var/cache/dnf`, sans changer la configuration). | Écart accepté et signalé (README, `nodes.yml`). | L2 (signalé) |

## 5. Points ouverts

### Bloquants pour L3

| ID | Point |
|---|---|
| P-01 | Motif de la décision D2 (aucun fencing). |
| P-02 | Acceptation de `wait_for_all` : après un arrêt complet, la VIP reste arrêtée tant que les 2 nœuds ne se sont pas vus (F-12). |
| P-03 | Nombre de cartes réseau par VM ; lien corosync dédié ou lien unique (point de défaillance unique). Détermine `corosync_addresses`. |
| P-04 | Les noms d'inventaire se résolvent-ils, sur chaque nœud, vers l'IP du cluster ? Sinon `corosync_addresses` / `pcs_address` sont nécessaires (`LSR/README.md:1454-1490`). |
| P-05 | Nom du cluster (valeur de `pacemaker_cluster_name`). |
| P-07 | Contre-mesures à la fuite du mot de passe côté contrôleur (`-vvv`, `log_path`) (F-13, F-23). |

### Non bloquants pour l'écriture, mais à lever avant la mise en service

| ID | Point |
|---|---|
| P-06 | Les redémarrages complets du cluster lors des changements de configuration sont-ils acceptables (F-08) ? Faut-il des fenêtres de maintenance ? |
| P-08 | Filtrage ARP/MAC du vSwitch Hyper-V (basculement de la VIP) : interlocuteur et réglages. |
| P-09 | Compatibilité réelle du rôle 1.88.9 avec pcs 0.11.11 et pacemaker 2.1.10 : **lue dans le code, jamais testée**. |
| P-10 | Variables `ANSIBLE_*` présentes dans l'environnement du contrôleur : elles priment sur `ansible.cfg`. |
| P-11 | Présence de `python3-netaddr` >= 0.10.1 sur le contrôleur. G3 le vérifie et échoue sinon ; à constater au premier lancement. |
| P-12 | Contenu du miroir interne : ol9_addons, ol9_appstream et ol9_baseos_latest (F-17). N6 le vérifie et échoue sinon ; à constater au premier lancement. |
| P-13 | Faut-il contrôler les sha256 des tarballs de collections (§2) ? G4 ne contrôle que les versions. |
| P-14 | Licence du dépôt : `meta/main.yml` porte `license: "À VÉRIFIER"`. |
| P-15 | Accès aux nœuds : utilisateur distant et `sudo` (le play nœuds est en `become: true`). |
| P-16 | IP de la VIP sur les vrais nœuds : NetworkManager présent, profil actif `100 (connected)` en `ipv4.method manual` (F-30) ; `arping -D` sur le vSwitch Hyper-V (F-29, P-08). Non validable hors cible. |
| P-17 | Décisions L2 à valider : VIP en /31 ou /32 refusée (G8) ; `pacemaker_vip_nic` identique sur les 2 nœuds (un seul paramètre `nic` IPaddr2) ; N6 exige les paquets dans les dépôts activés même s'ils sont déjà installés. |

## 6. Commandes à lancer (hors portée de la machine de développement)

### Sur le contrôleur (depuis la racine du dépôt)

```
rpm -q ansible-core python3-netaddr; ansible --version
/usr/bin/python3 -c 'import netaddr; print(netaddr.__version__)'
env | grep '^ANSIBLE_'
ls -ld .                                    # ne doit pas être world-writable (F-20)
ansible-config dump --only-changed          # CONFIG_FILE() = <racine>/ansible.cfg
ansible-galaxy collection install -r collections/requirements.yml -p collections
ansible-galaxy collection list              # collections hors ./collections (information)
ansible-galaxy collection list -p collections
ansible-playbook -i inventories/<env> preflight.yml --ask-vault-pass   # L2 : rc=0 attendu
```

### Sur chaque nœud

```
cat /etc/os-release; systemd-detect-virt
dnf repolist
dnf list pacemaker pcs corosync corosync-qnetd libknet1-plugins-all resource-agents fence-agents-all fence-virt
ip -o -4 addr show; ip -o -4 route list match <VIP>/<préfixe> scope link
sed -n '213,244p' /usr/lib/ocf/lib/heartbeat/findif.sh     # comparer à RA v4.10.0 (F-14)
systemctl is-enabled firewalld; getenforce
getent hosts <nom_nœud_1> <nom_nœud_2>
ls -l /dev/watchdog*; wdctl                                 # information (F-24)
nmcli --escape no --get-values GENERAL.CONNECTION,GENERAL.STATE device show <nic>   # 100 (connected) (F-30)
nmcli --escape no --get-values ipv4.method connection show <profil>                 # manual
sudo arping -D -c 2 -I <nic> <VIP>          # rc 0 + « Received 0 response(s) » (F-29)
dnf --quiet repoquery --available --queryformat '%{name}' pcs corosync-qnetd openssl corosync libknet1-plugins-all resource-agents pacemaker fence-agents-all fence-virt
```

### Après un premier déploiement (sur un nœud)

```
corosync-quorumtool -s                       # flags 2Node et WaitForAll (F-12)
pcs property config --all | grep stonith     # stonith-enabled = false (D2)
pcs status; corosync-cfgtool -n              # VIP Started ; port knet effectif (F-16)
journalctl --since today | grep -c 'Invoked with'   # attendu 0 avec no_target_syslog (F-23, journald)
```

## 7. Journal des séries

| Série | Date | Livrable | Résultat |
|---|---|---|---|
| L0 | 2026-11-05 | Résolution en lecture seule : faits Q1–Q9, sans code. | Faits F-01 à F-19, F-24, F-25. Repro locale (ipaddr, dépendances Galaxy, chargement des vars OL9). |
| L1 | 2026-11-05 | Socle : CLAUDE.md, ce fichier, `collections/requirements.yml`, `ansible.cfg`, `.gitignore`, `inventories/example/`, README. | Repro locale : installation des 5 versions exactes ; mismatch → ERROR (et warning hors cfg) ; piège du cfg démontré ; `no_target_syslog` démontré (rsyslog) ; inventaire et group_vars chargés ; yamllint rc=0 ; ansible-lint rc=0 (profil production, ansible-lint 26.11.0 / ansible-core 2.21.6, **pas** 2.14). Faits F-20 à F-23 ajoutés. |
| L2 | 2026-11-05 | Preflight : `preflight.yml` et rôle `pacemaker_preflight` (gardes G1-G10, N1-N6), section README, CLAUDE.md §4, §6 (corrigé) et §14. | Repro locale et repro conteneur, sur ansible-core 2.14.18 : `--syntax-check` rc=0 ; chaque garde vérifiée en nominal et en échec forcé (77 cas, journaux hors dépôt) ; secret absent des sorties `-vvv` (0 occurrence) ; `inventories/example` échoue en G7. yamllint rc=0 ; ansible-lint rc=0 (profil production, ansible-lint 26.11.0 / ansible-core 2.21.6, **pas** 2.14). Faits F-26 à F-34 ajoutés, F-17 corrigé. |
