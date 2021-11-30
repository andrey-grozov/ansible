# Домашнее задание к занятию "08.03 Использование Yandex Cloud"

## Подготовка к выполнению
1. Создайте свой собственный (или используйте старый) публичный репозиторий на github с произвольным именем.
2. Скачайте [playbook](./playbook/) из репозитория с домашним заданием и перенесите его в свой репозиторий.

## Основная часть
#### 1. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает kibana.

    - name: Install Kibana
      hosts: kibana
      handlers:
        - name: restart kibana
          become: true
          service:
            name: kibana
            state: restarted
      tasks:
        - name: "Download Kibana rpm"
          get_url:
            url: "https://artifacts.elastic.co/downloads/kibana/kibana-{{ elk_stack_version }}-x86_64.rpm"
            dest: "/tmp/kibana-{{ elk_stack_version }}-x86_64.rpm"
          register: download_kibana
          until: download_kibana is succeeded
        - name: Install Kibana
          become: true
          yum:
            name: "/tmp/kibana-{{ elk_stack_version }}-x86_64.rpm"
            state: present
        - name: Configure Kibana
          become: true
          template:
            src: kibana.yml.j2
            dest: /etc/kibana/kibana.yml
            mode: 0644
          notify: restart kibana

#### 2. При создании tasks рекомендую использовать модули: `get_url`, `template`, `yum`, `apt`.
#### 3. Tasks должны: скачать нужной версии дистрибутив, выполнить распаковку в выбранную директорию, сгенерировать конфигурацию с параметрами.
#### 4. Приготовьте свой собственный inventory файл `prod.yml`.
   
    all:
      vars:
        ansible_connection: ssh
        ansible_user: vagrant
    elasticsearch:
      hosts:
        el-instance:
          ansible_host: 51.250.27.58
    kibana:
      hosts:
        k-instance:
          ansible_host: 51.250.27.204
     
#### 5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.
   
    vagrant@vagrant:~/ansible/3$ ansible-lint site.yml
    WARNING  Overriding detected file kind 'yaml' with 'playbook' for given positional argument: site.yml
    WARNING  Listing 1 violation(s) that are fatal
    risky-octal: Octal file permissions must contain leading zero or be a string
    site.yml:22 Task/Handler: Configure Elasticsearch
    
    You can skip specific rules or tags by adding them to your configuration file:
    # .ansible-lint
    warn_list:  # or 'skip_list' to silence them completely
      - risky-octal  # Octal file permissions must contain leading zero or be a string
    
    Finished with 1 failure(s), 0 warning(s) on 1 files.
   
    Добавил разрешения на каталог. предупреждение ушло.
    vagrant@vagrant:~/ansible/3$ ansible-lint site.yml
    WARNING  Overriding detected file kind 'yaml' with 'playbook' for given positional argument: site.yml
   
#### 6. Попробуйте запустить playbook на этом окружении с флагом `--check`.
   
    vagrant@vagrant:~/ansible/3$ ansible-playbook -i inventory/prod site.yml --check
    
    С первого раза не проходит по причине отсутствия файла /tmp/elasticsearch-7.15.2-x86_64.rpm.
    PLAY [Install Elasticsearch] *******************************************************************************************
    TASK [Gathering Facts] *************************************************************************************************
    ok: [el-instance]
    
    TASK [Download Elasticsearch's rpm] ************************************************************************************
    changed: [el-instance]
    
    TASK [Install Elasticsearch] *******************************************************************************************
    fatal: [el-instance]: FAILED! => {"changed": false, "msg": "No RPM file matching '/tmp/elasticsearch-7.15.2-x86_64.rpm' 
    found on system", "rc": 127, "results": ["No RPM file matching '/tmp/elasticsearch-7.15.2-x86_64.rpm' found on system"]}
    PLAY RECAP *************************************************************************************************************
    el-instance                : ok=2    changed=1    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
   
#### 7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.

   
    vagrant@vagrant:~/ansible/3$ ansible-playbook -i inventory/prod site.yml --diff
    
    PLAY [Install Elasticsearch] **************************************************************************************************************************
    
    TASK [Gathering Facts] ********************************************************************************************************************************
    ok: [el-instance]
    
    TASK [Download Elasticsearch's rpm] *******************************************************************************************************************
    ok: [el-instance]
    
    TASK [Install Elasticsearch] **************************************************************************************************************************
    ok: [el-instance]
    
    TASK [Configure Elasticsearch] ************************************************************************************************************************
    ok: [el-instance]
    
    PLAY [Install Kibana] *********************************************************************************************************************************
    
    TASK [Gathering Facts] ********************************************************************************************************************************
    ok: [k-instance]
    
    TASK [Download Kibana rpm] ****************************************************************************************************************************
    ok: [k-instance]
    
    TASK [Install Kibana] *********************************************************************************************************************************
    ok: [k-instance]
    
    TASK [Configure Kibana] *******************************************************************************************************************************
    ok: [k-instance]
    
    PLAY RECAP ********************************************************************************************************************************************
    el-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    k-instance                 : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

#### 8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.
   
    
    PLAY RECAP ********************************************************************************************************************************************
    el-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    k-instance                 : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

#### 9. Проделайте шаги с 1 до 8 для создания ещё одного play, который устанавливает и настраивает filebeat.
#### 9.1. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает filebeat.

    - name: Install filebeat
      hosts: app
      handlers:
        - name: restart filebeat
          become: true
          systemd:
            name: filebeat
            state: restarted
      tasks:
        - name: "Download filebeat rpm"
          get_url:
            url: "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-{{ elk_stack_version }}-x86_64.rpm"
            dest: "/tmp/filebeat-{{ elk_stack_version }}-x86_64.rpm"
          register: download_filebeat
          until: download_filebeat is succeeded
        - name: Install filebeat
          become: true
          yum:
            name: "/tmp/filebeat-{{ elk_stack_version }}-x86_64.rpm"
            state: present
        - name: Configure filebeat
          become: true
          template:
            src: filebeat.yml.j2
            dest: /etc/filebeat/filebeat.yml
            mode: 0644
          notify: restart filebeat
#### 9.2. При создании tasks рекомендую использовать модули: `get_url`, `template`, `yum`, `apt`.
#### 9.3. Tasks должны: скачать нужной версии дистрибутив, выполнить распаковку в выбранную директорию, сгенерировать конфигурацию с параметрами.
#### 9.4. Приготовьте свой собственный inventory файл `prod.yml`.

    all:
      vars:
        ansible_connection: ssh
        ansible_user: vagrant
    elasticsearch:
      hosts:
        el-instance:
          ansible_host: 51.250.27.58
    kibana:
      hosts:
        k-instance:
          ansible_host: 51.250.27.204
    app:
      hosts:
        app-instance:
          ansible_host: 51.250.16.198
    
#### 9.5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.    
    
Ошибок нет

#### 9.6. Попробуйте запустить playbook на этом окружении с флагом `--check`.

    При первом запуске выдается ошибка о том что файл не существует.
    vagrant@vagrant:~/ansible/3$ ansible-playbook -i inventory/prod site.yml --check
    
    PLAY [Install Elasticsearch] **************************************************************************************************************************
    
    TASK [Gathering Facts] ********************************************************************************************************************************
    ok: [el-instance]
    
    TASK [Download Elasticsearch's rpm] *******************************************************************************************************************
    ok: [el-instance]
    
    TASK [Install Elasticsearch] **************************************************************************************************************************
    ok: [el-instance]
    
    TASK [Configure Elasticsearch] ************************************************************************************************************************
    ok: [el-instance]
    
    PLAY [Install Kibana] *********************************************************************************************************************************
    
    TASK [Gathering Facts] ********************************************************************************************************************************
    ok: [k-instance]
    
    TASK [Download Kibana rpm] ****************************************************************************************************************************
    ok: [k-instance]
    
    TASK [Install Kibana] *********************************************************************************************************************************
    ok: [k-instance]
    
    TASK [Configure Kibana] *******************************************************************************************************************************
    ok: [k-instance]
    
    PLAY [Install filebeat] *******************************************************************************************************************************
    
    TASK [Gathering Facts] ********************************************************************************************************************************
    ok: [app-instance]
    
    TASK [Download filebeat rpm] **************************************************************************************************************************
    changed: [app-instance]
    
    TASK [Install filebeat] *******************************************************************************************************************************
    fatal: [app-instance]: FAILED! => {"changed": false, "msg": "No RPM file matching '/tmp/filebeat-7.15.2-x86_64.rpm' found on system", "rc": 127, "results": ["No RPM file matching '/tmp/filebeat-7.15.2-x86_64.rpm' found on system"]}
    
    PLAY RECAP ********************************************************************************************************************************************
    app-instance               : ok=2    changed=1    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
    el-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    k-instance                 : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

#### 9.7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.

    vagrant@vagrant:~/ansible/3$ ansible-playbook -i inventory/prod site.yml --diff
    
    PLAY [Install Elasticsearch] **************************************************************************************************************************
    
    TASK [Gathering Facts] ********************************************************************************************************************************
    ok: [el-instance]
    
    TASK [Download Elasticsearch's rpm] *******************************************************************************************************************
    ok: [el-instance]
    
    TASK [Install Elasticsearch] **************************************************************************************************************************
    ok: [el-instance]
    
    TASK [Configure Elasticsearch] ************************************************************************************************************************
    ok: [el-instance]
    
    PLAY [Install Kibana] *********************************************************************************************************************************
    
    TASK [Gathering Facts] ********************************************************************************************************************************
    ok: [k-instance]
    
    TASK [Download Kibana rpm] ****************************************************************************************************************************
    ok: [k-instance]
    
    TASK [Install Kibana] *********************************************************************************************************************************
    ok: [k-instance]
    
    TASK [Configure Kibana] *******************************************************************************************************************************
    ok: [k-instance]
    
    PLAY [Install filebeat] *******************************************************************************************************************************
    
    TASK [Gathering Facts] ********************************************************************************************************************************
    ok: [app-instance]
    
    TASK [Download filebeat rpm] **************************************************************************************************************************
    ok: [app-instance]
    
    TASK [Install filebeat] *******************************************************************************************************************************
    ok: [app-instance]
    
    TASK [Configure filebeat] *****************************************************************************************************************************
    ok: [app-instance]
    
    PLAY RECAP ********************************************************************************************************************************************
    app-instance               : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    el-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    k-instance                 : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

#### 9.8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.

    PLAY RECAP ********************************************************************************************************************************************
    app-instance               : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    el-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    k-instance                 : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

10. Подготовьте README.md файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.
11. Готовый playbook выложите в свой репозиторий, в ответ предоставьте ссылку на него.

---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---
