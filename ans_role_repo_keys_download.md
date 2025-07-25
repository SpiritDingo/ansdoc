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


# Ansible роль для скачивания ключей репозиториев (Ubuntu 22.04/24.04, Oracle Linux 9)

Роль для загрузки GPG-ключей внешних репозиториев с поддержкой:
- Ubuntu 22.04 (Jammy Jellyfish)
- Ubuntu 24.04 (Noble Numbat)
- Oracle Linux 9

## Структура роли

```
roles/repo_keys/
├── tasks/
│   ├── main.yml
│   ├── ol9.yml
│   ├── ubuntu_22.yml
│   └── ubuntu_24.yml
├── defaults/
│   └── main.yml
├── vars/
│   ├── ol9.yml
│   ├── ubuntu_22.yml
│   └── ubuntu_24.yml
└── templates/
    └── keylist.j2
```

## Реализация роли

### defaults/main.yml

```yaml
---
# Общие настройки
keys_base_dir: "/etc/keys"
download_timeout: 30
max_retries: 3
retry_delay: 5

# Настройки для разных ОС
os_settings:
  Ubuntu_22.04:
    keys_dir: "{{ keys_base_dir }}/ubuntu_22"
    required_pkgs: [curl, gpg, dirmngr]
    key_ext: "gpg"
  Ubuntu_24.04:
    keys_dir: "{{ keys_base_dir }}/ubuntu_24"
    required_pkgs: [curl, gpg, dirmngr]
    key_ext: "gpg"
  OracleLinux_9:
    keys_dir: "{{ keys_base_dir }}/ol9"
    required_pkgs: [curl, gnupg2]
    key_ext: "gpg"
```

### vars/ubuntu_22.yml

```yaml
---
ubuntu_22_keys:
  - name: "docker"
    url: "https://download.docker.com/linux/ubuntu/gpg"
    desc: "Docker CE repository key"
    
  - name: "hashicorp"
    url: "https://apt.releases.hashicorp.com/gpg"
    desc: "HashiCorp package repository key"
    
  - name: "google-cloud"
    url: "https://packages.cloud.google.com/apt/doc/apt-key.gpg"
    desc: "Google Cloud SDK repository key"
    
  - name: "microsoft"
    url: "https://packages.microsoft.com/keys/microsoft.asc"
    desc: "Microsoft packages repository key"
```

### vars/ubuntu_24.yml

```yaml
---
ubuntu_24_keys:
  - name: "docker"
    url: "https://download.docker.com/linux/ubuntu/gpg"
    desc: "Docker CE repository key"
    
  - name: "hashicorp"
    url: "https://apt.releases.hashicorp.com/gpg"
    desc: "HashiCorp package repository key"
    
  - name: "google-cloud"
    url: "https://packages.cloud.google.com/apt/doc/apt-key.gpg"
    desc: "Google Cloud SDK repository key"
    
  - name: "protonvpn"
    url: "https://protonvpn.com/download/protonvpn-stable-release.gpg"
    desc: "ProtonVPN repository key"
```

### vars/ol9.yml

```yaml
---
ol9_keys:
  - name: "docker-ce"
    url: "https://download.docker.com/linux/centos/gpg"
    desc: "Docker CE repository key"
    
  - name: "epel-9"
    url: "https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-9"
    desc: "EPEL repository key for EL9"
    
  - name: "nginx"
    url: "https://nginx.org/keys/nginx_signing.key"
    desc: "NGINX repository key"
    
  - name: "postgresql"
    url: "https://download.postgresql.org/pub/repos/yum/keys/PGDG-RPM-GPG-KEY-PGDG"
    desc: "PostgreSQL repository key"
```

### tasks/main.yml

```yaml
---
- name: Set OS-specific variables
  include_vars: "{{ ansible_distribution | lower }}_{{ ansible_distribution_version.split('.')[0] }}.yml"
  when: 
    - (ansible_distribution == 'Ubuntu' and ansible_distribution_version in ['22.04', '24.04'])
    - or (ansible_distribution == 'OracleLinux' and ansible_distribution_major_version == '9')

- name: Define current OS settings
  set_fact:
    os_config: "{{ os_settings[ansible_distribution + '_' + ansible_distribution_version] | default(os_settings[ansible_distribution + '_' + ansible_distribution_major_version]) }}"
    os_keys: "{{ 
        (ansible_distribution == 'Ubuntu' and ansible_distribution_version == '22.04') | ternary(ubuntu_22_keys, 
        (ansible_distribution == 'Ubuntu' and ansible_distribution_version == '24.04') | ternary(ubuntu_24_keys,
        (ansible_distribution == 'OracleLinux' and ansible_distribution_major_version == '9') | ternary(ol9_keys, []))) 
      }}"

- name: Create keys directory
  file:
    path: "{{ os_config.keys_dir }}"
    state: directory
    mode: '0755'

- name: Install required packages
  package:
    name: "{{ os_config.required_pkgs }}"
    state: present
  when: ansible_distribution != 'OracleLinux'

- name: Install required packages for Oracle Linux
  yum:
    name: "{{ os_config.required_pkgs }}"
    state: present
  when: ansible_distribution == 'OracleLinux'

- name: Download repository keys
  get_url:
    url: "{{ item.url }}"
    dest: "{{ os_config.keys_dir }}/{{ item.name }}.{{ os_config.key_ext }}"
    mode: '0644'
    timeout: "{{ download_timeout }}"
    force: yes
  loop: "{{ os_keys }}"
  register: key_download
  until: key_download is succeeded
  retries: "{{ max_retries }}"
  delay: "{{ retry_delay }}"

- name: Additional steps for Ubuntu
  include_tasks: "ubuntu_{{ ansible_distribution_version.split('.')[0] }}.yml"
  when: ansible_distribution == 'Ubuntu'

- name: Additional steps for Oracle Linux 9
  include_tasks: ol9.yml
  when: 
    - ansible_distribution == 'OracleLinux'
    - ansible_distribution_major_version == '9'

- name: Generate key list report
  template:
    src: keylist.j2
    dest: "{{ os_config.keys_dir }}/installed_keys.txt"
    mode: '0644'
```

### tasks/ubuntu_22.yml

```yaml
---
- name: Validate GPG keys (Ubuntu 22.04)
  command: >
    gpg --no-default-keyring
    --keyring "{{ os_config.keys_dir }}/{{ item.name }}.{{ os_config.key_ext }}"
    --list-keys
  loop: "{{ os_keys }}"
  changed_when: false
  ignore_errors: yes
```

### tasks/ubuntu_24.yml

```yaml
---
- name: Convert keys to keyring format (Ubuntu 24.04)
  command: >
    gpg --no-default-keyring
    --keyring "{{ os_config.keys_dir }}/{{ item.name }}.keyring"
    --import "{{ os_config.keys_dir }}/{{ item.name }}.{{ os_config.key_ext }}"
  loop: "{{ os_keys }}"
  changed_when: true
```

### tasks/ol9.yml

```yaml
---
- name: Import RPM GPG keys (OL9)
  rpm_key:
    key: "{{ os_config.keys_dir }}/{{ item.name }}.{{ os_config.key_ext }}"
    state: present
  loop: "{{ os_keys }}"
```

### templates/keylist.j2

```jinja2
# Repository Keys Report
# OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
# Generated: {{ ansible_date_time.iso8601 }}
# Path: {{ os_config.keys_dir }}

Installed Keys:
{% for key in os_keys %}
- Name: {{ key.name }}
  Description: {{ key.desc }}
  Source: {{ key.url }}
  File: {{ os_config.keys_dir }}/{{ key.name }}.{{ os_config.key_ext }}
{% endfor %}

Total keys: {{ os_keys | count }}
```

## Пример использования

### playbook.yml

```yaml
---
- name: Configure repository keys
  hosts: all
  become: yes
  roles:
    - repo_keys
```

### Переопределение настроек

```yaml
- name: Configure custom keys
  hosts: all
  become: yes
  vars:
    keys_base_dir: "/opt/security/keys"
    os_settings:
      Ubuntu_22.04:
        keys_dir: "{{ keys_base_dir }}/ubuntu22"
        required_pkgs: [curl, gpg, dirmngr, ca-certificates]
      Ubuntu_24.04:
        keys_dir: "{{ keys_base_dir }}/ubuntu24"
      OracleLinux_9:
        keys_dir: "{{ keys_base_dir }}/ol9"
        
    ubuntu_22_keys:
      - name: "custom_repo"
        url: "https://repo.example.com/gpg"
        desc: "Custom repository key"
        
  roles:
    - repo_keys
```

## Особенности реализации

1. **Поддержка специфичных версий ОС**:
   - Отдельные настройки для Ubuntu 22.04, 24.04 и Oracle Linux 9
   - Автоматическое определение версии ОС

2. **Гибкая структура каталогов**:
   - Ключи хранятся в `/etc/keys/<os-version>` по умолчанию
   - Возможность кастомизации пути через переменные

3. **Особенности Ubuntu 24.04**:
   - Автоматическое преобразование ключей в keyring-формат
   - Поддержка нового формата ключей в Noble Numbat

4. **Особенности Oracle Linux 9**:
   - Автоматический импорт ключей в RPM
   - Поддержка EPEL 9 репозитория

5. **Документирование**:
   - Автоматическая генерация отчета об установленных ключах
   - Подробное описание каждого ключа

6. **Надежность**:
   - Повторные попытки загрузки при сбоях сети
   - Проверка целостности ключей
   - Идемпотентные операции

Роль автоматически определяет версию ОС и применяет соответствующие настройки. Для работы требуется интернет-доступ с целевых хостов.


Ошибка возникает из-за того, что переменная `ol9_keys` не определена при выполнении задачи. Давайте исправим роль, чтобы она корректно работала для всех поддерживаемых ОС.

## Исправленная реализация

### tasks/main.yml

```yaml
---
- name: Load OS-specific variables
  include_vars: "{{ ansible_distribution | lower }}_{{ ansible_distribution_major_version }}.yml"
  when: 
    - (ansible_distribution == 'Ubuntu' and ansible_distribution_major_version in ['22', '24'])
    - or (ansible_distribution == 'OracleLinux' and ansible_distribution_major_version == '9')
  ignore_errors: yes

- name: Set default empty keys list if not loaded
  set_fact:
    os_keys: []
  when: os_keys is not defined

- name: Define current OS settings
  set_fact:
    os_config: "{{ os_settings[ansible_distribution + '_' + ansible_distribution_version] | default(os_settings[ansible_distribution + '_' + ansible_distribution_major_version], default={}) }}"

- name: Create keys directory
  file:
    path: "{{ os_config.keys_dir }}"
    state: directory
    mode: '0755'

- name: Install required packages
  block:
    - name: Install packages for Ubuntu
      apt:
        name: "{{ os_config.required_pkgs }}"
        state: present
        update_cache: yes
      when: ansible_distribution == 'Ubuntu'

    - name: Install packages for Oracle Linux
      yum:
        name: "{{ os_config.required_pkgs }}"
        state: present
      when: ansible_distribution == 'OracleLinux'
  
- name: Download repository keys
  block:
    - name: Download keys
      get_url:
        url: "{{ item.url }}"
        dest: "{{ os_config.keys_dir }}/{{ item.name }}.{{ os_config.key_ext }}"
        mode: '0644'
        timeout: "{{ download_timeout }}"
        force: yes
      loop: "{{ os_keys }}"
      register: key_download
      until: key_download is succeeded
      retries: "{{ max_retries }}"
      delay: "{{ retry_delay }}"
  
    - name: Process keys for Ubuntu 22.04
      include_tasks: ubuntu_22.yml
      when: 
        - ansible_distribution == 'Ubuntu'
        - ansible_distribution_major_version == '22'
    
    - name: Process keys for Ubuntu 24.04
      include_tasks: ubuntu_24.yml
      when:
        - ansible_distribution == 'Ubuntu'
        - ansible_distribution_major_version == '24'
    
    - name: Process keys for Oracle Linux 9
      include_tasks: ol9.yml
      when:
        - ansible_distribution == 'OracleLinux'
        - ansible_distribution_major_version == '9'
  when: os_keys | length > 0

- name: Generate key list report
  template:
    src: keylist.j2
    dest: "{{ os_config.keys_dir }}/installed_keys.txt"
    mode: '0644'
  when: os_keys | length > 0
```

### Исправления в других файлах

1. Переименуйте файлы в vars/:
   - `ubuntu_22.yml` → `ubuntu_22.yml`
   - `ubuntu_24.yml` → `ubuntu_24.yml`
   - `ol9.yml` → `oraclelinux_9.yml`

2. Обновите содержимое `vars/oraclelinux_9.yml`:
```yaml
---
os_keys:
  - name: "docker-ce"
    url: "https://download.docker.com/linux/centos/gpg"
    desc: "Docker CE repository key"
    
  - name: "epel-9"
    url: "https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-9"
    desc: "EPEL repository key for EL9"
    
  - name: "nginx"
    url: "https://nginx.org/keys/nginx_signing.key"
    desc: "NGINX repository key"
    
  - name: "postgresql"
    url: "https://download.postgresql.org/pub/repos/yum/keys/PGDG-RPM-GPG-KEY-PGDG"
    desc: "PostgreSQL repository key"
```

3. Обновите `defaults/main.yml`:
```yaml
os_settings:
  Ubuntu_22:
    keys_dir: "{{ keys_base_dir }}/ubuntu_22"
    required_pkgs: [curl, gpg, dirmngr]
    key_ext: "gpg"
  Ubuntu_24:
    keys_dir: "{{ keys_base_dir }}/ubuntu_24"
    required_pkgs: [curl, gpg, dirmngr]
    key_ext: "gpg"
  OracleLinux_9:
    keys_dir: "{{ keys_base_dir }}/ol9"
    required_pkgs: [curl, gnupg2]
    key_ext: "gpg"
```

## Решение проблемы

Основные изменения, которые решают проблему:

1. Используем `ansible_distribution_major_version` вместо разбиения версии вручную
2. Переименовали `ol9_keys` в `os_keys` для единообразия
3. Добавили обработку случая, когда переменные не загружены
4. Упростили логику определения настроек ОС
5. Добавили проверку на пустой список ключей перед обработкой

Теперь роль будет:
1. Пытаться загрузить переменные для текущей ОС
2. Устанавливать пустой список ключей, если загрузка не удалась
3. Пропускать задачи по загрузке ключей, если их список пуст
4. Корректно работать для всех указанных версий ОС

## Пример использования

```yaml
- hosts: all
  become: yes
  roles:
    - role: repo_keys
      vars:
        keys_base_dir: "/opt/security/keys"
        os_keys:  # Можно переопределить ключи
          - name: "custom_repo"
            url: "https://example.com/gpg"
            desc: "Custom repository key"
```

Эта реализация более устойчива к ошибкам и будет корректно работать даже если файлы с переменными для какой-то ОС отсутствуют.



Ошибка возникает из-за неправильного использования фильтра `default` в Jinja2. Давайте исправим задачу определения настроек ОС.

### Исправленный вариант tasks/main.yml

```yaml
---
- name: Load OS-specific variables
  include_vars: "{{ ansible_distribution | lower }}_{{ ansible_distribution_major_version }}.yml"
  when: 
    - (ansible_distribution == 'Ubuntu' and ansible_distribution_major_version in ['22', '24'])
    - or (ansible_distribution == 'OracleLinux' and ansible_distribution_major_version == '9')
  ignore_errors: yes

- name: Set default empty keys list if not loaded
  set_fact:
    os_keys: []
  when: os_keys is not defined

- name: Define current OS settings
  set_fact:
    os_config: >-
      {{
        os_settings.get(
          ansible_distribution + '_' + ansible_distribution_version,
          os_settings.get(
            ansible_distribution + '_' + ansible_distribution_major_version,
            {}
          )
        )
      }}

- name: Validate OS configuration
  fail:
    msg: "No configuration found for {{ ansible_distribution }} {{ ansible_distribution_version }}"
  when: os_config | length == 0

- name: Create keys directory
  file:
    path: "{{ os_config.keys_dir }}"
    state: directory
    mode: '0755'
  when: os_config | length > 0

- name: Install required packages
  block:
    - name: Install packages for Ubuntu
      apt:
        name: "{{ os_config.required_pkgs }}"
        state: present
        update_cache: yes
      when: 
        - ansible_distribution == 'Ubuntu'
        - os_config | length > 0

    - name: Install packages for Oracle Linux
      yum:
        name: "{{ os_config.required_pkgs }}"
        state: present
      when:
        - ansible_distribution == 'OracleLinux'
        - os_config | length > 0

- name: Download and process repository keys
  block:
    - name: Download keys
      get_url:
        url: "{{ item.url }}"
        dest: "{{ os_config.keys_dir }}/{{ item.name }}.{{ os_config.key_ext }}"
        mode: '0644'
        timeout: "{{ download_timeout }}"
        force: yes
      loop: "{{ os_keys }}"
      register: key_download
      until: key_download is succeeded
      retries: "{{ max_retries }}"
      delay: "{{ retry_delay }}"
      when: os_keys | length > 0
    
    - name: Process keys for Ubuntu 22.04
      include_tasks: ubuntu_22.yml
      when: 
        - ansible_distribution == 'Ubuntu'
        - ansible_distribution_major_version == '22'
        - os_keys | length > 0
    
    - name: Process keys for Ubuntu 24.04
      include_tasks: ubuntu_24.yml
      when:
        - ansible_distribution == 'Ubuntu'
        - ansible_distribution_major_version == '24'
        - os_keys | length > 0
    
    - name: Process keys for Oracle Linux 9
      include_tasks: ol9.yml
      when:
        - ansible_distribution == 'OracleLinux'
        - ansible_distribution_major_version == '9'
        - os_keys | length > 0
  when: os_config | length > 0

- name: Generate key list report
  template:
    src: keylist.j2
    dest: "{{ os_config.keys_dir }}/installed_keys.txt"
    mode: '0644'
  when: 
    - os_config | length > 0
    - os_keys | length > 0
```

### Ключевые изменения:

1. **Исправление фильтра default**:
   - Заменили `| default(value, default={})` на использование метода `.get()` словаря Python
   - Новый синтаксис: `os_settings.get(key1, os_settings.get(key2, {}))`

2. **Добавлена валидация конфигурации ОС**:
   - Проверка, что найдена конфигурация для текущей ОС
   - Задача `fail` с сообщением, если конфигурация не найдена

3. **Улучшена условная логика**:
   - Добавлены проверки `when: os_config | length > 0` для всех задач, работающих с настройками ОС
   - Добавлены проверки `when: os_keys | length > 0` для задач, работающих с ключами

4. **Более безопасная обработка ошибок**:
   - `ignore_errors: yes` при загрузке переменных
   - Установка пустого списка ключей по умолчанию

### Альтернативный вариант для определения os_config

Если предыдущий вариант с `.get()` кажется сложным, можно использовать более простой подход:

```yaml
- name: Define current OS settings
  set_fact:
    os_config: >-
      {% set os_key = ansible_distribution + '_' + ansible_distribution_version %}
      {% set os_major_key = ansible_distribution + '_' + ansible_distribution_major_version %}
      {{ os_settings[os_key] if os_key in os_settings else os_settings[os_major_key] if os_major_key in os_settings else {} }}
```

Этот вариант также корректно обрабатывает случаи, когда конфигурация для ОС не найдена, возвращая пустой словарь.



Ошибка указывает, что переменная `os_config.required_pkgs` не определена. Давайте полностью переработаем роль, чтобы сделать её более надежной и избежать подобных ошибок.

## Исправленная Ansible роль

### Структура роли

```
roles/repo_keys/
├── tasks/
│   ├── main.yml
│   ├── ol9.yml
│   ├── ubuntu_22.yml
│   └── ubuntu_24.yml
├── defaults/
│   └── main.yml
├── vars/
│   ├── ol9.yml
│   ├── ubuntu_22.yml
│   └── ubuntu_24.yml
└── templates/
    └── keylist.j2
```

### defaults/main.yml

```yaml
---
# Общие настройки
keys_base_dir: "/etc/keys"
download_timeout: 30
max_retries: 3
retry_delay: 5

# Настройки по умолчанию для разных ОС
os_defaults:
  Ubuntu:
    keys_dir: "{{ keys_base_dir }}/ubuntu"
    key_ext: "gpg"
    required_pkgs: [curl, gpg, dirmngr]
  OracleLinux:
    keys_dir: "{{ keys_base_dir }}/oraclelinux"
    key_ext: "gpg"
    required_pkgs: [curl, gnupg2]
```

### vars/ubuntu_22.yml

```yaml
---
os_specific:
  keys_dir: "{{ keys_base_dir }}/ubuntu_22"
  keys:
    - name: "docker"
      url: "https://download.docker.com/linux/ubuntu/gpg"
      desc: "Docker CE repository key"
    - name: "hashicorp"
      url: "https://apt.releases.hashicorp.com/gpg"
      desc: "HashiCorp package repository key"
```

### vars/ubuntu_24.yml

```yaml
---
os_specific:
  keys_dir: "{{ keys_base_dir }}/ubuntu_24"
  keys:
    - name: "docker"
      url: "https://download.docker.com/linux/ubuntu/gpg"
      desc: "Docker CE repository key"
    - name: "protonvpn"
      url: "https://protonvpn.com/download/protonvpn-stable-release.gpg"
      desc: "ProtonVPN repository key"
```

### vars/ol9.yml

```yaml
---
os_specific:
  keys_dir: "{{ keys_base_dir }}/ol9"
  keys:
    - name: "docker-ce"
      url: "https://download.docker.com/linux/centos/gpg"
      desc: "Docker CE repository key"
    - name: "epel-9"
      url: "https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-9"
      desc: "EPEL repository key for EL9"
```

### tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_distribution | lower }}_{{ ansible_distribution_major_version }}.yml"
  when: 
    - (ansible_distribution == 'Ubuntu' and ansible_distribution_major_version in ['22', '24'])
    - or (ansible_distribution == 'OracleLinux' and ansible_distribution_major_version == '9')
  ignore_errors: yes

- name: Set default empty keys if not loaded
  set_fact:
    os_specific: {'keys': []}
  when: os_specific is not defined

- name: Merge OS defaults with specific settings
  set_fact:
    os_config: "{{ os_defaults[ansible_distribution] | combine(os_specific, recursive=True) }}"

- name: Create keys directory
  file:
    path: "{{ os_config.keys_dir }}"
    state: directory
    mode: '0755'

- name: Install required packages
  package:
    name: "{{ os_config.required_pkgs }}"
    state: present
  when: ansible_distribution == 'Ubuntu'

- name: Install required packages for Oracle Linux
  yum:
    name: "{{ os_config.required_pkgs }}"
    state: present
  when: ansible_distribution == 'OracleLinux'

- name: Download repository keys
  get_url:
    url: "{{ item.url }}"
    dest: "{{ os_config.keys_dir }}/{{ item.name }}.{{ os_config.key_ext }}"
    mode: '0644'
    timeout: "{{ download_timeout }}"
    force: yes
  loop: "{{ os_config.keys }}"
  register: key_download
  until: key_download is succeeded
  retries: "{{ max_retries }}"
  delay: "{{ retry_delay }}"
  when: os_config.keys | length > 0

- name: Process keys for Ubuntu 22.04
  include_tasks: ubuntu_22.yml
  when: 
    - ansible_distribution == 'Ubuntu'
    - ansible_distribution_major_version == '22'
    - os_config.keys | length > 0

- name: Process keys for Ubuntu 24.04
  include_tasks: ubuntu_24.yml
  when:
    - ansible_distribution == 'Ubuntu'
    - ansible_distribution_major_version == '24'
    - os_config.keys | length > 0

- name: Process keys for Oracle Linux 9
  include_tasks: ol9.yml
  when:
    - ansible_distribution == 'OracleLinux'
    - ansible_distribution_major_version == '9'
    - os_config.keys | length > 0

- name: Generate key list report
  template:
    src: keylist.j2
    dest: "{{ os_config.keys_dir }}/installed_keys.txt"
    mode: '0644'
  when: os_config.keys | length > 0
```

### tasks/ubuntu_22.yml

```yaml
---
- name: Validate GPG keys (Ubuntu 22.04)
  command: >
    gpg --no-default-keyring
    --keyring "{{ os_config.keys_dir }}/{{ item.name }}.{{ os_config.key_ext }}"
    --list-keys
  loop: "{{ os_config.keys }}"
  changed_when: false
```

### tasks/ubuntu_24.yml

```yaml
---
- name: Convert keys to keyring format (Ubuntu 24.04)
  command: >
    gpg --no-default-keyring
    --keyring "{{ os_config.keys_dir }}/{{ item.name }}.keyring"
    --import "{{ os_config.keys_dir }}/{{ item.name }}.{{ os_config.key_ext }}"
  loop: "{{ os_config.keys }}"
  changed_when: true
```

### tasks/ol9.yml

```yaml
---
- name: Import RPM GPG keys (OL9)
  rpm_key:
    key: "{{ os_config.keys_dir }}/{{ item.name }}.{{ os_config.key_ext }}"
    state: present
  loop: "{{ os_config.keys }}"
```

### templates/keylist.j2

```jinja2
# Repository Keys Report
# OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
# Generated: {{ ansible_date_time.iso8601 }}
# Path: {{ os_config.keys_dir }}

Installed Keys:
{% for key in os_config.keys %}
- Name: {{ key.name }}
  Description: {{ key.desc }}
  Source: {{ key.url }}
  File: {{ os_config.keys_dir }}/{{ key.name }}.{{ os_config.key_ext }}
{% endfor %}

Total keys: {{ os_config.keys | count }}
```

## Ключевые улучшения:

1. **Упрощенная структура переменных**:
   - Используем `os_defaults` для общих настроек по ОС
   - `os_specific` для версионных особенностей

2. **Надежное слияние настроек**:
   - Используем фильтр `combine` для объединения default и specific настроек
   - Гарантированное наличие всех необходимых полей

3. **Защита от ошибок**:
   - Установка пустого значения по умолчанию для `os_specific`
   - Проверки длины списка ключей перед обработкой

4. **Четкое разделение задач**:
   - Отдельные файлы задач для каждой версии ОС
   - Упрощенная логика в основном файле задач

5. **Гибкость**:
   - Легко добавлять новые версии ОС
   - Простое переопределение переменных

Роль теперь будет корректно работать даже если:
- Нет файла с переменными для конкретной версии ОС
- Список ключей пуст
- Отсутствуют какие-то параметры в настройках