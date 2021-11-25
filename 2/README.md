# Домашнее задание к занятию "08.02 Работа с Playbook"

## Подготовка к выполнению
1. Создайте свой собственный (или используйте старый) публичный репозиторий на github с произвольным именем.
2. Скачайте [playbook] из репозитория с домашним заданием и перенесите его в свой репозиторий.
3. Подготовьте хосты в соотвтествии с группами из предподготовленного playbook. 
4. Скачайте дистрибутив [java](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html) и положите его в директорию `playbook/files/`. 

## Основная часть
#### 1. Приготовьте свой собственный inventory файл `prod.yml`.

    ---
    elasticsearch:
      hosts:
        elastic0:
          ansible_connection: docker
    kibana:
      hosts:
        kibana0:
          ansible_connection: docker

#### 2. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает kibana.

    
    - name: Install kibana
      hosts: kibana
      tasks:
        - name: Upload tar.gz kibana from remote URL
          get_url:
            url: "https://artifacts.elastic.co/downloads/kibana/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
            dest: "/tmp/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
            mode: 0644
            timeout: 60
            force: true
            validate_certs: false
          register: get_kibana
          until: get_kibana is succeeded
          tags: kibana
        - name: Create directrory for kibana
          file:
            state: directory
            path: "{{ kibana_home }}"
            mode: 0755
          tags: kibana
        - name: Extract Kibana in the installation directory
          become: true
          unarchive:
            copy: false
            src: "/tmp/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
            dest: "{{ kibana_home }}"
            extra_opts: [--strip-components=1]
            creates: "{{ kibana_home }}/bin/kibana"
          tags: kibana
        - name: Set environment Kibana
          become: true
          template:
            src: templates/kib.sh.j2
            dest: /etc/profile.d/kib.sh
            mode: 0644
            tags: kibana

#### 3. При создании tasks рекомендую использовать модули: `get_url`, `template`, `unarchive`, `file`.
#### 4. Tasks должны: скачать нужной версии дистрибутив, выполнить распаковку в выбранную директорию, сгенерировать конфигурацию с параметрами.

    
#### 5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.

    При запуске были предупреждения:  risky-file-permissions: File permissions unset or incorrect
    После доавления разрешений для файлов и директорий предупреждения исчезли.
   
#### 6. Попробуйте запустить playbook на этом окружении с флагом `--check`.

    При первом запуске выдаст ошибку, т.к. отсутствуют каталоги для явы в контейнерах.
    root@vagrant:/home/vagrant/ansible/2# ansible-playbook -i inventory/prod.yml site.yml --check
    
    PLAY [Install Java] **********************************************************************************************************
    
    TASK [Gathering Facts] *******************************************************************************************************
    ok: [kibana0]
    ok: [elastic0]
    
    TASK [Set facts for Java 11 vars] ********************************************************************************************
    ok: [elastic0]
    ok: [kibana0]
    
    TASK [Upload .tar.gz file containing binaries from local storage] ************************************************************
    changed: [elastic0]
    changed: [kibana0]
    
    TASK [Ensure installation dir exists] ****************************************************************************************
    changed: [elastic0]
    changed: [kibana0]
    
    TASK [Extract java in the installation directory] ****************************************************************************
    An exception occurred during task execution. To see the full traceback, use -vvv. The error was: NoneType: None
    fatal: [kibana0]: FAILED! => {"changed": false, "msg": "dest '/opt/jdk/11.0.13' must be an existing dir"}
    An exception occurred during task execution. To see the full traceback, use -vvv. The error was: NoneType: None
    fatal: [elastic0]: FAILED! => {"changed": false, "msg": "dest '/opt/jdk/11.0.13' must be an existing dir"}
    
    PLAY RECAP *******************************************************************************************************************
    elastic0                   : ok=4    changed=2    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
    kibana0                    : ok=4    changed=2    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0    
#### 7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.

    root@vagrant:/home/vagrant/ansible/2# ansible-playbook -i inventory/prod.yml site.yml --diff
    
    PLAY [Install Java] **********************************************************************************************************
    
    TASK [Gathering Facts] *******************************************************************************************************
    [DEPRECATION WARNING]: Distribution Ubuntu 18.04 on host kibana0 should use /usr/bin/python3, but is using /usr/bin/python
    for backward compatibility with prior Ansible releases. A future Ansible release will default to using the discovered
    platform python for this host. See https://docs.ansible.com/ansible-core/2.11/reference_appendices/interpreter_discovery.html
     for more information. This feature will be removed in version 2.12. Deprecation warnings can be disabled by setting
    deprecation_warnings=False in ansible.cfg.
    ok: [kibana0]
    [DEPRECATION WARNING]: Distribution Ubuntu 18.04 on host elastic0 should use /usr/bin/python3, but is using /usr/bin/python
    for backward compatibility with prior Ansible releases. A future Ansible release will default to using the discovered
    platform python for this host. See https://docs.ansible.com/ansible-core/2.11/reference_appendices/interpreter_discovery.html
     for more information. This feature will be removed in version 2.12. Deprecation warnings can be disabled by setting
    deprecation_warnings=False in ansible.cfg.
    ok: [elastic0]
    
    TASK [Set facts for Java 11 vars] ********************************************************************************************
    ok: [elastic0]
    ok: [kibana0]
    
    TASK [Upload .tar.gz file containing binaries from local storage] ************************************************************
    diff skipped: source file size is greater than 104448
    changed: [kibana0]
    diff skipped: source file size is greater than 104448
    changed: [elastic0]
    
    TASK [Ensure installation dir exists] ****************************************************************************************
    --- before
    +++ after
    @@ -1,4 +1,4 @@
     {
         "path": "/opt/jdk/11.0.13",
    -    "state": "absent"
    +    "state": "directory"
     }
    
    changed: [kibana0]
    --- before
    +++ after
    @@ -1,4 +1,4 @@
     {
         "path": "/opt/jdk/11.0.13",
    -    "state": "absent"
    +    "state": "directory"
     }
    
    changed: [elastic0]
    
    TASK [Extract java in the installation directory] ****************************************************************************
    changed: [kibana0]
    changed: [elastic0]
    
    TASK [Export environment variables] ******************************************************************************************
    --- before
    +++ after: /root/.ansible/tmp/ansible-local-73206_rwartdy/tmp8sjxk8q6/jdk.sh.j2
    @@ -0,0 +1,5 @@
    +# Warning: This file is Ansible Managed, manual changes will be overwritten on next playbook run.
    +#!/usr/bin/env bash
    +
    +export JAVA_HOME=/opt/jdk/11.0.13
    +export PATH=$PATH:$JAVA_HOME/bin
    
    changed: [elastic0]
    --- before
    +++ after: /root/.ansible/tmp/ansible-local-73206_rwartdy/tmprgu4amyp/jdk.sh.j2
    @@ -0,0 +1,5 @@
    +# Warning: This file is Ansible Managed, manual changes will be overwritten on next playbook run.
    +#!/usr/bin/env bash
    +
    +export JAVA_HOME=/opt/jdk/11.0.13
    +export PATH=$PATH:$JAVA_HOME/bin
    
    changed: [kibana0]
    
    PLAY [Install Elasticsearch] *************************************************************************************************
    
    TASK [Gathering Facts] *******************************************************************************************************
    ok: [elastic0]
    
    TASK [Upload tar.gz Elasticsearch from remote URL] ***************************************************************************
    changed: [elastic0]
    
    TASK [Create directrory for Elasticsearch] ***********************************************************************************
    --- before
    +++ after
    @@ -1,4 +1,4 @@
     {
         "path": "/opt/elastic/7.15.2",
    -    "state": "absent"
    +    "state": "directory"
     }
    
    changed: [elastic0]
    
    TASK [Extract Elasticsearch in the installation directory] *******************************************************************
    changed: [elastic0]
    
    TASK [Set environment Elastic] ***********************************************************************************************
    --- before
    +++ after: /root/.ansible/tmp/ansible-local-73206_rwartdy/tmpmar8myar/elk.sh.j2
    @@ -0,0 +1,5 @@
    +# Warning: This file is Ansible Managed, manual changes will be overwritten on next playbook run.
    +#!/usr/bin/env bash
    +
    +export ES_HOME=/opt/elastic/7.15.2
    +export PATH=$PATH:$ES_HOME/bin
    
    changed: [elastic0]
    
    PLAY [Install kibana] ********************************************************************************************************
    
    TASK [Gathering Facts] *******************************************************************************************************
    ok: [kibana0]
    
    TASK [Upload tar.gz kibana from remote URL] **********************************************************************************
    changed: [kibana0]
    
    TASK [Create directrory for kibana] ******************************************************************************************
    --- before
    +++ after
    @@ -1,4 +1,4 @@
     {
         "path": "/opt/kibana/7.15.2",
    -    "state": "absent"
    +    "state": "directory"
     }
    
    changed: [kibana0]
    
    TASK [Extract Kibana in the installation directory] **************************************************************************
    changed: [kibana0]
    
    TASK [Set environment Kibana] ************************************************************************************************
    --- before
    +++ after: /root/.ansible/tmp/ansible-local-73206_rwartdy/tmprrma2t7z/kib.sh.j2
    @@ -0,0 +1,5 @@
    +# Warning: This file is Ansible Managed, manual changes will be overwritten on next playbook run.
    +#!/usr/bin/env bash
    +
    +export KIBANA_HOME=/opt/kibana/7.15.2
    +export PATH=$PATH:$KIBANA_HOME/bin
    
    changed: [kibana0]
    
    PLAY RECAP *******************************************************************************************************************
    elastic0                   : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    kibana0                    : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
#### 8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.

    PLAY RECAP *******************************************************************************************************************
    elastic0                   : ok=9    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
    kibana0                    : ok=9    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
#### 9. Подготовьте README.md файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.
#### 10. Готовый playbook выложите в свой репозиторий, в ответ предоставьте ссылку на него.

---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---
