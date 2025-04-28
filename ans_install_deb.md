# Ansible роль для установки deb-пакетов на Debian/Ubuntu

```yaml
# roles/deb_pkg_manager/defaults/main.yml
---
# Основные параметры
deb_manage_repos: true
deb_update_cache: true
deb_upgrade_system: false
deb_clean_cache: true

# Основные пакеты для установки
deb_base_packages:
  - apt-transport-https
  - ca-certificates
  - curl
  - wget
  - git
  - htop
  - vim
  - net-tools

# Дополнительные пакеты
deb_extra_packages: []

# Пакеты для удаления
deb_remove_packages: []

# Репозитории
deb_repositories:
  - "deb http://archive.ubuntu.com/ubuntu {{ ansible_distribution_release }} main restricted universe multiverse"
  - "deb http://archive.ubuntu.com/ubuntu {{ ansible_distribution_release }}-updates main restricted universe multiverse"

# GPG ключи
deb_repo_keys: []
```

```yaml
# roles/deb_pkg_manager/tasks/main.yml
---
- name: Проверить поддержку ОС
  assert:
    that:
      - ansible_distribution in ['Debian', 'Ubuntu']
    msg: "Эта роль работает только с Debian/Ubuntu системами"

- name: Обновить индекс пакетов
  apt:
    update_cache: "{{ deb_update_cache }}"
  when: deb_update_cache

- name: Добавить GPG ключи
  apt_key:
    url: "{{ item }}"
    state: present
  loop: "{{ deb_repo_keys }}"
  when: deb_manage_repos and deb_repo_keys | length > 0

- name: Добавить репозитории
  apt_repository:
    repo: "{{ item }}"
    state: present
    update_cache: yes
  loop: "{{ deb_repositories }}"
  when: deb_manage_repos

- name: Установить базовые пакеты
  apt:
    name: "{{ deb_base_packages }}"
    state: present
    update_cache: no

- name: Установить дополнительные пакеты
  apt:
    name: "{{ deb_extra_packages }}"
    state: present
  when: deb_extra_packages | length > 0

- name: Удалить указанные пакеты
  apt:
    name: "{{ deb_remove_packages }}"
    state: absent
    purge: true
  when: deb_remove_packages | length > 0

- name: Обновить систему
  apt:
    upgrade: dist
  when: deb_upgrade_system

- name: Очистить кеш
  apt:
    autoclean: yes
  when: deb_clean_cache
```

```yaml
# playbooks/install_deb_packages.yml
---
- name: Установка пакетов на Debian/Ubuntu
  hosts: debian_servers
  become: true

  vars:
    deb_extra_packages:
      - nginx
      - postgresql-13
      - python3-pip
    deb_remove_packages:
      - apache2
    deb_upgrade_system: true
    deb_repo_keys:
      - "https://www.postgresql.org/media/keys/ACCC4CF8.asc"
    deb_repositories:
      - "deb http://apt.postgresql.org/pub/repos/apt/ {{ ansible_distribution_release }}-pgdg main"

  roles:
    - deb_pkg_manager
```

## Использование

1. Создайте структуру роли:
```bash
mkdir -p roles/deb_pkg_manager/{defaults,tasks}
```

2. Разместите файлы конфигурации как показано выше

3. Создайте inventory файл (hosts.ini):
```ini
[debian_servers]
server1 ansible_host=192.168.1.10
server2 ansible_host=192.168.1.11

[debian_servers:vars]
ansible_user=admin
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

4. Запустите playbook:
```bash
ansible-playbook -i hosts.ini playbooks/install_deb_packages.yml
```

## Особенности роли

1. **Автоматическое определение релиза ОС**:
   - Использует переменную `ansible_distribution_release`
   - Поддерживает разные версии Debian/Ubuntu

2. **Гибкое управление пакетами**:
   - Установка базовых и дополнительных пакетов
   - Удаление ненужных пакетов
   - Обновление системы

3. **Управление репозиториями**:
   - Добавление дополнительных репозиториев
   - Импорт GPG ключей
   - Автоматическое обновление кеша

4. **Безопасность**:
   - Проверка типа ОС перед выполнением
   - Очистка кеша после операций

Для кастомизации просто измените переменные в playbook или создайте файл `group_vars` для конкретной группы серверов.