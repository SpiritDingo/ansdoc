# Ansible роль для установки пакетов через yum

Вот полная реализация Ansible роли для управления пакетами через yum на CentOS/RHEL серверах.

## Структура роли

```
roles/
  yum_packages/
    ├── defaults/
    │   └── main.yml
    ├── tasks/
    │   └── main.yml
    ├── vars/
    │   └── main.yml
    └── README.md
```

## 1. defaults/main.yml (переменные по умолчанию)

```yaml
---
# Список пакетов для установки
yum_packages_to_install:
  - vim-enhanced
  - htop
  - telnet
  - wget
  - curl
  - git

# Список пакетов для удаления
yum_packages_to_remove: []

# Список групп пакетов для установки
yum_groups_to_install: []

# Обновлять ли все пакеты
yum_update_all: no

# Очищать кеш после операций
yum_clean_cache: yes

# Удалять ли зависимости при удалении пакетов
yum_remove_leaf_only: no
```

## 2. tasks/main.yml (основные задачи)

```yaml
---
- name: Обновление кеша yum
  ansible.builtin.yum:
    name: "*"
    update_cache: yes
  when: yum_clean_cache

- name: Установка указанных пакетов
  ansible.builtin.yum:
    name: "{{ yum_packages_to_install }}"
    state: present
  when: yum_packages_to_install | length > 0

- name: Установка групп пакетов
  ansible.builtin.yum:
    name: "{{ yum_groups_to_install }}"
    state: present
    enablerepo: "*"
  when: yum_groups_to_install | length > 0

- name: Удаление указанных пакетов
  ansible.builtin.yum:
    name: "{{ yum_packages_to_remove }}"
    state: absent
    autoremove: "{{ yum_remove_leaf_only }}"
  when: yum_packages_to_remove | length > 0

- name: Полное обновление системы
  ansible.builtin.yum:
    name: "*"
    state: latest
  when: yum_update_all

- name: Очистка кеша yum
  ansible.builtin.yum:
    clean: all
  when: yum_clean_cache
```

## 3. vars/main.yml (переменные для переопределения)

```yaml
---
# Пример переопределения переменных для конкретного окружения
# yum_packages_to_install:
#   - nginx
#   - postgresql
#   - python3
```

## 4. README.md (документация роли)

````markdown
# Роль yum_packages

Роль для управления пакетами через yum на CentOS/RHEL серверах.

## Переменные

- `yum_packages_to_install` - список пакетов для установки
- `yum_packages_to_remove` - список пакетов для удаления
- `yum_groups_to_install` - список групп пакетов для установки
- `yum_update_all` - обновлять ли все пакеты (true/false)
- `yum_clean_cache` - очищать кеш после операций (true/false)
- `yum_remove_leaf_only` - удалять только листовые пакеты (true/false)

## Пример использования

```yaml
- hosts: servers
  roles:
    - role: yum_packages
      vars:
        yum_packages_to_install:
          - nginx
          - postgresql-server
          - python3-pip
        yum_update_all: yes
```
````

## Пример playbook для использования роли (install_packages.yml)

```yaml
---
- name: Установка пакетов на серверы
  hosts: all
  become: yes

  roles:
    - role: yum_packages
      vars:
        yum_packages_to_install:
          - nginx
          - postgresql-server
          - python3-pip
          - docker-ce
        yum_update_all: no
        yum_clean_cache: yes
```

## Дополнительные возможности

1. **Установка конкретных версий пакетов**:
```yaml
yum_packages_to_install:
  - name: nginx
    version: 1.20.1-1.el7
```

2. **Использование репозиториев**:
```yaml
- name: Установка пакета из конкретного репозитория
  ansible.builtin.yum:
    name: docker-ce
    enablerepo: "docker-ce-stable"
    state: present
```

3. **Установка локального RPM-пакета**:
```yaml
- name: Установка локального RPM
  ansible.builtin.yum:
    name: /path/to/package.rpm
    state: present
```

## Запуск playbook

```bash
ansible-playbook -i inventory.ini install_packages.yml
```

Эта роль предоставляет гибкий способ управления пакетами на ваших серверах с возможностью тонкой настройки под конкретные требования.