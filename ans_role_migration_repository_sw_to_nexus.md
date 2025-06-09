# Роль Ansible для миграции с Spacewalk на Nexus

Вот пример роли Ansible для миграции репозиториев с Spacewalk на Nexus для Oracle Linux 9 и Ubuntu.

## Структура роли

```
roles/nexus_migration/
├── tasks/
│   ├── main.yml
│   ├── oracle_linux.yml
│   └── ubuntu.yml
├── templates/
│   ├── oracle_nexus.repo.j2
│   └── ubuntu_nexus.list.j2
└── vars/
    └── main.yml
```

## Содержание файлов

### vars/main.yml

```yaml
nexus_url: "https://nexus.corporate.example.com"
nexus_oracle_repo: "oracle9-repo"
nexus_ubuntu_repo: "ubuntu-repo"
gpg_check: "no"  # Поскольку исключаем CE-сертификат
```

### tasks/main.yml

```yaml
- name: Detect OS and include specific tasks
  include_tasks: "{{ ansible_distribution | lower }}.yml"
  when: ansible_distribution in ['OracleLinux', 'Ubuntu']
  
- name: Fail if unsupported OS
  fail:
    msg: "Unsupported OS {{ ansible_distribution }}"
  when: ansible_distribution not in ['OracleLinux', 'Ubuntu']
```

### tasks/oracle_linux.yml

```yaml
- name: Remove Spacewalk client for Oracle Linux
  yum:
    name: rhn-client-tools
    state: absent

- name: Remove Spacewalk repository
  file:
    path: /etc/yum.repos.d/spacewalk.repo
    state: absent

- name: Install Nexus repository configuration for Oracle Linux
  template:
    src: oracle_nexus.repo.j2
    dest: /etc/yum.repos.d/nexus.repo
    owner: root
    group: root
    mode: '0644'

- name: Clean yum cache
  command: yum clean all
```

### tasks/ubuntu.yml

```yaml
- name: Remove Spacewalk client for Ubuntu
  apt:
    name: python3-rhn
    state: absent

- name: Remove Spacewalk repository
  file:
    path: /etc/apt/sources.list.d/spacewalk.list
    state: absent

- name: Install Nexus repository configuration for Ubuntu
  template:
    src: ubuntu_nexus.list.j2
    dest: /etc/apt/sources.list.d/nexus.list
    owner: root
    group: root
    mode: '0644'

- name: Update apt cache
  apt:
    update_cache: yes
```

### templates/oracle_nexus.repo.j2

```ini
[nexus]
name=Nexus Corporate Repository - Oracle Linux 9
baseurl={{ nexus_url }}/repository/{{ nexus_oracle_repo }}/
enabled=1
gpgcheck={{ gpg_check }}
sslverify=0
```

### templates/ubuntu_nexus.list.j2

```ini
deb [trusted=yes] {{ nexus_url }}/repository/{{ nexus_ubuntu_repo }}/ {{ ansible_distribution_release }} main
```

## Использование роли

1. Создайте playbook для применения роли:

```yaml
- hosts: all
  become: yes
  roles:
    - nexus_migration
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini migration_playbook.yml
```

## Примечания

1. Роль предполагает, что Nexus уже настроен и доступен по указанному URL.
2. Отключена проверка SSL (sslverify=0 для Oracle, [trusted=yes] для Ubuntu), так как требуется исключить CE-сертификат.
3. Порт 443 не указан явно, используется по умолчанию для HTTPS.
4. Удаляются клиенты Spacewalk (rhn-client-tools для Oracle, python3-rhn для Ubuntu).
5. Удаляются конфигурационные файлы Spacewalk.

Для корпоративного использования рекомендуется:
1. Добавить обработку ошибок
2. Реализовать проверку доступности Nexus перед миграцией
3. Добавить теги для возможности частичного выполнения
4. Реализовать откат изменений при необходимости


# Улучшенная роль Ansible для миграции с Spacewalk на Nexus

Вот усовершенствованная версия роли с обработкой ошибок, проверкой доступности Nexus, тегами и возможностью отката.

## Обновленная структура роли

```
roles/nexus_migration/
├── tasks/
│   ├── main.yml
│   ├── oracle_linux.yml
│   ├── ubuntu.yml
│   ├── pre_check.yml
│   ├── rollback.yml
├── templates/
│   ├── oracle_nexus.repo.j2
│   └── ubuntu_nexus.list.j2
└── vars/
    └── main.yml
```

## Обновленные файлы

### vars/main.yml

```yaml
nexus_url: "https://nexus.corporate.example.com"
nexus_oracle_repo: "oracle9-repo"
nexus_ubuntu_repo: "ubuntu-repo"
gpg_check: "no"
timeout_seconds: 10  # Таймаут для проверки доступности Nexus
backup_dir: "/tmp/nexus_migration_backup"  # Директория для бэкапов
```

### tasks/main.yml

```yaml
- name: Include pre-check tasks
  include_tasks: pre_check.yml
  tags: always

- name: Create backup directory
  file:
    path: "{{ backup_dir }}"
    state: directory
    mode: '0755'
  tags: backup

- name: Detect OS and include specific tasks
  block:
    - name: Include OS-specific tasks
      include_tasks: "{{ ansible_distribution | lower }}.yml"
      when: ansible_distribution in ['OracleLinux', 'Ubuntu']
  
    - name: Fail if unsupported OS
      fail:
        msg: "Unsupported OS {{ ansible_distribution }}"
      when: ansible_distribution not in ['OracleLinux', 'Ubuntu']
  tags: migration

- name: Notify success
  debug:
    msg: "Migration to Nexus completed successfully"
  when: not rollback_triggered | default(false)
  tags: notify
```

### tasks/pre_check.yml

```yaml
- name: Check Nexus availability
  uri:
    url: "{{ nexus_url }}/service/rest/v1/status"
    validate_certs: no
    timeout: "{{ timeout_seconds }}"
    status_code: 200
  register: nexus_check
  ignore_errors: yes
  tags: always

- name: Fail if Nexus is not available
  fail:
    msg: "Nexus server is not available at {{ nexus_url }}. Check connection or server status."
  when: nexus_check is failed
  tags: always
```

### tasks/rollback.yml

```yaml
- name: Rollback - Restore original repositories
  block:
    - name: Find backup files
      find:
        paths: "{{ backup_dir }}"
        patterns: "*.bak"
        recurse: yes
      register: backup_files
      ignore_errors: yes

    - name: Restore backup files
      copy:
        src: "{{ item.path }}"
        dest: "{{ item.path | regex_replace('.*/([^/]+)\\.bak$', '/etc/\\1') }}"
        remote_src: yes
        owner: root
        group: root
        mode: '0644'
      loop: "{{ backup_files.files }}"
      when: backup_files.matched > 0

    - name: Reinstall Spacewalk client if needed
      block:
        - name: Reinstall Spacewalk client for Oracle Linux
          yum:
            name: rhn-client-tools
            state: present
          when: ansible_distribution == 'OracleLinux'

        - name: Reinstall Spacewalk client for Ubuntu
          apt:
            name: python3-rhn
            state: present
          when: ansible_distribution == 'Ubuntu'
      ignore_errors: yes

  when: rollback_triggered | default(false)
  tags: rollback
```

### Обновленные OS-specific задачи (oracle_linux.yml и ubuntu.yml)

```yaml
# tasks/oracle_linux.yml
- name: Backup current Spacewalk repository config
  copy:
    src: /etc/yum.repos.d/spacewalk.repo
    dest: "{{ backup_dir }}/spacewalk.repo.bak"
    remote_src: yes
  ignore_errors: yes
  tags: backup

- name: Remove Spacewalk client for Oracle Linux
  yum:
    name: rhn-client-tools
    state: absent
  register: remove_spacewalk
  failed_when: false
  tags: removal

- name: Remove Spacewalk repository
  file:
    path: /etc/yum.repos.d/spacewalk.repo
    state: absent
  register: remove_repo
  failed_when: false
  tags: removal

- name: Install Nexus repository configuration for Oracle Linux
  template:
    src: oracle_nexus.repo.j2
    dest: /etc/yum.repos.d/nexus.repo
    owner: root
    group: root
    mode: '0644'
  tags: config

- name: Verify Nexus repository
  command: yum repolist enabled --disablerepo="*" --enablerepo="nexus"
  register: repo_check
  failed_when: "'nexus' not in repo_check.stdout"
  tags: verification
```

```yaml
# tasks/ubuntu.yml
- name: Backup current Spacewalk repository config
  copy:
    src: /etc/apt/sources.list.d/spacewalk.list
    dest: "{{ backup_dir }}/spacewalk.list.bak"
    remote_src: yes
  ignore_errors: yes
  tags: backup

- name: Remove Spacewalk client for Ubuntu
  apt:
    name: python3-rhn
    state: absent
  register: remove_spacewalk
  failed_when: false
  tags: removal

- name: Remove Spacewalk repository
  file:
    path: /etc/apt/sources.list.d/spacewalk.list
    state: absent
  register: remove_repo
  failed_when: false
  tags: removal

- name: Install Nexus repository configuration for Ubuntu
  template:
    src: ubuntu_nexus.list.j2
    dest: /etc/apt/sources.list.d/nexus.list
    owner: root
    group: root
    mode: '0644'
  tags: config

- name: Verify Nexus repository
  command: apt-cache policy
  register: repo_check
  failed_when: "'nexus' not in repo_check.stdout"
  tags: verification
```

## Пример playbook с обработкой ошибок и откатом

```yaml
- hosts: all
  become: yes
  vars:
    rollback_triggered: false
  roles:
    - role: nexus_migration
  rescue:
    - name: Set rollback flag
      set_fact:
        rollback_triggered: true

    - name: Include rollback tasks
      include_role:
        name: nexus_migration
        tasks_from: rollback.yml

    - name: Notify about rollback
      debug:
        msg: "Migration failed. Rollback has been performed."
```

## Использование с тегами

1. Выполнить только проверки:
```bash
ansible-playbook -i inventory.ini migration_playbook.yml --tags "always"
```

2. Выполнить миграцию без проверок:
```bash
ansible-playbook -i inventory.ini migration_playbook.yml --tags "migration,config"
```

3. Выполнить только откат:
```bash
ansible-playbook -i inventory.ini migration_playbook.yml --tags "rollback" -e "rollback_triggered=true"
```

4. Полный прогон (рекомендуется):
```bash
ansible-playbook -i inventory.ini migration_playbook.yml
```

## Улучшения:

1. **Обработка ошибок**:
   - Использование `ignore_errors` для некритичных операций
   - Регистрация результатов операций для последующей проверки
   - Явное указание `failed_when` для контроля над ошибками

2. **Проверка доступности Nexus**:
   - Проверка через HTTP запрос перед началом миграции
   - Проверка доступности репозитория после настройки

3. **Теги**:
   - Разделение задач на логические группы с тегами
   - Возможность выборочного выполнения частей миграции

4. **Откат изменений**:
   - Создание бэкапов перед изменениями
   - Отдельный файл задач для отката
   - Механизм активации отката через переменную

5. **Дополнительные улучшения**:
   - Уведомления о статусе выполнения
   - Проверка корректности настройки репозиториев
   - Гибкая настройка через переменные