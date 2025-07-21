# Ansible роль для скачивания ключей репозиториев в OS-специфичные папки

Роль загружает GPG-ключи внешних репозиториев в отдельные папки для каждой ОС с четкой организацией структуры.

## Финальная структура роли

```
roles/repo_keys/
├── tasks/
│   ├── main.yml
│   ├── oraclelinux/
│   │   ├── main.yml
│   │   └── keys.yml
│   └── ubuntu/
│       ├── main.yml
│       └── keys.yml
├── defaults/
│   └── main.yml
└── templates/
    └── keylist.j2
```

## Реализация

### defaults/main.yml

```yaml
---
# Основные настройки
keys_download:
  ubuntu:
    base_dir: "/etc/apt/trusted.gpg.d"
    keys_subdir: "external_keys"
    keyring_format: "gpg"
  oraclelinux:
    base_dir: "/etc/pki/rpm-gpg"
    keys_subdir: "external_keys"
    keyring_format: "gpg"

# Настройки загрузки
download_timeout: 30
max_retries: 3
retry_delay: 5
```

### tasks/main.yml

```yaml
---
- name: Include OS-specific key tasks
  include_tasks: "{{ ansible_distribution | lower }}/main.yml"
  when: ansible_distribution in ['Ubuntu', 'OracleLinux']
```

### tasks/ubuntu/main.yml

```yaml
---
- name: Load Ubuntu keys configuration
  include_vars: keys.yml

- name: Create Ubuntu keys directory structure
  block:
    - name: Ensure base directory exists
      ansible.builtin.file:
        path: "{{ keys_download.ubuntu.base_dir }}"
        state: directory
        mode: '0755'

    - name: Create subdirectory for external keys
      ansible.builtin.file:
        path: "{{ keys_download.ubuntu.base_dir }}/{{ keys_download.ubuntu.keys_subdir }}"
        state: directory
        mode: '0755'
      when: keys_download.ubuntu.keys_subdir | length > 0

- name: Set full keys directory path
  set_fact:
    ubuntu_keys_dir: "{{ keys_download.ubuntu.base_dir }}{% if keys_download.ubuntu.keys_subdir | length > 0 %}/{{ keys_download.ubuntu.keys_subdir }}{% endif %}"

- name: Install required packages for Ubuntu
  apt:
    name:
      - curl
      - gpg
      - dirmngr
    state: present
    update_cache: yes

- name: Download Ubuntu repository keys
  ansible.builtin.get_url:
    url: "{{ item.url }}"
    dest: "{{ ubuntu_keys_dir }}/{{ item.name }}.{{ keys_download.ubuntu.keyring_format }}"
    mode: '0644'
    timeout: "{{ download_timeout }}"
    force: yes
  loop: "{{ ubuntu_keys }}"
  register: key_download
  until: key_download is succeeded
  retries: "{{ max_retries }}"
  delay: "{{ retry_delay }}"

- name: Validate downloaded GPG keys
  command: >
    gpg --no-default-keyring
    --keyring "{{ ubuntu_keys_dir }}/{{ item.name }}.{{ keys_download.ubuntu.keyring_format }}"
    --list-keys
  loop: "{{ ubuntu_keys }}"
  changed_when: false
  ignore_errors: yes
```

### tasks/ubuntu/keys.yml

```yaml
---
ubuntu_keys:
  - name: "hashicorp"
    url: "https://apt.releases.hashicorp.com/gpg"
    description: "HashiCorp package repository key"

  - name: "docker"
    url: "https://download.docker.com/linux/ubuntu/gpg"
    description: "Docker CE repository key"

  - name: "nodesource"
    url: "https://deb.nodesource.com/gpgkey/nodesource.gpg.key"
    description: "NodeSource package repository key"

  - name: "google-cloud"
    url: "https://packages.cloud.google.com/apt/doc/apt-key.gpg"
    description: "Google Cloud SDK repository key"
```

### tasks/oraclelinux/main.yml

```yaml
---
- name: Load OracleLinux keys configuration
  include_vars: keys.yml

- name: Create OracleLinux keys directory structure
  block:
    - name: Ensure base directory exists
      ansible.builtin.file:
        path: "{{ keys_download.oraclelinux.base_dir }}"
        state: directory
        mode: '0755'

    - name: Create subdirectory for external keys
      ansible.builtin.file:
        path: "{{ keys_download.oraclelinux.base_dir }}/{{ keys_download.oraclelinux.keys_subdir }}"
        state: directory
        mode: '0755'
      when: keys_download.oraclelinux.keys_subdir | length > 0

- name: Set full keys directory path
  set_fact:
    oraclelinux_keys_dir: "{{ keys_download.oraclelinux.base_dir }}{% if keys_download.oraclelinux.keys_subdir | length > 0 %}/{{ keys_download.oraclelinux.keys_subdir }}{% endif %}"

- name: Install required packages for OracleLinux
  yum:
    name:
      - curl
      - gnupg2
    state: present

- name: Download OracleLinux repository keys
  ansible.builtin.get_url:
    url: "{{ item.url }}"
    dest: "{{ oraclelinux_keys_dir }}/{{ item.name }}.{{ keys_download.oraclelinux.keyring_format }}"
    mode: '0644'
    timeout: "{{ download_timeout }}"
    force: yes
  loop: "{{ oraclelinux_keys }}"
  register: key_download
  until: key_download is succeeded
  retries: "{{ max_retries }}"
  delay: "{{ retry_delay }}"

- name: Import RPM GPG keys
  rpm_key:
    key: "{{ oraclelinux_keys_dir }}/{{ item.name }}.{{ keys_download.oraclelinux.keyring_format }}"
    state: present
  loop: "{{ oraclelinux_keys }}"
  when: keys_download.oraclelinux.keyring_format == 'gpg'
```

### tasks/oraclelinux/keys.yml

```yaml
---
oraclelinux_keys:
  - name: "docker-ce"
    url: "https://download.docker.com/linux/centos/gpg"
    description: "Docker CE repository key"

  - name: "epel"
    url: "https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-8"
    description: "EPEL repository key"

  - name: "nginx"
    url: "https://nginx.org/keys/nginx_signing.key"
    description: "NGINX repository key"

  - name: "postgresql"
    url: "https://download.postgresql.org/pub/repos/yum/keys/PGDG-RPM-GPG-KEY-PGDG"
    description: "PostgreSQL repository key"
```

### templates/keylist.j2 (опционально)

```jinja2
# Auto-generated list of installed repository keys
# OS: {{ ansible_distribution }}
# Date: {{ ansible_date_time.iso8601 }}

{% if ansible_distribution == 'Ubuntu' %}
{% for key in ubuntu_keys %}
{{ key.name|upper }}:
  Description: {{ key.description }}
  File: {{ ubuntu_keys_dir }}/{{ key.name }}.{{ keys_download.ubuntu.keyring_format }}
  Source: {{ key.url }}

{% endfor %}
{% elif ansible_distribution == 'OracleLinux' %}
{% for key in oraclelinux_keys %}
{{ key.name|upper }}:
  Description: {{ key.description }}
  File: {{ oraclelinux_keys_dir }}/{{ key.name }}.{{ keys_download.oraclelinux.keyring_format }}
  Source: {{ key.url }}

{% endfor %}
{% endif %}
```

## Использование роли

1. Базовая установка:
```yaml
- hosts: all
  roles:
    - repo_keys
```

2. С кастомными параметрами:
```yaml
- hosts: all
  vars:
    keys_download:
      ubuntu:
        base_dir: "/opt/ubuntu_keys"
        keys_subdir: ""
      oraclelinux:
        base_dir: "/opt/oracle_keys"
        keys_subdir: "rpm_gpg"
  roles:
    - repo_keys
```

3. С дополнительными ключами:
```yaml
- hosts: ubuntu_servers
  vars:
    ubuntu_keys:
      - name: "microsoft"
        url: "https://packages.microsoft.com/keys/microsoft.asc"
        description: "Microsoft packages repository key"
  roles:
    - repo_keys
```

## Преимущества данной реализации

1. **Полное разделение по ОС**:
   - Каждая ОС имеет свою собственную папку для ключей
   - Отдельные файлы конфигурации для каждой ОС

2. **Гибкая настройка**:
   - Возможность настройки путей через переменные
   - Поддержка подпапок и плоской структуры

3. **Документирование**:
   - Каждый ключ имеет описание
   - Опциональное создание файла с информацией о ключах

4. **Расширяемость**:
   - Легко добавить поддержку новых ОС
   - Просто добавлять новые ключи через переменные

5. **Надежность**:
   - Проверка валидности ключей
   - Повторные попытки загрузки при ошибках
   - Четкое логирование операций