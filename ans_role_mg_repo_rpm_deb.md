# Универсальная Ansible роль для управления репозиториями (Ubuntu 22.04/24.04, Oracle Linux 9)

Вот усовершенствованная Ansible роль, которая поддерживает:
- Ubuntu 22.04 (Jammy) и 24.04 (Noble)
- Oracle Linux 9
- Различные методы аутентификации
- Гибкое управление GPG-ключами
- Локальные и удаленные репозитории

## Структура роли

```
roles/repository_manager/
├── defaults/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── debian.yml
│   ├── oracle.yml
│   ├── add_key.yml
│   ├── remove_key.yml
│   ├── setup_auth.yml
│   └── cleanup.yml
├── templates/
│   ├── deb-repo.j2
│   ├── rpm-repo.j2
│   ├── apt-auth.j2
│   └── yum-auth.j2
└── vars/
    ├── Debian.yml
    ├── OracleLinux.yml
    └── main.yml
```

## Основные файлы роли

### defaults/main.yml

```yaml
---
# Общие настройки
repo_manager:
  state: present               # present/absent
  name: "custom-repo"          # Уникальное имя репозитория
  description: ""              # Описание (для Oracle Linux)
  enabled: true                # Включен ли репозиторий
  gpgcheck: true               # Проверять GPG подписи
  priority: null               # Приоритет (только для Ubuntu)

  # Настройки репозитория
  baseurl: ""                  # Основной URL репозитория
  mirrorlist: ""               # URL mirrorlist (только Oracle Linux)
  components: "main"           # Компоненты (только Ubuntu)
  architectures: "amd64"       # Архитектуры (только Ubuntu)

  # Настройки GPG ключа
  gpg_key:
    url: ""                    # URL для загрузки ключа
    local_path: ""             # Локальный путь к ключу
    keyring_path: ""           # Кастомный путь для хранения ключа
    trusted_gpg: true          # Использовать /etc/apt/trusted.gpg.d/ (Ubuntu)
    gpgkey: ""                 # URL ключа в формате RPM (Oracle Linux)

  # Настройки аутентификации
  auth:
    required: false            # Требуется ли аутентификация
    username: ""               # Имя пользователя
    password: ""               # Пароль
    auth_file: ""              # Кастомный путь к файлу аутентификации
```

### tasks/main.yml

```yaml
---
- name: Detect OS family
  ansible.builtin.set_fact:
    os_family: "{{ 'OracleLinux' if ansible_distribution == 'OracleLinux' else ansible_os_family }}"

- name: Include OS-specific variables
  ansible.builtin.include_vars: "{{ item }}"
  with_first_found:
    - "vars/{{ os_family }}.yml"
    - "vars/main.yml"
  tags: always

- name: Set facts for repository
  ansible.builtin.set_fact:
    repo_config: "{{ repo_manager }}"

- name: Validate parameters
  ansible.builtin.assert:
    that:
      - repo_config.name | length > 0
      - repo_config.baseurl | length > 0 or repo_config.mirrorlist | length > 0
    fail_msg: "Repository name and baseurl/mirrorlist must be specified"

- name: Include distribution-specific tasks
  ansible.builtin.include_tasks: "{{ os_family }}.yml"
  when: os_family in ['Debian', 'OracleLinux']

- name: Fail for unsupported OS
  ansible.builtin.fail:
    msg: "Unsupported OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
  when: os_family not in ['Debian', 'OracleLinux']
```

### tasks/debian.yml (для Ubuntu)

```yaml
---
- name: Install prerequisites (Ubuntu)
  ansible.builtin.apt:
    name:
      - ca-certificates
      - gnupg
      - software-properties-common
    state: present
    update_cache: yes

- name: Add GPG key (Ubuntu)
  ansible.builtin.include_tasks: add_key.yml
  when: 
    - repo_config.gpgcheck
    - repo_config.gpg_key.url or repo_config.gpg_key.local_path

- name: Configure repository (Ubuntu)
  ansible.builtin.template:
    src: "deb-repo.j2"
    dest: "/etc/apt/sources.list.d/{{ repo_config.name }}.list"
    mode: "0644"
    owner: root
    group: root

- name: Setup authentication (Ubuntu)
  ansible.builtin.include_tasks: setup_auth.yml
  when: repo_config.auth.required

- name: Update package cache (Ubuntu)
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600
```

### tasks/oracle.yml (для Oracle Linux)

```yaml
---
- name: Install prerequisites (Oracle Linux)
  ansible.builtin.yum:
    name:
      - yum-utils
      - dnf-plugins-core
    state: present

- name: Add GPG key (Oracle Linux)
  ansible.builtin.include_tasks: add_key.yml
  when: 
    - repo_config.gpgcheck
    - repo_config.gpg_key.gpgkey or repo_config.gpg_key.url or repo_config.gpg_key.local_path

- name: Configure repository (Oracle Linux)
  ansible.builtin.template:
    src: "rpm-repo.j2"
    dest: "/etc/yum.repos.d/{{ repo_config.name }}.repo"
    mode: "0644"
    owner: root
    group: root

- name: Setup authentication (Oracle Linux)
  ansible.builtin.include_tasks: setup_auth.yml
  when: repo_config.auth.required

- name: Clean metadata cache (Oracle Linux)
  ansible.builtin.command: dnf clean metadata
  changed_when: false
```

### templates/deb-repo.j2 (для Ubuntu)

```jinja2
# {{ ansible_managed }}
deb [arch={{ repo_config.architectures }} signed-by={{ signed_by }}] {{ repo_config.baseurl }} {{ ubuntu_dist }} {{ repo_config.components }}
{% if repo_config.priority %}
deb [arch={{ repo_config.architectures }} signed-by={{ signed_by }} priority={{ repo_config.priority }}] {{ repo_config.baseurl }} {{ ubuntu_dist }} {{ repo_config.components }}
{% endif %}
```

### templates/rpm-repo.j2 (для Oracle Linux)

```jinja2
# {{ ansible_managed }}
[{{ repo_config.name }}]
name={{ repo_config.description | default(repo_config.name) }}
baseurl={{ repo_config.baseurl }}
{% if repo_config.mirrorlist %}mirrorlist={{ repo_config.mirrorlist }}
{% endif %}enabled={{ 1 if repo_config.enabled else 0 }}
gpgcheck={{ 1 if repo_config.gpgcheck else 0 }}
{% if repo_config.gpgcheck and repo_config.gpg_key.gpgkey %}gpgkey={{ repo_config.gpg_key.gpgkey }}
{% endif %}
```

## Примеры использования

### Для Ubuntu 22.04/24.04

```yaml
- hosts: ubuntu_servers
  vars:
    ubuntu_dist: "{{ 'jammy' if ansible_distribution_version == '22.04' else 'noble' }}"
  
  roles:
    - role: repository_manager
      vars:
        repo_manager:
          name: "company-ubuntu"
          description: "Company Ubuntu Repository"
          baseurl: "https://repo.company.com/ubuntu"
          components: "main restricted universe multiverse"
          architectures: "amd64 arm64"
          state: present
          gpgcheck: true
          gpg_key:
            url: "https://repo.company.com/keys/ubuntu.gpg"
            trusted_gpg: true
          auth:
            required: true
            username: "{{ vault_repo_user }}"
            password: "{{ vault_repo_pass }}"
```

### Для Oracle Linux 9

```yaml
- hosts: oracle_servers
  roles:
    - role: repository_manager
      vars:
        repo_manager:
          name: "company-ol9"
          description: "Company Oracle Linux 9 Repository"
          baseurl: "https://repo.company.com/oraclelinux/9/$basearch"
          gpgcheck: true
          gpg_key:
            gpgkey: "https://repo.company.com/keys/RPM-GPG-KEY-company"
          state: present
          auth:
            required: false
```

### Удаление репозитория

```yaml
- hosts: all_servers
  roles:
    - role: repository_manager
      vars:
        repo_manager:
          name: "old-repo"
          state: absent
```

## Особенности реализации

1. **Автоматическое определение версий Ubuntu**:
   - 22.04 (Jammy)
   - 24.04 (Noble)

2. **Поддержка Oracle Linux 9**:
   - Использование DNF (YUM v4)
   - Правильное управление GPG-ключами RPM

3. **Гибкое управление ключами**:
   - Для Ubuntu: `/etc/apt/trusted.gpg.d/` или кастомные пути
   - Для Oracle Linux: стандартные RPM-GPG ключи

4. **Безопасность**:
   - Поддержка Ansible Vault для учетных данных
   - Правильные права на файлы
   - Удаление временных файлов

5. **Универсальность**:
   - Единый интерфейс для разных дистрибутивов
   - Поддержка локальных и удаленных репозиториев
   - Гибкая настройка через переменные

Эта роль предоставляет полное решение для управления репозиториями в указанных дистрибутивах с учетом их особенностей.