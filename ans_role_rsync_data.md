Ansible роль sync_data (полная версия)

Структура роли

```
roles/
└── sync_data/
    ├── defaults/
    │   └── main.yml
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── vars/
    │   └── main.yml
    ├── meta/
    │   └── main.yml
    ├── README.md
    └── tests/
        └── test.yml
```

---

roles/sync_data/defaults/main.yml

```yaml
---
# === Источник и назначение ===
# Путь к исходному каталогу на сервере-источнике
sync_src: "/data/source/"

# Путь к целевому каталогу на сервере-назначении
sync_dest: "/data/destination/"

# Хост-источник (должен быть в inventory)
sync_source_host: "source-server"

# Хост-назначение (должен быть в inventory)
sync_dest_host: "dest-server"

# Направление: pull (тянуть с источника) или push (толкать на назначение)
sync_mode: "pull"

# Удалять файлы на назначении, которых нет в источнике (осторожно!)
sync_delete: false

# === Опции rsync ===
# Дополнительные опции rsync (исключения, сжатие и т.д.)
sync_rsync_opts:
  - "--compress"
  - "--exclude=.git/"
  - "--exclude=*.tmp"
  - "--exclude=node_modules/"

# SSH-опции для rsync
sync_ssh_args: "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o Compression=no"

# Таймаут соединения (секунды)
sync_timeout: 300

# === Ограничение скорости ===
# Ограничение скорости rsync в КБ/с (0 = без ограничения)
# Пример: 10240 = 10 МБ/с, 51200 = 50 МБ/с
sync_bwlimit: 0

# Отдельные лимиты для рабочих часов и ночного окна
# Работает только если sync_bwlimit == 0
sync_bwlimit_work: 5120      # 5 МБ/с днём
sync_bwlimit_night: 0        # без ограничения ночью
sync_business_hours_start: 9  # 09:00
sync_business_hours_end: 19   # 19:00

# === Сервисы на целевом хосте ===
# Список сервисов, которые нужно перезапустить после синхронизации
sync_services_to_restart: []

# Перезапускать сервисы только если были изменения
sync_restart_on_change_only: true
```

---

roles/sync_data/tasks/main.yml

```yaml
---
- name: Проверить наличие исходного каталога
  ansible.builtin.stat:
    path: "{{ sync_src }}"
  delegate_to: "{{ sync_source_host }}"
  register: sync_src_check
  run_once: true

- name: Завершить, если исходный каталог не найден
  ansible.builtin.fail:
    msg: "Исходный каталог {{ sync_src }} не существует на {{ sync_source_host }}"
  when: not sync_src_check.stat.exists
  run_once: true

- name: Создать целевой каталог
  ansible.builtin.file:
    path: "{{ sync_dest }}"
    state: directory
    mode: "0755"
  delegate_to: "{{ sync_dest_host }}"
  run_once: true

# === Вычисление лимита скорости ===
- name: Определить текущее время на источнике
  ansible.builtin.setup:
    gather_subset:
      - "!all"
      - "!min"
      - date_time
  delegate_to: "{{ sync_source_host }}"
  run_once: true
  register: sync_facts

- name: Рассчитать активный bwlimit
  ansible.builtin.set_fact:
    sync_active_bwlimit: >-
      {{
        sync_bwlimit | int
        if (sync_bwlimit | int) > 0
        else (
          sync_bwlimit_work | int
          if (sync_facts.ansible_facts.ansible_date_time.hour | int) >= (sync_business_hours_start | int)
             and (sync_facts.ansible_facts.ansible_date_time.hour | int) < (sync_business_hours_end | int)
          else sync_bwlimit_night | int
        )
      }}
  run_once: true

- name: Показать выбранный лимит скорости
  ansible.builtin.debug:
    msg: >-
      Активный bwlimit:
      {{ sync_active_bwlimit }} КБ/с
      {{ '(без ограничения)' if (sync_active_bwlimit | int) == 0 else '' }}
  run_once: true

# === Основная синхронизация ===
- name: Синхронизировать данные между серверами
  ansible.posix.synchronize:
    src: "{{ sync_src }}"
    dest: "{{ sync_dest }}"
    mode: "{{ sync_mode }}"
    delete: "{{ sync_delete }}"
    recursive: yes
    archive: yes
    compress: yes
    rsync_opts: >-
      {{
        sync_rsync_opts
        + (['--bwlimit=' ~ (sync_active_bwlimit | string)] if (sync_active_bwlimit | int) > 0 else [])
      }}
    ssh_args: "{{ sync_ssh_args }}"
    timeout: "{{ sync_timeout }}"
  delegate_to: "{{ sync_source_host }}"
  run_once: true
  register: sync_result
  changed_when: sync_result is changed

- name: Показать сводку синхронизации
  ansible.builtin.debug:
    msg: >-
      Синхронизация завершена.
      Изменения: {{ 'да' if sync_result is changed else 'нет' }}.
      Лимит: {{ sync_active_bwlimit }} КБ/с.
  run_once: true

# === Перезапуск сервисов на целевом хосте ===
- name: Перезапустить сервисы на целевом хосте
  ansible.builtin.service:
    name: "{{ item }}"
    state: restarted
  loop: "{{ sync_services_to_restart }}"
  delegate_to: "{{ sync_dest_host }}"
  when:
    - sync_services_to_restart | length > 0
    - (not sync_restart_on_change_only) or (sync_result is changed)
  run_once: true
```

---

roles/sync_data/handlers/main.yml

```yaml
---
- name: Перезапустить сервисы после синхронизации
  ansible.builtin.service:
    name: "{{ item }}"
    state: restarted
  loop: "{{ sync_services_to_restart | default([]) }}"
  delegate_to: "{{ sync_dest_host }}"
```

Handlers не вызываются автоматически из роли, чтобы не было двойного перезапуска. Они оставлены на случай, если вы захотите вызывать их вручную через notify.

---

roles/sync_data/vars/main.yml

```yaml
---
# Переменные роли, которые не должны переопределяться пользователем
# (внутренние значения). Пользовательские настройки — в defaults/main.yml.
```

---

roles/sync_data/meta/main.yml

```yaml
---
galaxy_info:
  author: "Ваше имя"
  description: "Синхронизация больших каталогов между серверами с помощью rsync"
  license: MIT
  min_ansible_version: "2.9"
  platforms:
    - name: EL
      versions:
        - all
    - name: Ubuntu
      versions:
        - all
    - name: Debian
      versions:
        - all
  galaxy_tags:
    - sync
    - rsync
    - filetransfer

collections:
  - ansible.posix

dependencies: []
```

Коллекция ansible.posix указана в collections, чтобы Ansible сам её подтянул при запуске плейбука. Установить её при необходимости: ansible-galaxy collection install ansible.posix.

---

roles/sync_data/README.md

```markdown
# sync_data

Роль для синхронизации больших каталогов между двумя серверами с помощью `rsync`
(через модуль `ansible.posix.synchronize`).

## Возможности

- Дельта-трансфер — передаются только изменения (быстрее, чем `copy`).
- Делегирование задачи на хост-источник — данные не идут через контроллер.
- Ограничение скорости (`--bwlimit`) с поддержкой отдельных лимитов
  для рабочих часов и ночного окна.
- Опциональное удаление лишних файлов на целевом хосте (`delete`).
- Перезапуск указанных сервисов на целевом хосте после синхронизации.

## Основные переменные

| Переменная | По умолчанию | Описание |
|---|---|---|
| `sync_src` | `/data/source/` | Исходный каталог |
| `sync_dest` | `/data/destination/` | Целевой каталог |
| `sync_source_host` | `source-server` | Хост-источник (из inventory) |
| `sync_dest_host` | `dest-server` | Хост-назначение (из inventory) |
| `sync_mode` | `pull` | `pull` или `push` |
| `sync_delete` | `false` | Удалять лишние файлы на назначении |
| `sync_rsync_opts` | `[...]` | Доп. опции rsync |
| `sync_bwlimit` | `0` | Жёсткий лимит КБ/с (0 = авто-режим) |
| `sync_bwlimit_work` | `5120` | Лимит в рабочие часы |
| `sync_bwlimit_night` | `0` | Лимит вне рабочих часов |
| `sync_business_hours_start` | `9` | Начало рабочих часов |
| `sync_business_hours_end` | `19` | Конец рабочих часов |
| `sync_services_to_restart` | `[]` | Сервисы для перезапуска |
| `sync_restart_on_change_only` | `true` | Только при изменениях |

## Пример плейбука

```yaml
- hosts: localhost
  gather_facts: false
  roles:
    - role: sync_data
      vars:
        sync_source_host: "server-a"
        sync_dest_host: "server-b"
        sync_src: "/var/www/html/"
        sync_dest: "/var/www/html/"
        sync_delete: true
        sync_bwlimit: 20480
        sync_services_to_restart:
          - nginx
          - php-fpm
```

```

---

## `roles/sync_data/tests/test.yml`

```yaml
---
- name: Тестовый запуск роли sync_data
  hosts: localhost
  gather_facts: false
  roles:
    - role: sync_data
      vars:
        sync_source_host: "server-a"
        sync_dest_host: "server-b"
        sync_src: "/tmp/test-src/"
        sync_dest: "/tmp/test-dest/"
        sync_bwlimit_work: 1024
        sync_bwlimit_night: 0
        sync_business_hours_start: 9
        sync_business_hours_end: 19
```

---

Пример плейбука для запуска

playbooks/sync.yml:

```yaml
---
# Жёсткий лимит 20 МБ/с
- name: Синхронизация данных (жёсткий лимит)
  hosts: localhost
  gather_facts: false
  roles:
    - role: sync_data
      vars:
        sync_source_host: "server-a"
        sync_dest_host: "server-b"
        sync_src: "/var/lib/postgresql/backups/"
        sync_dest: "/mnt/backup/pg/"
        sync_bwlimit: 20480

---
# Автоматический лимит по времени суток
- name: Синхронизация медиа (день/ночь)
  hosts: localhost
  gather_facts: false
  roles:
    - role: sync_data
      vars:
        sync_source_host: "server-a"
        sync_dest_host: "server-b"
        sync_src: "/data/media/"
        sync_dest: "/data/media/"
        sync_bwlimit: 0
        sync_bwlimit_work: 5120      # 5 МБ/с днём
        sync_bwlimit_night: 0        # без лимита ночью
        sync_business_hours_start: 9
        sync_business_hours_end: 19
```

---

Запуск

```bash
# Установить зависимость один раз
ansible-galaxy collection install ansible.posix

# Запустить плейбук
ansible-playbook -i inventory.ini playbooks/sync.yml

# Только проверить (dry-run)
ansible-playbook -i inventory.ini playbooks/sync.yml --check --diff
```

---

Ключевые моменты

· Делегирование (delegate_to) — задача synchronize выполняется на хосте-источнике, rsync сам подключается к назначению.
· run_once: true — синхронизация запускается один раз, а не для каждого хоста inventory.
· Ограничение скорости:
  · sync_bwlimit > 0 — жёсткий лимит;
  · sync_bwlimit == 0 — авто-режим: днём sync_bwlimit_work, ночью sync_bwlimit_night;
  · если итоговый лимит 0, опция --bwlimit в команду rsync не добавляется.
· delete: yes — делает назначение точной копией источника, удаляя лишние файлы. Используйте осторожно.
· sync_restart_on_change_only: true — сервисы перезапускаются только если rsync что-то изменил.
__________

Ansible роль sync_data с логированием (полная версия)

Добавлено логирование результатов синхронизации: сводка в системный журнал, отчёт в файл на управляющем хосте и опциональная запись JSON-лога на целевом сервере.

Структура роли

```
roles/
└── sync_data/
    ├── defaults/
    │   └── main.yml
    ├── tasks/
    │   ├── main.yml
    │   └── log.yml
    ├── templates/
    │   └── sync_report.json.j2
    ├── handlers/
    │   └── main.yml
    ├── vars/
    │   └── main.yml
    ├── meta/
    │   └── main.yml
    ├── README.md
    └── tests/
        └── test.yml
```

---

roles/sync_data/defaults/main.yml

```yaml
---
# === Источник и назначение ===
sync_src: "/data/source/"
sync_dest: "/data/destination/"
sync_source_host: "source-server"
sync_dest_host: "dest-server"
sync_mode: "pull"
sync_delete: false

# === Опции rsync ===
sync_rsync_opts:
  - "--compress"
  - "--exclude=.git/"
  - "--exclude=*.tmp"
  - "--exclude=node_modules/"

sync_ssh_args: "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o Compression=no"
sync_timeout: 300

# === Ограничение скорости ===
sync_bwlimit: 0
sync_bwlimit_work: 5120
sync_bwlimit_night: 0
sync_business_hours_start: 9
sync_business_hours_end: 19

# === Сервисы на целевом хосте ===
sync_services_to_restart: []
sync_restart_on_change_only: true

# === Логирование ===
# Включить логирование
sync_logging_enabled: true

# Путь к файлу лога на управляющем хосте (контроллере)
sync_log_path: "/var/log/ansible/sync_data.log"

# Формат: line (одна строка текста) или json
sync_log_format: "line"

# Отдельный JSON-отчёт на контроллере (по каждому запуску)
sync_report_dir: "/var/log/ansible/reports"

# Писать сводку в syslog целевого хоста (через logger)
sync_log_to_syslog: false

# Тег syslog
sync_syslog_tag: "ansible-sync-data"

# Уровень детализации: minimal | normal | verbose
sync_log_verbosity: "normal"

# Включить расширенную статистику rsync (--stats) для парсинга
sync_enable_stats: true
```

---

roles/sync_data/tasks/main.yml

```yaml
---
- name: Проверить наличие исходного каталога
  ansible.builtin.stat:
    path: "{{ sync_src }}"
  delegate_to: "{{ sync_source_host }}"
  register: sync_src_check
  run_once: true

- name: Завершить, если исходный каталог не найден
  ansible.builtin.fail:
    msg: "Исходный каталог {{ sync_src }} не существует на {{ sync_source_host }}"
  when: not sync_src_check.stat.exists
  run_once: true

- name: Создать целевой каталог
  ansible.builtin.file:
    path: "{{ sync_dest }}"
    state: directory
    mode: "0755"
  delegate_to: "{{ sync_dest_host }}"
  run_once: true

# === Вычисление лимита скорости ===
- name: Определить текущее время на источнике
  ansible.builtin.setup:
    gather_subset:
      - "!all"
      - "!min"
      - date_time
  delegate_to: "{{ sync_source_host }}"
  run_once: true
  register: sync_facts

- name: Рассчитать активный bwlimit
  ansible.builtin.set_fact:
    sync_active_bwlimit: >-
      {{
        sync_bwlimit | int
        if (sync_bwlimit | int) > 0
        else (
          sync_bwlimit_work | int
          if (sync_facts.ansible_facts.ansible_date_time.hour | int) >= (sync_business_hours_start | int)
             and (sync_facts.ansible_facts.ansible_date_time.hour | int) < (sync_business_hours_end | int)
          else sync_bwlimit_night | int
        )
      }}
  run_once: true

- name: Зафиксировать время начала синхронизации
  ansible.builtin.set_fact:
    sync_started_at: "{{ '%Y-%m-%dT%H:%M:%S%z' | strftime(ansible_date_time.epoch | int) }}"
    sync_started_epoch: "{{ ansible_date_time.epoch | int }}"
  run_once: true

- name: Показать выбранный лимит скорости
  ansible.builtin.debug:
    msg: >-
      Активный bwlimit: {{ sync_active_bwlimit }} КБ/с
      {{ '(без ограничения)' if (sync_active_bwlimit | int) == 0 else '' }}
  run_once: true

# === Основная синхронизация ===
- name: Синхронизировать данные между серверами
  ansible.posix.synchronize:
    src: "{{ sync_src }}"
    dest: "{{ sync_dest }}"
    mode: "{{ sync_mode }}"
    delete: "{{ sync_delete }}"
    recursive: yes
    archive: yes
    compress: yes
    rsync_opts: >-
      {{
        sync_rsync_opts
        + (['--bwlimit=' ~ (sync_active_bwlimit | string)] if (sync_active_bwlimit | int) > 0 else [])
        + (['--stats'] if sync_enable_stats | bool else [])
      }}
    ssh_args: "{{ sync_ssh_args }}"
    timeout: "{{ sync_timeout }}"
  delegate_to: "{{ sync_source_host }}"
  run_once: true
  register: sync_result
  changed_when: sync_result is changed
  failed_when: sync_result is failed

- name: Зафиксировать время окончания синхронизации
  ansible.builtin.set_fact:
    sync_finished_at: "{{ '%Y-%m-%dT%H:%M:%S%z' | strftime(ansible_date_time.epoch | int) }}"
    sync_finished_epoch: "{{ ansible_date_time.epoch | int }}"
  run_once: true

- name: Вычислить длительность синхронизации
  ansible.builtin.set_fact:
    sync_duration_sec: "{{ (sync_finished_epoch | int) - (sync_started_epoch | int) }}"
  run_once: true

# === Парсинг статистики rsync ===
- name: Извлечь статистику из вывода rsync
  ansible.builtin.set_fact:
    sync_stats: >-
      {{
        {
          'files_transferred': (sync_result.stdout | default('') | regex_findall('Number of regular files transferred: (\\d+)') | first | default('0')),
          'total_bytes_sent':  (sync_result.stdout | default('') | regex_findall('Total bytes sent: ([\\d,]+)') | first | default('0') | replace(',', '')),
          'total_bytes_recv':  (sync_result.stdout | default('') | regex_findall('Total bytes received: ([\\d,]+)') | first | default('0') | replace(',', '')),
          'speedup':           (sync_result.stdout | default('') | regex_findall('speedup is ([\\d.]+)') | first | default('n/a'))
        }
      }}
  when: sync_enable_stats | bool
  run_once: true

# === Логирование ===
- name: Записать результаты синхронизации в лог
  ansible.builtin.include_tasks: log.yml
  when: sync_logging_enabled | bool
  run_once: true

- name: Показать сводку синхронизации
  ansible.builtin.debug:
    msg: >-
      Синхронизация завершена.
      Статус: {{ 'изменения' if sync_result is changed else 'без изменений' }}.
      Время: {{ sync_duration_sec }} c.
      Лимит: {{ sync_active_bwlimit }} КБ/с.
  run_once: true

# === Перезапуск сервисов на целевом хосте ===
- name: Перезапустить сервисы на целевом хосте
  ansible.builtin.service:
    name: "{{ item }}"
    state: restarted
  loop: "{{ sync_services_to_restart }}"
  delegate_to: "{{ sync_dest_host }}"
  when:
    - sync_services_to_restart | length > 0
    - (not sync_restart_on_change_only) or (sync_result is changed)
  run_once: true
```

---

roles/sync_data/tasks/log.yml

```yaml
---
# === 1. Собрать структурированные данные о запуске ===
- name: Сформировать структуру записи лога
  ansible.builtin.set_fact:
    sync_log_record:
      timestamp: "{{ sync_finished_at }}"
      status: "{{ 'changed' if sync_result is changed else 'ok' }}"
      duration_sec: "{{ sync_duration_sec | int }}"
      source_host: "{{ sync_source_host }}"
      dest_host: "{{ sync_dest_host }}"
      src: "{{ sync_src }}"
      dest: "{{ sync_dest }}"
      mode: "{{ sync_mode }}"
      delete: "{{ sync_delete | bool }}"
      bwlimit_kbps: "{{ sync_active_bwlimit | int }}"
      files_transferred: "{{ sync_stats.files_transferred | default('0') }}"
      bytes_sent: "{{ sync_stats.total_bytes_sent | default('0') }}"
      bytes_recv: "{{ sync_stats.total_bytes_recv | default('0') }}"
      speedup: "{{ sync_stats.speedup | default('n/a') }}"
      run_by: "{{ lookup('env', 'USER') | default('unknown', true) }}"
      playbook: "{{ ansible_play_name | default('n/a') }}"
      task: "{{ ansible_play_hosts_all | length }} host(s)"
  run_once: true

# === 2. Текстовая строка для файла лога ===
- name: Сформировать строку лога
  ansible.builtin.set_fact:
    sync_log_line: >-
      {{ sync_log_record.timestamp }}
      status={{ sync_log_record.status }}
      duration={{ sync_log_record.duration_sec }}s
      files={{ sync_log_record.files_transferred }}
      sent={{ sync_log_record.bytes_sent }}B
      recv={{ sync_log_record.bytes_recv }}B
      bwlimit={{ sync_log_record.bwlimit_kbps }}KB/s
      src={{ sync_log_record.source_host }}:{{ sync_log_record.src }}
      dest={{ sync_log_record.dest_host }}:{{ sync_log_record.dest }}
  run_once: true

# === 3. Запись на управляющем хосте (контроллере) ===
- name: Убедиться, что каталог для логов существует
  ansible.builtin.file:
    path: "{{ sync_log_path | dirname }}"
    state: directory
    mode: "0755"
  delegate_to: localhost
  become: false
  run_once: true

- name: Записать строку лога в файл на контроллере
  ansible.builtin.lineinfile:
    path: "{{ sync_log_path }}"
    line: "{{ sync_log_line }}"
    create: yes
    mode: "0644"
    insertafter: EOF
  delegate_to: localhost
  become: false
  when: sync_log_format == 'line'
  run_once: true

# === 4. JSON-отчёт ===
- name: Убедиться, что каталог для отчётов существует
  ansible.builtin.file:
    path: "{{ sync_report_dir }}"
    state: directory
    mode: "0755"
  delegate_to: localhost
  become: false
  when: sync_log_format == 'json'
  run_once: true

- name: Записать JSON-отчёт о синхронизации
  ansible.builtin.copy:
    content: "{{ sync_log_record | to_nice_json }}\n"
    dest: >-
      {{ sync_report_dir }}/sync-{{ sync_finished_at | regex_replace('[:+]', '-') }}.json
    mode: "0644"
  delegate_to: localhost
  become: false
  when: sync_log_format == 'json'
  run_once: true

# === 5. Запись в syslog на целевом хосте ===
- name: Записать сводку в syslog целевого хоста
  ansible.builtin.shell: |
    logger -t {{ sync_syslog_tag | quote }} -- {{ sync_log_line | quote }}
  delegate_to: "{{ sync_dest_host }}"
  changed_when: false
  when: sync_log_to_syslog | bool
  run_once: true

# === 6. Verbose-вывод в консоль ===
- name: Verbose-вывод результатов синхронизации
  ansible.builtin.debug:
    var: sync_log_record
  when:
    - sync_log_verbosity == 'verbose'
  run_once: true
```

---

roles/sync_data/templates/sync_report.json.j2

Шаблон для тех, кто предпочитает генерировать отчёт через template (альтернатива copy + to_nice_json):

```jinja
{
  "timestamp": "{{ sync_log_record.timestamp }}",
  "status": "{{ sync_log_record.status }}",
  "duration_sec": {{ sync_log_record.duration_sec }},
  "source_host": "{{ sync_log_record.source_host }}",
  "dest_host": "{{ sync_log_record.dest_host }}",
  "src": "{{ sync_log_record.src }}",
  "dest": "{{ sync_log_record.dest }}",
  "mode": "{{ sync_log_record.mode }}",
  "delete": {{ sync_log_record.delete | to_json }},
  "bwlimit_kbps": {{ sync_log_record.bwlimit_kbps }},
  "files_transferred": {{ sync_log_record.files_transferred }},
  "bytes_sent": {{ sync_log_record.bytes_sent }},
  "bytes_recv": {{ sync_log_record.bytes_recv }},
  "speedup": "{{ sync_log_record.speedup }}",
  "run_by": "{{ sync_log_record.run_by }}",
  "playbook": "{{ sync_log_record.playbook }}"
}
```

---

roles/sync_data/handlers/main.yml

```yaml
---
- name: Перезапустить сервисы после синхронизации
  ansible.builtin.service:
    name: "{{ item }}"
    state: restarted
  loop: "{{ sync_services_to_restart | default([]) }}"
  delegate_to: "{{ sync_dest_host }}"
```

---

roles/sync_data/vars/main.yml

```yaml
---
# Внутренние переменные роли — не переопределяются пользователем.
```

---

roles/sync_data/meta/main.yml

```yaml
---
galaxy_info:
  author: "Ваше имя"
  description: "Синхронизация больших каталогов между серверами с логированием результатов"
  license: MIT
  min_ansible_version: "2.9"
  platforms:
    - name: EL
      versions: [all]
    - name: Ubuntu
      versions: [all]
    - name: Debian
      versions: [all]
  galaxy_tags:
    - sync
    - rsync
    - filetransfer
    - logging

collections:
  - ansible.posix

dependencies: []
```

---

roles/sync_data/README.md

```markdown
# sync_data

Роль для синхронизации больших каталогов между двумя серверами через `rsync`
с ограничением скорости и логированием результатов.

## Возможности

- Дельта-трансфер (rsync) — передаются только изменения.
- Делегирование на хост-источник, данные не идут через контроллер.
- Лимит скорости `--bwlimit` с разделением день/ночь.
- Опциональное удаление лишних файлов на цели.
- Перезапуск сервисов на цели после изменений.
- Логирование:
  - строка в файл на контроллере (`sync_log_format: line`);
  - JSON-отчёт по каждому запуску (`sync_log_format: json`);
  - запись в syslog целевого хоста (`sync_log_to_syslog: true`);
  - verbose-вывод в консоль (`sync_log_verbosity: verbose`).

## Переменные логирования

| Переменная | По умолчанию | Описание |
|---|---|---|
| `sync_logging_enabled` | `true` | Включить логирование |
| `sync_log_path` | `/var/log/ansible/sync_data.log` | Файл лога на контроллере |
| `sync_log_format` | `line` | `line` или `json` |
| `sync_report_dir` | `/var/log/ansible/reports` | Каталог JSON-отчётов |
| `sync_log_to_syslog` | `false` | Писать сводку в syslog цели |
| `sync_syslog_tag` | `ansible-sync-data` | Тег syslog |
| `sync_log_verbosity` | `normal` | `minimal` / `normal` / `verbose` |
| `sync_enable_stats` | `true` | Добавлять `--stats` в rsync |

## Пример

```yaml
- hosts: localhost
  gather_facts: false
  roles:
    - role: sync_data
      vars:
        sync_source_host: "server-a"
        sync_dest_host: "server-b"
        sync_src: "/var/www/html/"
        sync_dest: "/var/www/html/"
        sync_delete: true
        sync_bwlimit: 20480
        sync_log_format: "json"
        sync_log_to_syslog: true
        sync_log_verbosity: "verbose"
        sync_services_to_restart:
          - nginx
          - php-fpm
```

```

---

## `roles/sync_data/tests/test.yml`

```yaml
---
- name: Тестовый запуск роли sync_data
  hosts: localhost
  gather_facts: false
  roles:
    - role: sync_data
      vars:
        sync_source_host: "server-a"
        sync_dest_host: "server-b"
        sync_src: "/tmp/test-src/"
        sync_dest: "/tmp/test-dest/"
        sync_bwlimit_work: 1024
        sync_bwlimit_night: 0
        sync_log_path: "/tmp/sync_data.log"
        sync_report_dir: "/tmp/sync_reports"
        sync_log_format: "json"
```

---

Примеры плейбуков

С текстовым логом

```yaml
---
- name: Синхронизация с текстовым логом
  hosts: localhost
  gather_facts: false
  roles:
    - role: sync_data
      vars:
        sync_source_host: "server-a"
        sync_dest_host: "server-b"
        sync_src: "/var/lib/postgresql/backups/"
        sync_dest: "/mnt/backup/pg/"
        sync_bwlimit: 20480
        sync_log_path: "/var/log/ansible/sync_pg.log"
```

С JSON-отчётами и syslog

```yaml
---
- name: Синхронизация с JSON и syslog
  hosts: localhost
  gather_facts: false
  roles:
    - role: sync_data
      vars:
        sync_source_host: "server-a"
        sync_dest_host: "server-b"
        sync_src: "/data/media/"
        sync_dest: "/data/media/"
        sync_log_format: "json"
        sync_report_dir: "/var/log/ansible/reports"
        sync_log_to_syslog: true
        sync_log_verbosity: "verbose"
```

---

Формат записи

line (пример строки)

```
2026-10-13T14:32:11+0300 status=changed duration=142s files=1842 sent=734003200B recv=24576B bwlimit=20480KB/s src=server-a:/var/www/html/ dest=server-b:/var/www/html/
```

Удобно парсить awk/grep и передавать в системы мониторинга.

json (пример отчёта)

```json
{
    "timestamp": "2026-10-13T14:32:11+0300",
    "status": "changed",
    "duration_sec": 142,
    "source_host": "server-a",
    "dest_host": "server-b",
    "src": "/var/www/html/",
    "dest": "/var/www/html/",
    "mode": "pull",
    "delete": true,
    "bwlimit_kbps": 20480,
    "files_transferred": "1842",
    "bytes_sent": "734003200",
    "bytes_recv": "24576",
    "speedup": "3.42",
    "run_by": "ansible",
    "playbook": "Синхронизация с JSON и syslog"
}
```

Файлы: /var/log/ansible/reports/sync-2026-10-13T14-32-11-0300.json.

---

Запуск

```bash
# Установить зависимость
ansible-galaxy collection install ansible.posix

# Запуск
ansible-playbook -i inventory.ini playbooks/sync.yml

# Dry-run
ansible-playbook -i inventory.ini playbooks/sync.yml --check --diff
```

---

Ключевые моменты логирования

· Парсинг статистики rsync (sync_stats) выполняется регулярками из stdout. Поле speedup может быть n/a, если rsync не вывел его.
· Запись лога на контроллере идёт через delegate_to: localhost с become: false. Убедитесь, что у пользователя, запускающего Ansible, есть права на запись в sync_log_path.
· JSON-отчёты именуются по времени завершения, поэтому не перетирают друг друга. Для очистки старых отчётов используйте отдельный cron или find /var/log/ansible/reports -mtime +30 -delete.
· sync_log_to_syslog: true пишет в journald/syslog на целевом хосте, что удобно для централизованного сбора (rsyslog → SIEM).
· sync_log_verbosity: verbose печатает всю структуру sync_log_record в консоль Ansible — полезно при отладке плейбука.
· failed_when: sync_result is failed оставлено явно, чтобы отловить ошибки rsync и прервать роль до записи «ложноположительного» лога. При необходимости можно добавить отдельную ветку block/rescue для логирования сбоев.

________
________

Ошибка возникает из-за того, что модуль ansible.posix.synchronize не принимает параметры ssh_args и timeout. В поддерживаемых параметрах, которые перечислены в ошибке, есть _ssh_args и rsync_timeout. Нужно заменить их в задаче синхронизации.

Что исправить в tasks/main.yml

Найдите блок Синхронизировать данные между серверами и замените в нём:

· ssh_args: "{{ sync_ssh_args }}" → _ssh_args: "{{ sync_ssh_args }}"
· timeout: "{{ sync_timeout }}" → rsync_timeout: "{{ sync_timeout }}"

Вот исправленный фрагмент задачи:

```yaml
- name: Синхронизировать данные между серверами
  ansible.posix.synchronize:
    src: "{{ sync_src }}"
    dest: "{{ sync_dest }}"
    mode: "{{ sync_mode }}"
    delete: "{{ sync_delete }}"
    recursive: yes
    archive: yes
    compress: yes
    rsync_opts: >-
      {{
        sync_rsync_opts
        + (['--bwlimit=' ~ (sync_active_bwlimit | string)] if (sync_active_bwlimit | int) > 0 else [])
        + (['--stats'] if sync_enable_stats | bool else [])
      }}
    _ssh_args: "{{ sync_ssh_args }}"
    rsync_timeout: "{{ sync_timeout }}"
  delegate_to: "{{ sync_source_host }}"
  run_once: true
  register: sync_result
  changed_when: sync_result is changed
  failed_when: sync_result is failed
```

Альтернативный, более правильный способ для SSH-опций

Параметр _ssh_args помечен в документации как «Internal use only» (только для внутреннего использования). Хотя он присутствует в списке поддерживаемых параметров, полагаться на внутренний параметр не рекомендуется.

Лучший способ — использовать use_ssh_args: true и задать нужные SSH-опции через переменные Ansible. Для этого в defaults/main.yml можно добавить:

```yaml
# SSH-опции для rsync (передаются через use_ssh_args)
ansible_ssh_common_args: "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o Compression=no"
```

А в задаче синхронизации заменить _ssh_args на:

```yaml
    use_ssh_args: true
```

Тогда модуль сам подставит ansible_ssh_common_args (и другие SSH-настройки) при вызове rsync. Это официально поддерживаемый путь.

Итог

Минимальное исправление — заменить в задаче два параметра:

· ssh_args → _ssh_args
· timeout → rsync_timeout

Если хотите сделать «по-правильному» — уберите ssh_args/_ssh_args и используйте use_ssh_args: true вместе с ansible_ssh_common_args.