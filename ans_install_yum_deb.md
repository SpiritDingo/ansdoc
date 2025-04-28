# Ansible роль для установки пакетов (yum/deb) на любые серверы

Универсальная роль для управления пакетами на серверах с разными дистрибутивами (RHEL/CentOS и Debian/Ubuntu).

## Структура роли

```
roles/
  universal_packages/
    ├── defaults/
    │   └── main.yml
    ├── tasks/
    │   ├── main.yml
    │   ├── debian.yml
    │   └── redhat.yml
    ├── vars/
    │   └── main.yml
    └── README.md
```

## 1. defaults/main.yml (переменные по умолчанию)

```yaml
---
# Общие параметры
package_manager_autoremove: no
package_manager_update_cache: yes
package_manager_clean_cache: yes

# Пакеты для установки (общие для всех ОС)
common_packages:
  - curl
  - wget
  - git
  - htop
  - vim

# Пакеты для установки (специфичные для ОС)
os_specific_packages:
  debian:
    - apt-transport-https
  redhat:
    - yum-utils

# Пакеты для удаления
packages_to_remove: []

# Обновлять ли все пакеты
full_upgrade: no

# Игнорировать ошибки при установке
ignore_package_errors: no
```

## 2. tasks/main.yml (основной файл задач)

```yaml
---
- name: Определить дистрибутив ОС
  ansible.builtin.setup:
    filter: ansible_distribution*
  tags: always

- name: Включить задачи для Debian/Ubuntu
  ansible.builtin.include_tasks: debian.yml
  when: ansible_distribution in ['Debian', 'Ubuntu']

- name: Включить задачи для RHEL/CentOS
  ansible.builtin.include_tasks: redhat.yml
  when: ansible_distribution in ['RedHat', 'CentOS', 'Fedora', 'Amazon', 'OracleLinux']

- name: Вывести предупреждение для неподдерживаемых ОС
  ansible.builtin.debug:
    msg: "Неподдерживаемый дистрибутив {{ ansible_distribution }}"
  when: >
    ansible_distribution not in ['Debian', 'Ubuntu', 'RedHat', 'CentOS', 'Fedora', 'Amazon', 'OracleLinux']
  tags: always
```

## 3. tasks/debian.yml (задачи для Debian/Ubuntu)

```yaml
---
- name: Обновить индекс пакетов (APT)
  ansible.builtin.apt:
    update_cache: "{{ package_manager_update_cache }}"
    cache_valid_time: 3600

- name: Установить общие пакеты (Debian)
  ansible.builtin.apt:
    name: "{{ common_packages + os_specific_packages['debian'] }}"
    state: present
  ignore_errors: "{{ ignore_package_errors }}"

- name: Установить специфичные пакеты (если определены)
  ansible.builtin.apt:
    name: "{{ debian_specific_packages | default([]) }}"
    state: present
  when: debian_specific_packages is defined
  ignore_errors: "{{ ignore_package_errors }}"

- name: Удалить указанные пакеты (Debian)
  ansible.builtin.apt:
    name: "{{ packages_to_remove }}"
    state: absent
    autoremove: "{{ package_manager_autoremove }}"
  when: packages_to_remove | length > 0

- name: Полное обновление системы (Debian)
  ansible.builtin.apt:
    upgrade: dist
  when: full_upgrade

- name: Очистить кеш APT
  ansible.builtin.apt:
    autoclean: "{{ package_manager_clean_cache }}"
```

## 4. tasks/redhat.yml (задачи для RHEL/CentOS)

```yaml
---
- name: Обновить кеш YUM/DNF
  ansible.builtin.yum:
    name: "*"
    update_cache: "{{ package_manager_update_cache }}"

- name: Установить общие пакеты (RHEL)
  ansible.builtin.yum:
    name: "{{ common_packages + os_specific_packages['redhat'] }}"
    state: present
  ignore_errors: "{{ ignore_package_errors }}"

- name: Установить специфичные пакеты (если определены)
  ansible.builtin.yum:
    name: "{{ redhat_specific_packages | default([]) }}"
    state: present
  when: redhat_specific_packages is defined
  ignore_errors: "{{ ignore_package_errors }}"

- name: Удалить указанные пакеты (RHEL)
  ansible.builtin.yum:
    name: "{{ packages_to_remove }}"
    state: absent
    autoremove: "{{ package_manager_autoremove }}"
  when: packages_to_remove | length > 0

- name: Полное обновление системы (RHEL)
  ansible.builtin.yum:
    name: "*"
    state: latest
  when: full_upgrade

- name: Очистить кеш YUM/DNF
  ansible.builtin.yum:
    clean: all
  when: package_manager_clean_cache
```

## 5. Пример использования роли (playbook.yml)

```yaml
---
- name: Установка пакетов на все серверы
  hosts: all
  become: yes

  vars:
    # Переопределение переменных
    common_packages:
      - nano
      - tmux
      - net-tools
    redhat_specific_packages:
      - epel-release
      - nginx
    debian_specific_packages:
      - nginx-light
    full_upgrade: no

  roles:
    - role: universal_packages
```

## 6. README.md (документация роли)

````markdown
# Универсальная роль для управления пакетами

Устанавливает и удаляет пакеты на серверах с разными дистрибутивами Linux.

## Поддерживаемые ОС
- Debian/Ubuntu (APT)
- RHEL/CentOS/Fedora (YUM/DNF)

## Переменные

### Общие параметры
- `common_packages` - пакеты для установки на всех ОС
- `os_specific_packages` - пакеты специфичные для ОС
- `packages_to_remove` - пакеты для удаления
- `full_upgrade` - полное обновление системы (true/false)
- `package_manager_*` - параметры менеджера пакетов

### OS-specific
- `debian_specific_packages` - пакеты только для Debian/Ubuntu
- `redhat_specific_packages` - пакеты только для RHEL/CentOS

## Пример использования

```yaml
- hosts: all
  roles:
    - role: universal_packages
      vars:
        common_packages:
          - curl
          - htop
        redhat_specific_packages:
          - epel-release
        full_upgrade: yes
```
````

## Дополнительные возможности

1. **Установка конкретных версий**:
```yaml
debian_specific_packages:
  - nginx=1.18.0-0ubuntu1
```

2. **Установка из дополнительных репозиториев**:
```yaml
- name: Добавить репозиторий EPEL (RHEL)
  ansible.builtin.yum:
    name: epel-release
    state: present

- name: Добавить PPA (Ubuntu)
  ansible.builtin.apt_repository:
    repo: "ppa:ondrej/php"
```

3. **Установка локальных пакетов**:
```yaml
- name: Установка локального .deb пакета
  ansible.builtin.apt:
    deb: "/tmp/package.deb"

- name: Установка локального .rpm пакета
  ansible.builtin.yum:
    name: "/tmp/package.rpm"
```

Эта роль обеспечивает кросс-дистрибутивную совместимость и может быть легко расширена для поддержки других дистрибутивов Linux.