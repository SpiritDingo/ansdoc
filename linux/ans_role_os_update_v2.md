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