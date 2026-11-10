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

_________
_________

Добавим в роль collect_logs опцию сохранения собранных архивов в локальный Nexus (raw-репозиторий) через REST API. Загрузка будет выполняться с control-node (или с хоста, указанного в delegate_to) после того, как архивы уже собраны и скачаны.

---

📄 Обновлённый defaults/main.yml

```yaml
# ... (остальные переменные без изменений)

# --- Настройки загрузки в Nexus ---
# Включить/выключить загрузку
nexus_upload: false

# Базовый URL Nexus (без завершающего слэша)
nexus_url: "https://nexus.example.com"

# Имя raw-репозитория, куда складывать архивы
nexus_repository: "logs-raw"

# Учётные данные (лучше хранить в Ansible Vault)
nexus_username: "admin"
nexus_password: "admin123"

# Путь внутри репозитория (опционально). Если задан, файлы будут лежать в подкаталоге.
# Пример: "collected/{{ ansible_date_time.date }}"
nexus_path_prefix: ""
```

---

📄 Обновлённый tasks/main.yml

```yaml
---
# ... (все предыдущие задачи сбора и скачивания без изменений)

- name: Upload archive to Nexus (raw repository)
  ansible.builtin.uri:
    url: "{{ nexus_url }}/repository/{{ nexus_repository }}/{{ nexus_path_prefix | default('') }}{{ nexus_path_prefix | default('') | ternary('/', '') }}{{ collect_archive_name }}"
    method: PUT
    src: "{{ collect_dest }}/{{ collect_archive_name }}"
    user: "{{ nexus_username }}"
    password: "{{ nexus_password }}"
    force_basic_auth: yes
    headers:
      Content-Type: application/octet-stream
    status_code: 201
  when:
    - nexus_upload | bool
    - collect_method == 'fetch'          # если архивы на control-node
  delegate_to: localhost
  run_once: false

- name: Upload archive to Nexus from dedicated log-server (if synchronize)
  ansible.builtin.uri:
    url: "{{ nexus_url }}/repository/{{ nexus_repository }}/{{ nexus_path_prefix | default('') }}{{ nexus_path_prefix | default('') | ternary('/', '') }}{{ collect_archive_name }}"
    method: PUT
    src: "{{ collect_dest }}/{{ collect_archive_name }}"
    user: "{{ nexus_username }}"
    password: "{{ nexus_password }}"
    force_basic_auth: yes
    headers:
      Content-Type: application/octet-stream
    status_code: 201
  when:
    - nexus_upload | bool
    - collect_method == 'synchronize'
  delegate_to: "{{ collect_delegate_host }}"
```

Пояснение по URL: при nexus_path_prefix = "collected/2026-10-27" файл попадёт в .../repository/logs-raw/collected/2026-10-27/web01_2026-10-27.tar.gz. Если префикс не нужен — оставьте его пустым, и архив ляжет в корень репозитория.

---

⚙️ Пример плейбука с загрузкой в Nexus

```yaml
- name: Collect logs and upload archives to local Nexus
  hosts: all
  become: true
  gather_facts: true
  roles:
    - collect_logs
  vars:
    collect_dest: "/central-logs"
    collect_method: synchronize
    collect_delegate_host: "log-server.internal"

    # Включаем загрузку в Nexus
    nexus_upload: true
    nexus_url: "https://nexus.internal.company"
    nexus_repository: "logs-raw"
    nexus_path_prefix: "collected/{{ ansible_date_time.date }}"
    nexus_username: "svc-ansible"
    # пароль лучше вынести в Ansible Vault:
    # nexus_password: !vault |
    #   $ANSIBLE_VAULT;1.1;AES256
    #   ...
```

---

🧠 Что учтено

· Два сценария доставки. Если архивы скачиваются через fetch (на control-node), загрузка в Nexus идёт с localhost. Если через synchronize (на выделенный log-server), загрузка выполняется с этого же сервера (delegate_to: collect_delegate_host), чтобы не перекачивать архивы повторно на control-node.
· Метод PUT для raw-репозитория. Nexus 3 поддерживает прямую загрузку в raw-репозитории через PUT /repository/<repo>/<path> — это самый простой и надёжный способ. Модуль uri с src полностью заменяет curl --upload-file.
· force_basic_auth: yes — обязательный параметр для корректной аутентификации в Nexus через uri (частая причина ошибок 403).
· Content-Type: application/octet-stream — гарантирует, что Nexus не попытается интерпретировать .tar.gz как форму или JSON.
· Статус-код 201 — ожидаемый ответ Nexus при успешной загрузке нового файла. Если файл уже существует, Nexus вернёт 400/409; при необходимости можно добавить --keep или обрабатывать перезапись отдельно (по умолчанию raw-репозиторий может запрещать перезапись — см. поведение Nexus).
· nexus_path_prefix — позволяет автоматически раскладывать архивы по датам внутри репозитория, что удобно для последующего поиска.
· Пароль рекомендуется хранить в Ansible Vault или передавать через переменные окружения (например, NEXUS_PASSWORD), а не в открытом виде в defaults.

---

📁 Итоговая структура роли

```text
roles/collect_logs/
├── defaults/
│   └── main.yml          # добавлены nexus_upload, nexus_url, nexus_repository, nexus_path_prefix и учётные данные
├── tasks/
│   └── main.yml          # добавлены две задачи Upload archive to Nexus (для fetch и synchronize)
├── meta/
│   └── main.yml
└── README.md             # можно дополнить разделом «Загрузка в Nexus»
```

Теперь роль умеет не только собирать логи в один архив с именем <hostname>_<date>.tar.gz и складывать его на центральный сервер, но и автоматически публиковать эти архивы в локальный Nexus — достаточно установить nexus_upload: true и заполнить параметры подключения.