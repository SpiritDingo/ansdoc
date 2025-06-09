# Универсальная Ansible роль для миграции репозиториев с Spacewalk на Nexus (Oracle Linux 9 и Ubuntu)

Эта роль обеспечивает кросс-дистрибутивную миграцию репозиториев с Spacewalk на Sonatype Nexus для Oracle Linux 9 и Ubuntu, работающую через порт 443.

## Особенности решения

1. Поддержка как Oracle Linux 9 (dnf/yum), так и Ubuntu (apt)
2. Работа через HTTPS (порт 443)
3. Удаление Spacewalk-клиента
4. Проверка доступности Nexus перед миграцией
5. Автоматическое определение дистрибутива
6. Поддержка корпоративных сертификатов

## Структура роли

```
roles/nexus_migration/
├── tasks/
│   ├── main.yml
│   ├── common/
│   │   ├── preflight.yml
│   │   ├── remove_spacewalk.yml
│   │   └── verify.yml
│   ├── oraclelinux/
│   │   ├── configure.yml
│   │   └── cleanup.yml
│   └── ubuntu/
│       ├── configure.yml
│       └── cleanup.yml
├── templates/
│   ├── oraclelinux/
│   │   └── nexus-repo.j2
│   └── ubuntu/
│       └── nexus-sources.list.j2
├── handlers/
│   └── main.yml
├── vars/
│   ├── main.yml
│   ├── oraclelinux.yml
│   └── ubuntu.yml
└── defaults/
    └── main.yml
```

## Ключевые переменные (defaults/main.yml)

```yaml
# Общие настройки Nexus
nexus:
  base_url: "https://nexus.corp.com:443/repository"
  ssl_verify: true
  ssl_cert_path: "/etc/ssl/certs/corporate-ca.pem"
  gpg:
    enabled: true
    oraclelinux_key: "https://nexus.corp.com:443/repository/keys/RPM-GPG-KEY-oracle"
    ubuntu_key: "https://nexus.corp.com:443/repository/keys/UBUNTU-GPG-KEY"

# Настройки миграции
migration:
  backup: true
  cleanup_spacewalk: true
  test_package: "curl"  # Тестовый пакет для проверки работы репозиториев

# Настройки для Oracle Linux
oraclelinux:
  repos:
    baseos:
      enabled: true
      path: "ol9-baseos"
    appstream:
      enabled: true
      path: "ol9-appstream"

# Настройки для Ubuntu
ubuntu:
  repos:
    main:
      enabled: true
      path: "ubuntu-main"
      components: "main restricted universe multiverse"
      arch: "amd64"
    updates:
      enabled: true
      path: "ubuntu-updates"
      components: "main restricted universe multiverse"
      arch: "amd64"
```

## Основной tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  ansible.builtin.include_vars: "{{ ansible_distribution | lower }}.yml"

- name: Include common preflight tasks
  ansible.builtin.import_tasks: "common/preflight.yml"

- name: Remove Spacewalk client
  ansible.builtin.import_tasks: "common/remove_spacewalk.yml"
  when: migration.cleanup_spacewalk

- name: Configure Nexus repositories
  ansible.builtin.import_tasks: "{{ ansible_distribution | lower }}/configure.yml"

- name: Verify migration
  ansible.builtin.import_tasks: "common/verify.yml"
```

## Общие задачи (common/preflight.yml)

```yaml
---
- name: Check Nexus accessibility
  ansible.builtin.uri:
    url: "{{ nexus.base_url }}/healthcheck"
    validate_certs: "{{ nexus.ssl_verify }}"
    ca_path: "{{ nexus.ssl_cert_path }}"
    timeout: 30
    status_code: 200
  register: nexus_health
  ignore_errors: yes

- name: Fail if Nexus is not accessible
  ansible.builtin.fail:
    msg: "Nexus is not accessible at {{ nexus.base_url }}"
  when: nexus_health is failed

- name: Install corporate CA certificate
  ansible.builtin.copy:
    src: "files/corporate-ca.pem"
    dest: "{{ nexus.ssl_cert_path }}"
    owner: root
    group: root
    mode: '0644'
  when: nexus.ssl_verify and nexus.ssl_cert_path is defined
```

## Удаление Spacewalk (common/remove_spacewalk.yml)

```yaml
---
- name: Check if Spacewalk client is installed
  ansible.builtin.command: |
    if [ -f /etc/sysconfig/rhn/systemid ]; then
      echo "installed"
    else
      echo "not_installed"
    fi
  register: spacewalk_check
  changed_when: false
  ignore_errors: true

- name: Remove Spacewalk client (Oracle Linux/RHEL)
  block:
    - name: Remove Spacewalk packages
      ansible.builtin.package:
        name:
          - rhn-client-tools
          - rhn-check
          - rhn-setup
          - rhnsd
          - yum-rhn-plugin
        state: absent
    
    - name: Cleanup Spacewalk configuration
      ansible.builtin.file:
        path: "/etc/sysconfig/rhn"
        state: absent
  when: 
    - spacewalk_check.stdout == "installed"
    - ansible_distribution in ['OracleLinux', 'RedHat', 'CentOS']

- name: Remove Spacewalk client (Ubuntu)
  block:
    - name: Remove Spacewalk packages
      ansible.builtin.apt:
        name:
          - rhn-client-tools
          - python3-rhn
        state: absent
  when: 
    - spacewalk_check.stdout == "installed"
    - ansible_distribution == 'Ubuntu'
```

## Конфигурация для Oracle Linux (oraclelinux/configure.yml)

```yaml
---
- name: Cleanup existing repositories
  ansible.builtin.file:
    path: "/etc/yum.repos.d/{{ item }}"
    state: absent
  loop:
    - "rhn.repo"
    - "spacewalk.repo"
    - "oracle*.repo"

- name: Configure Nexus repositories
  ansible.builtin.template:
    src: "oraclelinux/nexus-repo.j2"
    dest: "/etc/yum.repos.d/nexus-{{ item.key }}.repo"
    mode: '0644'
  loop: "{{ oraclelinux.repos | dict2items }}"
  when: item.value.enabled

- name: Import GPG key
  ansible.builtin.rpm_key:
    state: present
    key: "{{ nexus.gpg.oraclelinux_key }}"
  when: nexus.gpg.enabled
```

## Конфигурация для Ubuntu (ubuntu/configure.yml)

```yaml
---
- name: Cleanup existing sources
  ansible.builtin.file:
    path: "/etc/apt/sources.list.d/{{ item }}"
    state: absent
  loop:
    - "rhn.list"
    - "spacewalk.list"

- name: Configure Nexus repositories
  ansible.builtin.template:
    src: "ubuntu/nexus-sources.list.j2"
    dest: "/etc/apt/sources.list.d/nexus-{{ item.key }}.list"
    mode: '0644'
  loop: "{{ ubuntu.repos | dict2items }}"
  when: item.value.enabled

- name: Import GPG key
  ansible.builtin.apt_key:
    url: "{{ nexus.gpg.ubuntu_key }}"
    validate_certs: no
  when: nexus.gpg.enabled

- name: Update apt cache
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600
```

## Проверка миграции (common/verify.yml)

```yaml
---
- name: Verify repositories (Oracle Linux)
  block:
    - name: Check repository list
      ansible.builtin.command: dnf repolist
      register: repo_list
      changed_when: false
    
    - name: Validate repositories
      ansible.builtin.assert:
        that:
          - "'nexus-{{ item.key }}' in repo_list.stdout"
        fail_msg: "Repository nexus-{{ item.key }} not configured"
      loop: "{{ oraclelinux.repos | dict2items }}"
      when: item.value.enabled
  when: ansible_distribution in ['OracleLinux', 'RedHat', 'CentOS']

- name: Verify repositories (Ubuntu)
  block:
    - name: Check repository list
      ansible.builtin.command: apt-cache policy
      register: apt_policy
      changed_when: false
    
    - name: Validate repositories
      ansible.builtin.assert:
        that:
          - "nexus.corp.com:443/repository/{{ ubuntu.repos[item.key].path }} in apt_policy.stdout"
        fail_msg: "Repository {{ item.key }} not configured"
      loop: "{{ ubuntu.repos | dict2items }}"
      when: item.value.enabled
  when: ansible_distribution == 'Ubuntu'

- name: Test package installation
  ansible.builtin.package:
    name: "{{ migration.test_package }}"
    state: present
```

## Шаблоны репозиториев

### Oracle Linux (templates/oraclelinux/nexus-repo.j2)

```
[nexus-{{ item.key }}]
name=Nexus {{ item.key }} repository
baseurl={{ nexus.base_url }}/{{ item.value.path }}
enabled=1
gpgcheck={% if nexus.gpg.enabled %}1{% else %}0{% endif %}
gpgkey={{ nexus.gpg.oraclelinux_key }}
sslverify={% if nexus.ssl_verify %}1{% else %}0{% endif %}
{% if nexus.ssl_verify and nexus.ssl_cert_path is defined %}
sslcacert={{ nexus.ssl_cert_path }}
{% endif %}
metadata_expire=300
```

### Ubuntu (templates/ubuntu/nexus-sources.list.j2)

```
deb [arch={{ item.value.arch }} {% if nexus.gpg.enabled %}signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg{% endif %} {{ nexus.base_url }}/{{ item.value.path }} {{ ansible_distribution_release }} {{ item.value.components }}
```

## Использование роли

1. Пример playbook:

```yaml
---
- name: Migrate servers to Nexus
  hosts: all
  become: yes
  vars:
    nexus:
      base_url: "https://nexus.corp.com:443/repository"
      ssl_cert_path: "/etc/ssl/certs/corp-ca.pem"
  
  roles:
    - nexus_migration
```

2. Запуск с дополнительными параметрами:

```bash
ansible-playbook -i inventory.yml migrate.yml \
  -e "nexus_base_url=https://nexus.corp.com:443/repository" \
  -e "migration_test_package=wget"
```

## Особенности реализации

1. **Безопасность**: Все подключения идут через HTTPS на порту 443
2. **Гибкость**: Поддержка разных дистрибутивов через переменные
3. **Надежность**: Проверка на каждом этапе миграции
4. **Корпоративная поддержка**: Работа с корпоративными сертификатами
5. **Идемпотентность**: Роль можно запускать многократно без побочных эффектов

Для полной реализации необходимо:
1. Разместить корпоративный CA-сертификат в `files/corporate-ca.pem`
2. Настроить переменные для конкретного окружения
3. Проверить доступность Nexus из целевых систем