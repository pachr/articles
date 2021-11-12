# Test de vie et Ansible : un exemple de réalisation pour mieux comprendre l'outil 

[Ansible est un outil fantastique](https://github.com/ansible/ansible), que l'éditeur (Red Hat) et sa communauté - présente comme "_radicalement simple_" ("_Ansible is a radically simple IT automation platform that makes your applications and systems easier to deploy and maintain._". C'est vrai par son approche descriptive d'un état désiré ("_as code_" : on décrit un état désiré et non des opérations à effectuer), par la généralisation des "livres de recettes" (_playbook_) et autres rôles qui sont réutilisables (presque) sans restriction. 

Mais ce n'est pas seulement un outil de déploiement : sa généricité va au-delà de la volonté de ne pas réinventer la roue pour une action d'installation. 

Les tests de vie - en résumé, savoir si un service est opérationnel ou pas -, rentre parfaitement dans ses cordes. Ce sont souvent les indicateurs avancés pour l'état de santé d'une infra : niveau de consommation des ressources (CPU, RAM, disques), si un processus est exécuté ou non, etc. 

Ce sont aussi des besoins différents suivant notre casquette, particulièrement en entreprise où les rôles sont bien séparés : 
  - comme responsable d'infra, nous allons privilégié le bon état des machines ; 
  - comme responsable de dév celle des processus ; 
  - comme responsable d'affaire celle des applications. 

La différence est parfois subtile mais c'est bien __cet ensemble__ qui compte. Imaginez par exemple que vous mettez en ligne un site Wordpress personnel : votre premier souhait est de savoir si le site est bien accessible. Vous êtes en qualité ici de "responsable d'affaire" : vos "clients" (visiteurs) peuvent-ils y accéder ? Peut-être que oui, peut-être que non, avec quelle performance pour les pages appelées... peu importe que vous soyez salarié, bénévole, amateur. La position critique est la meme. 

S'il n'est pas accessible ou difficilement (délai, erreur), pourquoi ? Un test sur une URL n'est pas suffisante : est-ce le Nginx frontal, est-ce PHP-FPM, est-ce la base de données, est-ce un parefeu ou tout autre chose... ? Est-ce votre développement ou votre exploitation qui pêche ? La machine-même, est-elle accessible ; est-elle suffisamment dimensionnée ? 

Si vous travaillez en direct sur cette dernière, vous allez parcourir chaque point : c'est long, laborieux, non-satisfaisant pour plus d'une ou deux machines. On voit alors l'intérêt d'Ansible pour automatiser ces tâches et les jouer avec régulièrement via AWX - Ansible Tower communautaire -, ou via une simple tâche CRON. 

C'est cet exercice mental que je vous propose, mais comme prétexte pour servir de support afin de comprendre / approfondir certains aspects d'Ansible sans se perdre. 

## Préparation son dossier de travail 

Il est de bon goût d'avoir un environnement virtuel Python pour travailler en démo / dév / test. Ansible est en Python, lors ne cherchons pas d'excuse: 

```bash
julien@julien-Vostro-7580:~/Developpement$ virtualenv ansible-test-vie 
created virtual environment CPython3.8.10.final.0-64 in 75ms
  creator CPython3Posix(dest=/home/julien/Developpement/ansible-test-vie , clear=False, global=False)
  seeder FromAppData(download=False, pip=bundle, setuptools=bundle, wheel=bundle, via=copy, app_data_dir=/home/julien/.local/share/virtualenv)
    added seed packages: pip==20.1.1, pkg_resources==0.0.0, setuptools==44.0.0, wheel==0.34.2
  activators BashActivator,CShellActivator,FishActivator,PowerShellActivator,PythonActivator,XonshActivator
julien@julien-Vostro-7580:~/Developpement$ cd ansible-test-vie 
julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ source ./bin/activate
(ansible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ python3 -m pip install ansible
Collecting ansible
  Downloading ansible-4.8.0.tar.gz (36.1 MB)
     |████████████████████████████████| 36.1 MB 459 kB/s 
Collecting ansible-core<2.12,>=2.11.6
  Downloading ansible-core-2.11.6.tar.gz (7.0 MB)
     |████████████████████████████████| 7.0 MB 657 kB/s 
Collecting PyYAML
  Downloading PyYAML-6.0-cp38-cp38-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_12_x86_64.manylinux2010_x86_64.whl (701 kB)
     |████████████████████████████████| 701 kB 597 kB/s 
Collecting cryptography
  Downloading cryptography-35.0.0-cp36-abi3-manylinux_2_17_x86_64.manylinux2014_x86_64.whl (3.7 MB)
     |████████████████████████████████| 3.7 MB 775 kB/s 
Collecting jinja2
  Downloading Jinja2-3.0.3-py3-none-any.whl (133 kB)
     |████████████████████████████████| 133 kB 800 kB/s 
Collecting packaging
  Downloading packaging-21.2-py3-none-any.whl (40 kB)
     |████████████████████████████████| 40 kB 863 kB/s 
Collecting resolvelib<0.6.0,>=0.5.3
  Downloading resolvelib-0.5.4-py2.py3-none-any.whl (12 kB)
Collecting cffi>=1.12
  Downloading cffi-1.15.0-cp38-cp38-manylinux_2_12_x86_64.manylinux2010_x86_64.whl (446 kB)
     |████████████████████████████████| 446 kB 330 kB/s 
Collecting MarkupSafe>=2.0
  Using cached MarkupSafe-2.0.1-cp38-cp38-manylinux2010_x86_64.whl (30 kB)
Collecting pyparsing<3,>=2.0.2
  Using cached pyparsing-2.4.7-py2.py3-none-any.whl (67 kB)
Collecting pycparser
  Downloading pycparser-2.21-py2.py3-none-any.whl (118 kB)
     |████████████████████████████████| 118 kB 677 kB/s 
Building wheels for collected packages: ansible, ansible-core
  Building wheel for ansible (setup.py) ... done
  Created wheel for ansible: filename=ansible-4.8.0-py3-none-any.whl size=59454442 sha256=ac2be274d30521793513160041ae9cf88d06283060672d85dc62071fffb97e8f
  Stored in directory: /home/julien/.cache/pip/wheels/3d/db/ed/6c8b0ffe3008371db04f2098374dfa21b22823bd7bf30fe5b1
  Building wheel for ansible-core (setup.py) ... done
  Created wheel for ansible-core: filename=ansible_core-2.11.6-py3-none-any.whl size=1959295 sha256=0ee112235f0eb31fcf398b43042c6d6c46abe65835d0d552b96857dbe1326786
  Stored in directory: /home/julien/.cache/pip/wheels/11/e3/41/39946a5f418fe614f955a9ba4de0edee55a209f46a53089846
Successfully built ansible ansible-core
Installing collected packages: PyYAML, pycparser, cffi, cryptography, MarkupSafe, jinja2, pyparsing, packaging, resolvelib, ansible-core, ansible
Successfully installed MarkupSafe-2.0.1 PyYAML-6.0 ansible-4.8.0 ansible-core-2.11.6 cffi-1.15.0 cryptography-35.0.0 jinja2-3.0.3 packaging-21.2 pycparser-2.21 pyparsing-2.4.7 resolvelib-0.5.4
(ansible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ subl . 
(ansible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ 
```

Notre poste et notre éditeur de texte préférés sont prêts ! Vous pouvez créé avec VirtualBox, un Windows Server quelconque, qui sera notre machine cible. Vous comprendrez tout à l'heure pourquoi j'ai utilisé des machines Windows dans cette article.  

## "Gather facts" 

C'est ce qui frappe la première fois qu'on lance l'outil. Une ligne apparaît, non souhaitée, et de prime abord on tente à l'oublier : "_TASK [Gathering Facts]..._". C'est pourtant le coeur nucléaire du système. 

Le _recueil des faits_ en bon français, est une opération qui est exécutée par défaut lorsque vous lançez Ansible en mode _playbook_ - sauf à spécifier le contraire donc. C'est notre premier appui à l'exercice. Réalisation un livre de recettes rapide : 

```yaml lancement.yml
- name: Tester l'activité de mes applicatifs 
  gather_facts: yes 
  hosts: all 
```

Avec un inventaire tout aussi rapide : 

```yaml inventaire.yml
all:
  hosts: 
    WIN-AD: 
      ansible_host: 192.168.56.3 
  vars: 
    ansible_user: "Administrateur"
    ansible_password: "Azerty01"
    ansible_connection: winrm
    ansible_port: 5985
    ansible_winrm_transport: ntlm
```

Lancez-le : 

```bash
(anible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ ansible-playbook -i inventaire.yml ./lancement.yml 

PLAY [Tester l'activité de mes applicatifs] ************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************************************************************************************
ok: [WIN-AD]

PLAY RECAP *********************************************************************************************************************************************************************************************************************************
WIN-AD                     : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Si l'on modifie maintenant le reccueil à "non"... 

```yaml lancement.yml
- name: Tester l'activité de mes applicatifs 
  gather_facts: no 
  hosts: all 
```

... , cette ligne habituelle disparaît (et le livre de recette va considérablement plus vite !) : 

```bash
(anible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ ansible-playbook -i inventaire.yml ./lancement.yml 

PLAY [Tester l'activité de mes applicatifs] ************************************************************************************************************************************************************************************************

PLAY RECAP *********************************************************************************************************************************************************************************************************************************
```

Mais que fait-elle, cette tâche ? Et bien exactement ce que son nom dit : elle recueil des faits sur les machines cibles, des faits stockés dans une variables Ansible liées à l'hôte. Tout simplement, Ansible prend en charge directement ce qui est courant et essentiel de savoir sur celle-ci. Suivant l'OS, ces données peuvent varier en nature et volume (pour Windows, aucune notion de montage par exemple, mais plus d'information sur la jonction à un domaine). 

Une étape de débogue vous permettra d'y voir clair : 

```yaml lancement.yml
- name: Tester l'activité de mes applicatifs 
  gather_facts: yes 
  hosts: all 
  tasks: 
    - debug: msg="{{ ansible_facts }}" 
```

Voici le résultat (j'ai gardé toutes les lignes pour le côté exhaustif, mais on se fiche des valeurs ici) : 

```bash
(anible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ ansible-playbook -i inventaire.yml ./lancement.yml 

PLAY [Tester l'activité de mes applicatifs] ************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************************************************************************************
ok: [WIN-AD]

TASK [debug] *******************************************************************************************************************************************************************************************************************************
ok: [WIN-AD] => {
    "msg": {
        "architecture": "64 bits",
        "bios_date": "12/01/2006",
        "bios_version": "VirtualBox",
        "date_time": {
            "date": "2021-11-10",
            "day": "10",
            "epoch": "1636532337,09129",
            "hour": "08",
            "iso8601": "2021-11-10T07:18:57Z",
            "iso8601_basic": "20211110T081857091287",
            "iso8601_basic_short": "20211110T081857",
            "iso8601_micro": "2021-11-10T07:18:57.091287Z",
            "minute": "18",
            "month": "11",
            "second": "57",
            "time": "08:18:57",
            "tz": "Romance Standard Time",
            "tz_offset": "+01:00",
            "weekday": "Wednesday",
            "weekday_number": "3",
            "weeknumber": "44",
            "year": "2021"
        },
        "distribution": "Microsoft Windows Server 2016 Standard Evaluation",
        "distribution_major_version": "10",
        "distribution_version": "10.0.14393.0",
        "domain": "",
        "env": {
            "ALLUSERSPROFILE": "C:\\ProgramData",
            "APPDATA": "C:\\Users\\Administrateur\\AppData\\Roaming",
            "COMPUTERNAME": "WIN-EMCILDHJ6MJ",
            "ComSpec": "C:\\Windows\\system32\\cmd.exe",
            "CommonProgramFiles": "C:\\Program Files\\Common Files",
            "CommonProgramFiles(x86)": "C:\\Program Files (x86)\\Common Files",
            "CommonProgramW6432": "C:\\Program Files\\Common Files",
            "HOMEDRIVE": "C:",
            "HOMEPATH": "\\Users\\Administrateur",
            "LOCALAPPDATA": "C:\\Users\\Administrateur\\AppData\\Local",
            "LOGONSERVER": "\\\\WIN-EMCILDHJ6MJ",
            "NUMBER_OF_PROCESSORS": "3",
            "OS": "Windows_NT",
            "PATHEXT": ".COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH;.MSC;.CPL",
            "PROCESSOR_ARCHITECTURE": "AMD64",
            "PROCESSOR_IDENTIFIER": "Intel64 Family 6 Model 158 Stepping 10, GenuineIntel",
            "PROCESSOR_LEVEL": "6",
            "PROCESSOR_REVISION": "9e0a",
            "PROMPT": "$P$G",
            "PSExecutionPolicyPreference": "Unrestricted",
            "PSModulePath": "C:\\Users\\Administrateur\\Documents\\WindowsPowerShell\\Modules;C:\\Program Files\\WindowsPowerShell\\Modules;C:\\Windows\\system32\\WindowsPowerShell\\v1.0\\Modules",
            "PUBLIC": "C:\\Users\\Public",
            "Path": "C:\\Windows\\system32;C:\\Windows;C:\\Windows\\System32\\Wbem;C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\;C:\\Users\\Administrateur\\AppData\\Local\\Microsoft\\WindowsApps",
            "ProgramData": "C:\\ProgramData",
            "ProgramFiles": "C:\\Program Files",
            "ProgramFiles(x86)": "C:\\Program Files (x86)",
            "ProgramW6432": "C:\\Program Files",
            "SystemDrive": "C:",
            "SystemRoot": "C:\\Windows",
            "TEMP": "C:\\Users\\ADMINI~1\\AppData\\Local\\Temp",
            "TMP": "C:\\Users\\ADMINI~1\\AppData\\Local\\Temp",
            "USERDOMAIN": "WIN-EMCILDHJ6MJ",
            "USERDOMAIN_ROAMINGPROFILE": "WIN-EMCILDHJ6MJ",
            "USERNAME": "Administrateur",
            "USERPROFILE": "C:\\Users\\Administrateur",
            "windir": "C:\\Windows"
        },
        "fqdn": "WIN-EMCILDHJ6MJ",
        "gather_subset": [
            "all"
        ],
        "hostname": "WIN-EMCILDHJ6MJ",
        "interfaces": [
            {
                "connection_name": "Ethernet",
                "default_gateway": null,
                "dns_domain": null,
                "interface_index": 2,
                "interface_name": "Intel(R) PRO/1000 MT Desktop Adapter",
                "macaddress": "08:00:27:18:B8:1C"
            }
        ],
        "ip_addresses": [
            "192.168.56.3",
            "fe80::c55c:4c4a:f16b:63b"
        ],
        "kernel": "10.0.14393.0",
        "lastboot": "2021-11-10 00:42:18Z",
        "machine_id": "S-1-5-21-160161560-88724368-22562144",
        "memtotal_mb": 4096,
        "module_setup": true,
        "netbios_name": "WIN-EMCILDHJ6MJ",
        "nodename": "WIN-EMCILDHJ6MJ",
        "os_family": "Windows",
        "os_name": "Microsoft Windows Server 2016 Standard Evaluation",
        "os_product_type": "server",
        "owner_contact": "",
        "owner_name": "Utilisateur Windows",
        "powershell_version": 5,
        "processor": [
            "GenuineIntel",
            "Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz",
            "GenuineIntel",
            "Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz",
            "GenuineIntel",
            "Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz"
        ],
        "processor_cores": 3,
        "processor_count": 1,
        "processor_threads_per_core": 1,
        "processor_vcpus": 3,
        "product_name": "VirtualBox",
        "product_serial": "0",
        "reboot_pending": null,
        "swaptotal_mb": 0,
        "system": "Win32NT",
        "system_description": "",
        "system_vendor": "innotek GmbH",
        "uptime_seconds": 27400,
        "user_dir": "C:\\Users\\Administrateur",
        "user_gecos": "",
        "user_id": "Administrateur",
        "user_sid": "S-1-5-21-160161560-88724368-22562144-500",
        "virtualization_role": "guest",
        "virtualization_type": "VirtualBox",
        "windows_domain": "WORKGROUP",
        "windows_domain_member": false,
        "windows_domain_role": "Stand-alone server"
    }
}

PLAY RECAP *********************************************************************************************************************************************************************************************************************************
WIN-AD                     : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```

Avec cette ligne sur votre console, même sans le débogue, vous savez déjà plusieurs choses : 
  - votre machine est accessible par le réseau déclaré (elle est donc allumée et opérationnelle sur l'accès distant via l'IP déclarée) ; 
  - votre inventaire est donc correct, car on a pu s'authentifier ; 
  - vous pouvez démarrer un shell, vous avez donc un minimum de droits pour agir... 

Car évidemment ces faits ne viennent pas de WinRM ou SSH : ce sont des scripts qui sont joués localement et dont les résultats "remontent" à la machine Ansible. C'est l'intérêt d'avoir ici du Windows : Ansible ne gère pas directement du Python pour l'exécution locale sur la machine cible, mais du Powershell. On comprend bien qu'il y a autre chose que simplement "une connexion" qui est ouverte. 

Pour Linux, un interpréteur Python est cherché - car il est généralement disponible en standard sur cet OS (au moins feu Python 2). 

Ainsi même un simple "ping", même sans passer par un livre de recettes, suit cette logique. Exemple toujours avec Windows : 
  - la machine est contactée, via le port cible de connexion à distance (WinRM dans notre cas) ; 
  - Ansible s'authentifie (grâce à NTLM dans notre cas) ; 
  - Ansible envoie un emballage ("_wrapper_") pour les modules des tâches en Powershell, puis les différents modules Powershell eux-mêmes, ceux qui sont appelés via des tâches ou en direct dans notre cas ; 
  - les scripts sont exécutés localement ; 
  - le résultat est récupéré, sérialisé en JSON et renvoyé à Ansible, qui le traite à son tour. 

C'est cela, un fonctionnement "sans agent" local : il n'est pas installé, mais il y a bien une exécution local au travers d'un ensemble coordonné. L'absence d'installation d'agent, ne veut pas dire pour autant anarchie ou une bête commmande. 

## Qu'est-ce qui est effectivement joué ? 

Pour bien comprendre que cette distinction entre Shells, modules et connexion est loin d'être mineure, si je tente [un "ping" avec le module par défaut](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ping_module.html) vers mon inventaire Windows, cela échoue : 

```bash
(ansible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ ansible -i inventaire.yml -m ping all
[WARNING]: No python interpreters found for host WIN-AD (tried ['/usr/bin/python', 'python3.7', 'python3.6', 'python3.5', 'python2.7', 'python2.6',
'/usr/libexec/platform-python', '/usr/bin/python3', 'python'])
WIN-AD | FAILED! => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "module_stderr": "Exception lors de l'appel de « Create » avec « 1 » argument(s) : « Au caractère Ligne:4 : 21\r\n+ def _ansiballz_main():\r\n+                     ~\r\nUne expression est attendue après « ( ».\r\nAu caractère Ligne:13 : 27\r\n+     except (AttributeError, OSError):\r\n+                           ~\r\nArgument manquant dans la liste de paramètres.\r\nAu caractère Ligne:15 : 7\r\n+     if scriptdir is not None:\r\n+       ~\r\nParenthèse ouvrante « ( » manquante après « if » dans l’instruction If.\r\nAu caractère Ligne:22 : 7\r\n+     if sys.version_info < (3,):\r\n+       ~\r\nParenthèse ouvrante « ( » manquante après « if » dans l’instruction If.\r\nAu caractère Ligne:22 : 30\r\n+     if sys.version_info < (3,):\r\n+                              ~\r\nExpression manquante après « , ».\r\nAu caractère Ligne:22 : 25\r\n+     if sys.version_info < (3,):\r\n+                         ~\r\nL’opérateur « < » est réservé à une utilisation future.\r\nAu caractère Ligne:27 : 34\r\n+     def invoke_module(modlib_path, temp_path, json_params):\r\n+                                  ~\r\nArgument manquant dans la liste de paramètres.\r\nAu caractère Ligne:28 : 40\r\n+         z = zipfile.ZipFile(modlib_path, mode='a')\r\n+                                        ~\r\nArgument manquant dans la liste de paramètres.\r\nAu caractère Ligne:31 : 33\r\n+         zinfo = zipfile.ZipInfo()\r\n+                                 ~\r\nUne expression est attendue après « ( ».\r\nAu caractère Ligne:34 : 25\r\n+         z.writestr(zinfo, sitecustomize)\r\n+                         ~\r\nArgument manquant dans la liste de paramètres.\r\nLes erreurs d’analyse n’ont pas toutes été signalées. Corrigez les erreurs signalées, puis recommencez. »\r\nAu caractère Ligne:6 : 1\r\n+ $exec_wrapper = [ScriptBlock]::Create($split_parts[0])\r\n+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~\r\n    + CategoryInfo          : NotSpecified: (:) [], MethodInvocationException\r\n    + FullyQualifiedErrorId : ParseException\r\n \r\nL’expression située après «&» dans un élément de pipeline a généré un objet non valide. Elle doit générer un nom de \r\ncommande, un bloc de script ou un objet CommandInfo.\r\nAu caractère Ligne:7 : 2\r\n+ &$exec_wrapper\r\n+  ~~~~~~~~~~~~~\r\n    + CategoryInfo          : InvalidOperation : (:) [], RuntimeException\r\n    + FullyQualifiedErrorId : BadExpression\r\n ",
    "module_stdout": "",
    "msg": "MODULE FAILURE\nSee stdout/stderr for the exact error",
    "rc": 1
}
```

Si par contre je passe par [le module "_win_ping_", celui dédié à l'OS de Microsoft](https://docs.ansible.com/ansible/latest/collections/ansible/windows/win_ping_module.html), plus de problème... et plus d'alerte "_No python interpreters found for host WIN-AD_". 

Ce module est d'ailleurs très facile de reproduire localement, ce que nous allons faire, pour bien comprendre ce qui se passe. 

Si vous êtes attentif sur le dépôt Git du projet, vous aurez noté que le module _win_ping_ a deux fichiers : 
 - un [fichier Python](https://github.com/ansible-collections/ansible.windows/blob/main/plugins/modules/win_ping.py) ; 
 - un [fichier Powershell](https://github.com/ansible-collections/ansible.windows/blob/main/plugins/modules/win_ping.ps1). 

Le fichier Python ___n'est pas___ jouée sur la machine cible dans notre cas : lorsqu'un interpréteur Python n'est pas trouvé mais dans le cas présent, il existe l'alternative d'un script pouvant être exécuté sur le shell local, c'est celui-ci qui est privilégié. Simple, pratique. Le script Python pour _win_ping_ n'existe [que pour porter la documentation](https://docs.ansible.com/ansible/latest/plugins/action.html#plugin-list). 

Jouons un peu autour de ça : 

```bash
(ansible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ mkdir -p plugins/modules
(ansible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ touch plugins/modules/win_ping.ps1
(ansible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ ANSIBLE_LIBRARY=./ ansible -i inventaire.yml -m win_ping all
WIN-AD | FAILED! => {
    "msg": "module (win_ping) is missing interpreter line"
}
``` 

Cette erreur nous montre bien comment on peut très facilement surcharger un module interne à Ansible, quelque soit la cible. Donne-nous lui du corps : 

```powershell
#!powershell

# j'insère la bibliothèque Ansible Windows 
#AnsibleRequires -CSharpUtil Ansible.Basic

$spec = @{
    options = @{
        data = @{ type = "str"; default = "pong" }
        # "data2" est ajouté, en facultatif ("required" sinon) 
        data2 = @{ type = "str" ; default = "toto" } 
    }
    supports_check_mode = $true
}
# j'instancie mon module - notez que '$args' est fourni par l'emballage 
$module = [Ansible.Basic.AnsibleModule]::Create($args, $spec)

# je récupères des données du modules, celles sérialisées initialement via le contrôle Ansible 
$data = $module.Params.data
$data2 = $module.Params.data2

$module.Result.ping = "$data - $data2" 

# j'indique de manière factice que le système cible a changé 
$module.Result.changed = $true 

# je sors, en produisant un JSON valide 
$module.ExitJson() 
```

L'exécution se joue sans erreur et l'orangé indique bien un changement sur le système cible (le vert indiquant une action réussite mais sans nécessairement changement d'état, différent de "_skipped_" qui est la non-réalisation pour une cause conditionnelle). 

```bash 
(ansible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ ANSIBLE_LIBRARY=./ ansible -i inventaire.yml -m win_ping all
WIN-AD | CHANGED => {
    "changed": true,
    "ping": "pong - toto"
}
``` 

Pour nos tests de vie, nous avons donc une piste sur le "comment" : l'usage de modules existants ([et la liste est longue](https://docs.ansible.com/ansible/2.8/modules/list_of_all_modules.html)), comme produire les siens, propres à son métier / à son besoin. Que ce soit directement en Python, soit en Powershell (ou [en ce que vous voulez !](https://docs.ansible.com/ansible/latest/plugins/shell.html)). 

## Gérer les faits d'un parc 

Nous avons vu deux points essentiels : 
  - récupérer automatiquement un grand nombre d'informations, à partir de l'équivalent du module par défaut "gather_facts" ; 
  - créer les siens et les stocker localement sur la machine Ansible. 

Cependant cette histoire de faits récupérés, d'exécution locale, peut s'avérer compliquée au quotidien. Il faut comprendre que la sérialisation des variables et du contexte de la machine Ansible vers la machine cible, et la remontée des résultats, ne veut pas dire que les machines cibles communiquent directement elles (pas du tout). 

De la même façon, la portée des faits est particulière : la machine Ansible est appelable comme machine cible, "indifféremment" des autres, via "_hosts: localhost_". Elle n'a donc pas par défaut, une autre machine cible, avec qui elle ne partage pas des variables autres que celles globales et dans l'instant où elles sont au moment de l'appel. 

Illustrons-ça (petite précision sur les noms en commentaire du code) : 

```yaml lancement2.yml
- name: Tester quelque chose (1) 
  gather_facts: yes 
  hosts: WIN-AD, localhost # attention, là c'est le nom d'inventaire... 
  vars: 
    ma_variable: "toto" 
  tasks: 
    - debug: msg="{{ ma_variable }}"
    - set_fact: 
        ma_variable: "titi"
      when: ansible_hostname == 'WIN-EMCILDHJ6MJ' # et ici c'est le nom d'hôte local (sinon prendre 'inventory_hostname')
    - set_fact: 
        ma_variable: "tata"
      when: ansible_hostname == 'julien-Vostro-7580'
    - debug: msg="{{ ma_variable }}"

```

L'exécution est sans appel, les deux sont bien divergents : 

```bash
(ansible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ ansible-playbook -i inventaire.yml ./lancement2.yml 

PLAY [Tester quelque chose (1)] *******************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************
ok: [localhost]
ok: [WIN-AD]

TASK [debug] **************************************************************************************************************************************************************
ok: [WIN-AD] => {
    "msg": "toto"
}
ok: [localhost] => {
    "msg": "toto"
}

TASK [set_fact] ***********************************************************************************************************************************************************
ok: [WIN-AD]
skipping: [localhost]

TASK [set_fact] ***********************************************************************************************************************************************************
skipping: [WIN-AD]
ok: [localhost]

TASK [debug] **************************************************************************************************************************************************************
ok: [WIN-AD] => {
    "msg": "titi"
}
ok: [localhost] => {
    "msg": "tata"
}

PLAY RECAP ****************************************************************************************************************************************************************
WIN-AD                     : ok=4    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
localhost                  : ok=4    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```

C'est-à-dire que suivant l'appel, soit on prend la machine Ansible comme une instance indépendante (c-à-d le contrôleur Ansible), soit comme une machine cible. Pour preuve, si vous relance en mode plus verbeux (`-vvv`), vous pouvez suivre la connexion à la machine cible "_localhost_" comme vers celle Windows : 

```bash
# ( ... ) 
TASK [Gathering Facts] ****************************************************************************************************************************************************
task path: /home/julien/Developpement/ansible-test-vie/lancement2.yml:3
<127.0.0.1> ESTABLISH LOCAL CONNECTION FOR USER: julien
<127.0.0.1> EXEC /bin/sh -c 'echo ~julien && sleep 0'
<127.0.0.1> EXEC /bin/sh -c '( umask 77 && mkdir -p "` echo /home/julien/.ansible/tmp `"&& mkdir /home/julien/.ansible/tmp/ansible-tmp-1636541247.5407162-99491-187805259433 && echo ansible-tmp-1636541247.5407162-99491-187805259433="` echo /home/julien/.ansible/tmp/ansible-tmp-1636541247.5407162-99491-187805259433 `" ) && sleep 0'
Using module file /usr/lib/python3/dist-packages/ansible/modules/windows/setup.ps1
Pipelining is enabled.
<192.168.56.3> ESTABLISH WINRM CONNECTION FOR USER: Administrateur on PORT 5985 TO 192.168.56.3
EXEC (via pipeline wrapper)
Using module file /usr/lib/python3/dist-packages/ansible/modules/system/setup.py
<127.0.0.1> PUT /home/julien/.ansible/tmp/ansible-local-99486k5ok0xvy/tmp0l38mgx1 TO /home/julien/.ansible/tmp/ansible-tmp-1636541247.5407162-99491-187805259433/AnsiballZ_setup.py
<127.0.0.1> EXEC /bin/sh -c 'chmod u+x /home/julien/.ansible/tmp/ansible-tmp-1636541247.5407162-99491-187805259433/ /home/julien/.ansible/tmp/ansible-tmp-1636541247.5407162-99491-187805259433/AnsiballZ_setup.py && sleep 0'
<127.0.0.1> EXEC /bin/sh -c '/usr/bin/python3 /home/julien/.ansible/tmp/ansible-tmp-1636541247.5407162-99491-187805259433/AnsiballZ_setup.py && sleep 0'
<127.0.0.1> EXEC /bin/sh -c 'rm -f -r /home/julien/.ansible/tmp/ansible-tmp-1636541247.5407162-99491-187805259433/ > /dev/null 2>&1 && sleep 0'
ok: [localhost]
ok: [WIN-AD]
META: ran handlers
``` 

Avoir une remontée d'information vers le contrôleur passe donc par le retour de votre module et de son emballage (son "_wrapper_", qui est un mécanisme interne à Ansible). Et la machine cible "_localhost_" n'est pas réellement dans l'inventaire déclaré, mais une sorte d'entrée fantôche, toujours présente. __Ne vous y méprenez pas, cette distinction a un impact.__ Voyons le passage des modifications entre les différentes recettes ("_plays_") : 

```yaml lancement2.yml
- name: Tester quelque chose (1) 
  gather_facts: yes 
  hosts: WIN-AD, localhost 
  vars: 
    mon_inventaire: "toto" 
  tasks: 

    - debug: msg="{{ mon_inventaire }}"

    - set_fact: 
        mon_inventaire: "titi"
      when: ansible_hostname == 'WIN-EMCILDHJ6MJ'

    - set_fact: 
        mon_inventaire: "tata"
      when: ansible_hostname == 'julien-Vostro-7580'

    - debug: msg="{{ mon_inventaire }}" 

- name: Tester quelque chose (2) 
  hosts: localhost 
  gather_facts: yes
  tasks: 

    - with_items: "{{ hostvars }}" 
      debug: msg="{{ hostvars[item]['mon_inventaire'] }}"
```

Vous vous attendez à avoir deux entrées de débogue ? Que nenni : 

```bash
(ansible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ ansible-playbook -i inventaire.yml ./lancement2.yml 

PLAY [Tester quelque chose (1)] *******************************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************
ok: [localhost]
ok: [WIN-AD]

TASK [debug] **************************************************************************************************************************************************************
ok: [WIN-AD] => {
    "msg": "toto"
}
ok: [localhost] => {
    "msg": "toto"
}

TASK [set_fact] ***********************************************************************************************************************************************************
ok: [WIN-AD]
skipping: [localhost]

TASK [set_fact] ***********************************************************************************************************************************************************
skipping: [WIN-AD]
ok: [localhost]

TASK [debug] **************************************************************************************************************************************************************
ok: [WIN-AD] => {
    "msg": "titi"
}
ok: [localhost] => {
    "msg": "tata"
}

PLAY [Tester quelque chose (2)] *******************************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************
ok: [localhost]

TASK [debug] **************************************************************************************************************************************************************
ok: [localhost] => (item=WIN-AD) => {
    "msg": "titi"
}

PLAY RECAP ****************************************************************************************************************************************************************
WIN-AD                     : ok=4    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
localhost                  : ok=6    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
```

Si l'on voulait véritablement considéré la machine locale (celle qui Ansible) comme une autre, il faudrait la déclarer comme les autres : 

```yaml inventaire.yml
all:
  hosts: 
    WIN-AD: 
      ansible_host: 192.168.56.3 
      ansible_user: "Administrateur"
      ansible_password: "Azerty01"
      ansible_connection: winrm
      ansible_port: 5985
      ansible_winrm_transport: ntlm 
    LINUX-LOCAL: 
      ansible_connection: local  
```

... Et elle apparaîtrait bien, comme ses modifications de valeur propres. Notez bien ici que j'ai bien spécifié "_hosts: localhost_" et non "_host: LOCAL-LINUX_". Notez bien aussi le rapport dans les dernières lignes, qui distingue la connexion locale ("_LOCAL-LINUX_") de l'hôte générique local ("_localhost_") : 

```yaml lancement2.yml
- name: Tester quelque chose (1) 
  gather_facts: yes 
  hosts: WIN-AD, LINUX-LOCAL 
  vars: 
    mon_inventaire: "toto" 
  tasks: 
    - debug: msg="{{ mon_inventaire }}"
    - set_fact: 
        mon_inventaire: "titi"
      when: ansible_hostname == 'WIN-EMCILDHJ6MJ'
    - set_fact: 
        mon_inventaire: "tata"
      when: ansible_hostname == 'julien-Vostro-7580'
    - debug: msg="{{ mon_inventaire }}" 

- name: Tester quelque chose (2) 
  hosts: localhost
  gather_facts: yes
  tasks: 
    - with_items: "{{ hostvars }}" 
      debug: msg="{{ hostvars[item]['mon_inventaire'] }}"
```

Et voici le résultat : 

```bash
(ansible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ ansible-playbook -i inventaire.yml ./lancement2.yml 

PLAY [Tester quelque chose (1)] *******************************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************
[DEPRECATION WARNING]: Distribution Ubuntu 20.10 on host LINUX-LOCAL should use /usr/bin/python3, but is using /usr/bin/python for backward compatibility with prior 
Ansible releases. A future Ansible release will default to using the discovered platform python for this host. See 
https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more information. This feature will be removed in version 2.12. Deprecation 
warnings can be disabled by setting deprecation_warnings=False in ansible.cfg.
ok: [LINUX-LOCAL]
ok: [WIN-AD]

TASK [debug] **************************************************************************************************************************************************************
ok: [WIN-AD] => {
    "msg": "toto"
}
ok: [LINUX-LOCAL] => {
    "msg": "toto"
}

TASK [set_fact] ***********************************************************************************************************************************************************
ok: [WIN-AD]
skipping: [LINUX-LOCAL]

TASK [set_fact] ***********************************************************************************************************************************************************
skipping: [WIN-AD]
ok: [LINUX-LOCAL]

TASK [debug] **************************************************************************************************************************************************************
ok: [WIN-AD] => {
    "msg": "titi"
}
ok: [LINUX-LOCAL] => {
    "msg": "tata"
}

PLAY [Tester quelque chose (2)] *******************************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************
ok: [localhost]

TASK [debug] **************************************************************************************************************************************************************
ok: [localhost] => (item=WIN-AD) => {
    "msg": "titi"
}
ok: [localhost] => (item=LINUX-LOCAL) => {
    "msg": "tata"
}

PLAY RECAP ****************************************************************************************************************************************************************
LINUX-LOCAL                : ok=4    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
WIN-AD                     : ok=4    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
localhost                  : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

C'est cet aspect qui est intéressant : notre test de vie va remonter des informations des machines cibles, vers l'hôte Ansible, en passant par des valeurs (des faits), qui sont liés à __la déclaration__ des machines. 

## Produire un rapport dans un format déterminé (ici XML) 

Bien, nous avons désormais pleins d'astuces pour (comprendre et) assurer l'organisation de nos tests de vie avec cet outil fabuleux. Ce sont des modules par défaut ou communautaires (ou les nôtres) qui assureront les tests (connexions à la base de données, ping de machine ou encore CURL d'URL, etc.). Les données remonteront au contrôleur Ansible et passeront de tâche en tâche, pour déterminer un état suivant une logique métier. 

Mais ces résultats, qu'en faire ? Faut-il envoyer par exemple une alerte pour chaque machine, même en cas de retour positif ? Si vous avez suivi des cours de logique floue (ou un peu de Bayes) et des théories de l'info-com', plusieurs points sont à considérer pour répondre à cette question : 
  - l'information, c'est ce qui sort de "l'ordinaire" dans un environnement donné : dans notre cas, on alerte lors d'un changement d'état (une perte de service alors que tout allait bien ; ou inversement en cas de reprise). Inutile d'alerte pour dire "ça va toujours (pas) bien" ; 
  - il faut des états intermédiaires, particulièrement si vous n'avez pas une machine formant un noeud unique. En cas de distribution, on alertera différemment l'état d'une machine ou d'un service de noeud, de l'ensemble d'un parc ; en privilégiant dans le second, l'alerte en cas de dépassement à la hausse ou à la baisse d'un seuil. Exemple : 80% d'activité, en baisse, de mes machines Web. 
  - on peut définir un état de santé d'un SI (donc d'au moins deux composants, eux-même pouvant être 1 ou _n_ processus), au travers d'une formule. Pour l'exemple précédent, l'état d'un SI, cela peut être la multiplication du taux de réussite des tests individuels, addionné ensuite pour chacune des machines. En dessous d'un certain délai, de tous les tests, et ce taux serait considéré valide à partir d'une certaine valeur. 

On voit donc l'intérêt d'Ansible : 
  - ne pas ré-inventer la roue : un module par grand test (BdD, service Web, etc.) ; 
  - variabiliser pour que le réusage des modules, s'adapte au travers des rôles ; 
  - définir des livres de recettes qui utilisent intelligemment les rôles, pour cibler un intérêt métier (dont les 3 classiques : responsabilité d'affaire ; d'exploitation ; de développement) ; 
  - agir sur de nombreuses machines à la fois et remontée - ou non - la bonne information, au bon moment, pour la bonne cible. 

Chaque lancement d'un livre de recette doit alimenter un jeu de données, par définition un jeu historisé. Ansible peut aussi s'en charger, même si c'est loin d'être sa cible première. Pour l'exercice, j'ai choisi [XML, car la manipulation via CRUD est disponible](https://docs.ansible.com/ansible/2.8/modules/xml_module.html) nativement. 


```yaml lancement.yml
- name: Tester l'activité de mes applicatifs 
  gather_facts: yes 
  hosts: all 
  tasks: 
    - name: Simulation d'un test de vie qui renvoi une valeur...  
      set_fact: 
        mon_retour: "toto !" 

- name: Générer un rapport 
  gather_facts: yes 
  hosts: localhost 
  tasks: 
    - name: Ecrire le retour en cas de succès 
      when: "hostvars[item]['mon_retour'] is defined "
      with_items: "{{ hostvars }}" 
      xml: 
        path: ./rapport.xml 
        xpath: /hotes 
        input_type: xml 
        add_children: 
          - "
            <{{ item }} date=\"{{ ansible_date_time.iso8601 }}\" erreur=\"non\">
              <particulier>
                {{ hostvars[item]['mon_retour'] }}
              </particulier>
              <general>
                {{ hostvars[item]['ansible_windows_domain_role'] }} ( et d'autres valeurs si on veut...) 
              </general>
            </{{ item }}>" 
    - name: Ecrire le retour en cas d'échec 
      when: "hostvars[item]['mon_retour'] is not defined "
      with_items: "{{ hostvars }}" 
      xml: 
        path: ./rapport.xml 
        xpath: /hotes 
        input_type: xml 
        add_children: 
          - "<{{ item }} date=\"{{ ansible_date_time.iso8601 }}\" erreur=\"oui\" />" 
``` 

Nous aurons besoin d'un caneva XML pour que les noeuds puissent ajoutés : 
```xml rapport.xml
<?xml version='1.0' encoding='UTF-8'?><hotes></hotes>
```

Si vous lancer la première fois le livre de recettes, vous allez vous heurtez à une erreur banale de module manquant : 

```bash
TASK [Remove the 'subjective' attribute of the 'rating' element] ***************************************************************************************************************************************************************************
An exception occurred during task execution. To see the full traceback, use -vvv. The error was: ModuleNotFoundError: No module named 'lxml'
failed: [localhost] (item=WIN-AD) => {"ansible_loop_var": "item", "changed": false, "item": "WIN-AD", "msg": "Failed to import the required Python library (lxml) on julien-Vostro-7580's Python /usr/bin/python3. Please read module documentation and install in the appropriate location. If the required library is installed, but Ansible is using the wrong Python interpreter, please consult the documentation on ansible_python_interpreter"}

PLAY RECAP *********************************************************************************************************************************************************************************************************************************
WIN-AD                     : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
localhost                  : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0 
```

Au choix : résoudre le cas sur la machine Ansible une fois pour toute... ou rajouter une étape pour valider que le paquet est bien présent. Il va s'en dire que ce second choix est celui à privilégier : il est un poil plus lent, mais rend votre livre de recettes générique. Ce serait bien un drame s'il n'était pas fichu de gérer son propre environnement ! 

Cela étant fait, l'exécution se déroule désormais sans difficulté : 

```bash
(ansible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ ansible-playbook -i inventaire.yml ./lancement.yml 

PLAY [Tester l'activité de mes applicatifs] *******************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************
ok: [WIN-AD]

TASK [simulation d'un test de vie qui renvoi une valeur...] ***************************************************************************************************************
ok: [WIN-AD]

PLAY [Générer un rapport] *************************************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************
ok: [localhost]

TASK [Ecrire le retour en cas de succès] **********************************************************************************************************************************
changed: [localhost] => (item=WIN-AD)

TASK [Ecrire le retour en cas d'échec] ************************************************************************************************************************************
skipping: [localhost] => (item=WIN-AD) 

PLAY RECAP ****************************************************************************************************************************************************************
WIN-AD                     : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
localhost                  : ok=2    changed=1    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
``` 

C'est parfait, notre rapport a été alimenté : 

```xml rapport.xml
<?xml version='1.0' encoding='UTF-8'?>
<hotes><WIN-AD date="2021-11-10T11:54:37Z" erreur="non"> <particulier> toto ! </particulier> <general> Stand-alone server ( et d'autres valeurs si on veut...) </general> </WIN-AD></hotes>
```

Voyons maintenant si l'on ajoute un hote en erreur, par exemple si ce dernier n'est pas encore (ou plus) accessible : 

```yaml inventaire.yml
all:
  hosts: 
    WIN-AD: 
      ansible_host: 192.168.56.3 
    WIN-IIS: # je l'ai éteinte, elle n'est donc pas accessible 
      ansible_host: 192.168.56.4 
    # LINUX-LOCAL:
    #   ansible_connection: local 
  vars: 
    ansible_user: "Administrateur"
    ansible_password: "Azerty01"
    ansible_connection: winrm
    ansible_port: 5985
    ansible_winrm_transport: ntlm
```

L'exécution indique bien une erreur, mais notre rapport restera complet : 

```bash
(ansible-test-vie) julien@julien-Vostro-7580:~/Developpement/ansible-test-vie$ ansible-playbook -i inventaire.yml ./lancement.yml 

PLAY [Tester l'activité de mes applicatifs] *******************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************
fatal: [WIN-IIS]: UNREACHABLE! => {"changed": false, "msg": "ntlm: HTTPConnectionPool(host='192.168.56.4', port=5985): Max retries exceeded with url: /wsman (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f40996c9460>: Failed to establish a new connection: [Errno 113] No route to host'))", "unreachable": true}
ok: [WIN-AD]

TASK [simulation d'un test de vie qui renvoi une valeur...] ***************************************************************************************************************
ok: [WIN-AD]

PLAY [Générer un rapport] *************************************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************
ok: [localhost]

TASK [Ecrire le retour en cas de succès] **********************************************************************************************************************************
changed: [localhost] => (item=WIN-AD)
skipping: [localhost] => (item=WIN-IIS) 

TASK [Ecrire le retour en cas d'échec] ************************************************************************************************************************************
skipping: [localhost] => (item=WIN-AD) 
changed: [localhost] => (item=WIN-IIS)

PLAY RECAP ****************************************************************************************************************************************************************
WIN-AD                     : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
WIN-IIS                    : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0   
localhost                  : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
``` 

On note d'ailleurs qu'il y a le résultat de l'exécution précédente. J'ai mis les tabulations pour la lisibilité : 

```xml rapport.xml
<?xml version='1.0' encoding='UTF-8'?>
<hotes>
	<!-- ici la première exécution --> 
	<WIN-AD date="2021-11-10T11:57:42Z" erreur="non"> 
		<particulier> toto ! </particulier> <general> Stand-alone server ( et d'autres valeurs si on veut...) </general> 
	</WIN-AD>
	<!-- ici la deuxième exécution --> 
	<WIN-AD date="2021-11-10T11:58:02Z" erreur="non"> 
		<particulier> toto ! </particulier> <general> Stand-alone server ( et d'autres valeurs si on veut...) </general> 
	</WIN-AD>
	<WIN-IIS date="2021-11-10T11:58:02Z" erreur="oui"/>
	<!-- les prochaines s'ajouteront après ce commentaire --> 
</hotes>
```

Voilà, il est prêt à partir : envoi par courriel, par API ou simplement le journaliser localement. 

Il existe encore d'autres notions essentielles, comme le mise en réserve des faits (pour accéler la récupération des faits, éventuellement sans passer par "_gather_facts_") ainsi que les notions de délégation. Cependant vous avez ici tout le caneva pour réussir les premiers pas d'un SI dont les tests de vie sont intelligents, efficaces et centralisés. 

Bon code ! 

---

_PS : je ne porte pas Windows dans mon coeur ; c'est juste un support._ 


