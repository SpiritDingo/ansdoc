Ниже представлена Ansible роль для обновления операционных систем Oracle Linux, Ubuntu 22.04 и Ubuntu 24.04. Роль выполняет:

1. Проверку блокировок обновления:
   · Для Debian‑семейства (Ubuntu) – наличие активных процессов apt/dpkg, а также удерживаемых пакетов (apt-mark hold).
   · Для RedHat‑семейства (Oracle Linux) – наличие процессов yum/dnf и пакетов, заблокированных через yum versionlock.
2. Снятие блокировок:
   · Ожидание завершения запущенных менеджеров пакетов (до 5 минут).
   · Снятие удержаний пакетов в Ubuntu (apt-mark unhold).
   · Очистка versionlock в Oracle Linux (yum versionlock clear).
3. Обновление системы:
   · Обновление кэша пакетов.
   · Обновление всех установленных пакетов до последних версий.

Роль безопасна – удаление файлов блокировок не выполняется, только ожидание завершения процессов. Команды снятия удержаний выполняются только при их наличии.

Структура роли

```
os_update/
├── tasks/
│   ├── main.yml
│   ├── debian.yml
│   └── redhat.yml
└── meta/
    └── main.yml
```

Файлы роли

tasks/main.yml

```yaml
---
- name: Include OS‑specific variables (optional)
  include_vars: "{{ ansible_os_family }}.yml"
  ignore_errors: yes

- name: Update system for Debian family
  include_tasks: debian.yml
  when: ansible_os_family == "Debian"

- name: Update system for RedHat family
  include_tasks: redhat.yml
  when: ansible_os_family == "RedHat"
```

tasks/debian.yml

```yaml
---
- name: Check for running apt or dpkg processes
  ansible.builtin.command: pgrep -f "apt|dpkg"
  register: apt_processes
  changed_when: false
  failed_when: false

- name: Wait for package managers to finish
  ansible.builtin.wait_for:
    path: /var/lib/dpkg/lock
    state: absent
    timeout: 300
  when: apt_processes.stdout != ""
  ignore_errors: yes

- name: Get held packages
  ansible.builtin.command: apt-mark showhold
  register: held_packages
  changed_when: false
  failed_when: false

- name: Unhold packages if any
  ansible.builtin.command: apt-mark unhold {{ held_packages.stdout_lines | join(' ') }}
  when: held_packages.stdout != ""
  changed_when: true

- name: Update apt cache
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Upgrade all packages
  ansible.builtin.apt:
    upgrade: dist
    update_cache: no
```

tasks/redhat.yml

```yaml
---
- name: Check for running yum or dnf processes
  ansible.builtin.command: pgrep -f "yum|dnf"
  register: rpm_processes
  changed_when: false
  failed_when: false

- name: Wait for package managers to finish
  ansible.builtin.wait_for:
    path: /var/run/yum.pid
    state: absent
    timeout: 300
  when: rpm_processes.stdout != ""
  ignore_errors: yes

- name: Get versionlock packages
  ansible.builtin.command: yum versionlock list
  register: versionlock_packages
  changed_when: false
  failed_when: false
  ignore_errors: yes

- name: Clear versionlock if any
  ansible.builtin.command: yum versionlock clear
  when: versionlock_packages.stdout != ""
  changed_when: true
  ignore_errors: yes

- name: Update all packages
  ansible.builtin.yum:
    name: '*'
    state: latest
    update_cache: yes
```

meta/main.yml

```yaml
---
galaxy_info:
  author: your_name
  description: Update OS packages, removing update locks
  license: MIT
  min_ansible_version: "2.9"
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
        - noble
    - name: OracleLinux
      versions:
        - "7"
        - "8"
        - "9"
  galaxy_tags:
    - update
    - upgrade
    - security
    - apt
    - yum
    - dnf
```

Использование

Пример playbook:

```yaml
- hosts: all
  become: yes
  roles:
    - os_update
```

Пояснения

· Блокировки процессов: проверяется наличие запущенных apt/dpkg или yum/dnf. Если есть, роль ждёт исчезновения файла блокировки (/var/lib/dpkg/lock или /var/run/yum.pid). Таймаут 5 минут.
· Удерживаемые пакеты (Ubuntu): apt-mark showhold возвращает список. Если он не пуст, снимаем удержание через apt-mark unhold.
· Versionlock (Oracle Linux): yum versionlock list показывает заблокированные пакеты. Если есть, очищаем блокировку командой yum versionlock clear.
· Обновление: для Ubuntu используется apt upgrade dist (обновление дистрибутива), для Oracle Linux – yum update '*' (все пакеты). Модуль apt и yum идемпотентны, повторный запуск ничего не изменит, если все пакеты уже актуальны.

Роль можно расширить:

· Добавить перезагрузку при необходимости.
· Использовать модуль package_facts для определения менеджера пакетов.
· Обрабатывать отдельно Oracle Linux 8+ через dnf (модуль yum автоматически использует dnf, если он установлен).