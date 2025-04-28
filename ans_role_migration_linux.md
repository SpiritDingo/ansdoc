# Ansible роль для переноса установленных пакетов и сервисов между серверами Linux

Эта роль позволяет мигрировать установленное ПО и сервисы с исходного сервера на новый.

## Структура роли

```
roles/migrate_packages/
├── defaults
│   └── main.yml
├── files
│   └── exclude_packages.list
├── tasks
│   ├── main.yml
│   ├── gather_source.yml
│   ├── transfer_packages.yml
│   ├── transfer_services.yml
│   └── verify_migration.yml
└── templates
    └── service_override.conf.j2
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Основные параметры
source_host: "old-server"           # Имя исходного сервера (из inventory)
package_manager: "auto"            # auto/apt/yum/dnf/zypper
exclude_packages_file: "exclude_packages.list"  # Файл с исключаемыми пакетами

# Настройки переноса
transfer_method: "direct"          # direct/offline (через файл)
offline_cache_dir: "/var/cache/package_migration"
include_services: true             # Переносить ли настройки сервисов
include_repos: true                # Переносить ли репозитории
verify_installation: true          # Проверять ли установку

# Параметры для разных менеджеров пакетов
apt_options:
  include_debs: true
  include_ppas: true

yum_options:
  include_gpgkeys: true
  include_groups: true
```

### tasks/main.yml

```yaml
---
- name: Include package manager detection
  include_tasks: detect_pkg_manager.yml
  when: package_manager == 'auto'

- name: Gather source server data
  include_tasks: gather_source.yml
  delegate_to: "{{ source_host }}"
  run_once: true

- name: Transfer and install packages
  include_tasks: transfer_packages.yml

- name: Transfer and configure services
  include_tasks: transfer_services.yml
  when: include_services

- name: Verify migration
  include_tasks: verify_migration.yml
  when: verify_installation
```

### tasks/detect_pkg_manager.yml

```yaml
---
- name: Detect package manager
  setup:
    gather_subset:
      - '!all'
      - 'pkg_mgr'
  delegate_to: "{{ source_host }}"
  run_once: true
  register: pkg_mgr_facts

- name: Set detected package manager
  set_fact:
    package_manager: "{{ ansible_pkg_mgr }}"
  when: package_manager == 'auto'
```

### tasks/gather_source.yml

```yaml
---
- name: Gather installed packages
  command: |
    {% if package_manager == 'apt' %}
    apt list --installed | awk -F/ '{print $1}'
    {% elif package_manager in ['yum', 'dnf'] %}
    rpm -qa --queryformat '%{NAME}\n'
    {% elif package_manager == 'zypper' %}
    rpm -qa --queryformat '%{NAME}\n'
    {% endif %}
  register: installed_packages
  changed_when: false

- name: Filter excluded packages
  set_fact:
    packages_to_transfer: |
      {{ installed_packages.stdout_lines | 
         difference(lookup('file', exclude_packages_file).split('\n')) |
         reject('match', '^\\s*$') |
         reject('match', '^#') |
         list }}

- name: Gather enabled services
  command: systemctl list-unit-files --state=enabled --no-legend | awk '{print $1}'
  register: enabled_services
  when: include_services
  changed_when: false

- name: Gather repository list
  command: |
    {% if package_manager == 'apt' %}
    find /etc/apt/sources.list.d/ -name '*.list' -exec cat {} \;
    {% elif package_manager in ['yum', 'dnf'] %}
    yum repolist enabled -v
    {% elif package_manager == 'zypper' %}
    zypper repos -E
    {% endif %}
  register: repository_list
  when: include_repos
  changed_when: false
```

### tasks/transfer_packages.yml

```yaml
---
- name: Install required packages for migration
  package:
    name:
      - "sshpass"  # Для передачи между серверами
      - "rsync"    # Для копирования файлов
    state: present

- name: Create offline cache directory
  file:
    path: "{{ offline_cache_dir }}"
    state: directory
    mode: '0755'
  when: transfer_method == 'offline'

- name: Download packages for offline transfer (APT)
  apt:
    name: "{{ packages_to_transfer }}"
    state: present
    download_only: yes
    install_recommends: no
    dpkg_options: 'force-confdef,force-confold'
    dest: "{{ offline_cache_dir }}"
  when:
    - package_manager == 'apt'
    - transfer_method == 'offline'
  delegate_to: "{{ source_host }}"

- name: Download packages for offline transfer (YUM/DNF)
  command: |
    dnf download --destdir={{ offline_cache_dir }} {{ packages_to_transfer | join(' ') }}
  when:
    - package_manager in ['yum', 'dnf']
    - transfer_method == 'offline'
  delegate_to: "{{ source_host }}"

- name: Transfer packages (direct method)
  package:
    name: "{{ packages_to_transfer }}"
    state: present
  when: transfer_method == 'direct'

- name: Transfer packages (offline method)
  block:
    - name: Copy downloaded packages
      synchronize:
        src: "{{ offline_cache_dir }}/"
        dest: "{{ offline_cache_dir }}/"
        mode: pull
        rsync_opts:
          - "--progress"
          - "--size-only"

    - name: Install local packages (APT)
      apt:
        deb: "{{ offline_cache_dir }}/*.deb"
        install_recommends: no
        dpkg_options: 'force-confdef,force-confold'
      when: package_manager == 'apt'

    - name: Install local packages (RPM)
      command: |
        rpm -Uvh {{ offline_cache_dir }}/*.rpm --nodeps --force
      when: package_manager in ['yum', 'dnf', 'zypper']
  when: transfer_method == 'offline'
```

### tasks/transfer_services.yml

```yaml
---
- name: Transfer service configurations
  synchronize:
    src: "/etc/systemd/system/{{ item }}.service"
    dest: "/etc/systemd/system/{{ item }}.service"
    mode: pull
  with_items: "{{ enabled_services.stdout_lines }}"
  ignore_errors: yes
  register: service_transfers

- name: Create service override directories
  file:
    path: "/etc/systemd/system/{{ item.item }}.d"
    state: directory
    mode: '0755'
  with_items: "{{ service_transfers.results }}"
  when: item is success

- name: Transfer service overrides
  template:
    src: "service_override.conf.j2"
    dest: "/etc/systemd/system/{{ item.item }}.d/override.conf"
  with_items: "{{ service_transfers.results }}"
  when: item is success

- name: Reload systemd
  systemd:
    daemon_reload: yes

- name: Enable transferred services
  systemd:
    name: "{{ item }}"
    enabled: yes
    state: started
  with_items: "{{ enabled_services.stdout_lines }}"
```

### tasks/verify_migration.yml

```yaml
---
- name: Verify installed packages
  command: |
    {% if package_manager == 'apt' %}
    dpkg -l | grep '^ii' | awk '{print $2}'
    {% elif package_manager in ['yum', 'dnf'] %}
    rpm -qa --queryformat '%{NAME}\n'
    {% elif package_manager == 'zypper' %}
    rpm -qa --queryformat '%{NAME}\n'
    {% endif %}
  register: installed_on_target
  changed_when: false

- name: Check missing packages
  debug:
    msg: "Missing packages: {{ packages_to_transfer | difference(installed_on_target.stdout_lines) }}"
    level: warning
  when: packages_to_transfer | difference(installed_on_target.stdout_lines) | length > 0

- name: Check enabled services
  command: systemctl list-unit-files --state=enabled --no-legend | awk '{print $1}'
  register: enabled_on_target
  changed_when: false

- name: Check missing services
  debug:
    msg: "Missing enabled services: {{ enabled_services.stdout_lines | difference(enabled_on_target.stdout_lines) }}"
    level: warning
  when: enabled_services.stdout_lines | difference(enabled_on_target.stdout_lines) | length > 0
```

### templates/service_override.conf.j2

```ini
[Service]
# Ansible managed - migrated from {{ source_host }}
Restart=always
```

### files/exclude_packages.list

```
# Системные пакеты, которые не нужно переносить
linux-image-*
linux-headers-*
linux-modules-*
kernel-*
*-firmware
*-utils
```

## Пример использования

1. Создайте playbook `migrate_server.yml`:

```yaml
---
- hosts: new-server
  become: yes
  roles:
    - migrate_packages
  vars:
    source_host: "old-server"
    package_manager: "apt"
    exclude_packages_file: "custom_excludes.list"  # Ваш собственный файл исключений
```

2. Запустите playbook:
```bash
ansible-playbook -i inventory migrate_server.yml
```

## Особенности

- Поддержка основных менеджеров пакетов (APT, YUM/DNF, Zypper)
- Два метода переноса: прямой и через оффлайн-кэш
- Перенос конфигураций сервисов и их состояний
- Проверка результатов миграции
- Гибкая система исключений пакетов
- Автоматическое определение менеджера пакетов

Для работы с другими дистрибутивами или специфическими пакетами можно расширить роль, добавив соответствующие задачи и фильтры.