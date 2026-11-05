# CLAUDE.md — règles persistantes du projet pacemaker_vip

Objet : un playbook Ansible qui crée un cluster Pacemaker à 2 nœuds Oracle Linux 9
(VMs Hyper-V) portant une ressource VIP unique, via le rôle `ha_cluster` de
`fedora.linux_system_roles`.

Les faits, sources, décisions, risques et points ouverts sont dans
[`docs/pacemaker-facts.md`](docs/pacemaker-facts.md). Les références ci-dessous
(« L0 Qn », « F-xx ») renvoient à ce fichier.

Abréviations de sources :
- **LSR** = `fedora.linux_system_roles` 1.88.9, `roles/ha_cluster/` ;
- **AC** = ansible-core 2.14.18, `lib/ansible/`.

---

## 1. Sourçage

- **Règle** : toute affirmation technique (nom de variable, comportement, valeur par
  défaut, paquet, dépôt, port, version, contrainte) est lue dans un artefact : fichier
  de code (chemin:ligne + version), sortie de commande, ou URL précise. Sinon elle
  porte le marqueur `# À VÉRIFIER : <où le confirmer>`. Aucune valeur plausible non
  sourcée, aucun chiffre ni nom de dépôt posé de mémoire. Le code livré fait foi sur
  sa doc.
- **Raison** : le L0 a montré plusieurs écarts entre comportement attendu et
  comportement réel, tous silencieux (repli de variables sur OL9, purge du CIB,
  dépendances non bornées). Voir F-04, F-06, F-08.

## 2. Frontière développement / cible

- **Règle** : le contrôleur et les nœuds ne sont pas joignables depuis la machine de
  développement. Rien n'est simulé, aucune sortie n'est fabriquée. Une reproduction
  en conteneur est étiquetée « repro locale » et n'est jamais présentée comme le
  contrôleur ou un nœud. Ce qui ne se valide que sur cible est écrit comme tel, avec
  la commande exacte et l'endroit où la lancer (§ « Commandes » du fichier de faits).
- **Raison** : décision D7.

## 3. Collections : versions exactes, chemin du projet, incompatibilité fatale

- **Règle 3.1** : toutes les collections sont épinglées en `==` dans
  `collections/requirements.yml`, y compris les dépendances déclarées par
  `fedora.linux_system_roles` (ansible.posix, community.general, containers.podman).
  - **Raison** : `MANIFEST.json` de 1.88.9 déclare ces dépendances en `"*"`.
    `ansible-galaxy` ne lit pas `requires_ansible` (aucune occurrence dans
    `AC/galaxy/`). En repro L0, il a installé community.general 14.0.0 (`>=2.19.0`)
    et ansible.posix 2.2.2 (`>=2.16.0`) sans avertissement. Voir F-03.
- **Règle 3.2** : `collections_on_ansible_version_mismatch = error` dans `ansible.cfg`.
  - **Raison** : le défaut est `warning` (`AC/config/base.yml:223-233`). Le contrôle
    n'a lieu qu'au chargement d'un plugin (`AC/plugins/loader.py:1411-1417`).
- **Règle 3.3** : collections installées et lues uniquement depuis `./collections`
  (`collections_path`), avec `collections_scan_sys_path = False`.
  - **Raison** : sur le contrôleur, le chemin par défaut inclut
    `/root/.ansible/collections` et `/usr/share/ansible/collections`
    (`AC/config/base.yml:211`), où d'autres versions masqueraient les nôtres. De plus,
    `sys.path` est ajouté après nos chemins si le balayage est actif
    (`AC/utils/collection_loader/_collection_finder.py:94-98`).
- **Règle 3.4** : on ne monte pas de version d'ansible-core, et pas de venv (D1). La
  dernière version de `fedora.linux_system_roles` compatible 2.14 est 1.88.9 (F-01).

## 4. ansible.cfg n'est pas toujours lu

- **Règle** : lancer `ansible*` **depuis la racine du dépôt**, sans `ANSIBLE_CONFIG`,
  dans un répertoire qui n'est pas accessible en écriture à tous. Vérifier par
  `ansible-config dump --only-changed` (ligne `CONFIG_FILE()`). Garde automatique :
  G1 du preflight, qui vérifie `ansible_config_file` et les réglages effectifs.
- **Raison** : `AC/config/manager.py:212-248`. L'ordre de recherche est
  `ANSIBLE_CONFIG`, puis `./ansible.cfg` (ignoré si le CWD est world-writable,
  l.226-232), puis `~/.ansible.cfg`, puis `/etc/ansible/ansible.cfg`. Le premier
  fichier trouvé est utilisé, sans fusion. Hors racine, la configuration du
  contrôleur s'applique en silence (démontré en repro L1, F-20).
- Les variables `ANSIBLE_*` de l'environnement priment sur le fichier.

## 5. Inventaire explicite

- **Règle** : l'inventaire est toujours passé avec `-i inventories/<env>`.
  `ansible.cfg` fixe `inventory = /dev/null`.
- **Raison** : sans réglage, c'est `/etc/ansible/hosts` qui est utilisé
  (`AC/config/base.yml:729-740`). Une valeur vide est résolue en chemin relatif au
  `.cfg` : la racine du dépôt devient l'inventaire (repro L1, F-21).

## 6. Le play cible exactement les 2 nœuds — `--limit` interdit

- **Règle** : le play porte sur le groupe `pacemaker_nodes`, qui contient exactement
  2 hôtes. Il ne s'exécute jamais avec `--limit`, ni avec un nœud injoignable.
  Garde G6 du preflight : `ansible_limit` non défini, et `ansible_play_hosts_all`
  égal à `groups['pacemaker_nodes']` ; plus `any_errors_fatal`.
- **Piège** : comparer `ansible_play_hosts` à `ansible_play_hosts_all` ne détecte
  pas `--limit`, car `ansible_play_hosts_all` est lui aussi réduit
  (`AC/plugins/strategy/__init__.py:274`, `AC/inventory/manager.py:388-389,404-407`,
  repro L2, F-26).
- **Raison** : le rôle construit corosync.conf à partir de `ansible_play_hosts`
  (`LSR/tasks/shell_pcs/check-and-prepare-role-variables.yml:9`,
  `LSR/tasks/shell_pcs/pcs-cluster-setup-pcs-0.10.yml:14-15`). Avec un seul hôte dans
  le play, il génère un corosync.conf à 1 nœud, le distribue, puis redémarre le
  cluster (L0 Q2, F-06).

## 7. Variables du rôle toujours déclarées explicitement

- **Règle** : `ha_cluster_enable_repos: false`, `ha_cluster_manage_firewall: false` et
  `ha_cluster_manage_selinux: false` sont toujours déclarées, jamais laissées au
  défaut.
- **Raisons** :
  - `enable_repos` vaut `true` par défaut (`LSR/defaults/main.yml:7`). Sur OL9, le
    rôle charge `vars/RedHat.yml`, où `__ha_cluster_repos: []` (l.11) : l'activation
    est un saut silencieux (L0 Q1/Q3, F-04).
  - `manage_firewall: true` installe, démasque, active et démarre firewalld
    (`fedora.linux_system_roles` 1.88.9 `roles/firewall/tasks/firewalld.yml:31-36`,
    `roles/firewall/tasks/main.yml:22-32`), ce qui contredit D4 (L0 Q8, F-15).
  - `manage_selinux: true` est sauté en silence si firewalld ne tourne pas
    (`LSR/tasks/selinux.yml:6-10`).
- Le nom de cluster est toujours fourni : le défaut du rôle est `my-cluster`
  (`LSR/defaults/main.yml:28`).

## 8. Le CIB est déclaratif : tout ce qui doit exister est déclaré

- **Règle** : chaque ressource, contrainte et propriété du cluster est déclarée dans
  les variables du rôle, y compris **`stonith-enabled: false`** (D2).
- **Raison** : à chaque exécution, le rôle purge du CIB les ressources, les
  contraintes et toutes les propriétés `crm_config`, sauf une liste blanche
  (`LSR/tasks/shell_pcs/create-and-push-cib.yml:51-70`). Il reconstruit ensuite à
  partir des variables, puis applique un patch (l.257-288). Il ne pose jamais
  `stonith-enabled` lui-même. Le défaut de Pacemaker est `true`, et alors aucune
  ressource ne démarre sans équipement de fencing (Pacemaker Explained 2.1,
  cluster-options). Un réglage manuel est effacé au passage suivant (L0 Q2/Q5,
  F-07, F-11).

## 9. Modifications de corosync.conf, des clés ou du nom de cluster

- **Règle** : toute modification qui touche corosync.conf (nom de cluster, transport,
  totem, quorum, liste des nœuds) ou les clés (`ha_cluster_regenerate_keys`) se fait
  en fenêtre de maintenance, après un passage `--check --diff`.
- **Raison** : si corosync.conf ou une authkey change, le rôle arrête `pacemaker` et
  `corosync` sur **tous** les nœuds à la fois, puis les redémarre
  (`LSR/tasks/shell_pcs/cluster-start-and-reload.yml:24-42`). La VIP est coupée
  (L0 Q2, F-08).

## 10. L'idempotence se teste sur l'état, pas sur le compteur `changed`

- **Règle** : un second passage est jugé sur l'état (contenu de corosync.conf, CIB,
  ressources démarrées), pas sur `changed=0`.
- **Raison** : le rôle signale `changed` à chaque passage. pcsd est arrêté puis
  redémarré systématiquement (`LSR/tasks/shell_pcs/configure-shell.yml:6-9`,
  `:88-92`), et de nombreuses tâches ont `changed_when: not ansible_check_mode`
  (L0 Q2, F-09).

## 11. Mot de passe hacluster : Vault uniquement

- **Règle** : la valeur vient uniquement d'Ansible Vault (D5). Aucun mot de passe en
  clair dans le dépôt, aucun fichier de mot de passe Vault versionné (`.gitignore`).
- **Raison et risque** : le rôle passe le mot de passe en `stdin` de
  `pcs host auth` sans `no_log` (`LSR/tasks/shell_pcs/cluster-auth-pcs-0.10.yml:15`).
  - Côté cible, `no_target_syslog = True` supprime la ligne « Invoked with »
    (`AC/plugins/action/__init__.py:969-970`, `AC/module_utils/basic.py:526-527`).
  - Côté contrôleur, le paramètre reste visible en `-vvv`
    (`AC/plugins/callback/__init__.py:234-235`).
  - Les autres contre-mesures restent à établir en L3 (point ouvert P-07).

## 12. Hygiène git

- **Règle** : une branche par série (`l<N>-<sujet>`) et un commit par livrable.
  L'intégration dans `main` se fait par merge `--no-ff`, uniquement après validation
  de l'utilisateur. Avant chaque commit, contrôler `git diff --stat` (puis
  `git diff --cached --stat`) : un ajout de contenu ne doit jamais faire diminuer un
  fichier. **Jamais** de ligne `Co-Authored-By`, ni aucune attribution à un
  assistant, dans les commits, PR ou fichiers. Pas de push sans demande explicite.
- **Raison** : traçabilité livrable par livrable, et détection des écrasements
  accidentels lors des éditions.

## 13. Le fichier de faits se met à jour en dernier

- **Règle** : dans une série, `docs/pacemaker-facts.md` est mis à jour **en dernier**,
  une fois les validations faites. Il ne contient rien de ce que git sait déjà (état
  de branche, position par rapport à origin).
- **Raison** : le fichier de faits est le tableau de bord de ce qui est prouvé. Il
  ne doit refléter que des résultats obtenus.

## 14. Preflight en tête de tout playbook qui touche les nœuds

- **Règle 14.1** : tout playbook qui modifie les nœuds commence par
  `import_playbook: preflight.yml`. Le rôle `pacemaker_preflight` y est appelé avec
  `tags: always`.
  - **Raison** : les gardes contrôleur tournent sur `localhost`. Avec
    `inventory = /dev/null`, un play sur `pacemaker_nodes` sans `-i` ne trouve aucun
    hôte et se termine sans erreur (R-03). Quand le play 1 échoue, le play 2 ne
    s'exécute pas (repro L2, F-34). `include_tasks` avec `apply: tags: [always]`
    garde les gardes actives sous `--tags` et `--skip-tags` (repro L2, F-31).
- **Règle 14.2** : une garde ne teste jamais une variable de gabarit lue via
  `hostvars` avec `is defined` ou `default()`. Elle teste la variable source.
  - **Raison** : `hostvars` rend les valeurs avec `fail_on_undefined=False`
    (`AC/vars/hostvars.py:140`). Si la source manque, la valeur reste le texte brut
    `"{{ … }}"` et `is defined` vaut vrai (repro L2, F-27).
- **Règle 14.3** : une garde qui dépend d'un code retour vérifie aussi la sortie
  quand l'outil renvoie 0 sans avoir fait le test.
  - **Raison** : `arping -D` sort en 0 sur une interface sans ARP
    (iputils-20210202 `arping.c:562-570,1006-1009`, repro L2, F-29).
- **Règle 14.4** : chaque garde est démontrée dans les deux sens, nominal et échec
  forcé, avec le message obtenu. Les inventaires de test vivent hors du dépôt.
