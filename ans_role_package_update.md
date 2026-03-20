Ниже представлена структура и содержимое Ansible-роли package_updater, которая умеет обновлять пакеты на Oracle Linux (RedHat-семейство) и Ubuntu (Debian-семейство). Роль поддерживает как обновление всех установленных пакетов, так и выборочное обновление конкретного пакета.

Структура роли

```
package_updater/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
└── README.md (опционально)
```

Файл defaults/main.yml

Здесь задаются значения по умолчанию. Если нужно обновить конкретный пакет, его имя передаётся через переменную package_name.

```yaml
---
# Имя пакета для выборочного обновления.
# Если не определено, обновляются все пакеты.
package_name: null
```

Файл tasks/main.yml

Основная логика: обновление кэша (только для Debian) и обновление пакетов через соответствующий модуль.

```yaml
---
- name: Update package cache (Debian/Ubuntu)
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
  when: ansible_os_family == "Debian"

- name: Upgrade all packages (Debian/Ubuntu)
  ansible.builtin.apt:
    upgrade: dist
  when:
    - ansible_os_family == "Debian"
    - package_name is not defined

- name: Upgrade all packages (RedHat/Oracle Linux)
  ansible.builtin.yum:
    name: '*'
    state: latest
  when:
    - ansible_os_family == "RedHat"
    - package_name is not defined

- name: Upgrade specific package (Debian/Ubuntu)
  ansible.builtin.apt:
    name: "{{ package_name }}"
    state: latest
    update_cache: true
  when:
    - ansible_os_family == "Debian"
    - package_name is defined

- name: Upgrade specific package (RedHat/Oracle Linux)
  ansible.builtin.yum:
    name: "{{ package_name }}"
    state: latest
  when:
    - ansible_os_family == "RedHat"
    - package_name is defined
```

Пояснения

1. Определение семейства ОС – через ansible_os_family. Для Ubuntu это Debian, для Oracle Linux – RedHat.
2. Обновление кэша выполняется только на Debian-системах (apt). Кэш считается актуальным в течение 3600 секунд, чтобы не делать лишних обращений к репозиториям.
3. Обновление всех пакетов – для Debian используется upgrade: dist (аналог apt dist-upgrade), для RedHat – yum с name: '*' и state: latest.
4. Обновление конкретного пакета – если задана переменная package_name, роль выполнит только его обновление. Для Debian перед этим дополнительно обновляется кэш.

Использование

Пример плейбука:

```yaml
- hosts: all
  roles:
    - package_updater
```

Если нужно обновить только конкретный пакет, передайте переменную:

```yaml
- hosts: all
  roles:
    - role: package_updater
      vars:
        package_name: nginx
```

Альтернативы

· Вместо yum можно использовать dnf для Oracle Linux 8+, но модуль yum совместим с обеими версиями.
· Модуль package не подходит для полного обновления системы на Debian, так как не умеет обновлять кэш. Поэтому используется раздельная логика.

Роль готова к использованию и может быть легко расширена (например, добавлена поддержка zypper для SUSE).