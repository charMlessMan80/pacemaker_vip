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
| **PM** | SRPM OL9 `pacemaker-2.1.10-3.0.1.el9_8.src.rpm` (oss.oracle.com, sha256 `d1f96df6414a4ac25dd5805b0aa13a33bf1eb40330069ad2199389f1c90dda3b`) : `pacemaker.spec`, et l'archive `pacemaker-5693eaeee.tar.gz` (code et `doc/sphinx/Pacemaker_Explained`) |
| **iputils** | archive GitHub `iputils-20210202.tar.gz` (sha256 `3f557ecfd2ace873801231d2c1f42de73ced9fbc1ef3a438d847688b5fb0e8ab`), `arping.c` ; page de manuel `arping.8` extraite du RPM OL9 `iputils-20210202-15.0.1.el9_7` (sha256 `ff498fa2625d945dde3b50f88e367d794c1f6b628da30d8e0885aac5df9f703e`) |

**Types de preuve**

| Mention | Signification |
|---|---|
| repoquery | `dnf repoquery` exécuté le 2026-11-05 sur `https://yum.oracle.com/repo/OracleLinux/OL9/{baseos/latest,appstream,addons}/x86_64/` |
| repro locale | conteneur `docker.io/library/oraclelinux:9` (Oracle Linux 9.8) avec le RPM `ansible-core-2.14.18-3.el9_8.1` et `python3-3.9.25-7.0.1.el9_8.2`. **Ce n'est ni le contrôleur ni un nœud.** |
| repro conteneur (L2) | contrôleur = repro locale + `python3-netaddr-0.10.1-3.el9` + `podman-remote` ; 2 nœuds `oraclelinux:9` avec systemd, NetworkManager et sudo (pv-n1 192.0.2.11/24, pv-n2 192.0.2.12/24, réseau podman 192.0.2.0/24, RFC 5737), connexion `containers.podman.podman` 1.21.2. Dépôts : yum.oracle.com le 2026-11-05. **Ni le contrôleur, ni les nœuds, ni Hyper-V.** |

| repro conteneur (L3a/L3b) | idem L2, plus les RPM OL9 `pacemaker-2.1.10-3.0.1.el9_8`, `pcs-0.11.11-2.el9_8.1`, `corosync-3.1.10-1.el9_8.2`, `resource-agents-4.10.0-111.el9_8.19`, `openssh-clients-9.9p1-7.0.2.el9_8` ; L3b : rôle ha_cluster 1.88.9 **réellement exécuté** sur pv-n1/pv-n2 par un playbook jetable hors dépôt (cluster formé, VIP démarrée). **Ni le contrôleur, ni les nœuds, ni Hyper-V.** |
| repro conteneur (L3c) | idem L2, nœuds **vierges** recréés depuis `oraclelinux:9` + NetworkManager + sudo, dépôt `ol9_addons` activé, **sans** pcs/pacemaker/corosync ni `/var/lib/pcsd` ; `cluster.yml` du dépôt exécuté de bout en bout (preflight, C5, D21, rôle 1.88.9, V1-V5). Arrêt/démarrage d'un nœud = `podman stop`/`podman start` (arrêt propre par systemd). **Ni le contrôleur, ni les nœuds, ni Hyper-V.** |
| cible (utilisateur) | sorties fournies par l'utilisateur le 2026-11-05, lancées sur le contrôleur et les nœuds réels. |

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

### Constats sur cible (2026-11-05)

Source : sorties fournies par l'utilisateur le 2026-11-05 (contrôleur `<contrôleur>`, nœuds `<nœud-1>` et `<nœud-2>`).

| ID | Constat | Statut |
|---|---|---|
| C-01 | Contrôleur OL9, ansible-core 2.14.18 : preflight G1-G10 OK (version L2). | constaté (cible) |
| C-02 | Nœuds <IP nœud 1>/24 et <IP nœud 2>/24, inventaire hors dépôt avec `ansible_host` : N1-N6 OK ; SSH en root, `become` OK. | constaté (cible) |
| C-03 | N5 démontrée dans les deux sens sur le vSwitch Hyper-V réel : avec la VIP égale à l'IP du contrôleur (<IP contrôleur>), échec « répond déjà sur eth0 » (réponse ARP reçue). | constaté (cible) |
| C-04 | Une seule interface par nœud : `eth0` (plus `lo`). | constaté (cible) |
| C-05 | `/etc/hosts` ne contient pas les noms des nœuds ; nsswitch `hosts: files dns myhostname` ; `getent ahosts <nom court>` donne l'IPv4 par DNS (FQDN `.<domaine interne>`) ; `getent hosts <nom propre>` donne `fe80::…` (fourni par myhostname). | constaté (cible) |
| C-06 | `uname -n` = nom d'inventaire exact sur chaque nœud (`<nœud-1>`, `<nœud-2>`). | constaté (cible) |
| C-07 | Interpréteur découvert sur les nœuds : `/usr/bin/python3.9`, avec l'avertissement « interpreter discovery ». | constaté (cible) |
| C-08 | Harnais de session (conservé) : `ANSIBLE_SSH_EXTRA_ARGS` (`UserKnownHostsFile` éphémère, `StrictHostKeyChecking=yes`), `ANSIBLE_SSH_CONTROL_PATH_DIR`, `ANSIBLE_LOCAL_TEMP`, `ANSIBLE_LOG_PATH` (répertoire `/tmp/tmp.*` en 755, journal en 644). G1 (L2) ne les voyait pas. | constaté (cible) |
| C-09 | Preflight L3b complet OK sur la cible (contrôleur + 2 nœuds) : G1 env, G1 ssh, C1, C2, C4, G2-G12, N1-N7. **C3 sautée** : ce lancement ne portait pas le harnais (aucune variable `ANSIBLE_*`), C3 n'est donc pas encore exercée sur cible. | constaté (cible, 2026-11-05) |
| C-10 | Inventaire réel : `pacemaker_node_ip` = <IP nœud 1> et <IP nœud 2>, égal à `ansible_host` ; N7 OK. | constaté (cible, 2026-11-05) |
| C-11 | Nœuds : `rpm -q pcs` → non installé ; `/var/lib/pcsd` absent (**R-32 confirmé sur cible**) ; `getenforce` → `Disabled`. | constaté (cible, 2026-11-05) |
| C-12 | Le contrôleur a fait `git pull` en HTTPS sans authentification : le dépôt est **public**. Motif supplémentaire pour garder l'inventaire réel hors du dépôt. | constaté (cible, 2026-11-05) |

### Résolution avant le playbook cluster (L3a)

| ID | Fait | Source | Statut |
|---|---|---|---|
| F-35 | Nom de nœud = `ha_cluster.node_name`, sinon `inventory_hostname`. corosync identifie le nœud local en comparant `uname -n` (exact, puis sans domaine) au `name` du nodelist ; à défaut, il appelle `getnameinfo()` sur chaque adresse locale (**résolution inverse**). Pacemaker lit le `name` du nodelist par nodeid, sans résolution. pcs joint les nœuds par l'adresse de `known-hosts` (`dest_list`). | `LSR/tasks/shell_pcs/check-and-prepare-role-variables.yml:3-9` ; `corosync-src exec/totemconfig.c:648-860` ; `PM lib/cluster/corosync.c:170-211` ; repro conteneur L3a | PROUVÉ |
| F-36 | `ip_version` vaut `ipv6-4` par défaut avec knet (`ipv4` seulement avec udp) et ne sert qu'à résoudre les `ringX_addr`. pcs ne résout pas un littéral IP. `ip_version=ipv4` en transport knet est accepté et écrit dans `totem`. | `corosync-src exec/totemconfig.c:916-939`, `exec/totemip.c:306-366` ; `pcs-src pcs/lib/corosync/node.py:84-96` ; repro locale L3a | PROUVÉ (absence de requête DNS sur un littéral : À VÉRIFIER, `strace` sur cible) |
| F-37 | Sans `corosync_addresses`, `pcs cluster setup --corosync_conf` prend l'adresse de `known-hosts`, sinon le nom (qui doit alors se résoudre : « Unable to resolve addresses », erreur sans `--force`). Le rôle authentifie **après** avoir généré corosync.conf : l'adresse change entre le premier et le second passage. | `pcs-src pcs/lib/commands/cluster.py:513-530` ; `LSR/tasks/main.yml:72-79` ; repro locale L3a (cas B et C) | PROUVÉ |
| F-38 | Le rôle s'authentifie seulement là où `pcs status pcsd -- <noms>` rend 2 : nœud non authentifié **ou** pcsd arrêté. Les jetons ne sont synchronisés que si corosync.conf existe ; ils survivent au redémarrage de pcsd, à la régénération de son certificat (pcs ne le vérifie pas) et au changement du mot de passe hacluster. Aucune variable du rôle n'évite l'authentification par mot de passe. | `LSR/tasks/shell_pcs/cluster-auth.yml:3-31` ; `pcs-src pcs/pcsd.py:224-285`, `pcsd/pcs.rb:1049-1053`, `pcs/common/node_communicator.py:556-557` ; repro conteneur L3a | PROUVÉ |
| F-39 | `ansible_verbosity` = verbosité effective (`-v`, `ANSIBLE_VERBOSITY`, `verbosity` du cfg cumulés). `invocation` (dont un `stdin`) s'affiche dès la verbosité 3. Le journal reprend l'écran ; il est créé au démarrage avec les droits de l'umask (644 avec 022). Un chemin non inscriptible ne donne qu'un avertissement. | `AC/utils/vars.py:211-230` ; `AC/cli/arguments/option_helpers.py:212` ; `AC/plugins/callback/__init__.py:234-235` ; `AC/utils/display.py:132-144,310-322` ; repro locale L3a | PROUVÉ |
| F-40 | `wait_for_all` n'accepte que `"0"` ou `"1"` (un booléen YAML `false` est refusé). pcs 0.11.11 accepte `two_node: 1` avec `wait_for_all: 0` sans avertissement. Repro du rôle (L3b) : `corosync-quorumtool -s` affiche `Flags: 2Node Quorate`, sans `WaitForAll`. | `pcs-src pcs/lib/corosync/config_validators.py:1698-1830` ; `corosync-src tools/corosync-quorumtool.c:629-637,707` ; repro locale L3a ; repro conteneur L3b | PROUVÉ |
| F-41 | La purge du rôle supprime `rsc_defaults` et `op_defaults` en entier. Le build OL9 crée les CIB neufs avec `resource-stickiness=1` (`build-resource-defaults`), que cette purge efface ; le défaut de Pacemaker est alors 0. Sans contrainte de localisation, la VIP ne revient pas sur un nœud redémarré, que la stickiness vaille 0 ou 1 (`crm_simulate`). `multiple-active` vaut `stop_start` : arrêt sur les 2 nœuds, puis démarrage sur un seul. | `LSR/tasks/shell_pcs/create-and-push-cib.yml:55-60` ; `PM pacemaker.spec:181-184`, `lib/cib/cib_utils.c:248-260`, `doc/sphinx/Pacemaker_Explained/resources.rst:443-449,521-536` ; repro locale L3a | PROUVÉ |
| F-42 | IPaddr2 (RPM OL9) : opérations copiées de l'agent par défaut (`copy_operations_from_agent: true`) : `monitor interval=10s timeout=20s`, `start timeout=20s`, `stop timeout=20s`. | `LSR/tasks/shell_pcs/pcs-cib-resource-primitive.yml:23-28` ; repro locale L3a (`pcs -f`, métadonnées de l'agent) | PROUVÉ |
| F-43 | Précédence : défauts du rôle < group/host vars < play vars < … < extra vars. `-e var=false` passe la **chaîne** `"false"` ; `when: ha_cluster_enable_repos` (sans `bool`) exécute alors la branche. | `AC/vars/manager.py:203-420` ; `LSR/tasks/main.yml:9` ; repro locale L3a | PROUVÉ |
| F-44 | Oracle Linux (`ID=ol`) est absent de la table de découverte de l'interpréteur : Ansible prend le premier trouvé de `INTERPRETER_PYTHON_FALLBACK`, qui commence par python3.11. `/usr/bin/python3` reste un lien vers `python3.9` (RPM `python3`, sans alternatives). Avec `ansible_python_interpreter: /usr/bin/python3.9` en vars du play, l'interpréteur reste 3.9 même avec python3.11 installé, sans avertissement. | `AC/config/base.yml:1531-1563` ; `AC/executor/interpreter_discovery.py:114-117,160-166` ; repro locale L3a ; repro conteneur L3b (ad hoc sans variable : `/usr/bin/python3.11` et avertissement ; play 2 du preflight : `/usr/bin/python3.9`, 0 avertissement) | PROUVÉ |
| F-45 | En `--check`, un premier déploiement échoue à « Fetch pcs capabilities » (pcs absent). Sur un cluster existant, le diff du CIB est calculé mais n'est visible qu'en `-v`, et le push est sauté. | `LSR/tasks/shell_pcs/check-and-prepare-role-variables.yml:141-146` ; `create-and-push-cib.yml:257-288` ; `AC/modules/command.py:319-331` | PROUVÉ (code) |
| F-46 | `openssl passwd -6 -salt … <mot de passe>` met le mot de passe hacluster dans l'argv du processus, sur le nœud ; `no_log` ne le couvre pas. | `LSR/tasks/main.yml:22-36` | PROUVÉ (code) |

### Preflight étendu (L3b)

| ID | Fait | Source | Statut |
|---|---|---|---|
| F-47 | **Origine des réglages.** Elle est calculée par `ConfigManager.get_config_value_and_origin` (`env: <NOM>`, chemin du cfg, `cli: …`). Le lookup `config` ne rend que la valeur. `ansible-config dump --only-changed -t all -f json` la publie (entrées `name`, `value`, `origin`, `type`), en 0,2 s en repro, mais il **n'énumère pas les plugins des collections** et ignore un nom `ANSIBLE_*` qui n'est pas un réglage. G1 combine donc ce dump et les noms `ANSIBLE_*` de l'environnement. `--ssh-extra-args`, les extra vars et le cfg sont vus par le lookup `config` avec `plugin_type`. | `AC/config/manager.py:461-515` ; `AC/plugins/lookup/config.py:14-35` ; `AC/cli/config.py:85-91,398-545` ; `AC/plugins/loader.py:947` ; repro locale L3b (`ANSIBLE_PODMAN_EXECUTABLE`, `ANSIBLE_FOO_TYPO` absents du dump) | PROUVÉ |
| F-48 | `ssh -o StrictHostKeyChecking=` : `no`, `off` et `false` donnent « false », `accept-new` accepte toute clé nouvelle ; la casse est ignorée ; les formes `X=Y`, `X Y` et `-oX=Y` sont valides ; avec deux `-o`, la première valeur l'emporte. `UserKnownHostsFile` accepte une liste de fichiers. `host_key_checking` faux ajoute `-o StrictHostKeyChecking=no`. | repro conteneur L3b (`ssh -G`, openssh-clients-9.9p1-7.0.2.el9_8) ; `AC/plugins/connection/ssh.py:37-55,722-724` | PROUVÉ |
| F-49 | `ansible_facts['nodename']` = `platform.node()` (nodename d'uname) ; `hostname` n'en garde que la partie avant le premier point. `ansible_facts[<nic>]['ipv4']` est l'adresse **principale** (`ip addr show primary`) ; la VIP posée par IPaddr2 dans le même réseau est **secondaire** (`ipv4_secondaries`). | `AC/module_utils/facts/system/platform.py:55-56` ; `AC/module_utils/facts/network/linux.py:191-214,263-276` ; repro conteneur L3b (VIP 192.0.2.100 secondaire, N7 OK) | PROUVÉ |
| F-50 | **Rôle 1.88.9 sur un nœud vierge** : `/var/lib/pcsd` est `%ghost` dans `pcs-0.11.11-2.el9_8.1` (non créé à l'installation) ; il est créé par `StateDirectory=pcsd` (0700) au premier démarrage de pcsd. Le rôle arrête pcsd puis écrit `pcs_settings.conf` : **échec** « Destination directory /var/lib/pcsd does not exist ». Après un démarrage préalable de pcsd, le rôle termine (rc=0) : cluster formé, VIP `Started` sur un nœud, adresses IPv4, `stonith-enabled=false`, `resource-stickiness=1`. | repro conteneur L3b (`rpm -q --qf '[%{FILENAMES} %{FILEFLAGS:fflags}]' pcs`, `pcsd.service`) ; `LSR/tasks/shell_pcs/configure-shell.yml:6-9,79-86` | PROUVÉ (repro) ; sur cible À VÉRIFIER (§6) |
| F-51 | Avec `ha_cluster_fence_agent_packages: []`, ni `fence-agents-all` ni `fence-virt` ne sont installés ; mais `/etc/cluster/fence_xvm.key` (512 octets) est **toujours distribuée** sur x86_64. Aucune variable ne l'empêche : seule l'architecture conditionne la tâche, et `ha_cluster_fence_virt_key_src` ne change que la source. | `LSR/vars/RedHat.yml:48` ; `LSR/defaults/main.yml:17` ; `LSR/tasks/main.yml:63-67` ; `LSR/tasks/distribute-fence-virt-key.yml:3-23` ; repro conteneur L3b | PROUVÉ |
| F-52 | **Preuve D18** (rôle réel, `--check --diff`, verbosité 0, clés marquées sans octet nul). (a) Cluster existant, clés inchangées : 0 occurrence. (b) Clés supprimées sur pv-n2 : les clés corosync, pacemaker et fence_xvm de pv-n1 s'affichent en « after ». (c) `ha_cluster_regenerate_keys: true` : les 3 clés en place s'affichent en « before » (2 lignes chacune). Seule une clé contenant un octet nul est masquée (environ 63 % des clés de 256 octets). | repro conteneur L3b ; `AC/plugins/action/__init__.py:1371-1405` ; `LSR/tasks/presharedkey.yml` | PROUVÉ |

### Playbook cluster (L3c)

| ID | Fait | Source | Statut |
|---|---|---|---|
| F-53 | **hostvars et play vars.** `hostvars[h]` appelle `get_vars(host=h)` sans play : il contient l'inventaire, les extra vars, les faits, `set_fact`/`register`, mais **pas** les play vars. Le rôle lit `ha_cluster` localement (`set_fact` de `__ha_cluster_node_name` et `__ha_cluster_local_node`) et ne lit par `hostvars` que ces faits dérivés : une variable de play `ha_cluster` construite avec `inventory_hostname` et `pacemaker_node_ip` donne le bon nodelist (V4 : `ring0_addr` = `pacemaker_node_ip`). | `AC/vars/hostvars.py:78,93-104` ; `AC/vars/manager.py:319-320,403-407` ; `LSR/tasks/shell_pcs/check-and-prepare-role-variables.yml:3-9,30-40`, `pcs-cluster-setup-pcs-0.10.yml:14-18`, `cluster-auth-pcs-0.10.yml:9-12` ; repro conteneur L3c (playbook jetable : variable de play absente de `hostvars` propre et du pair, `set_fact` dérivé visible chez le pair, extra var présente) | PROUVÉ |
| F-54 | **Unité pcsd (RPM OL9 `pcs-0.11.11-2.el9_8.1.x86_64`, sha256 `e3df73fae74b61277098580aca779d5146174313ee68398eb7d8960f5b9cb4c6`).** `/usr/lib/systemd/system/pcsd.service:13-14` : `StateDirectory=pcsd`, `StateDirectoryMode=0700` ; pas de `User=` (root). Le RPM déclare `/var/lib/pcsd` en `%ghost` (`g drwx------ root:root`). D21 (installer pcs, démarrer pcsd si le répertoire manque) produit `drwx------ root:root /var/lib/pcsd` ; le rôle termine ensuite sans erreur sur nœud vierge. | RPM téléchargé le 2026-11-05 (`dnf download pcs`, `rpm -qp --qf '%{FILEFLAGS:fflags} …'`, `rpm2cpio`) ; repro conteneur L3c | PROUVÉ |
| F-55 | **Syntaxe `ha_cluster_resource_defaults`** : `meta_attrs:` liste d'ensembles, chacun avec `attrs:` (`name`/`value`), `id`, `rule` et `score` facultatifs. Résultat effectif : `<rsc_defaults><meta_attributes id="rsc_defaults-meta_attributes"><nvpair … name="resource-stickiness" value="1"/>`. | `LSR/README.md:850-886` ; `LSR/tasks/shell_pcs/create-and-push-cib.yml:116-122` ; repro conteneur L3c (`cibadmin --query --scope rsc_defaults`) | PROUVÉ |
| F-56 | **Mécanique Ansible utilisée par C5 et V1-V5.** La clé de fusion YAML `<<` est gérée (`flatten_mapping`). Une liste construite dans une boucle Jinja (`{% set l = [] %}…{{ l }}`, bloc `>-`) est rendue comme une liste. Un `until` épuisé marque la tâche en échec (sortie brute) : d'où la condition « conforme ou `attempts >= N` » suivie d'un `assert`. Dans un bloc `>-`, une référence arrière s'écrit `'\1'` (une barre) ; `'\\1'` provoque `'NoneType' object has no attribute 'group'`. Couper une chaîne Jinja en fin de ligne d'un bloc `>-` y insère une espace. | `AC/parsing/yaml/constructor.py:50-58` ; `AC/executor/task_executor.py:610-619,724-781` ; repro locale L3c | PROUVÉ |
| F-57 | **Second passage sans changement** (repro conteneur L3c, fichiers définitifs) : rc=0 ; tâche « Stop cluster daemons to reload configuration » **sautée** sur les 2 nœuds ; corosync.conf sha256 identique sur chaque nœud (`32b7f1ad…`) ; CIB `--scope configuration` identique octet pour octet (1822 octets) ; historique d'opérations de `vip` inchangé (`start` call 6, même `last-rc-change`), même porteur. Le compteur `changed` reste non nul (18 et 5). D21 sauté (répertoire présent). | repro conteneur L3c (`final.out` §3) | PROUVÉ (repro) |
| F-58 | **D11** : `podman stop` du porteur → VIP `Started` sur l'autre nœud en 1 à 2 s, `Flags: 2Node Quorate` ; au retour du nœud (en ligne en 4 s), la VIP **reste** sur le nœud qui l'a reprise (historique d'opérations identique 30 s après). `corosync` et `pacemaker` `enabled` (`start_on_boot`). | repro conteneur L3c (`d11.out`, `final.out` §7) | PROUVÉ (repro, arrêt propre) |
| F-59 | **D8** : les deux nœuds arrêtés, un seul redémarré → `Nodes: 1`, `Total votes: 1`, `Quorate: Yes`, `Flags: 2Node Quorate`, VIP `Started` en 23 à 25 s ; au retour du pair, la VIP ne bouge pas ; V1-V5 OK. | repro conteneur L3c (`d8.out`, `final.out` §8) | PROUVÉ (repro) |
| F-60 | **Sorties lues par V1-V5** (pacemaker 2.1.10, corosync 3.1.10) : `crm_mon --one-shot --output-as=xml` donne `<cluster_options stonith-enabled=…>` et `<resource id="vip" … role="Started" … nodes_running_on="1">` suivi de `<node name=…>` ; `corosync-cmapctl -b nodelist.node` donne `nodelist.node.N.name (str) = …` et `.ring0_addr (str) = …` ; `corosync-cmapctl -g totem.ip_version` → `ipv4` ; `corosync-cfgtool -n` → `LINK: 0 udp (<local>-><pair>) enabled connected` ; `cibadmin --query --xpath` sans résultat rend **105** (« No such device or address »). | repro conteneur L3c (`state_after_deploy1.txt`) | PROUVÉ |
| F-61 | **NetworkManager et VIP** : `nmcli con up <profil actif>` sur le nœud porteur **retire la VIP** (adresse posée hors NetworkManager) ; le moniteur IPaddr2 (10 s, F-42) la déclare `not running` et Pacemaker la repose sur le même nœud, avec `fail-count=1`. Artefact de repro distinct : après `podman start`, netavark configure eth0 avant NetworkManager, qui reprend l'interface en connexion externe (`connected (externally)`), ce qui fait échouer N2 ; sans objet sur une VM. | repro conteneur L3c (`nm_reactivation.out`, journal NetworkManager `connection-assumed`) | PROUVÉ (repro) ; sur cible À VÉRIFIER |
| F-62 | `ansible-lint --exclude` prend plusieurs valeurs : placé avant les chemins, il les avale comme exclusions et l'outil analyse d'autres fichiers (« 3 files processed »). Les chemins se placent avant `--exclude=…`. | repro locale L3c (ansible-lint 26.11.0, `-vv` « Examining ») | PROUVÉ |

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
| D2 | 2026-11-05 | **Aucun fencing.** Risque accepté : split-brain possible, donc VIP active sur les deux nœuds ; configuration non supportée par Red Hat (F-25). `stonith-enabled: false` sera déclaré dans les variables du rôle (F-07, F-11). | Utilisateur, 2026-11-05 : les services tournent sur les deux nœuds ; une IP en double, dans les cas qui peuvent y conduire, ne pose pas de problème. |
| D3 | 2026-11-05 | VIP fournie en une variable au format CIDR, interface en variable. Nœuds et VIP dans le même /24. | Prémisse utilisateur. |
| D4 | 2026-11-05 | firewalld désactivé sur les nœuds ; filtrage géré par l'infra réseau. | Prémisse utilisateur. Conséquence : `ha_cluster_manage_firewall: false` obligatoire (F-15). |
| D5 | 2026-11-05 | Mot de passe hacluster via Ansible Vault. | Prémisse utilisateur. |
| D6 | 2026-11-05 | Projet indépendant : aucune convention héritée d'un autre dépôt. | Décision utilisateur. |
| D7 | 2026-11-05 | Contrôleur distinct de la machine de développement ; ni contrôleur ni nœuds joignables depuis celle-ci. | Prémisse utilisateur. |
| D8 | 2026-11-05 | `wait_for_all: "0"` (avec `two_node: 1`). Arbitrage exact : un nœud qui démarre sans voir un pair pourtant vivant devient quorate seul et duplique la VIP, puis `multiple-active=stop_start` la coupe brièvement au retour du lien ; en échange, si un nœud est mort et que l'autre redémarre, la VIP redémarre sans intervention. | Décision utilisateur (F-12, F-40, F-41). |
| D9 | 2026-11-05 | Lien corosync unique (`eth0`). | Décision utilisateur ; une seule interface par nœud (C-04). |
| D10 | 2026-11-05 | Adresses IPv4 explicites : variable par hôte `pacemaker_node_ip`, adresse corosync et pcs, contrôlée contre l'IPv4 principale de `pacemaker_vip_nic` (G11, N7). | Décision utilisateur ; aucune dépendance à la résolution de noms (C-05, F-35 à F-37). |
| D11 | 2026-11-05 | Mises à jour et redémarrages des nœuds en décalé ; le nœud restant reprend la VIP. | Décision utilisateur (F-41). |
| D12 | 2026-11-05 | **Révisée** : pas de pré-authentification pcs. Contre-mesures : gardes C1-C4 (verbosité, débogage, journal, diff). `-vvv` interdit à l'écran comme dans le journal. | Décision utilisateur (F-38, F-39). |
| D13 | 2026-11-05 | `ha_cluster_resource_defaults` avec `resource-stickiness = 1` déclaré. | Décision utilisateur : défaut du build OL9, sinon purgé par le rôle (F-41, R-26). |
| D14 | 2026-11-05 | `ip_version = ipv4` dans le transport knet dès le premier déploiement. | Décision utilisateur (F-36) ; l'ajouter plus tard modifierait corosync.conf (F-08). |
| D15 | 2026-11-05 | `ansible_python_interpreter = /usr/bin/python3.9` en vars des plays nœuds. | Décision utilisateur (F-44, C-07). |
| D16 | 2026-11-05 | Journal Ansible autorisé seulement s'il est protégé (garde C3). | Décision utilisateur (F-39). |
| D17 | 2026-11-05 | G1 généralisée : toute origine `env:` est refusée, sauf `ANSIBLE_SSH_EXTRA_ARGS`, `ANSIBLE_SSH_CONTROL_PATH_DIR`, `ANSIBLE_LOCAL_TEMP`, `ANSIBLE_LOG_PATH` ; vérification des clés SSH jamais désactivée (G1, G12). | Décision utilisateur (C-08, F-47, F-48). |
| D18 | 2026-11-05 | `--diff` autorisé seulement avec `--check`, et seulement s'il est prouvé qu'il n'affiche pas les authkeys ; sinon interdit en toutes circonstances. **Résultat L3b : interdit en toutes circonstances** (F-52, garde C4). | Décision utilisateur ; preuve L3b. |
| D19 | 2026-11-05 | `ha_cluster_fence_agent_packages: []` : aucun paquet de fencing installé. | Décision utilisateur (D2) ; `fence_xvm.key` reste distribuée (F-51, R-33). |
| D20 | 2026-11-05 | Nom de nœud = `inventory_hostname`, égal à `uname -n` (N7). Identifiant de la ressource VIP : `vip` (L3c). | Décision utilisateur (C-06, F-35). |
| D21 | 2026-11-05 | Contournement de R-32 par l'option (i) : en `pre_tasks`, si `/var/lib/pcsd` est absent, installer pcs et démarrer pcsd une fois, pour que systemd crée le répertoire (`StateDirectory`) ; vérifier 0700 root:root. Option (ii) (créer le répertoire soi-même) écartée : elle réimplémenterait mode, propriétaire et contexte SELinux de systemd. | Décision utilisateur (P-19, C-11) ; validé en repro L3c (F-54). |
| D22 | 2026-11-05 | R-33 accepté : `/etc/cluster/fence_xvm.key` distribuée par le rôle sur x86_64, inerte sans `fence_virtd` ; ne pas la supprimer (le rôle la redistribuerait à chaque passage). | Décision utilisateur (P-21, F-51). |

## 4. Risques d'échec silencieux et gardes

| ID | Risque | Garde prévue | Livrable |
|---|---|---|---|
| R-01 | Dépendances Galaxy incompatibles installées sans avertissement (F-03). | Épinglage `==` de toutes les collections, plus `collections_on_ansible_version_mismatch = error` (repro L1 : `ERROR! Collection community.general does not support Ansible version 2.14.18`, rc=1). | **L1 (fait)** |
| R-02 | `ansible.cfg` du projet non lu (CWD, `ANSIBLE_CONFIG`, répertoire world-writable) : la configuration du contrôleur s'applique (F-20). | G1 : `ansible_config_file` = cfg du dépôt, et réglages effectifs non écrasés par `ANSIBLE_*`. | **L2 (fait)** |
| R-03 | Inventaire oublié : avec `/dev/null`, le play ne trouve aucun hôte et se termine sans rien faire. Une garde placée sur `hosts: pacemaker_nodes` **ne s'exécuterait pas**. | G5 sur `localhost` : `groups['pacemaker_nodes']` existe et contient exactement 2 hôtes. | **L2 (fait)** |
| R-04 | Play partiel (`--limit`, nœud injoignable) : corosync.conf à 1 nœud, puis redémarrage (F-06). | G6 : `ansible_limit` non défini sur les 2 plays, et `ansible_play_hosts_all` = `groups['pacemaker_nodes']` sur le play nœuds. La comparaison prévue au L0 (`ansible_play_hosts == ansible_play_hosts_all`) **ne détecte pas `--limit`** (F-26). Plus `any_errors_fatal` (F-33). | **L2 (fait)** ; L3 : `import_playbook: preflight.yml` |
| R-05 | `enable_repos` sans effet sur OL9 (F-04). | N6 : paquets disponibles dans les dépôts activés (**L2, fait**) ; `ha_cluster_enable_repos: false` explicite (L3). | L2 / L3 |
| R-06 | `stonith-enabled` absent : la VIP ne démarre jamais, alors que le rôle termine en succès (F-11). | Propriété déclarée (D2) ; V2 (options effectives de crm_mon) puis V1 (VIP `Started` sur un seul nœud, adresse posée sur ce seul nœud). | **L3c (fait)** |
| R-07 | Ressources ou propriétés manuelles purgées à chaque passage (F-07). | Tout déclarer dans `cluster.yml` (C5 refuse les variables `ha_cluster*` hors déclaration) ; second passage : CIB identique (F-57). Consigne : aucune configuration manuelle (README). | **L3c (fait)** |
| R-08 | IPaddr2 avec `nic` et `cidr_netmask` fournis : aucune vérification de sous-réseau. Sans `nic` : choix automatique de l'interface (F-14). | N2 (interface, IPv4 statique) et N3 (même réseau, même préfixe), sur les faits des nœuds. | **L2 (fait)** |
| R-09 | `ipaddr` renvoie `False` ou `/32` sans erreur (F-19). | G8 : `/` obligatoire, octets et préfixe (1 à 30) vérifiés par expression régulière, recoupés par `ansible.utils.ipv4`, ni réseau ni broadcast ; N3 : préfixe égal à celui de l'interface. | **L2 (fait)** |
| R-10 | `manage_firewall: true` réactive firewalld ; `manage_selinux` est sauté si firewalld est arrêté (F-15). | Variables déclarées `false` dans `cluster.yml` ; C5 compare valeur et type (`-e '{"ha_cluster_manage_firewall": true}'` refusé). | **L3c (fait)** |
| R-11 | Mot de passe hacluster journalisé sur la cible (F-13). | `no_target_syslog = True` (repro L1). | **L1 (fait)** |
| R-12 | Mot de passe affiché côté contrôleur en `-vvv` (F-23, limite). | C1 (verbosité 0), C2, C3 (D12 révisée). | **L3b (fait)** |
| R-13 | Une modification de corosync.conf ou des clés arrête les 2 nœuds à la fois, et le rôle termine en succès (F-08). | Fenêtre de maintenance ; modification par `cluster.yml` ou l'inventaire, jamais `-e` (C5) ; **ni `--check` ni `--diff`** (D18, R-28). Constaté en repro L3c (`wait_for_all` remis à 0 : « Stop cluster daemons » `changed`). | **L3c (procédure, CLAUDE.md §9, README)** |
| R-14 | Nom de cluster par défaut `my-cluster` (`LSR/defaults/main.yml:28`). | G7 : `pacemaker_cluster_name` non vide, différent de `my-cluster`, conforme à pcs (F-28). | **L2 (fait)** |
| R-15 | Test d'idempotence naïf faussé : `changed` vaut toujours vrai (F-09). | Test sur l'état : sha256 de corosync.conf, CIB, historique d'opérations de `vip`, tâche d'arrêt sautée (F-57). | **L3c (fait)** |
| R-16 | `collections_scan_sys_path` actif : une collection hors `./collections` serait trouvée dans `sys.path` (F-22). | `collections_scan_sys_path = False`. | **L1 (fait)** |
| R-17 | Variables de l'exemple chargées depuis un mauvais emplacement, sans erreur. | `group_vars/` à côté de `hosts.yml` ; chargement démontré par `ansible-inventory --host` (repro L1). Les variables nulles font échouer le preflight en G7 (repro L2). | **L1 / L2 (fait)** |
| R-18 | Garde sur une variable de gabarit via `hostvars` : `is defined` vrai alors que la source Vault manque (F-27). | G10 teste `vault_pacemaker_hacluster_password`, puis l'égalité `pacemaker_hacluster_password == vault_…`. Les valeurs contenant `{{`/`{%` sont refusées (G9, G10). | **L2 (fait)** |
| R-19 | `arping -D` renvoie 0 sans test sur une interface sans ARP (F-29). | N5 exige aussi « Sent N probes » (N >= 1) et « Received 0 response ». | **L2 (fait)** |
| R-20 | Preflight sauté par `--skip-tags always`. | Aucune garde technique possible : la consigne d'exploitation est de ne jamais l'utiliser (README). | ouvert |
| R-21 | N6 lance `dnf repoquery`, qui peut rafraîchir le cache des métadonnées dnf du nœud (écriture dans `/var/cache/dnf`, sans changer la configuration). | Écart accepté et signalé (README, `nodes.yml`). | L2 (signalé) |
| R-22 | `pcs_address` sans `corosync_addresses` : l'adresse corosync change entre le premier et le second passage, d'où un redémarrage complet (F-37). | D10 : `ha_cluster.pcs_address` et `corosync_addresses` = `pacemaker_node_ip` ; G11, N7 ; V4 (`ring0_addr`, lien knet connecté). | **L3c (fait)** |
| R-23 | `-e var=false` passe une chaîne : la branche `enable_repos` s'exécute (F-43). | C5 : comparaison JSON (valeur et type) à la source unique de `cluster.yml`. | **L3c (fait)** |
| R-24 | `--diff` affiche les clés corosync, pacemaker et fence_xvm quand elles changent, même en `--check` (F-52). | C4 : `--diff` refusé en toutes circonstances (D18). | **L3b (fait)** |
| R-25 | Un python3.11 installé change l'interpréteur des modules (F-44). | D15 en vars du play nœuds du preflight et du play cluster. | **L3c (fait)** |
| R-26 | La purge du rôle efface `resource-stickiness=1` du build OL9 (F-41). | D13 : stickiness déclarée ; V5 (`rsc_defaults` = 1, non redéfini sur `vip`). | **L3c (fait)** |
| R-27 | Mot de passe hacluster dans l'argv d'`openssl` sur le nœud (F-46). | Aucune : propre au rôle. | **accepté** (2026-11-05) |
| R-28 | Premier déploiement non simulable en `--check` (F-45). | Aucune. | **accepté** (2026-11-05) |
| R-29 | Nom de nœud différent de `uname -n` : corosync passe par une résolution inverse (F-35). | N7 : `inventory_hostname == ansible_facts['nodename']` (D20). | **L3b (fait)** |
| R-30 | Journal Ansible créé avec les droits de l'umask (F-39) ; cas du harnais : 755/644 (C-08). | C3 (D16). | **L3b (fait)** |
| R-31 | `pcs status pcsd` rend 2 aussi quand pcsd est arrêté : le rôle repasse le mot de passe en `stdin` (F-38). | C1 : verbosité 0, le `stdin` n'est pas affiché. | **L3b (couvert)** |
| R-32 | Rôle 1.88.9 en échec sur un nœud vierge : `/var/lib/pcsd` absent (F-50 ; constaté absent sur cible, C-11). | D21 (`tasks/pcsd_state_dir.yml`) : démarrage préalable de pcsd si le répertoire manque ; repro L3c sur nœuds vierges : rc=0 (F-54). | **L3c (fait)** ; cible À VÉRIFIER |
| R-33 | `/etc/cluster/fence_xvm.key` distribuée malgré D19 (F-51). | Aucune variable ; la clé n'est plus affichée grâce à C4. | **accepté** (D22, 2026-11-05) |
| R-34 | `nmcli con up` (réactivation du profil) sur le nœud porteur retire la VIP jusqu'au moniteur suivant (≤ 10 s) et laisse un `fail-count` (F-61). | Aucune garde possible : consigne CLAUDE.md §17.5 et README. | ouvert (repro) ; cible À VÉRIFIER |
| R-35 | `--tags <x>` : le rôle n'a aucun tag ; `cluster.yml` termine en rc=0 après le seul preflight, sans déployer ni vérifier (repro L3c). | Consigne README : jamais `--tags`/`--skip-tags`. | ouvert (consigne) |

## 5. Points ouverts

### Levés le 2026-11-05

| ID | Point | Levé par |
|---|---|---|
| P-01 | Motif de D2. | D2 (motif) |
| P-02 | Acceptation de `wait_for_all`. | D8 |
| P-03 | Lien corosync dédié ou unique. | D9, C-04 |
| P-04 | Résolution des noms d'inventaire. | D10, D20, C-05, C-06 |
| P-07 | Fuite du mot de passe côté contrôleur. | D12 révisée, C1-C4 |
| P-11 | `python3-netaddr` sur le contrôleur. | C-01 (G3 OK) |
| P-12 | Contenu du miroir. | C-02 (N6 OK) |
| P-15 | Accès aux nœuds. | C-02 (root, `become` OK) |
| P-16 | NetworkManager et `arping` sur Hyper-V. | C-02, C-03 (N2, N5 OK) |
| P-19 | Contournement de R-32. | D21 (validé en repro L3c, F-54) |
| P-21 | Distribution de `fence_xvm.key`. | D22 |

### Bloquants pour le premier déploiement réel

| ID | Point |
|---|---|
| P-05 | Valeur de `pacemaker_cluster_name` dans l'inventaire réel (hors dépôt). |
| P-18 | Journal du harnais : en 755/644 (C-08), il fait échouer C3 (non encore exercée sur cible, C-09). Le placer dans un répertoire 0700 (journal 0600), ou retirer `ANSIBLE_LOG_PATH`. |

### Non bloquants pour l'écriture, mais à lever avant la mise en service

| ID | Point |
|---|---|
| P-06 | Les redémarrages complets du cluster lors des changements de configuration sont-ils acceptables (F-08) ? Faut-il des fenêtres de maintenance ? |
| P-08 | Filtrage ARP/MAC du vSwitch Hyper-V lors de la **bascule** de la VIP (ARP gratuit d'IPaddr2) : à constater après le L3c. La détection N5 fonctionne (C-03). |
| P-09 | Compatibilité réelle du rôle 1.88.9 avec pcs 0.11.11 et pacemaker 2.1.10 : `cluster.yml` complet validé en repro conteneur L3c sur nœuds vierges (F-54, F-57 à F-59), **jamais sur cible**. |
| P-10 | Variables `ANSIBLE_*` du contrôleur : désormais refusées par G1 hors liste blanche (D17) ; à constater avec le harnais réel. |
| P-13 | Faut-il contrôler les sha256 des tarballs de collections (§2) ? G4 ne contrôle que les versions. |
| P-14 | Licence du dépôt : `meta/main.yml` porte `license: "À VÉRIFIER"`. |
| P-17 | Décisions L2 à valider : VIP en /31 ou /32 refusée (G8) ; `pacemaker_vip_nic` identique sur les 2 nœuds (un seul paramètre `nic` IPaddr2) ; N6 exige les paquets dans les dépôts activés même s'ils sont déjà installés. |
| P-20 | C3 ne couvre pas les ACL POSIX, une sortie redirigée (`tee`, `script`, ordonnanceur) ni l'historique du terminal. |
| P-22 | Changement limité à la CIB (paramètres de `vip`, stickiness) : la VIP est-elle redémarrée ? Non testé (README). |
| P-23 | R-34 sur la VM Hyper-V : effet de `nmcli con up`/`nmcli device reapply` sur la VIP ; bascule réelle de la VIP et ARP gratuit (P-08). |
| P-24 | D11 en repro = arrêt propre (`podman stop`). Une coupure brutale du porteur (VM éteinte) n'a pas été testée ; sans fencing, le nœud restant reprend la VIP (stonith désactivé), à constater. |

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
env | grep -E '^(ANSIBLE_|NO_COLOR)' | cut -d= -f1                      # L3b : seulement la liste blanche D17
python3 -m ansible.cli.config dump --only-changed -t all -f json | grep '"origin": "env'   # G1 (F-47)
ls -ld "$(dirname "$ANSIBLE_LOG_PATH")" "$ANSIBLE_LOG_PATH"             # C3 : 0700 et 0600 (P-18)
ssh -G <nœud> | grep -iE '^(stricthostkeychecking|userknownhostsfile)' # true + fichier réel (F-48)
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
uname -n; ip -o -4 addr show dev eth0       # = nom d'inventaire ; IPv4 principale = pacemaker_node_ip (N7)
rpm -q --qf '[%{FILENAMES} %{FILEFLAGS:fflags}\n]' pcs | grep '/var/lib/pcsd '; ls -ld /var/lib/pcsd   # R-32 (F-50)
dnf --quiet repoquery --available --queryformat '%{name}' pcs corosync-qnetd openssl corosync libknet1-plugins-all resource-agents pacemaker fence-agents-all fence-virt
```

### Premier déploiement (contrôleur, racine du dépôt, après un preflight vert)

```
ansible-playbook -i <inventaire hors dépôt> cluster.yml --ask-vault-pass   # sans -v, --check, --diff, --limit, --tags, -e ha_cluster*
```

Attendu : D21 `changed` (pcs installé, pcsd démarré) puis `/var/lib/pcsd` 0700 root:root ; V1-V5 `ok` ; `failed=0`. Second passage identique : « Stop cluster daemons to reload configuration » sauté.

### Après un premier déploiement (sur un nœud)

```
corosync-quorumtool -s                       # rc 0 ; « Flags: 2Node Quorate », sans WaitForAll (D8, F-40)
corosync-cmapctl -b nodelist.node | grep addr  # ring0_addr = pacemaker_node_ip (D10)
crm_mon --output-as=xml                      # resource id="vip" role="Started" nodes_running_on="1"
pcs property config --all | grep stonith     # stonith-enabled = false (D2)
pcs status; corosync-cfgtool -n              # VIP Started ; port knet effectif (F-16)
journalctl --since today | grep -c 'Invoked with'   # attendu 0 avec no_target_syslog (F-23, journald)
sha256sum /etc/corosync/corosync.conf; cibadmin --query --scope configuration | sha256sum   # avant/après un second passage : identiques (F-57)
crm_mon -1 --output-as=xml --include=operations | grep -A3 'resource_history id="vip"'   # call et last-rc-change du start inchangés
stat -c '%A %U:%G' /var/lib/pcsd             # drwx------ root:root (D21)
```

## 7. Journal des séries

| Série | Date | Livrable | Résultat |
|---|---|---|---|
| L0 | 2026-11-05 | Résolution en lecture seule : faits Q1–Q9, sans code. | Faits F-01 à F-19, F-24, F-25. Repro locale (ipaddr, dépendances Galaxy, chargement des vars OL9). |
| L1 | 2026-11-05 | Socle : CLAUDE.md, ce fichier, `collections/requirements.yml`, `ansible.cfg`, `.gitignore`, `inventories/example/`, README. | Repro locale : installation des 5 versions exactes ; mismatch → ERROR (et warning hors cfg) ; piège du cfg démontré ; `no_target_syslog` démontré (rsyslog) ; inventaire et group_vars chargés ; yamllint rc=0 ; ansible-lint rc=0 (profil production, ansible-lint 26.11.0 / ansible-core 2.21.6, **pas** 2.14). Faits F-20 à F-23 ajoutés. |
| L2 | 2026-11-05 | Preflight : `preflight.yml` et rôle `pacemaker_preflight` (gardes G1-G10, N1-N6), section README, CLAUDE.md §4, §6 (corrigé) et §14. | Repro locale et repro conteneur, sur ansible-core 2.14.18 : `--syntax-check` rc=0 ; chaque garde vérifiée en nominal et en échec forcé (77 cas, journaux hors dépôt) ; secret absent des sorties `-vvv` (0 occurrence) ; `inventories/example` échoue en G7. yamllint rc=0 ; ansible-lint rc=0 (profil production, ansible-lint 26.11.0 / ansible-core 2.21.6, **pas** 2.14). Faits F-26 à F-34 ajoutés, F-17 corrigé. |
| L3a | 2026-11-05 | Résolution en lecture seule avant le playbook cluster (R1-R10), sans commit. | Repro locale (pcs `--corosync_conf`, `pcs -f`, `crm_simulate`, verbosité/journal/précédence/interpréteur) et repro conteneur (jetons pcsd). Faits F-35 à F-46, risques R-22 à R-31. |
| L3b | 2026-11-05 | Preflight étendu : G1 généralisée (origine env, clés SSH), C1-C4, G11, G12, N7, N6 sans fencing, interpréteur du play 2 ; inventaire d'exemple ; README ; CLAUDE.md §4, §11, §15, §16. | Repro locale et repro conteneur sur ansible-core 2.14.18 : `--syntax-check` rc=0 (nominal et `inventories/example`) ; nouvelles gardes : 75 cas (nominal et échecs forcés, messages exacts) ; non-régression : 75 cas L2 rejoués, 70 identiques, 5 écarts attendus (3 cas `-vvv` arrêtés par C1, 2 cas N3/N4 arrêtés plus tôt par G11) ; yamllint rc=0 ; ansible-lint rc=0 (profil production, ansible-lint 26.11.0 / ansible-core 2.21.6, **pas** 2.14). Rôle ha_cluster 1.88.9 exécuté réellement en repro conteneur (F-50) ; preuve D18 (F-52) : `--diff` interdit. Constats sur cible C-01 à C-08 consignés. |
| L3c | 2026-11-05 | Playbook cluster : `cluster.yml` (preflight importé, C5, D21, rôle ha_cluster 1.88.9, V1-V5), `tasks/declared_vars.yml`, `tasks/pcsd_state_dir.yml`, `tasks/verify.yml` ; README (Playbook cluster, Exploitation) ; CLAUDE.md §9 (sans `--check --diff`) et §17. Constats cible C-09 à C-12, décisions D21, D22. | Repro conteneur L3c, ansible-core 2.14.18, nœuds vierges : `--syntax-check` rc=0 (test et `inventories/example`) ; premier déploiement rc=0 en 127 s, V1-V5 OK, 0 avertissement ; second passage : état identique (F-57) ; V1-V5 : 8 échecs forcés (dont V1 en borne réelle : 75 s) ; reconvergence de `wait_for_all` ; C5 : 9 échecs forcés + 1 nominal complet ; D21 nœud vierge et répertoire présent ; D11 et D8 conformes ; non-régression : 20 cas (preflight et `cluster.yml`) ; secret hacluster : 0 occurrence dans 43 journaux. yamllint rc=0 ; ansible-lint rc=0 (profil production, 13 fichiers, ansible-lint 26.11.0 / ansible-core 2.21.6, **pas** 2.14). Faits F-53 à F-62, risques R-34, R-35. |
