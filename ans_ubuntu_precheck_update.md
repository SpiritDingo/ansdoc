Я подготовил для вас роль Ansible, которая выполняет всестороннюю проверку системы Ubuntu перед запуском обновления. Роль останавливает выполнение (fail), если обнаруживает проблемы, и выдаёт предупреждения, если есть риски.

Структура роли

```
pre_update_checks/
├── defaults/
│   └── main.yml          # Пороговые значения и флаги включения проверок
├── tasks/
│   └── main.yml          # Основной список задач
└── README.md             # Описание (при необходимости)
```

1. defaults/main.yml

Переменные по умолчанию — вы можете переопределить их в плейбуке.

```yaml
---
# Пороги свободного места (в мегабайтах)
precheck_disk_boot_min_mb: 100
precheck_disk_root_min_mb: 500
precheck_disk_apt_cache_min_mb: 200
precheck_disk_tmp_min_mb: 100

# Проверять наличие удерживаемых (held) пакетов
precheck_held_packages: true

# Проверять сломанные пакеты (dpkg --audit)
precheck_broken_packages: true

# Проверять незавершённые конфигурации dpkg
precheck_pending_config: true

# Проверять, что нет блокировок apt/dpkg
precheck_no_locks: true

# Проверять, требуется ли перезагрузка
precheck_reboot_required: true

# Проверять доступность репозиториев (ping до archive.ubuntu.com)
precheck_repo_connectivity: true

# Проверять, не запущен ли unattended-upgrades
precheck_unattended_upgrades: true

# Максимально допустимое количество ожидающих обновлений (0 – только предупреждение, >0 – ошибка при превышении)
precheck_max_upgradable: 0

# Если true – роль упадёт при превышении precheck_max_upgradable, иначе только предупредит
precheck_strict_upgradable_limit: false
```

2. tasks/main.yml

Файл с логикой проверок. Каждая проверка изолирована и снабжена понятными сообщениями.

```yaml
---
- name: Проверка свободного места на /boot
  ansible.builtin.assert:
    that:
      - item.mount == '/boot' and (item.size_available | int) >= (precheck_disk_boot_min_mb * 1024 * 1024)
    fail_msg: >
      Недостаточно места на /boot!
      Доступно {{ ((item.size_available | int) / 1024 / 1024) | round(2) }} МБ,
      требуется минимум {{ precheck_disk_boot_min_mb }} МБ.
    success_msg: "Место на /boot в норме."
  loop: "{{ ansible_mounts }}"
  when: item.mount == '/boot'
  tags: [disk]

- name: Проверка свободного места на /
  ansible.builtin.assert:
    that:
      - item.mount == '/' and (item.size_available | int) >= (precheck_disk_root_min_mb * 1024 * 1024)
    fail_msg: >
      Недостаточно места на корневом разделе!
      Доступно {{ ((item.size_available | int) / 1024 / 1024) | round(2) }} МБ,
      требуется минимум {{ precheck_disk_root_min_mb }} МБ.
    success_msg: "Место на / в норме."
  loop: "{{ ansible_mounts }}"
  when: item.mount == '/'
  tags: [disk]

- name: Проверка свободного места в /var/cache/apt
  ansible.builtin.assert:
    that:
      - item.mount == '/var/cache/apt' and (item.size_available | int) >= (precheck_disk_apt_cache_min_mb * 1024 * 1024)
    fail_msg: >
      Недостаточно места в кэше apt!
      Доступно {{ ((item.size_available | int) / 1024 / 1024) | round(2) }} МБ,
      требуется минимум {{ precheck_disk_apt_cache_min_mb }} МБ.
    success_msg: "Место в /var/cache/apt в норме."
  loop: "{{ ansible_mounts }}"
  when:
    - item.mount == '/var/cache/apt' or item.mount == '/var' or item.mount == '/'
    # Уточняем: если отдельный раздел для /var/cache/apt или /var, иначе корень
    - (item.mount == '/var/cache/apt') or
      (item.mount == '/var' and '/var/cache/apt' not in ansible_mounts | map(attribute='mount') | list) or
      (item.mount == '/' and '/var' not in ansible_mounts | map(attribute='mount') | list)
  tags: [disk]
  # Этот блок упрощён: если /var/cache/apt не примонтирован отдельно, используем раздел, содержащий этот путь.
  # Для более точной проверки можно заменить на модуль find или shell, но здесь достаточно.

- name: Проверка блокировок apt/dpkg
  when: precheck_no_locks
  block:
    - name: Проверить наличие lock-файлов
      ansible.builtin.stat:
        path: "{{ item }}"
      loop:
        - /var/lib/dpkg/lock-frontend
        - /var/lib/dpkg/lock
        - /var/cache/apt/archives/lock
      register: lock_files

    - name: Убедиться, что lock-файлы отсутствуют
      ansible.builtin.assert:
        that:
          - not item.stat.exists
        fail_msg: "Обнаружена блокировка: {{ item.item }}. Возможно, работает другой менеджер пакетов."
        success_msg: "Блокировки apt/dpkg отсутствуют."
      loop: "{{ lock_files.results }}"
  tags: [locks]

- name: Проверка удерживаемых пакетов (held)
  when: precheck_held_packages
  block:
    - name: Получить список held-пакетов
      ansible.builtin.command: apt-mark showhold
      register: held_packages
      changed_when: false

    - name: Предупредить о наличии held-пакетов
      ansible.builtin.assert:
        that:
          - held_packages.stdout_lines | length == 0
        fail_msg: >
          Обнаружены удерживаемые пакеты. Это может помешать обновлению.
          Список: {{ held_packages.stdout_lines | join(', ') }}
        success_msg: "Удерживаемых пакетов нет."
  tags: [packages]

- name: Проверка сломанных пакетов
  when: precheck_broken_packages
  block:
    - name: Запустить dpkg --audit
      ansible.builtin.command: dpkg --audit
      register: broken_check
      changed_when: false
      failed_when: false  # возвращает 1, если есть проблемы

    - name: Остановить выполнение при сломанных пакетах
      ansible.builtin.assert:
        that:
          - broken_check.rc == 0 and broken_check.stdout == ''
        fail_msg: "dpkg обнаружил проблемы с пакетами. Вывод: {{ broken_check.stdout }}"
        success_msg: "Сломанные пакеты отсутствуют."
  tags: [packages]

- name: Проверка незавершённых конфигураций dpkg
  when: precheck_pending_config
  block:
    - name: Узнать, есть ли пакеты в состоянии half-configured
      ansible.builtin.command: dpkg --configure --pending --dry-run
      register: pending_config
      changed_when: false

    - name: Убедиться, что нет отложенных конфигураций
      ansible.builtin.assert:
        that:
          - pending_config.stdout == ''
        fail_msg: "Есть пакеты, ожидающие настройки. Запустите 'dpkg --configure -a'."
        success_msg: "Отложенных конфигураций dpkg нет."
  tags: [packages]

- name: Проверка необходимости перезагрузки
  when: precheck_reboot_required
  block:
    - name: Проверить наличие флага перезагрузки
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_flag

    - name: Предупредить, если требуется перезагрузка
      ansible.builtin.fail:
        msg: "Система требует перезагрузки (файл /var/run/reboot-required). Обновление может быть небезопасным."
      when: reboot_flag.stat.exists
  tags: [system]

- name: Проверка доступности репозиториев
  when: precheck_repo_connectivity
  block:
    - name: Пинг archive.ubuntu.com
      ansible.builtin.command: ping -c 2 -W 3 archive.ubuntu.com
      register: ping_result
      changed_when: false
      failed_when: false

    - name: Убедиться, что репозиторий доступен
      ansible.builtin.assert:
        that:
          - ping_result.rc == 0
        fail_msg: "Нет связи с archive.ubuntu.com. Проверьте сеть и DNS."
        success_msg: "Репозиторий archive.ubuntu.com доступен."
  tags: [network]

- name: Проверка, не запущен ли unattended-upgrades
  when: precheck_unattended_upgrades
  block:
    - name: Поиск процесса unattended-upgrades
      ansible.builtin.shell: pgrep -f unattended-upgrade || true
      register: uu_process
      changed_when: false

    - name: Убедиться, что unattended-upgrades не выполняется
      ansible.builtin.assert:
        that:
          - uu_process.stdout == ''
        fail_msg: "Фоновый процесс unattended-upgrades активен. Дождитесь его завершения."
        success_msg: "unattended-upgrades не запущен."
  tags: [packages]

- name: Проверка количества доступных обновлений (опционально)
  when: precheck_max_upgradable > 0
  block:
    - name: Обновить кэш apt
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600
      changed_when: false

    - name: Получить список обновляемых пакетов
      ansible.builtin.command: apt list --upgradable 2>/dev/null
      register: upgradable_list
      changed_when: false

    - name: Подсчитать количество обновлений
      ansible.builtin.set_fact:
        upgradable_count: "{{ (upgradable_list.stdout_lines | length) - 1 }}"  # первая строка "Listing..."

    - name: Сравнить с лимитом
      ansible.builtin.fail:
        msg: >
          Количество доступных обновлений ({{ upgradable_count }}) превышает допустимый лимит ({{ precheck_max_upgradable }}).
      when: precheck_strict_upgradable_limit and (upgradable_count | int) > (precheck_max_upgradable | int)

    - name: Предупреждение о большом количестве обновлений
      ansible.builtin.debug:
        msg: >
          Внимание: доступно {{ upgradable_count }} обновлений (лимит {{ precheck_max_upgradable }}).
          Обновление может занять много времени.
      when: not precheck_strict_upgradable_limit and (upgradable_count | int) > (precheck_max_upgradable | int)
  tags: [updates]
```

Как использовать роль

Пример плейбука:

```yaml
---
- hosts: ubuntu_servers
  become: yes
  vars:
    precheck_disk_root_min_mb: 1024   # 1 ГБ минимум на /
    precheck_reboot_required: true
    precheck_max_upgradable: 50
    precheck_strict_upgradable_limit: true   # упадёт, если обновлений больше 50
  roles:
    - pre_update_checks

- hosts: ubuntu_servers
  become: yes
  tasks:
    - name: Обновление пакетов
      ansible.builtin.apt:
        upgrade: dist
        update_cache: yes
      when: not ansible_check_mode  # защита на случай повторного запуска в dry-run
```

Запуск проверки отдельно:

```bash
ansible-playbook -i inventory precheck.yml --tags disk,locks,packages
```

Примечания

· Проверка свободного места для /var/cache/apt реализована упрощённо. В промышленном варианте можно заменить на модуль ansible.builtin.shell с df или использовать факты ansible_mounts более аккуратно.
· Роль безопасна для check_mode (все команды changed_when: false).
· Если хотите использовать роль в CI, проверьте, что целевая система доступна по SSH и у неё есть Python.

Если нужно добавить другие проверки (например, целостность файловой системы или наличие конкретных сервисов), дайте знать — я дополню.