Ansible роль: Обновление ОС Linux (Oracle Linux, Ubuntu)

Ниже представлена полная структура Ansible-роли с реализацией всех шести шагов. Роль поддерживает как Oracle Linux (семейство RedHat), так и Ubuntu (семейство Debian), используя условные конструкции when и соответствующие модули пакетных менеджеров.

Структура роли

```
roles/os_update/
├── defaults/
│   └── main.yml
├── vars/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── gather_facts.yml
│   ├── stop_services.yml
│   ├── check_locks.yml
│   ├── perform_update.yml
│   ├── verify_services.yml
│   └── generate_report.yml
├── templates/
│   └── update_report.j2
└── handlers/
    └── main.yml
```

defaults/main.yml — переменные по умолчанию

```yaml
---
# Список сервисов для остановки перед обновлением
os_update_services_to_stop: []

# Останавливать ли сервисы перед обновлением
os_update_stop_services: true

# Запускать ли сервисы после обновления
os_update_start_services: true

# Директория для отчетов
os_update_report_dir: /var/log/ansible-reports

# Таймаут ожидания снятия блокировки (секунды)
os_update_lock_timeout: 600

# Выполнять ли перезагрузку после обновления
os_update_reboot_after: false

# Исключаемые пакеты (например, kernel)
os_update_exclude_packages: []

# Только проверка без реального обновления (dry-run)
os_update_check_mode: false
```

vars/main.yml — внутренние переменные

```yaml
---
# Пути к файлам блокировок для разных семейств ОС
os_update_apt_lock_files:
  - /var/lib/dpkg/lock-frontend
  - /var/lib/dpkg/lock
  - /var/lib/apt/lists/lock
  - /var/cache/apt/archives/lock

os_update_dnf_lock_pattern: "/var/cache/dnf/*_lock.pid"
```

tasks/main.yml — главный файл задач

```yaml
---
- name: "Шаг 1: Сбор фактов ОС"
  ansible.builtin.include_tasks: gather_facts.yml

- name: "Шаг 2: Остановка сервисов"
  ansible.builtin.include_tasks: stop_services.yml
  when: os_update_stop_services | bool

- name: "Шаг 3: Проверка и устранение блокировок"
  ansible.builtin.include_tasks: check_locks.yml

- name: "Шаг 4: Запуск обновления"
  ansible.builtin.include_tasks: perform_update.yml

- name: "Шаг 5: Проверка и запуск сервисов"
  ansible.builtin.include_tasks: verify_services.yml
  when: os_update_start_services | bool

- name: "Шаг 6: Создание отчета"
  ansible.builtin.include_tasks: generate_report.yml
```

tasks/gather_facts.yml — Шаг 1: Сбор фактов

```yaml
---
- name: "Сбор фактов о системе"
  ansible.builtin.setup:
    gather_subset:
      - "!all"
      - "distribution"
      - "pkg_mgr"
      - "date_time"
  register: os_update_facts

- name: "Определение семейства ОС"
  ansible.builtin.set_fact:
    os_update_os_family: "{{ ansible_os_family }}"
    os_update_distribution: "{{ ansible_distribution }}"
    os_update_version: "{{ ansible_distribution_version }}"
    os_update_pkg_mgr: "{{ ansible_pkg_mgr }}"

- name: "Вывод информации об ОС"
  ansible.builtin.debug:
    msg: >
      Хост: {{ inventory_hostname }}
      ОС: {{ os_update_distribution }} {{ os_update_version }}
      Семейство: {{ os_update_os_family }}
      Пакетный менеджер: {{ os_update_pkg_mgr }}
```

Ansible автоматически собирает факты о дистрибутиве через модуль setup. Для получения только информации об ОС можно использовать фильтр ansible_distribution*.

tasks/stop_services.yml — Шаг 2: Остановка сервисов

```yaml
---
- name: "Остановка сервисов перед обновлением"
  ansible.builtin.service:
    name: "{{ item }}"
    state: stopped
  loop: "{{ os_update_services_to_stop }}"
  register: os_update_stopped_services
  failed_when: false
  when: os_update_services_to_stop | length > 0

- name: "Сохранение списка остановленных сервисов"
  ansible.builtin.set_fact:
    os_update_stopped_list: >-
      {{
        os_update_stopped_services.results
        | selectattr('changed', 'defined')
        | selectattr('changed', 'equalto', true)
        | map(attribute='item')
        | list
        if os_update_stopped_services.results is defined
        else []
      }}

- name: "Сервисы для остановки не указаны"
  ansible.builtin.debug:
    msg: "Список сервисов для остановки пуст — пропускаем"
  when: os_update_services_to_stop | length == 0
```

Модуль service поддерживает состояния started, stopped, restarted, reloaded и является идемпотентным — команды не выполняются, если сервис уже в нужном состоянии.

tasks/check_locks.yml — Шаг 3: Проверка блокировок

```yaml
---
# === Для Ubuntu / Debian ===
- name: "Ожидание снятия блокировки apt (Ubuntu/Debian)"
  ansible.builtin.shell: |
    while fuser {{ item }} >/dev/null 2>&1; do
      sleep 5
    done
  loop: "{{ os_update_apt_lock_files }}"
  changed_when: false
  timeout: "{{ os_update_lock_timeout }}"
  when: os_update_os_family == "Debian"

- name: "Остановка таймеров автообновления (Ubuntu/Debian)"
  ansible.builtin.systemd:
    name: "{{ item }}"
    state: stopped
  loop:
    - apt-daily.timer
    - apt-daily-upgrade.timer
  failed_when: false
  when: os_update_os_family == "Debian"

# === Для Oracle Linux / RHEL ===
- name: "Ожидание снятия блокировки dnf/yum (Oracle Linux)"
  ansible.builtin.shell: |
    while ls {{ os_update_dnf_lock_pattern }} >/dev/null 2>&1; do
      sleep 5
    done
  changed_when: false
  timeout: "{{ os_update_lock_timeout }}"
  when: os_update_os_family == "RedHat"

- name: "Проверка активных процессов dnf/yum"
  ansible.builtin.shell: |
    set -o pipefail
    if pgrep -x "dnf|yum" >/dev/null 2>&1; then
      echo "ACTIVE"
    else
      echo "FREE"
    fi
  register: os_update_dnf_process_check
  changed_when: false
  failed_when: false
  when: os_update_os_family == "RedHat"

- name: "Ожидание завершения активных dnf/yum процессов"
  ansible.builtin.shell: |
    while pgrep -x "dnf|yum" >/dev/null 2>&1; do
      sleep 5
    done
  changed_when: false
  timeout: "{{ os_update_lock_timeout }}"
  when:
    - os_update_os_family == "RedHat"
    - "'ACTIVE' in os_update_dnf_process_check.stdout"
```

Для Ubuntu надёжным решением является использование опции lock_timeout на задачах apt, которая обрабатывает случай фоновых автообновлений. Для RedHat-систем Ansible использует файл блокировки /var/cache/dnf/*_lock.pid.

tasks/perform_update.yml — Шаг 4: Запуск обновления

```yaml
---
# === Oracle Linux / RHEL ===
- name: "Обновление всех пакетов (Oracle Linux)"
  ansible.builtin.dnf:
    name: "*"
    state: latest
    update_cache: true
    exclude: "{{ os_update_exclude_packages | join(',') if os_update_exclude_packages | length > 0 else omit }}"
    lock_timeout: "{{ os_update_lock_timeout }}"
  register: os_update_dnf_result
  when: os_update_os_family == "RedHat"
  notify: reboot required

# === Ubuntu / Debian ===
- name: "Обновление кэша пакетов (Ubuntu)"
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
    lock_timeout: "{{ os_update_lock_timeout }}"
  register: os_update_apt_cache
  when: os_update_os_family == "Debian"

- name: "Безопасное обновление пакетов (Ubuntu)"
  ansible.builtin.apt:
    upgrade: safe
    lock_timeout: "{{ os_update_lock_timeout }}"
  register: os_update_apt_result
  when: os_update_os_family == "Debian"

- name: "Полное обновление пакетов (Ubuntu)"
  ansible.builtin.apt:
    upgrade: dist
    lock_timeout: "{{ os_update_lock_timeout }}"
  register: os_update_apt_dist_result
  when:
    - os_update_os_family == "Debian"
    - os_update_apt_full_upgrade | default(false) | bool

- name: "Сохранение результатов обновления"
  ansible.builtin.set_fact:
    os_update_changed: >-
      {{
        (os_update_dnf_result.changed | default(false)) or
        (os_update_apt_result.changed | default(false))
      }}
    os_update_updated_packages: >-
      {{
        os_update_dnf_result.results | default([])
        if os_update_os_family == "RedHat"
        else os_update_apt_result.results | default([])
      }}
```

Модуль dnf (заменяющий yum в новых версиях Ansible) используется для управления пакетами на RedHat-системах. При state: latest с name: "*" выполняется полное обновление аналогично dnf -y update. Для Ubuntu модуль apt с upgrade: safe выполняет безопасное обновление, не удаляя пакеты.

tasks/verify_services.yml — Шаг 5: Проверка и запуск сервисов

```yaml
---
- name: "Проверка статуса остановленных сервисов"
  ansible.builtin.service_facts:

- name: "Запуск сервисов, которые были остановлены"
  ansible.builtin.service:
    name: "{{ item }}"
    state: started
  loop: "{{ os_update_stopped_list | default([]) }}"
  register: os_update_started_services
  failed_when: false

- name: "Проверка успешности запуска сервисов"
  ansible.builtin.assert:
    that:
      - item.state == "running"
    fail_msg: "Сервис {{ item.item }} НЕ запущен!"
    success_msg: "Сервис {{ item.item }} успешно запущен"
  loop: "{{ os_update_started_services.results | default([]) }}"
  loop_control:
    label: "{{ item.item | default('unknown') }}"
  when: os_update_started_services.results is defined

- name: "Восстановление таймеров автообновления (Ubuntu)"
  ansible.builtin.systemd:
    name: "{{ item }}"
    state: started
  loop:
    - apt-daily.timer
    - apt-daily-upgrade.timer
  failed_when: false
  when: os_update_os_family == "Debian"
```

tasks/generate_report.yml — Шаг 6: Создание отчета

```yaml
---
- name: "Создание директории для отчетов"
  ansible.builtin.file:
    path: "{{ os_update_report_dir }}"
    state: directory
    mode: "0755"
  delegate_to: localhost
  become: false

- name: "Генерация отчета об обновлении"
  ansible.builtin.template:
    src: update_report.j2
    dest: "{{ os_update_report_dir }}/{{ inventory_hostname }}_{{ ansible_date_time.date }}.txt"
    mode: "0644"
  delegate_to: localhost
  become: false
  register: os_update_report

- name: "Вывод пути к отчету"
  ansible.builtin.debug:
    msg: "Отчет сохранен: {{ os_update_report.dest }}"
```

templates/update_report.j2 — Шаблон отчета

```jinja2
================================================================================
           ОТЧЕТ ОБ ОБНОВЛЕНИИ ОПЕРАЦИОННОЙ СИСТЕМЫ
================================================================================
Хост:              {{ inventory_hostname }}
Дата и время:      {{ ansible_date_time.iso8601 }}
ОС:                {{ os_update_distribution }} {{ os_update_version }}
Семейство ОС:      {{ os_update_os_family }}
Пакетный менеджер: {{ os_update_pkg_mgr }}

--------------------------------------------------------------------------------
ШАГ 2: ОСТАНОВЛЕННЫЕ СЕРВИСЫ
--------------------------------------------------------------------------------
{% if os_update_stopped_list | default([]) | length > 0 %}
{% for svc in os_update_stopped_list %}
  - {{ svc }}
{% endfor %}
{% else %}
  Сервисы не останавливались
{% endif %}

--------------------------------------------------------------------------------
ШАГ 4: РЕЗУЛЬТАТЫ ОБНОВЛЕНИЯ
--------------------------------------------------------------------------------
Статус:            {{ 'Обновление выполнено' if os_update_changed | default(false) else 'Обновления не требовались' }}
Изменения внесены: {{ 'ДА' if os_update_changed | default(false) else 'НЕТ' }}

{% if os_update_updated_packages | default([]) | length > 0 %}
Обновленные пакеты:
{% for pkg in os_update_updated_packages %}
  - {{ pkg }}
{% endfor %}
{% endif %}

--------------------------------------------------------------------------------
ШАГ 5: СТАТУС СЕРВИСОВ ПОСЛЕ ОБНОВЛЕНИЯ
--------------------------------------------------------------------------------
{% if os_update_started_services is defined and os_update_started_services.results | length > 0 %}
{% for svc in os_update_started_services.results %}
  {{ svc.item }}: {{ 'ЗАПУЩЕН' if svc.state | default('') == 'started' else 'ОШИБКА' }}
{% endfor %}
{% else %}
  Нет данных о запуске сервисов
{% endif %}

--------------------------------------------------------------------------------
ИТОГОВЫЙ СТАТУС
--------------------------------------------------------------------------------
{{ 'УСПЕШНО' if (os_update_changed | default(false)) or not (os_update_stop_services | bool) else 'ЗАВЕРШЕНО' }}
================================================================================
```

handlers/main.yml

```yaml
---
- name: reboot required
  ansible.builtin.debug:
    msg: "Требуется перезагрузка для применения обновлений ядра"

- name: reboot host
  ansible.builtin.reboot:
    reboot_timeout: 600
    msg: "Перезагрузка после обновления ОС"
  when: os_update_reboot_after | bool
```

Пример использования (playbook)

```yaml
---
- name: Обновление ОС на серверах
  hosts: all
  become: true
  gather_facts: true

  vars:
    os_update_services_to_stop:
      - nginx
      - postgresql
    os_update_report_dir: /opt/reports/os_updates
    os_update_lock_timeout: 900

  roles:
    - role: os_update
```

Ключевые особенности реализации

1. Кроссплатформенность — роль автоматически определяет семейство ОС (ansible_os_family) и применяет соответствующие модули: dnf для RedHat, apt для Debian.
2. Управление блокировками — для Ubuntu используется ожидание снятия файлов блокировок /var/lib/dpkg/lock-frontend и остановка таймеров apt-daily. Для Oracle Linux — ожидание освобождения lock-файлов dnf и завершения активных процессов.
3. Идемпотентность — модули service, dnf, apt являются идемпотентными, что позволяет безопасно перезапускать роль без побочных эффектов.
4. Отчетность — отчет генерируется на control-node через delegate_to: localhost с использованием Jinja2-шаблона, содержащего полную информацию о ходе выполнения всех шагов.