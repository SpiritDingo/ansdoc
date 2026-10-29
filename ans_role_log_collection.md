Готовая Ansible-роль collect_logs — финальная версия

Ниже — полная роль со всеми изменениями: сбор логов в один архив на сервере, имя которого содержит inventory_hostname и дату. Поддерживаются оба способа доставки — fetch (на control-node) и synchronize (rsync-pull на выделенный log-server).

---

📁 Структура роли

```text
roles/collect_logs/
├── defaults/
│   └── main.yml
├── tasks/
│   └── main.yml
├── meta/
│   └── main.yml
└── README.md
```

---

📄 roles/collect_logs/defaults/main.yml

```yaml
---
# Что собираем — файлы, каталоги и glob-шаблоны
collect_log_paths:
  - /var/log/syslog
  - /var/log/messages
  - /var/log/nginx
  - "/var/log/myapp/*.log"

# Исключения из архива (шаблоны tar)
collect_log_excludes:
  - "*.gz"
  - "*.old"

# Имя архива: <hostname>_<date>.tar.gz
# Для безопасного варианта с временем замените на:
#   "{{ inventory_hostname }}_{{ ansible_date_time.iso8601_basic_short }}.tar.gz"
collect_archive_name: "{{ inventory_hostname }}_{{ ansible_date_time.date }}.tar.gz"

# Временный каталог на управляемом хосте
collect_remote_tmp: "/tmp/logs-collect"

# Куда складывать архивы
collect_dest: "/backup/logs"

# Метод доставки: fetch (на control-node) или synchronize (rsync-pull)
collect_method: fetch

# Только для synchronize: сервер-приёмник (должен иметь SSH-доступ к остальным хостам)
collect_delegate_host: "log-server.internal"
```

---

📄 roles/collect_logs/tasks/main.yml

```yaml
---
- name: Create remote temp directory
  ansible.builtin.file:
    path: "{{ collect_remote_tmp }}"
    state: directory
    mode: "0750"

- name: Build tar excludes argument
  ansible.builtin.set_fact:
    _collect_excludes: >-
      {{ collect_log_excludes | map('regex_replace', '^(.*)$', '--exclude=\1') | join(' ') }}

- name: Create single compressed log archive (hostname + date in name)
  ansible.builtin.shell: |
    tar --ignore-failed-read -czf "{{ collect_remote_tmp }}/{{ collect_archive_name }}" \
        {{ _collect_excludes }} {{ collect_log_paths | join(' ') }} 2>/dev/null || true
    test -s "{{ collect_remote_tmp }}/{{ collect_archive_name }}"
  args:
    executable: /bin/bash
  become: true
  register: archive_result
  changed_when: archive_result.rc == 0

- name: Make archive readable by ansible user
  ansible.builtin.file:
    path: "{{ collect_remote_tmp }}/{{ collect_archive_name }}"
    mode: "0644"
  become: true

- name: Ensure destination directory exists
  ansible.builtin.file:
    path: "{{ collect_dest }}"
    state: directory
    mode: "0755"
  delegate_to: "{{ 'localhost' if collect_method == 'fetch' else collect_delegate_host }}"
  run_once: true

- name: Download archive to control-node (fetch)
  ansible.builtin.fetch:
    src: "{{ collect_remote_tmp }}/{{ collect_archive_name }}"
    dest: "{{ collect_dest }}/"
    flat: true                       # имя уже уникально (hostname + date)
    fail_on_missing: true
  when: collect_method == 'fetch'

- name: Download archive to dedicated log-server (rsync pull)
  ansible.posix.synchronize:
    src: "{{ collect_remote_tmp }}/{{ collect_archive_name }}"
    dest: "{{ collect_dest }}/"
    mode: pull
    rsync_opts:
      - "--compress"
  when: collect_method == 'synchronize'
  delegate_to: "{{ collect_delegate_host }}"

- name: Cleanup remote temp directory
  ansible.builtin.file:
    path: "{{ collect_remote_tmp }}"
    state: absent
```

---

📄 roles/collect_logs/meta/main.yml

```yaml
---
galaxy_info:
  role_name: collect_logs
  author: your_name
  description: Collect logs from managed hosts into a single archive (hostname + date) on a central server
  license: MIT
  min_ansible_version: "2.10"
  platforms:
    - name: EL
      versions: [all]
    - name: Ubuntu
      versions: [all]
    - name: Debian
      versions: [all]
  galaxy_tags:
    - logs
    - backup
    - archive
dependencies: []
```

---

📄 roles/collect_logs/README.md

```markdown
# collect_logs

Собирает логи с управляемых хостов в **один `.tar.gz`-архив** на каждом хосте и
скачивает его на центральный сервер. Имя архива содержит имя хоста и дату:
`web01_2026-10-27.tar.gz`.

## Переменные

| Переменная              | По умолчанию                                  | Описание                                |
|-------------------------|-----------------------------------------------|-----------------------------------------|
| `collect_log_paths`     | список логов (syslog, nginx, ...)             | Что упаковывать в архив                 |
| `collect_log_excludes`  | `["*.gz", "*.old"]`                            | Шаблоны исключений для tar              |
| `collect_archive_name`  | `{{ inventory_hostname }}_{{ date }}.tar.gz`   | Имя архива                              |
| `collect_remote_tmp`    | `/tmp/logs-collect`                            | Временный каталог на хосте              |
| `collect_dest`          | `/backup/logs`                                | Куда складывать архивы                  |
| `collect_method`        | `fetch`                                        | `fetch` или `synchronize`               |
| `collect_delegate_host` | `log-server.internal`                          | Сервер-приёмник для `synchronize`       |

## Требования

- `become: true` в плейбуке (логи в `/var/log/*` требует root).
- Для `synchronize`: `log-server.internal` должен иметь SSH-доступ к остальным
  хостам под тем же `remote_user` (pull выполняется **с** log-server).
- Для `synchronize` нужна коллекция `ansible.posix` (входит в штатный набор
  `ansible-core`, но при необходимости: `ansible-galaxy collection install ansible.posix`).

## Пример использования

```yaml
- hosts: all
  become: true
  roles:
    - collect_logs
  vars:
    collect_dest: "/central-logs"
    collect_method: synchronize
    collect_delegate_host: "log-server.internal"
```
```

---

⚙️ Пример плейбука

```yaml
# playbooks/collect_logs.yml
- name: Collect all logs into single archives on central server
  hosts: all
  become: true
  gather_facts: true
  roles:
    - collect_logs
  vars:
    collect_dest: "/central-logs"
    collect_method: synchronize
    collect_delegate_host: "log-server.internal"
```

Запуск:

```bash
ansible-playbook -i inventory playbooks/collect_logs.yml
```

---

📦 Результат на log-server.internal

```text
/central-logs/
├── web01_2026-10-27.tar.gz
├── web02_2026-10-27.tar.gz
├── db01_2026-10-27.tar.gz
└── ...
```

Проверить содержимое архива:

```bash
tar -tzf /central-logs/web01_2026-10-27.tar.gz
```

---

🧠 Что учтено

· Один архив с одного хоста. Все указанные в collect_log_paths файлы/каталоги попадают в один .tar.gz.
· Имя содержит hostname и дату. inventory_hostname_YYYY-MM-DD.tar.gz — уникально и наглядно.
· --ignore-failed-read — tar не падает, если части логов нет (например, нет myapp/*.log). Файл всё равно создаётся из существующих логов.
· test -s — если в итоге архив не создан или пуст, задача завершится ошибкой (защита от пустых скачиваний).
· mode: 0644 после создания — архив создан под root, но должен быть читаем для fetch/rsync от имени remote_user.
· flat: true у fetch — не создаёт hostname/-подкаталог, потому что имя сервера уже в имени архива.
· rsync --compress — экономит трафик при больших архивах.
· Очистка /tmp/logs-collect — удаляется только после успешной доставки. Если доставка упала — архив останется для отладки.
· delegate_to для synchronize — rsync выполняется на самом log-server.internal, pull тянет данные с управляемого хоста. Соответственно, log-server должен иметь SSH-доступ к остальным хостам.
· run_once: true для создания collect_dest — каталог создаётся один раз, а не N раз параллельно.