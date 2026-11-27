Ansible-роль os_update (финальная версия)

Ниже — полная структура роли со всеми исправлениями. Роль определяет Ubuntu / Oracle Linux, корректно обрабатывает блокировки пакетных менеджеров, идемпотентна и логирует результаты.

📁 Структура

```text
os_update/
├── README.md
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── debian.yml
│   └── redhat.yml
└── vars/
    └── main.yml
```

---

📄 defaults/main.yml

```yaml
---
# Директория для логов обновления на управляющем узле
os_update_log_dir: "{{ playbook_dir }}/logs"

# Перезагружать ли хост после обновления при необходимости
os_update_reboot: true

# Таймауты перезагрузки (секунды)
os_update_reboot_timeout: 600
os_update_post_reboot_delay: 30

# Тип обновления APT: dist (полное) или safe (только upgrade)
os_update_apt_upgrade_type: dist
```

📄 vars/main.yml

```yaml
---
# Внутренние параметры (не предназначены для переопределения)
os_update_dpkg_lock_retries: 60
os_update_dpkg_lock_delay: 5
os_update_dnf_lock_timeout: 300
```

📄 tasks/main.yml

```yaml
---
- name: Ensure OS facts are gathered
  ansible.builtin.setup:
    gather_subset:
      - min
  when: ansible_distribution is not defined

- name: Validate that OS is supported
  ansible.builtin.assert:
    that:
      - ansible_distribution in ['Ubuntu', 'OracleLinux']
    fail_msg: >-
      Unsupported OS: {{ ansible_distribution | default('unknown') }}
      {{ ansible_distribution_version | default('') }}.
      This role supports only Ubuntu and Oracle Linux.

- name: Warn about untested OS versions
  ansible.builtin.debug:
    msg: >-
      WARNING: {{ ansible_distribution }} {{ ansible_distribution_version }}
      is not in tested versions (Ubuntu 22.04/24.04, Oracle Linux 8/9).
      Proceeding with the generic update procedure.
  when: >
    (ansible_distribution == 'Ubuntu' and ansible_distribution_major_version not in ['22', '24']) or
    (ansible_distribution == 'OracleLinux' and ansible_distribution_major_version not in ['8', '9'])

- name: Ensure log directory exists on control node
  ansible.builtin.file:
    path: "{{ os_update_log_dir }}"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true

- name: Include OS-specific update tasks
  ansible.builtin.include_tasks: "{{ ansible_os_family | lower }}.yml"

- name: Check for reboot-required flag file (Ubuntu)
  ansible.builtin.stat:
    path: /var/run/reboot-required
  register: reboot_required_file
  when: ansible_distribution == 'Ubuntu'

- name: Check if reboot is required (Oracle Linux)
  ansible.builtin.command:
    cmd: needs-restarting -r
  register: reboot_required_cmd
  changed_when: false
  failed_when: false
  when: ansible_distribution == 'OracleLinux'

- name: Reboot host if required
  ansible.builtin.debug:
    msg: "Reboot is required — triggering handler"
  changed_when: true
  notify: Reboot host
  when:
    - os_update_reboot | bool
    - >
      (ansible_distribution == 'Ubuntu' and reboot_required_file.stat.exists) or
      (ansible_distribution == 'OracleLinux' and reboot_required_cmd.rc | default(0) == 1)
```

📄 tasks/debian.yml

```yaml
---
- name: Update apt cache (with lock retries)
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600
  register: apt_cache_result
  until: apt_cache_result is success
  retries: "{{ os_update_dpkg_lock_retries }}"
  delay: "{{ os_update_dpkg_lock_delay }}"

- name: Ensure psmisc is installed (provides fuser)
  ansible.builtin.apt:
    name: psmisc
    state: present
  register: psmisc_install
  until: psmisc_install is success
  retries: "{{ os_update_dpkg_lock_retries }}"
  delay: "{{ os_update_dpkg_lock_delay }}"

- name: Wait for dpkg locks to be released
  ansible.builtin.shell: >
    fuser /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock
    /var/cache/apt/archives/lock 2>/dev/null
  register: dpkg_lock_check
  until: dpkg_lock_check.rc != 0
  retries: "{{ os_update_dpkg_lock_retries }}"
  delay: "{{ os_update_dpkg_lock_delay }}"
  changed_when: false
  failed_when: false

- name: Get list of held packages
  ansible.builtin.command: apt-mark showhold
  register: held_packages
  changed_when: false
  failed_when: false

- name: Unhold packages if any are held
  ansible.builtin.command: >-
    apt-mark unhold {{ held_packages.stdout_lines | join(' ') }}
  when: held_packages.stdout | default('') | trim | length > 0
  register: unhold_result
  changed_when: true

- name: Update apt cache before upgrade
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 0
  register: apt_update_result
  until: apt_update_result is success
  retries: "{{ os_update_dpkg_lock_retries }}"
  delay: "{{ os_update_dpkg_lock_delay }}"

- name: Upgrade all packages
  ansible.builtin.apt:
    upgrade: "{{ os_update_apt_upgrade_type }}"
    update_cache: no
    autoremove: yes
  register: apt_upgrade_result
  until: apt_upgrade_result is success
  retries: 5
  delay: 30
  environment:
    DEBIAN_FRONTEND: noninteractive

- name: Write update log to control node
  ansible.builtin.copy:
    content: |
      Timestamp: {{ ansible_date_time.iso8601 }}
      Host: {{ inventory_hostname }}
      OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
      === Held packages ===
      {{ held_packages.stdout | default('No held packages') }}
      === Unhold operation ===
      {{ unhold_result.stdout | default('No unhold performed') }}
      {{ unhold_result.stderr | default('') }}
      === APT update ===
      {{ apt_update_result.stdout | default('') }}
      {{ apt_update_result.stderr | default('') }}
      === APT upgrade ===
      {{ apt_upgrade_result.stdout | default('') }}
      {{ apt_upgrade_result.stderr | default('') }}
    dest: "{{ os_update_log_dir }}/{{ inventory_hostname }}_os_update_{{ ansible_date_time.epoch }}.log"
    mode: '0644'
  delegate_to: localhost
```

📄 tasks/redhat.yml

```yaml
---
- name: Wait for yum/dnf lock file to disappear
  ansible.builtin.wait_for:
    path: /var/run/yum.pid
    state: absent
    timeout: "{{ os_update_dnf_lock_timeout }}"

- name: Ensure dnf-plugins-core is installed (provides versionlock)
  ansible.builtin.dnf:
    name: dnf-plugins-core
    state: present
  register: dnf_plugins_install
  until: dnf_plugins_install is success
  retries: 30
  delay: 10

- name: Try to install dnf-utils (provides needs-restarting)
  ansible.builtin.dnf:
    name: dnf-utils
    state: present
  register: dnf_utils_install
  failed_when: false
  changed_when: dnf_utils_install is success

- name: Check for dnf versionlock entries
  ansible.builtin.command: dnf versionlock list
  register: versionlock_list
  changed_when: false
  failed_when: false

- name: Clear versionlock entries if any are present
  ansible.builtin.command: dnf versionlock clear
  when: versionlock_list.stdout | default('') | trim | length > 0
  register: versionlock_clear_result
  changed_when: true

- name: Update all installed packages
  ansible.builtin.dnf:
    name: '*'
    state: latest
    update_cache: yes
    update_only: yes
  register: dnf_upgrade_result

- name: Write update log to control node
  ansible.builtin.copy:
    content: |
      Timestamp: {{ ansible_date_time.iso8601 }}
      Host: {{ inventory_hostname }}
      OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
      === Versionlock entries before update ===
      {{ versionlock_list.stdout | default('No versionlock entries') }}
      === Versionlock clear operation ===
      {{ versionlock_clear_result.stdout | default('No clear performed') }}
      {{ versionlock_clear_result.stderr | default('') }}
      === DNF upgrade ===
      {{ dnf_upgrade_result.stdout | default('') }}
      {{ dnf_upgrade_result.stderr | default('') }}
    dest: "{{ os_update_log_dir }}/{{ inventory_hostname }}_os_update_{{ ansible_date_time.epoch }}.log"
    mode: '0644'
  delegate_to: localhost
```

📄 handlers/main.yml

```yaml
---
- name: Reboot host
  ansible.builtin.reboot:
    reboot_timeout: "{{ os_update_reboot_timeout }}"
    post_reboot_delay: "{{ os_update_post_reboot_delay }}"
```

📄 meta/main.yml

```yaml
---
galaxy_info:
  author: your_name
  description: >-
    Update OS packages for Ubuntu 22.04/24.04 and Oracle Linux 8/9.
    Handles package-manager locks, versionlock, held packages and reboot detection.
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: Ubuntu
      versions:
        - jammy
        - noble
    - name: OracleLinux
      versions:
        - "8"
        - "9"
  galaxy_tags:
    - update
    - upgrade
    - security
    - ubuntu
    - oraclelinux
dependencies: []
```

📄 README.md

```markdown
# os_update

Ansible-роль для обновления пакетов в **Ubuntu 22.04/24.04** и **Oracle Linux 8/9**.

## Что делает роль

1. Валидирует ОС (только Ubuntu и Oracle Linux).
2. Определяет семейство ОС (`ansible_os_family`) и подключает `debian.yml` / `redhat.yml`.
3. **Ubuntu**: ждёт снятия блокировок `dpkg`, снимает `apt-mark hold`, обновляет кэш, делает `apt upgrade` (`dist` по умолчанию).
4. **Oracle Linux**: ждёт снятия `yum.pid`, устанавливает `dnf-plugins-core`, очищает `versionlock`, обновляет `*` через `dnf` с `update_only: yes`.
5. Проверяет необходимость перезагрузки (`/var/run/reboot-required` на Ubuntu, `needs-restarting -r` на Oracle Linux).
6. Пишет подробные логи на управляющий узел.

## Переменные (`defaults/main.yml`)

| Переменная | По умолчанию | Описание |
|---|---|---|
| `os_update_log_dir` | `{{ playbook_dir }}/logs` | Куда писать логи |
| `os_update_reboot` | `true` | Перезагружать ли при необходимости |
| `os_update_reboot_timeout` | `600` | Таймаут ожидания перезагрузки |
| `os_update_post_reboot_delay` | `30` | Пауза после перезагрузки |
| `os_update_apt_upgrade_type` | `dist` | `dist` (полное) или `safe` (только upgrade) |

## Пример плейбука

```yaml
- hosts: all
  become: true
  roles:
    - os_update
```

Требования

· Ansible ≥ 2.14.
· Дополнительные коллекции не требуются (используются только ansible.builtin).
· Для Oracle Linux: dnf-plugins-core (роль установит сама).
· Для Ubuntu: psmisc (роль установит сама).

Ограничения

· Для Oracle Linux 9 needs-restarting может отсутствовать в dnf-utils — в этом случае
  роль не будет инициировать перезагрузку по needs-restarting (безопасное поведение).
  При необходимости установите пакет, содержащий needs-restarting, вручную.
· apt upgrade: dist может удалять пакеты, если того требует разрешение зависимостей.
  Используйте os_update_apt_upgrade_type: safe, чтобы исключить удаление.

```

---

### 🛠️ Итоговые исправления по сравнению с первой версией

| Проблема | Решение |
|---|---|
| В `main.yml` лог-директория создавалась **после** включения задач обновления — логи не писались | Создание директории перенесено **до** `include_tasks` |
| `wait_for: state=absent` на lock-файлы dpkg **не работает** (файлы существуют всегда) | Заменено на `fuser` + `until: rc != 0` (ждём снятия реальной блокировки) |
| `fuser` может отсутствовать | Добавлена установка `psmisc` с `retries` |
| `apt` и `dnf` падали при занятой блокировке | Добавлены `retries`/`until` на все вызовы пакетных менеджеров |
| `dnf versionlock clear` сообщал `changed` даже при отсутствии блокировок | Сначала проверка `dnf versionlock list`, затем условный `clear` |
| `needs-restarting` мог не установиться (`dnf-utils` отсутствует в OL9) | Установка `dnf-utils` с `failed_when: false`; проверка `rc == 1` безопасно пропускается |
| Валидация не различала Ubuntu и Oracle Linux явно | В `assert` проверяется `ansible_distribution in ['Ubuntu', 'OracleLinux']` |
| Нет предупреждения о непротестированных версиях | Добавлена задача `Warn about untested OS versions` |
| Переменные были захардкожены в `vars` | Вынесены в `defaults`, внутренние — в `vars` |

Роль готова к использованию, поддерживает `--check` (за исключением обновления пакетов — стандартное поведение модулей `apt`/`dnf`), корректно работает с `become: true` и не требует внешних коллекций.
```
__________
__________

Чтобы роль не зависала на интерактивном запросе при обновлении PAM (и любых других конфигурационных файлов), нужно автоматически выбирать вариант «оставить текущую версию» (N / keep). Решение отличается для Ubuntu и Oracle Linux, поэтому добавляем соответствующую логику в обе ветки роли.

⚙️ 1. Новые переменные в defaults/main.yml

```yaml
---
# ... существующие переменные ...

# Стратегия разрешения конфликтов конфигурационных файлов в dpkg (Ubuntu)
# force-confold: сохранить текущую версию (N)
# force-confdef: если файл не изменялся, принять новую версию автоматически
os_update_dpkg_options: "force-confold,force-confdef"

# Отключать ли плагин rpmconf в DNF (Oracle Linux),
# который вызывает интерактивные запросы при обновлении
os_update_disable_rpmconf: true
```

📄 2. Обновлённый tasks/debian.yml (Ubuntu)

```yaml
---
- name: Update apt cache (with lock retries)
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600
    dpkg_options: "{{ os_update_dpkg_options }}"   # <-- ключевое изменение
  register: apt_cache_result
  until: apt_cache_result is success
  retries: "{{ os_update_dpkg_lock_retries }}"
  delay: "{{ os_update_dpkg_lock_delay }}"

- name: Ensure psmisc is installed (provides fuser)
  ansible.builtin.apt:
    name: psmisc
    state: present
    dpkg_options: "{{ os_update_dpkg_options }}"
  register: psmisc_install
  until: psmisc_install is success
  retries: "{{ os_update_dpkg_lock_retries }}"
  delay: "{{ os_update_dpkg_lock_delay }}"

- name: Wait for dpkg locks to be released
  ansible.builtin.shell: >
    fuser /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock
    /var/cache/apt/archives/lock 2>/dev/null
  register: dpkg_lock_check
  until: dpkg_lock_check.rc != 0
  retries: "{{ os_update_dpkg_lock_retries }}"
  delay: "{{ os_update_dpkg_lock_delay }}"
  changed_when: false
  failed_when: false

- name: Get list of held packages
  ansible.builtin.command: apt-mark showhold
  register: held_packages
  changed_when: false
  failed_when: false

- name: Unhold packages if any are held
  ansible.builtin.command: >-
    apt-mark unhold {{ held_packages.stdout_lines | join(' ') }}
  when: held_packages.stdout | default('') | trim | length > 0
  register: unhold_result
  changed_when: true

- name: Update apt cache before upgrade
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 0
    dpkg_options: "{{ os_update_dpkg_options }}"
  register: apt_update_result
  until: apt_update_result is success
  retries: "{{ os_update_dpkg_lock_retries }}"
  delay: "{{ os_update_dpkg_lock_delay }}"

- name: Upgrade all packages
  ansible.builtin.apt:
    upgrade: "{{ os_update_apt_upgrade_type }}"
    update_cache: no
    autoremove: yes
    dpkg_options: "{{ os_update_dpkg_options }}"   # <-- ключевое изменение
  register: apt_upgrade_result
  until: apt_upgrade_result is success
  retries: 5
  delay: 30
  environment:
    DEBIAN_FRONTEND: noninteractive
    UCF_FORCE_CONFOLD: "1"          # <-- дополнительная страховка для ucf-файлов
```

Что делает dpkg_options: "force-confold,force-confdef":

· force-confold — при конфликте всегда оставляет текущую версию файла (выбор N), а новый файл сохраняется с суффиксом .dpkg-dist.
· force-confdef — позволяет dpkg самостоятельно принимать новую версию, если файл не был изменён локально, и запрашивать только при реальном конфликте. В сочетании с force-confold это исключает любые интерактивные запросы.

📄 3. Обновлённый tasks/redhat.yml (Oracle Linux)

```yaml
---
- name: Wait for yum/dnf lock file to disappear
  ansible.builtin.wait_for:
    path: /var/run/yum.pid
    state: absent
    timeout: "{{ os_update_dnf_lock_timeout }}"

- name: Ensure dnf-plugins-core is installed (provides versionlock)
  ansible.builtin.dnf:
    name: dnf-plugins-core
    state: present
  register: dnf_plugins_install
  until: dnf_plugins_install is success
  retries: 30
  delay: 10

- name: Try to install dnf-utils (provides needs-restarting)
  ansible.builtin.dnf:
    name: dnf-utils
    state: present
  register: dnf_utils_install
  failed_when: false
  changed_when: dnf_utils_install is success

# --- Отключаем rpmconf, чтобы DNF не задавал вопросы по PAM и другим файлам ---
- name: Ensure /etc/dnf/plugins directory exists
  ansible.builtin.file:
    path: /etc/dnf/plugins
    state: directory
    mode: '0755'
  when: os_update_disable_rpmconf | bool

- name: Disable rpmconf plugin to avoid interactive prompts
  ansible.builtin.copy:
    dest: /etc/dnf/plugins/rpmconf.conf
    content: |
      [main]
      enabled = 0
    mode: '0644'
  when: os_update_disable_rpmconf | bool

- name: Check for dnf versionlock entries
  ansible.builtin.command: dnf versionlock list
  register: versionlock_list
  changed_when: false
  failed_when: false

- name: Clear versionlock entries if any are present
  ansible.builtin.command: dnf versionlock clear
  when: versionlock_list.stdout | default('') | trim | length > 0
  register: versionlock_clear_result
  changed_when: true

- name: Update all installed packages
  ansible.builtin.dnf:
    name: '*'
    state: latest
    update_cache: yes
    update_only: yes
  register: dnf_upgrade_result

# ... остальные задачи (логирование) без изменений ...
```

Почему это решает проблему:

· В Oracle Linux за интерактивные запросы при обновлении PAM (и других конфигурационных файлов) отвечает плагин rpmconf из пакета dnf-plugins-extras.
· Отключение плагина через /etc/dnf/plugins/rpmconf.conf (enabled = 0) заставляет DNF работать в штатном неинтерактивном режиме. При этом RPM по умолчанию сохраняет текущую версию конфигурационного файла (аналог выбора N), а новая версия сохраняется как .rpmnew.

✅ Итог

После этих изменений роль:

· На Ubuntu передаёт dpkg_options: "force-confold,force-confdef" во все вызовы apt и дополнительно выставляет UCF_FORCE_CONFOLD=1, поэтому на запрос по PAM автоматически выбирается «оставить текущую версию» (N).
· На Oracle Linux отключает плагин rpmconf, и DNF больше не показывает интерактивных запросов, сохраняя текущие конфигурационные файлы.

Если вы хотите, чтобы роль всегда принимала новую версию конфигурации (выбор Y), измените os_update_dpkg_options на "force-confnew" для Ubuntu и не отключайте rpmconf для Oracle Linux — но в контексте вашего запроса (выбор отрицательный) текущая настройка является правильной.


________
________
________

Объединённая роль os_update с управлением KESL

Роль теперь временно останавливает файловый монитор KESL (если он установлен) на время обновления ОС и гарантированно возвращает его в исходное состояние через block/always, даже при ошибке обновления. Если KESL не установлен — обновление выполняется в обычном режиме.

📁 Структура

```text
os_update/
├── README.md
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── debian.yml
│   └── redhat.yml
└── vars/
    └── main.yml
```

---

📄 defaults/main.yml

```yaml
---
# Директория для логов обновления на управляющем узле
os_update_log_dir: "{{ playbook_dir }}/logs"

# Перезагружать ли хост после обновления при необходимости
os_update_reboot: true
os_update_reboot_timeout: 600
os_update_post_reboot_delay: 30

# Тип обновления APT: dist (полное) или safe (только upgrade)
os_update_apt_upgrade_type: dist

# Стратегия разрешения конфликтов конфигов dpkg (Ubuntu)
# force-confold сохраняет текущую версию (N), force-confdef — авто-принятие если файл не менялся
os_update_dpkg_options: "force-confold,force-confdef"

# Отключать ли плагин rpmconf в DNF (Oracle Linux) — он вызывает интерактивные запросы
os_update_disable_rpmconf: true

# --- KESL (Kaspersky Endpoint Security for Linux) ---
# Управлять ли задачами KESL во время обновления
os_update_kesl_manage: true

# Путь к kesl-control
os_update_kesl_bin: /usr/bin/kesl-control

# Шаблон имени/типа задачи для поиска ID (File_Threat_Protection или OAS)
os_update_kesl_task_pattern: "/Name: File_Threat_Protection|Type: OAS/"
```

📄 vars/main.yml

```yaml
---
os_update_dpkg_lock_retries: 60
os_update_dpkg_lock_delay: 5
os_update_dnf_lock_timeout: 300
```

📄 tasks/main.yml

```yaml
---
# --- 1. Подготовка ---
- name: Ensure OS facts are gathered
  ansible.builtin.setup:
    gather_subset:
      - min
  when: ansible_distribution is not defined

- name: Validate that OS is supported
  ansible.builtin.assert:
    that:
      - ansible_distribution in ['Ubuntu', 'OracleLinux']
    fail_msg: >-
      Unsupported OS: {{ ansible_distribution | default('unknown') }}
      {{ ansible_distribution_version | default('') }}.
      This role supports only Ubuntu and Oracle Linux.

- name: Warn about untested OS versions
  ansible.builtin.debug:
    msg: >-
      WARNING: {{ ansible_distribution }} {{ ansible_distribution_version }}
      is not in tested versions (Ubuntu 22.04/24.04, Oracle Linux 8/9).
      Proceeding with the generic update procedure.
  when: >
    (ansible_distribution == 'Ubuntu' and ansible_distribution_major_version not in ['22', '24']) or
    (ansible_distribution == 'OracleLinux' and ansible_distribution_major_version not in ['8', '9'])

- name: Ensure log directory exists on control node
  ansible.builtin.file:
    path: "{{ os_update_log_dir }}"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true

# --- 2. Определение KESL ---
- name: Detect KESL installation
  ansible.builtin.stat:
    path: "{{ os_update_kesl_bin }}"
  register: kesl_bin

# --- 3. Блок обновления с управлением KESL ---
- name: OS update with KESL file threat protection management
  block:
    # --- 3.1. Остановка KESL ---
    - name: Get KESL OAS task ID
      ansible.builtin.shell: |
        set -o pipefail
        {{ os_update_kesl_bin }} --get-task-list \
          | awk '{{ os_update_kesl_task_pattern }} {found=1} found && /ID:/ {print $2; exit}'
      register: kesl_oas_task
      changed_when: false
      failed_when: false
      when:
        - os_update_kesl_manage | bool
        - kesl_bin.stat.exists

    - name: Get KESL OAS task state
      ansible.builtin.shell: >-
        {{ os_update_kesl_bin }} --get-task-state {{ kesl_oas_task.stdout }}
      register: kesl_oas_state
      changed_when: false
      failed_when: false
      when:
        - os_update_kesl_manage | bool
        - kesl_bin.stat.exists
        - kesl_oas_task.stdout | default('') | trim | length > 0

    - name: Stop KESL OAS task before OS update
      ansible.builtin.command:
        cmd: "{{ os_update_kesl_bin }} --stop-task {{ kesl_oas_task.stdout }}"
      register: kesl_stopped
      when:
        - os_update_kesl_manage | bool
        - kesl_bin.stat.exists
        - kesl_oas_task.stdout | default('') | trim | length > 0
        - (kesl_oas_state.stdout | default('')) is search('Started|Running')

    # --- 3.2. Обновление ОС ---
    - name: Include OS-specific update tasks
      ansible.builtin.include_tasks: "{{ ansible_os_family | lower }}.yml"

    # --- 3.3. Проверка необходимости перезагрузки ---
    - name: Check for reboot-required flag file (Ubuntu)
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_required_file
      when: ansible_distribution == 'Ubuntu'

    - name: Check if reboot is required (Oracle Linux)
      ansible.builtin.command:
        cmd: needs-restarting -r
      register: reboot_required_cmd
      changed_when: false
      failed_when: false
      when: ansible_distribution == 'OracleLinux'

  # --- 3.4. ГАРАНТИРОВАННЫЙ возврат KESL ---
  always:
    - name: Ensure KESL OAS task is running after OS update
      ansible.builtin.command:
        cmd: "{{ os_update_kesl_bin }} --start-task {{ kesl_oas_task.stdout }}"
      register: kesl_started
      when:
        - os_update_kesl_manage | bool
        - kesl_bin.stat.exists
        - kesl_oas_task is defined
        - kesl_oas_task.stdout | default('') | trim | length > 0
        - kesl_stopped is defined
        - kesl_stopped is changed

# --- 4. Перезагрузка при необходимости ---
- name: Reboot host if required
  ansible.builtin.debug:
    msg: "Reboot is required — triggering handler"
  changed_when: true
  notify: Reboot host
  when:
    - os_update_reboot | bool
    - >
      (ansible_distribution == 'Ubuntu' and reboot_required_file.stat.exists | default(false)) or
      (ansible_distribution == 'OracleLinux' and reboot_required_cmd.rc | default(0) == 1)
```

📄 tasks/debian.yml (с dpkg_options против запросов PAM)

```yaml
---
- name: Update apt cache (with lock retries)
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600
    dpkg_options: "{{ os_update_dpkg_options }}"
  register: apt_cache_result
  until: apt_cache_result is success
  retries: "{{ os_update_dpkg_lock_retries }}"
  delay: "{{ os_update_dpkg_lock_delay }}"

- name: Ensure psmisc is installed (provides fuser)
  ansible.builtin.apt:
    name: psmisc
    state: present
    dpkg_options: "{{ os_update_dpkg_options }}"
  register: psmisc_install
  until: psmisc_install is success
  retries: "{{ os_update_dpkg_lock_retries }}"
  delay: "{{ os_update_dpkg_lock_delay }}"

- name: Wait for dpkg locks to be released
  ansible.builtin.shell: >
    fuser /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock
    /var/cache/apt/archives/lock 2>/dev/null
  register: dpkg_lock_check
  until: dpkg_lock_check.rc != 0
  retries: "{{ os_update_dpkg_lock_retries }}"
  delay: "{{ os_update_dpkg_lock_delay }}"
  changed_when: false
  failed_when: false

- name: Get list of held packages
  ansible.builtin.command: apt-mark showhold
  register: held_packages
  changed_when: false
  failed_when: false

- name: Unhold packages if any are held
  ansible.builtin.command: >-
    apt-mark unhold {{ held_packages.stdout_lines | join(' ') }}
  when: held_packages.stdout | default('') | trim | length > 0
  register: unhold_result
  changed_when: true

- name: Update apt cache before upgrade
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 0
    dpkg_options: "{{ os_update_dpkg_options }}"
  register: apt_update_result
  until: apt_update_result is success
  retries: "{{ os_update_dpkg_lock_retries }}"
  delay: "{{ os_update_dpkg_lock_delay }}"

- name: Upgrade all packages
  ansible.builtin.apt:
    upgrade: "{{ os_update_apt_upgrade_type }}"
    update_cache: no
    autoremove: yes
    dpkg_options: "{{ os_update_dpkg_options }}"
  register: apt_upgrade_result
  until: apt_upgrade_result is success
  retries: 5
  delay: 30
  environment:
    DEBIAN_FRONTEND: noninteractive
    UCF_FORCE_CONFOLD: "1"

- name: Write update log to control node
  ansible.builtin.copy:
    content: |
      Timestamp: {{ ansible_date_time.iso8601 }}
      Host: {{ inventory_hostname }}
      OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
      === Held packages ===
      {{ held_packages.stdout | default('No held packages') }}
      === Unhold operation ===
      {{ unhold_result.stdout | default('No unhold performed') }}
      {{ unhold_result.stderr | default('') }}
      === APT update ===
      {{ apt_update_result.stdout | default('') }}
      {{ apt_update_result.stderr | default('') }}
      === APT upgrade ===
      {{ apt_upgrade_result.stdout | default('') }}
      {{ apt_upgrade_result.stderr | default('') }}
    dest: "{{ os_update_log_dir }}/{{ inventory_hostname }}_os_update_{{ ansible_date_time.epoch }}.log"
    mode: '0644'
  delegate_to: localhost
```

📄 tasks/redhat.yml (с отключением rpmconf против запросов PAM)

```yaml
---
- name: Wait for yum/dnf lock file to disappear
  ansible.builtin.wait_for:
    path: /var/run/yum.pid
    state: absent
    timeout: "{{ os_update_dnf_lock_timeout }}"

- name: Ensure dnf-plugins-core is installed
  ansible.builtin.dnf:
    name: dnf-plugins-core
    state: present
  register: dnf_plugins_install
  until: dnf_plugins_install is success
  retries: 30
  delay: 10

- name: Try to install dnf-utils (provides needs-restarting)
  ansible.builtin.dnf:
    name: dnf-utils
    state: present
  register: dnf_utils_install
  failed_when: false
  changed_when: dnf_utils_install is success

# --- Отключаем rpmconf, чтобы DNF не задавал вопросов по PAM ---
- name: Ensure /etc/dnf/plugins directory exists
  ansible.builtin.file:
    path: /etc/dnf/plugins
    state: directory
    mode: '0755'
  when: os_update_disable_rpmconf | bool

- name: Disable rpmconf plugin to avoid interactive prompts
  ansible.builtin.copy:
    dest: /etc/dnf/plugins/rpmconf.conf
    content: |
      [main]
      enabled = 0
    mode: '0644'
  when: os_update_disable_rpmconf | bool

- name: Check for dnf versionlock entries
  ansible.builtin.command: dnf versionlock list
  register: versionlock_list
  changed_when: false
  failed_when: false

- name: Clear versionlock entries if any are present
  ansible.builtin.command: dnf versionlock clear
  when: versionlock_list.stdout | default('') | trim | length > 0
  register: versionlock_clear_result
  changed_when: true

- name: Update all installed packages
  ansible.builtin.dnf:
    name: '*'
    state: latest
    update_cache: yes
    update_only: yes
  register: dnf_upgrade_result

- name: Write update log to control node
  ansible.builtin.copy:
    content: |
      Timestamp: {{ ansible_date_time.iso8601 }}
      Host: {{ inventory_hostname }}
      OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
      === Versionlock entries ===
      {{ versionlock_list.stdout | default('No versionlock entries') }}
      === Versionlock clear ===
      {{ versionlock_clear_result.stdout | default('No clear performed') }}
      {{ versionlock_clear_result.stderr | default('') }}
      === DNF upgrade ===
      {{ dnf_upgrade_result.stdout | default('') }}
      {{ dnf_upgrade_result.stderr | default('') }}
    dest: "{{ os_update_log_dir }}/{{ inventory_hostname }}_os_update_{{ ansible_date_time.epoch }}.log"
    mode: '0644'
  delegate_to: localhost
```

📄 handlers/main.yml

```yaml
---
- name: Reboot host
  ansible.builtin.reboot:
    reboot_timeout: "{{ os_update_reboot_timeout }}"
    post_reboot_delay: "{{ os_update_post_reboot_delay }}"
```

📄 meta/main.yml

```yaml
---
galaxy_info:
  author: your_name
  description: >-
    Update OS packages for Ubuntu 22.04/24.04 and Oracle Linux 8/9,
    with optional KESL file threat protection management, non-interactive
    config conflict resolution (PAM) and reboot handling.
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: Ubuntu
      versions:
        - jammy
        - noble
    - name: OracleLinux
      versions:
        - "8"
        - "9"
  galaxy_tags:
    - update
    - upgrade
    - security
    - kesl
    - ubuntu
    - oraclelinux
dependencies: []
```

📄 README.md

```markdown
# os_update

Обновление пакетов в **Ubuntu 22.04/24.04** и **Oracle Linux 8/9** с:
- корректной обработкой блокировок пакетных менеджеров;
- неинтерактивным разрешением конфликтов конфигов (PAM и др.);
- **временной остановкой KESL** (если установлен) на время обновления
  и гарантированным возвратом через `block/always`;
- проверкой необходимости перезагрузки и логированием.

## Переменные (`defaults/main.yml`)

| Переменная | По умолчанию | Описание |
|---|---|---|
| `os_update_log_dir` | `{{ playbook_dir }}/logs` | Куда писать логи |
| `os_update_reboot` | `true` | Перезагружать при необходимости |
| `os_update_reboot_timeout` | `600` | Таймаут перезагрузки |
| `os_update_post_reboot_delay` | `30` | Пауза после перезагрузки |
| `os_update_apt_upgrade_type` | `dist` | `dist` или `safe` |
| `os_update_dpkg_options` | `force-confold,force-confdef` | Стратегия конфликтов конфигов (PAM → N) |
| `os_update_disable_rpmconf` | `true` | Отключение `rpmconf` (OL) |
| `os_update_kesl_manage` | `true` | Управлять ли KESL во время обновления |
| `os_update_kesl_bin` | `/usr/bin/kesl-control` | Путь к `kesl-control` |

## Логика KESL

1. Если `/usr/bin/kesl-control` существует — определяется ID задачи `OAS`
   (`File_Threat_Protection`), проверяется её состояние.
2. Если задача **запущена** — она останавливается перед обновлением.
3. Обновление ОС выполняется в блоке `block`.
4. В `always` (срабатывает **всегда**, даже при ошибке обновления) задача
   **возвращается в running**, если мы её останавливали. Если она изначально
   была выключена — роль её **не запускает** (уважаем намеренное отключение).
5. Если KESL не установлен — обновление выполняется без изменений.

## Пример плейбука

```yaml
- hosts: all
  become: true
  roles:
    - os_update
```

## Важно

- Роль требует `become: true`.
- При использовании KSC с «замком» на локальное управление задачами
  `kesl-control --stop-task` вернёт ошибку доступа. Роль не упадёт на этом
  (используется `failed_when: false` при обнаружении), но KESL не будет
  остановлен — обновление при этом выполнится.
- Для Ubuntu `dist-upgrade` может удалять пакеты; используйте
  `os_update_apt_upgrade_type: safe`, если это недопустимо.
```

---

🔍 Ключевые решения при объединении

Вопрос Решение
Как гарантировать запуск KESL обратно? block/always вокруг всего блока обновления (включая include_tasks, reboot-check)
Что если KESL не установлен? Задачи KESL помечены when: kesl_bin.stat.exists, include_tasks выполняется всегда
Что если первая KESL-задача упала? failed_when: false на discovery + проверки kesl_oas_task.stdout \| default('') \| length > 0 — последующие задачи просто пропускаются
Что если KESL был изначально остановлен? Условие остановки search('Started\|Running') не сработает, kesl_stopped.changed = False, always не запустит задачу (уважаем намеренное отключение)
Что если include_tasks упал? always всё равно выполнится → KESL вернётся в работу
Что если KESL остановить не удалось (KSC lock)? Stop-task упадёт, always не запустит (т.к. kesl_stopped.changed = False), задача KESL останется в исходном состоянии
Как избежать вопросов по PAM? Ubuntu: dpkg_options: force-confold,force-confdef + UCF_FORCE_CONFOLD=1; OL: отключение rpmconf
Где выполняется reboot? После always (KESL уже возвращён), через handler

Роль готова к запуску. Достаточно скопировать файлы и при необходимости переопределить os_update_kesl_manage: false, если управление KESL не требуется.

_____
_____
_____
_____

Роль дополнена очисткой старых ядер с сохранением только двух последних версий. Логика встроена в существующий block/always с KESL: очистка выполняется после обновления пакетов, но до проверки необходимости перезагрузки, чтобы не удалить только что установленное ядро.

⚙️ 1. Новые переменные в defaults/main.yml

```yaml
---
# ... существующие переменные ...

# --- Очистка старых ядер ---
# Удалять ли старые версии ядра после обновления
os_update_remove_old_kernels: true

# Сколько последних версий ядра оставлять (минимум 2)
os_update_keep_kernel_nr: 2

# Настраивать ли installonly_limit в dnf.conf (Oracle Linux)
os_update_configure_dnf_installonly: true
```

📄 2. Обновлённый tasks/debian.yml (Ubuntu)

Добавляем задачи в конец debian.yml (после Upgrade all packages и до логирования):

```yaml
# ... существующие задачи (apt cache, psmisc, locks, held, upgrade) ...

# --- Очистка старых ядер (Ubuntu) ---
- name: Get list of installed kernel packages
  ansible.builtin.shell: |
    set -o pipefail
    dpkg-query -W -f='${Package} ${Status}\n' 'linux-image-*' \
      | awk '$NF == "installed" {print $1}' \
      | grep -vE 'linux-image-(generic|virtual|aws|azure|gcp|kvm|oem|oracle|lowlatency)$' \
      | sort -V
  register: ubuntu_kernels_raw
  changed_when: false
  failed_when: false
  when: os_update_remove_old_kernels | bool

- name: Determine kernels to keep (current + previous + latest)
  ansible.builtin.set_fact:
    ubuntu_kernels_to_remove: >-
      {{
        ubuntu_kernels_raw.stdout_lines
        | reject('search', ansible_kernel)
        | list
        | sort
        | slice(0, ([0, (ubuntu_kernels_raw.stdout_lines | length) - (os_update_keep_kernel_nr | int)] | max))
      }}
  when:
    - os_update_remove_old_kernels | bool
    - ubuntu_kernels_raw.stdout_lines | default([]) | length > (os_update_keep_kernel_nr | int)

- name: Remove old kernel packages (keep last {{ os_update_keep_kernel_nr }})
  ansible.builtin.apt:
    name: "{{ ubuntu_kernels_to_remove }}"
    state: absent
    purge: true
    autoremove: true
    dpkg_options: "{{ os_update_dpkg_options }}"
  when:
    - os_update_remove_old_kernels | bool
    - ubuntu_kernels_to_remove | default([]) | length > 0
  register: ubuntu_kernel_cleanup

- name: Log kernel cleanup result
  ansible.builtin.debug:
    msg: >-
      Removed old kernels: {{ ubuntu_kernels_to_remove | default([]) | join(', ') }}
      (kept {{ os_update_keep_kernel_nr }} newest + running kernel)
  when:
    - os_update_remove_old_kernels | bool
    - ubuntu_kernel_cleanup is changed
```

Почему это безопасно для Ubuntu:

· Исключается текущее работающее ядро (reject('search', ansible_kernel)).
· Мета-пакеты (linux-image-generic и т.п.) не удаляются — только конкретные версии.
· Удаляются только те версии, что старше последних os_update_keep_kernel_nr.
· purge: true удаляет и конфигурационные файлы, освобождая /boot.
· Если ядер меньше или равно лимиту — очистка не выполняется.

📄 3. Обновлённый tasks/redhat.yml (Oracle Linux)

Добавляем задачи в конец redhat.yml:

```yaml
# ... существующие задачи (lock, dnf-plugins-core, versionlock, upgrade) ...

# --- Очистка старых ядер (Oracle Linux) ---
- name: Configure installonly_limit in dnf.conf (keep {{ os_update_keep_kernel_nr }} kernels)
  ansible.builtin.lineinfile:
    path: /etc/dnf/dnf.conf
    regexp: '^installonly_limit\s*='
    line: "installonly_limit={{ os_update_keep_kernel_nr }}"
    state: present
  when:
    - os_update_remove_old_kernels | bool
    - os_update_configure_dnf_installonly | bool

- name: Remove old installonly packages (kernels) exceeding limit
  ansible.builtin.command:
    cmd: >-
      dnf remove -y --oldinstallonly
      --setopt installonly_limit={{ os_update_keep_kernel_nr }} kernel
  register: redhat_kernel_cleanup
  changed_when: redhat_kernel_cleanup.stdout | default('') is search('Removed|Удалено')
  failed_when: false
  when: os_update_remove_old_kernels | bool

- name: Log kernel cleanup result
  ansible.builtin.debug:
    msg: >-
      DNF oldinstallonly cleanup: {{ redhat_kernel_cleanup.stdout | default('no output') }}
  when:
    - os_update_remove_old_kernels | bool
    - redhat_kernel_cleanup is defined
    - redhat_kernel_cleanup.stdout | default('') | trim | length > 0
```

Почему это безопасно для Oracle Linux:

· dnf remove --oldinstallonly автоматически пропускает текущее работающее ядро (документированное поведение).
· --setopt installonly_limit=2 гарантирует, что останутся ровно две последние версии (включая текущую).
· Настройка installonly_limit в /etc/dnf/dnf.conf предотвращает накопление старых ядер в будущем.
· failed_when: false — если удалять нечего, задача не считается упавшей.

📄 4. Обновлённый tasks/main.yml (фрагмент с очисткой)

Очистка ядер должна выполняться внутри block, после include_tasks, но до проверки перезагрузки:


```yaml
    # --- 3.2. Обновление ОС (включает очистку ядер в debian.yml / redhat.yml) ---
    - name: Include OS-specific update tasks
      ansible.builtin.include_tasks: "{{ ansible_os_family | lower }}.yml"

    # --- 3.3. Проверка необходимости перезагрузки ---
    - name: Check for reboot-required flag file (Ubuntu)
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_required_file
      when: ansible_distribution == 'Ubuntu'
    # ... остальные задачи блока ...
```

Очистка уже встроена в debian.yml / redhat.yml, поэтому в main.yml дополнительных задач не требуется — include_tasks выполнит их в нужном порядке.

📄 5. Обновлённый README.md (фрагмент)

Добавьте в таблицу переменных:

Переменная По умолчанию Описание
os_update_remove_old_kernels true Удалять старые ядра после обновления
os_update_keep_kernel_nr 2 Сколько последних версий ядра оставлять (мин. 2)
os_update_configure_dnf_installonly true Настраивать installonly_limit в dnf.conf (OL)

И раздел:

```markdown
## Очистка старых ядер

После обновления пакетов роль удаляет старые версии ядра,
оставляя `os_update_keep_kernel_nr` последних (по умолчанию **2**).

- **Ubuntu**: удаляются конкретные пакеты `linux-image-<version>`,
  кроме текущего работающего ядра и последних N версий. Мета-пакеты
  (`linux-image-generic` и т.п.) не затрагиваются.
- **Oracle Linux**: выполняется `dnf remove --oldinstallonly kernel`
  с `installonly_limit=2`, а также настраивается `installonly_limit=2`
  в `/etc/dnf/dnf.conf` для предотвращения накопления ядер в будущем.
  Текущее работающее ядро никогда не удаляется.

Чтобы отключить очистку:

```yaml
os_update_remove_old_kernels: false
```

```

---

### ✅ Итог

| ОС | Команда очистки | Защита текущего ядра | Лимит |
|---|---|---|---|
| **Ubuntu** | `apt purge` конкретных `linux-image-*` | Исключается через `ansible_kernel` | `os_update_keep_kernel_nr` |
| **Oracle Linux** | `dnf remove --oldinstallonly` | Встроено в DNF[reference:1] | `installonly_limit=2` + `dnf.conf` |
````
Роль теперь полностью автоматизирует цикл **обновление → очистка старых ядер → проверка перезагрузки → возврат KESL**, сохраняя только две последние версии ядра на обеих поддерживаемых ОС.

_______
_______
_______
_______


Ошибка fatal: FAILED! при выполнении apt-mark unhold возникает из-за того, что один или несколько перечисленных пакетов не установлены в системе (например, ядра для gke или azure на текущем хосте), либо отсутствуют в кэше APT. Команда apt-mark завершается с ненулевым кодом возврата, если не может найти хотя бы один пакет из аргументов, что Ansible воспринимает как фатальный сбой.

Чтобы роль не падала и корректно снимала удержание только с реально существующих пакетов, необходимо отфильтровать список перед вызовом apt-mark.

🛠️ Исправленная задача в tasks/debian.yml

Замените блок задач, связанный с удержанием (Get list of held packages и Unhold packages if any are held), на следующий:

```yaml
- name: Get list of held packages
  ansible.builtin.command: apt-mark showhold
  register: held_packages
  changed_when: false
  failed_when: false

- name: Filter held packages that are actually installed
  ansible.builtin.set_fact:
    installed_held_packages: >-
      {{
        held_packages.stdout_lines
        | select('in', ansible_facts.packages.keys() | default([]))
        | list
      }}
  when: held_packages.stdout | default('') | trim | length > 0

- name: Unhold packages if any are held and installed
  ansible.builtin.command: >-
    apt-mark unhold {{ installed_held_packages | join(' ') }}
  when:
    - installed_held_packages is defined
    - installed_held_packages | length > 0
  register: unhold_result
  changed_when: true
  failed_when: false
```

Важно: Для работы ansible_facts.packages необходимо, чтобы в системе был установлен пакет python3-apt (обычно он уже присутствует в стандартной поставке Ansible для Ubuntu). Если факты о пакетах не собраны, их можно предварительно собрать задачей ansible.builtin.package_facts: manager=auto.

💡 Альтернативный (упрощённый) вариант

Если не хочется использовать package_facts, можно просто игнорировать ошибку снятия удержания, логируя её:

```yaml
- name: Unhold packages if any are held
  ansible.builtin.command: >-
    apt-mark unhold {{ held_packages.stdout_lines | join(' ') }}
  when: held_packages.stdout | default('') | trim | length > 0
  register: unhold_result
  changed_when: true
  failed_when: false

- name: Warn if unhold failed
  ansible.builtin.debug:
    msg: "Не удалось снять удержание с некоторых пакетов (возможно, они не установлены): {{ unhold_result.stderr | default('') }}"
  when:
    - unhold_result is defined
    - unhold_result.rc is defined
    - unhold_result.rc != 0
```

В этом случае роль продолжит обновление, но в логах останется предупреждение о том, какие именно пакеты не удалось обработать.

🔍 Почему это произошло на вашем хосте

На скриншоте видны пакеты linux-headers-5.15.0-1032-gke, linux-headers-5.15.0-1038-azure, linux-headers-5.15.0-1044-azure — это заголовки ядер для специфических облачных платформ (Google Kubernetes Engine, Microsoft Azure). На вашем сервере они помечены как «удерживаемые» (hold), но фактически отсутствуют в системе. apt-mark showhold показывает их, потому что запись об удержании осталась в базе dpkg, а команда unhold не может найти сами пакеты для обработки.

Исправленная задача выше решит эту проблему: она будет снимать удержание только с тех пакетов, которые реально установлены, и не будет падать на отсутствующих.


___



Ошибка 'dict object' has no attribute 'packages' возникает из-за того, что встроенный факт ansible_facts.packages не заполняется автоматически при стандартном сборе фактов. Его нужно явно собрать с помощью модуля ansible.builtin.package_facts.

Также важно учитывать: если вызвать package_facts и сразу в том же задании обратиться к ansible_facts.packages, факт может не успеть обновиться в текущем контексте. Решение — разделить сбор фактов и фильтрацию на два отдельных шага.

🛠️ Исправленный фрагмент для tasks/debian.yml

Замените текущий блок фильтрации (Filter held packages that are actually installed) на следующие три задачи:

```yaml
- name: Get list of held packages
  ansible.builtin.command: apt-mark showhold
  register: held_packages
  changed_when: false
  failed_when: false

- name: Gather installed packages facts
  ansible.builtin.package_facts:
    manager: auto
  when: held_packages.stdout | default('') | trim | length > 0

- name: Filter held packages that are actually installed
  ansible.builtin.set_fact:
    installed_held_packages: >-
      {{
        held_packages.stdout_lines
        | select('in', ansible_facts.packages | default({}) | list)
        | list
      }}
  when: held_packages.stdout | default('') | trim | length > 0

- name: Unhold packages if any are held and installed
  ansible.builtin.command: >-
    apt-mark unhold {{ installed_held_packages | join(' ') }}
  when:
    - installed_held_packages is defined
    - installed_held_packages | length > 0
  register: unhold_result
  changed_when: true
  failed_when: false
```

Что изменилось:

· Добавлен ansible.builtin.package_facts (manager: auto) — он заполняет ansible_facts.packages списком установленных пакетов.
· Добавлена защита | default({}) на случай, если факт по какой-то причине не собрался — тогда фильтр просто вернёт пустой список вместо падения.
· Сбор фактов вынесен в отдельную задачу перед set_fact, чтобы избежать проблемы с асинхронным обновлением фактов.

После этого роль будет снимать удержание только с тех пакетов, которые реально установлены, и не будет падать на отсутствующих пакетах (как в случае с linux-headers-...-gke и ...-azure на ваших серверах).

_____


Ошибка на скриншоте — это не сбой сети и не поломка роли, а классическая ситуация, когда роль корректно ждёт освобождения блокировки dpkg, но на целевых хостах в этот момент уже работает фоновое обновление.

Сообщения sftp transfer mechanism failed — это лишь предупреждение о резервном методе копирования файлов (Ansible переключается на scp), оно не ломает обновление. Основная проблема — таймаут ожидания dpkg.

🔍 Почему происходит таймаут блокировки

1. Фоновые обновления Ubuntu: на серверах работают таймеры unattended-upgrades (apt-daily.timer, apt-daily-upgrade.timer), которые удерживают dpkg долгое время. Ваша проверка fuser видит активный процесс и начинает ждать.
2. Слишком короткое окно ожидания: 60 попыток по 5 секунд дают всего 5 минут. Установка обновлений безопасности (особенно ядра) может занимать больше времени.
3. Неэффективный цикл: конструкция с until: dpkg_lock_check.rc != 0 может работать нестабильно, если fuser завершается с кодом 0 даже при отсутствии блокировки (например, из-за прав доступа).

🛠️ Исправление в tasks/debian.yml

Замените блок Wait for dpkg locks to be released на следующий, а также добавьте lock_timeout в модули apt:

```yaml
- name: Wait for dpkg locks to be released
  ansible.builtin.shell: |
    set -o pipefail
    while fuser /var/lib/dpkg/lock-frontend \
                /var/lib/dpkg/lock \
                /var/cache/apt/archives/lock >/dev/null 2>&1; do
      sleep 5
    done
  register: dpkg_lock_wait
  retries: 120           # 120 * 10 сек = 20 минут
  delay: 10
  until: dpkg_lock_wait.rc == 0
  changed_when: false
  failed_when: false
```

И добавьте lock_timeout в задачи Update apt cache before upgrade и Upgrade all packages:

```yaml
- name: Update apt cache before upgrade
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 0
    dpkg_options: "{{ os_update_dpkg_options }}"
    lock_timeout: 300          # <-- ключевое добавление
  register: apt_update_result
  until: apt_update_result is success
  retries: 10
  delay: 30

- name: Upgrade all packages
  ansible.builtin.apt:
    upgrade: "{{ os_update_apt_upgrade_type }}"
    update_cache: no
    autoremove: yes
    dpkg_options: "{{ os_update_dpkg_options }}"
    lock_timeout: 600          # <-- ключевое добавление
  register: apt_upgrade_result
  until: apt_upgrade_result is success
  retries: 10
  delay: 30
  environment:
    DEBIAN_FRONTEND: noninteractive
```

lock_timeout передаёт apt опцию -o DPkg::Lock::Timeout=..., и apt сам ждёт освобождения блокировки вместо мгновенного падения.

⚙️ Устранение предупреждений sftp

Чтобы убрать предупреждения, добавьте в ansible.cfg (или в переменные окружения):

```ini
[ssh_connection]
scp_if_ssh = True
```

Это заставит Ansible сразу использовать scp вместо sftp, что часто работает стабильнее на серверах с нестандартной конфигурацией SSH.

💡 Дополнительная рекомендация

Если обновления выполняются на многих хостах одновременно, добавьте serial: 1 (или serial: 10%) в плейбук — это снизит конкуренцию за блокировки на каждом хосте.

После этих изменений роль будет ждать до 20 минут снятия блокировки и не упадёт, если фоновое обновление затянется.