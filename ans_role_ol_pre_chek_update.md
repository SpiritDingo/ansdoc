Ниже представлена Ansible-роль для проверки состояния системы Oracle Linux перед обновлением. Она выполняет ряд диагностических тестов (свободное место, целостность БД RPM, блокировки dnf/yum, совместимость ядра UEK/RHCK, состояние ksplice и др.) и сигнализирует о потенциальных проблемах до того, как будет запущен dnf update.

Роль можно использовать как часть плейбука обновления или как самостоятельную проверку.

Структура роли

```
roles/
└── pre_update_check/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── defaults/
    │   └── main.yml
    └── meta/
        └── main.yml
```

1. defaults/main.yml – настройки по умолчанию

```yaml
---
# Пороги проверок
pre_update_min_free_space_mb: 1024        # Минимальный объём в МБ на /boot (обычно 200-500 МБ)
pre_update_min_free_space_root_mb: 2048   # Минимальный объём в МБ для /
pre_update_min_memory_mb: 512             # Минимальный объём ОЗУ
pre_update_check_ksplice: true            # Проверять состояние ksplice
pre_update_check_uek_compat: true         # Проверять совместимость текущего ядра с UEK
pre_update_reboot_required_check: true    # Проверять, нужна ли перезагрузка
pre_update_ignore_services: []            # Список сервисов, не вызывающих предупреждений при остановке
pre_update_required_repos:                # Репозитории, которые должны быть доступны
  - ol8_baseos_latest
  - ol8_appstream
  # Для OL7:
  # - ol7_latest
  # - ol7_UEKR6
```

2. tasks/main.yml – основные проверки

```yaml
---
- name: Факты о системе
  setup:

- name: Проверка версии Oracle Linux
  assert:
    that:
      - ansible_distribution == 'OracleLinux'
    fail_msg: "Целевая система не является Oracle Linux. Найдено: {{ ansible_distribution }}"
    success_msg: "Система Oracle Linux подтверждена (версия {{ ansible_distribution_version }})"

- name: Проверка свободного места на /
  assert:
    that:
      - mount_info.size_available > (pre_update_min_free_space_root_mb * 1024 * 1024)
    fail_msg: >
      Слишком мало места на корневом разделе (доступно: {{ (mount_info.size_available / 1024 / 1024) | int }} MB,
      требуется минимум {{ pre_update_min_free_space_root_mb }} MB)
    success_msg: "На корневом разделе достаточно свободного места"
  vars:
    mount_info: "{{ ansible_mounts | selectattr('mount', 'equalto', '/') | first }}"

- name: Проверка свободного места на /boot
  assert:
    that:
      - boot_info.size_available > (pre_update_min_free_space_mb * 1024 * 1024)
    fail_msg: >
      Мало места на /boot (доступно: {{ (boot_info.size_available / 1024 / 1024) | int }} MB,
      требуется минимум {{ pre_update_min_free_space_mb }} MB)
    success_msg: "На /boot достаточно свободного места"
  vars:
    boot_info: "{{ ansible_mounts | selectattr('mount', 'equalto', '/boot') | first | default({}) }}"
  when: boot_info | default({}) | length > 0

- name: Проверка оперативной памяти
  assert:
    that:
      - ansible_memtotal_mb > pre_update_min_memory_mb
    fail_msg: >
      Недостаточно оперативной памяти ({{ ansible_memtotal_mb }} MB, требуется минимум {{ pre_update_min_memory_mb }} MB)
    success_msg: "Достаточный объём ОЗУ: {{ ansible_memtotal_mb }} MB"

- name: Проверка наличия блокировки dnf/yum (pid файл)
  shell:
    cmd: >
      if [ -f /var/run/yum.pid ] || [ -f /var/run/dnf.pid ]; then exit 1; else exit 0; fi
  register: lock_check
  changed_when: false
  failed_when: false

- name: Сообщить об активной блокировке пакетного менеджера
  fail:
    msg: "Обнаружена активная блокировка пакетного менеджера (pid-файл существует). Дождитесь завершения или удалите блокировку."
  when: lock_check.rc != 0

- name: Проверка целостности базы RPM (dnf check)
  command: dnf check
  register: dnf_check
  changed_when: false
  failed_when: false

- name: Сообщить о проблемах в базе RPM
  fail:
    msg: >
      Обнаружены проблемы с зависимостями пакетов:
      {{ dnf_check.stderr }}
  when:
    - dnf_check.rc != 0
    - '"is ok" not in dnf_check.stderr'

- name: Проверка доступности обязательных репозиториев
  command: dnf repolist --enabled
  register: repolist
  changed_when: false
  failed_when: false

- name: Убедиться, что все обязательные репозитории включены
  fail:
    msg: "Отсутствуют обязательные репозитории: {{ missing_repos | join(', ') }}"
  vars:
    repolist_output: "{{ repolist.stdout }}"
    missing_repos: "{{ pre_update_required_repos | reject('in', repolist_output) | list }}"
  when: missing_repos | length > 0

- name: Проверка, требуется ли перезагрузка (если задано)
  block:
    - name: Поиск файла /var/run/reboot-required
      stat:
        path: /var/run/reboot-required
      register: reboot_required_file

    - name: Вывести предупреждение о необходимости перезагрузки
      debug:
        msg: >
          ВНИМАНИЕ: Система ожидает перезагрузки (найден /var/run/reboot-required).
          Рекомендуется перезагрузить систему перед обновлением.
      when: reboot_required_file.stat.exists
  when: pre_update_reboot_required_check | bool

- name: Блок Oracle-специфичных проверок
  block:
    - name: Определить тип текущего ядра (UEK / RHCK)
      shell: >
        if uname -r | grep -q 'uek'; then echo "UEK"; else echo "RHCK"; fi
      register: kernel_type
      changed_when: false

    - name: Информация о типе ядра
      debug:
        msg: "Текущее ядро: {{ kernel_type.stdout }}"

    - name: Проверить совместимость загрузочного ядра с пакетами UEK (при UEK-ядре)
      block:
        - name: Получить список установленных UEK-пакетов
          package_facts:
            manager: rpm
          register: pkg_facts

        - name: Проверить, что пакеты kernel-uek соответствуют версии работающего ядра
          assert:
            that:
              - "uname_version in installed_uek_versions"
            fail_msg: >
              Установленное ядро UEK не соответствует ни одному из пакетов kernel-uek.
              Возможен сбой загрузки после обновления.
            success_msg: "Пакет kernel-uek соответствует работающему ядру."
          vars:
            uname_version: "{{ ansible_kernel }}"
            installed_uek_versions: "{{ ansible_facts.packages | select('match', '^kernel-uek$') | map('extract', ansible_facts.packages) | flatten | map(attribute='version') | map('regex_replace', '([0-9]+\\.[0-9]+\\.[0-9]+-[0-9]+\\.[0-9]+\\.[0-9]+).*', '\\1') | list }}"
      when: kernel_type.stdout == 'UEK'

    - name: Проверить состояние Ksplice (если включено)
      block:
        - name: Установлен ли пакет uptrack-updates?
          package_facts:
            manager: rpm

        - name: Проверить состояние Ksplice
          command: uptrack-show --available
          register: ksplice_status
          changed_when: false
          failed_when: false
          when: "'uptrack-updates' in ansible_facts.packages"

        - name: Вывести предупреждение, если доступны обновления Ksplice
          debug:
            msg: >
              Ksplice доступны обновления. Рекомендуется применить их перед обновлением пакетов
              (команда: uptrack-upgrade -y).
          when:
            - "'uptrack-updates' in ansible_facts.packages"
            - ksplice_status.rc == 0
            - "'No updates' not in ksplice_status.stdout"
      when: pre_update_check_ksplice | bool

  when: ansible_distribution == 'OracleLinux'

- name: Вывод итогового сообщения
  debug:
    msg:
      - "Все предполётные проверки успешно пройдены."
      - "Система готова к обновлению."
```

3. handlers/main.yml (заглушка, если понадобятся обработчики)

```yaml
---
# В рамках проверок обработчики не используются, файл оставлен для возможного расширения.
```

4. meta/main.yml – метаинформация

```yaml
---
galaxy_info:
  author: your_name
  description: Проверка состояния Oracle Linux перед обновлением пакетов
  company: your_company
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: OracleLinux
      versions:
        - 7
        - 8
        - 9
  galaxy_tags:
    - oraclelinux
    - update
    - precheck
dependencies: []
```

Пример использования в плейбуке

```yaml
---
- name: Предобновленная проверка Oracle Linux
  hosts: ol_servers
  become: true
  roles:
    - role: pre_update_check
      vars:
        pre_update_min_free_space_mb: 500
        pre_update_required_repos:
          - ol8_baseos_latest
          - ol8_appstream
```

Запуск:

```bash
ansible-playbook -i inventory precheck.yml
```

Роль остановит выполнение с ошибкой (fail), если какое-либо критическое условие не выполнено. Некритичные замечания выводятся через debug (например, о необходимости перезагрузки). При желании можно заменить fail на debug с failed_when: false для сбора всех предупреждений без прерывания.
____

Отлично, уточняю решение именно под Oracle Linux 9.
Ниже — полностью адаптированная роль pre_update_check для OL9, с учётом специфики dnf, репозиториев, проверки UEK-ядра, ksplice, необходимости перезагрузки через dnf needs-restarting и других деталей.

Структура роли (без изменений)

```
roles/pre_update_check/
├── defaults/main.yml
├── tasks/main.yml
├── handlers/main.yml
└── meta/main.yml
```

Все файлы обновлены под OL9.

---

1. defaults/main.yml – пороги и репозитории для OL9

```yaml
---
# Пороги проверок
pre_update_min_free_space_mb: 1024        # Минимум МБ на /boot
pre_update_min_free_space_root_mb: 2048   # Минимум МБ на /
pre_update_min_memory_mb: 512             # Минимум МБ ОЗУ

pre_update_check_ksplice: true            # Проверять наличие обновлений Ksplice
pre_update_ignore_services: []            # (не используется, но оставлено для расширения)

# Репозитории, которые должны быть активны (стандартные для OL9)
pre_update_required_repos:
  - ol9_baseos_latest
  - ol9_appstream
  # Если используется UEK, можно добавить:
  # - ol9_UEKR7
```

2. tasks/main.yml – проверки специально для OL9

```yaml
---
- name: Сбор системных фактов
  setup:

- name: Проверка, что это Oracle Linux 9
  assert:
    that:
      - ansible_distribution == 'OracleLinux'
      - ansible_distribution_major_version == '9'
    fail_msg: >
      Система не Oracle Linux 9.
      Обнаружено: {{ ansible_distribution }} {{ ansible_distribution_version }}
    success_msg: "Oracle Linux 9 подтверждён (версия {{ ansible_distribution_version }})"

- name: Проверка свободного места на корневом разделе
  vars:
    root_mount: "{{ ansible_mounts | selectattr('mount', 'equalto', '/') | first }}"
  assert:
    that:
      - root_mount.size_available > (pre_update_min_free_space_root_mb * 1024 * 1024)
    fail_msg: >
      Мало места на / (доступно {{ (root_mount.size_available / 1024 / 1024) | int }} MB,
      требуется минимум {{ pre_update_min_free_space_root_mb }} MB)
    success_msg: "Достаточно места на /"

- name: Проверка свободного места на /boot
  vars:
    boot_mount: "{{ ansible_mounts | selectattr('mount', 'equalto', '/boot') | list | first | default({}) }}"
  when: boot_mount.mount is defined
  assert:
    that:
      - boot_mount.size_available > (pre_update_min_free_space_mb * 1024 * 1024)
    fail_msg: >
      Мало места на /boot (доступно {{ (boot_mount.size_available / 1024 / 1024) | int }} MB,
      требуется минимум {{ pre_update_min_free_space_mb }} MB)
    success_msg: "Достаточно места на /boot"

- name: Проверка оперативной памяти
  assert:
    that:
      - ansible_memtotal_mb > pre_update_min_memory_mb
    fail_msg: >
      Недостаточно ОЗУ ({{ ansible_memtotal_mb }} MB, требуется минимум {{ pre_update_min_memory_mb }} MB)
    success_msg: "ОЗУ в норме: {{ ansible_memtotal_mb }} MB"

- name: Поиск pid-файла dnf
  shell:
    cmd: "test -f /var/run/dnf.pid && exit 1 || exit 0"
  register: dnf_lock
  changed_when: false
  failed_when: false

- name: Блокировка пакетного менеджера
  fail:
    msg: "Обнаружена активная блокировка dnf (pid-файл). Завершите процесс или удалите файл."
  when: dnf_lock.rc != 0

- name: Проверка целостности базы RPM
  command: dnf check
  register: dnf_check
  changed_when: false
  failed_when: false

- name: Ошибка при проблемах с зависимостями
  fail:
    msg: >
      Проблемы в базе RPM:
      {{ dnf_check.stderr }}
  when:
    - dnf_check.rc != 0
    - '"Check completed" not in dnf_check.stderr'

- name: Проверка доступности обязательных репозиториев
  command: dnf repolist --enabled
  register: repolist
  changed_when: false

- name: Убедиться, что все нужные репозитории включены
  fail:
    msg: "Отсутствуют обязательные репозитории: {{ missing_repos | join(', ') }}"
  vars:
    repolist_text: "{{ repolist.stdout }}"
    missing_repos: "{{ pre_update_required_repos | reject('in', repolist_text) | list }}"
  when: missing_repos | length > 0

- name: Проверка необходимости перезагрузки (через dnf needs-restarting -r)
  command: dnf needs-restarting -r
  register: reboot_needed
  changed_when: false
  failed_when: false
  # Команда возвращает 0, если перезагрузка нужна, 1 — если нет, и 2 при ошибке
  # В случае ошибки просто предупредим
  ignore_errors: yes

- name: Предупреждение о необходимости перезагрузки
  debug:
    msg: >
      ВНИМАНИЕ: требуется перезагрузка системы перед обновлением.
      (выполните reboot или проверьте dnf needs-restarting -r)
  when:
    - reboot_needed.rc == 0

# ---------- Специфичные проверки Oracle Linux 9 ----------
- name: Блок проверок Oracle Linux 9
  when: ansible_distribution == 'OracleLinux'
  block:

    - name: Определение типа ядра (UEK / RHCK)
      shell: uname -r | grep -q uek && echo UEK || echo RHCK
      register: kernel_type
      changed_when: false

    - debug:
        msg: "Текущее ядро: {{ kernel_type.stdout }}"

    - name: Проверить соответствие установленного UEK‑пакета работающему ядру
      block:
        - name: Узнать версию установленного пакета kernel-uek
          command: rpm -q kernel-uek
          register: kernel_uek_pkg
          changed_when: false
          failed_when: false

        - name: Сравнить версию UEK с работающим ядром
          assert:
            that:
              - ansible_kernel in kernel_uek_pkg.stdout
            fail_msg: >
              Установленный пакет kernel-uek ({{ kernel_uek_pkg.stdout }}) не соответствует
              работающему ядру ({{ ansible_kernel }}). Возможен сбой загрузки после обновления.
            success_msg: "Пакет kernel-uek совпадает с работающим ядром."
      when: kernel_type.stdout == 'UEK'

    - name: Проверка состояния Ksplice
      when: pre_update_check_ksplice | bool
      block:
        - name: Проверить наличие пакета uptrack-updates
          package_facts:
            manager: rpm

        - name: Получить доступные ksplice-обновления
          command: uptrack-show --available
          register: ksplice_output
          changed_when: false
          failed_when: false
          when: "'uptrack-updates' in ansible_facts.packages"

        - name: Предупредить о неприменённых Ksplice-обновлениях
          debug:
            msg: >
              Доступны обновления Ksplice. Рекомендуется применить их до обновления пакетов:
              uptrack-upgrade -y
          when:
            - "'uptrack-updates' in ansible_facts.packages"
            - ksplice_output.rc == 0
            - "'No updates' not in ksplice_output.stdout"
            - "'Effective kernel version is' in ksplice_output.stdout"

- name: Итоговое сообщение
  debug:
    msg:
      - "Все предполётные проверки пройдены успешно."
      - "Система Oracle Linux 9 готова к обновлению."
```

3. handlers/main.yml

Оставлено пустым — в данной роли обработчики не требуются.

```yaml
---
# Обработчики не задействованы, файл для возможного расширения.
```

4. meta/main.yml

```yaml
---
galaxy_info:
  author: your_name
  description: Предобновленная проверка Oracle Linux 9
  company: your_company
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: OracleLinux
      versions:
        - 9
  galaxy_tags:
    - oraclelinux
    - ol9
    - update
    - precheck
dependencies: []
```

---

Пример использования

```yaml
---
- name: Проверка OL9 перед обновлением
  hosts: ol9_servers
  become: yes
  roles:
    - role: pre_update_check
      vars:
        pre_update_min_free_space_mb: 512
        pre_update_required_repos:
          - ol9_baseos_latest
          - ol9_appstream
          - ol9_UEKR7
```

Запуск:

```bash
ansible-playbook -i inventory precheck-ol9.yml
```

Роль прервётся с ошибкой при любом критическом несоответствии, либо выведет предупреждения (например, о необходимости перезагрузки или о доступных ksplice) и продолжит работу. При желании поведение можно сделать полностью информационным, заменив fail на debug с failed_when: false.