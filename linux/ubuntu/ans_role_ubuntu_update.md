Ниже представлена улучшенная роль Ansible system_update для безопасного обновления Ubuntu 22.04 / 24.04 с формированием детального отчёта. Роль решает типичные проблемы: блокировки dpkg, неудовлетворённые зависимости, нехватку места, необходимость перезагрузки, и делает процесс идемпотентным.

Структура роли

```
roles/system_update/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── report.j2
└── handlers
    └── main.yml
```

defaults/main.yml

```yaml
---
# Тип обновления: safe, full, dist
update_upgrade_type: safe

# Автоматическое удаление старых пакетов и очистка кэша
update_autoremove: true
update_autoclean: true

# Разрешить перезагрузку, если требуется (по умолчанию нет)
update_reboot_allowed: false

# Генерировать отчёт
update_generate_report: true
update_report_dir: /var/log/ansible-update

# Таймауты и повторы
update_lock_timeout: 300  # секунд на ожидание освобождения блокировок
update_retries: 3
update_delay: 10
```

tasks/main.yml

```yaml
---
- name: Проверка совместимости
  assert:
    that:
      - ansible_distribution == 'Ubuntu'
      - ansible_distribution_version in ['22.04', '24.04']
    fail_msg: "Роль поддерживает только Ubuntu 22.04 / 24.04. Текущая ОС: {{ ansible_distribution }} {{ ansible_distribution_version }}"

- name: Убедиться, что директория для отчётов существует
  file:
    path: "{{ update_report_dir }}"
    state: directory
    mode: '0755'
  when: update_generate_report | bool

- name: Ожидание снятия блокировок APT/DPKG
  shell: |
    for lock in /var/lib/dpkg/lock /var/lib/apt/lists/lock /var/cache/apt/archives/lock; do
      while fuser "$lock" >/dev/null 2>&1; do
        echo "Ожидание освобождения $lock..."
        sleep 5
      done
    done
  changed_when: false
  register: wait_lock
  timeout: "{{ update_lock_timeout }}"
  args:
    executable: /bin/bash

- name: Проверка свободного места на важных разделах
  shell: |
    df -h / /boot --output=target,avail | tail -n +2
  register: disk_space
  changed_when: false

- name: Вывести предупреждение при нехватке места (<1G)
  debug:
    msg: >
      ⚠️ Внимание: мало свободного места на разделах:
      {% for line in disk_space.stdout_lines %}
      {{ line }}{% if not loop.last %}, {% endif %}
      {% endfor %}
  when: >
    disk_space.stdout_lines |
    map('regex_replace', '.*\s(\d+\.?\d*)(G|M)', '\1 \2') |
    select('match', '^\d') |
    map('split') |
    map('select', 'float') |
    map('float') |
    min < 1.0

- name: Получить список пакетов до обновления
  shell: dpkg-query -W -f='${Package} ${Version}\n' | sort > /tmp/pre-update-packages.txt
  changed_when: false

- name: Обновление кэша APT
  apt:
    update_cache: yes
    cache_valid_time: 3600
  register: apt_update
  retries: "{{ update_retries }}"
  delay: "{{ update_delay }}"
  until: apt_update is success

- name: Безопасное обновление пакетов
  block:
    - name: Выполнение upgrade
      apt:
        upgrade: "{{ update_upgrade_type }}"
        dpkg_options: 'force-confold,force-confdef'
      register: apt_upgrade
      retries: "{{ update_retries }}"
      delay: "{{ update_delay }}"
      until: apt_upgrade is success

  rescue:
    - name: Попытка исправить сломанные зависимости
      apt:
        name: '--fix-broken'
        state: present
      register: fix_broken

    - name: Повторное обновление после исправления
      apt:
        upgrade: "{{ update_upgrade_type }}"
        dpkg_options: 'force-confold,force-confdef'
      register: apt_upgrade
      retries: "{{ update_retries }}"
      delay: "{{ update_delay }}"
      until: apt_upgrade is success

- name: Автоудаление и очистка кэша
  apt:
    autoremove: "{{ update_autoremove }}"
    autoclean: "{{ update_autoclean }}"

- name: Получить список пакетов после обновления
  shell: dpkg-query -W -f='${Package} ${Version}\n' | sort > /tmp/post-update-packages.txt
  changed_when: false

- name: Сравнить версии пакетов
  shell: diff /tmp/pre-update-packages.txt /tmp/post-update-packages.txt || true
  register: diff_output
  changed_when: false

- name: Проверить необходимость перезагрузки
  stat:
    path: /var/run/reboot-required
  register: reboot_required

- name: Перезагрузить систему (если разрешено и требуется)
  reboot:
    reboot_timeout: 600
    pre_reboot_delay: 10
    post_reboot_delay: 20
    test_command: uptime
  when:
    - reboot_required.stat.exists
    - update_reboot_allowed | bool
  notify: log reboot event

- name: Создать отчёт об обновлении
  template:
    src: report.j2
    dest: "{{ update_report_dir }}/update_report_{{ ansible_date_time.iso8601_basic_short }}.txt"
    mode: '0644'
  when: update_generate_report | bool

- name: Удалить временные файлы
  file:
    path: "{{ item }}"
    state: absent
  loop:
    - /tmp/pre-update-packages.txt
    - /tmp/post-update-packages.txt
  changed_when: false
```

handlers/main.yml

```yaml
---
- name: log reboot event
  lineinfile:
    path: "{{ update_report_dir }}/reboot.log"
    line: "{{ ansible_date_time.iso8601 }}: Система {{ inventory_hostname }} перезагружена после обновления"
    create: yes
    mode: '0644'
  when: update_generate_report | bool
```

templates/report.j2

```jinja2
Отчёт об обновлении системы
=============================
Хост: {{ inventory_hostname }}
ОС: {{ ansible_distribution }} {{ ansible_distribution_version }}
Ядро: {{ ansible_kernel }}
Дата/время: {{ ansible_date_time.date }} {{ ansible_date_time.time }}

Результаты операций
-------------------
Обновление кэша APT: {{ 'Успешно' if apt_update is success else 'Ошибка' }}
Обновление пакетов: {{ 'Успешно' if apt_upgrade is success else 'Ошибка' }}
Исправление зависимостей: {{ 'Выполнено' if (fix_broken is defined and fix_broken is success) else 'Не требовалось' }}
Автоудаление и очистка: {{ 'Выполнено' if update_autoremove or update_autoclean else 'Отключено' }}

Статус перезагрузки
-------------------
Требуется перезагрузка: {{ 'Да' if reboot_required.stat.exists else 'Нет' }}
Перезагрузка выполнена: {{ 'Да' if (update_reboot_allowed and reboot_required.stat.exists) else 'Нет' }}

Изменения в пакетах
-------------------
{% if diff_output.stdout_lines is defined %}
{% for line in diff_output.stdout_lines %}
{{ line }}
{% endfor %}
{% else %}
Информация недоступна.
{% endif %}

Сообщения об ошибках (если есть)
--------------------------------
{% if wait_lock is defined and wait_lock.rc != 0 %}
Ошибка ожидания блокировок: {{ wait_lock.stderr | default('timeout') }}
{% endif %}
{% if apt_update is defined and apt_update.failed %}
Ошибка обновления кэша: {{ apt_update.msg }}
{% endif %}
{% if apt_upgrade is defined and apt_upgrade.failed %}
Ошибка обновления пакетов: {{ apt_upgrade.msg }}
{% endif %}
{% if fix_broken is defined and fix_broken.failed %}
Ошибка исправления зависимостей: {{ fix_broken.msg }}
{% endif %}
```

Пример playbook, использующего роль

```yaml
---
- name: Плановое обновление серверов Ubuntu
  hosts: ubuntu_servers
  become: yes
  roles:
    - role: system_update
      vars:
        update_upgrade_type: safe
        update_reboot_allowed: false   # перезагрузка вручную после проверки
        update_generate_report: true
```

Ключевые улучшения и обработка проблем

1. Блокировки APT/DPKG – ожидание освобождения файлов блокировок с таймаутом, а не жёсткая ошибка.
2. Сломанные зависимости – блок rescue автоматически выполняет apt --fix-broken install и повторяет обновление.
3. Нехватка места – предупреждение, если на / или /boot осталось менее 1 ГБ.
4. Контролируемая перезагрузка – выполняется только при наличии флага /var/run/reboot-required и явном разрешении update_reboot_allowed: true. Хендлер логирует факт перезагрузки.
5. Идемпотентность – модули apt с dpkg_options: 'force-confold,force-confdef' гарантируют неизменность конфигурационных файлов; повторы операций не приводят к изменениям.
6. Детальный отчёт – фиксируются версии пакетов до/после, diff изменений, ошибки, статус перезагрузки. Отчёт сохраняется в /var/log/ansible-update/.

При необходимости можно легко расширить роль, добавив нотификации (Slack, email) или дополнительные проверки (например, целостность загрузчика).


Ошибка возникает из-за двух проблем в условии для таски Вывести предупреждение при нехватке места (<1G):

1. Некорректная фильтрация – map('select', 'float') некорректен, т.к. select ожидает тест, а 'float' не является встроенным тестом Jinja2. После split элементы – это строки, и дальнейшее преобразование в числа сделано неправильно.
2. Пустая последовательность – вывод команды df может не содержать ожидаемых значений (например, если /boot не является отдельной точкой монтирования, или числа не содержат единиц G/M). Фильтры оставляют пустой список, а оператор min выдаёт ошибку «No aggregated item, sequence was empty».

Исправление (рекомендуемый вариант с использованием фактов Ansible)

Замените таски, связанные с проверкой места, на использование встроенных фактов ansible_mounts. Это надёжнее, не требует парсинга вывода df и работает без ошибок.

Удалите (или закомментируйте) старые таски:

```yaml
# - name: Проверка свободного места на важных разделах
#   shell: ...
#
# - name: Вывести предупреждение при нехватке места (<1G)
#   debug: ...
#   when: ...
```

Добавьте новую таску:

```yaml
- name: Проверка свободного места и предупреждение
  vars:
    check_mounts: ["/", "/boot"]
    min_free_bytes: "{{ 1 * 1024**3 }}"  # 1 ГБ
  debug:
    msg: >
      ⚠️ Внимание: на разделе {{ item.mount }} свободно
      {{ (item.size_available / 1024**3) | round(2) }} ГБ,
      что меньше {{ min_free_bytes / 1024**3 }} ГБ!
  loop: "{{ ansible_mounts | selectattr('mount', 'in', check_mounts) | list }}"
  when:
    - item.size_available < min_free_bytes
    - item.mount in check_mounts
  loop_control:
    label: "{{ item.mount }}"
```

Пояснение:

· Используются собранные факты о монтированиях (ansible_mounts).
· Проверяем только точки монтирования / и /boot (если /boot отсутствует как отдельный раздел, он просто не попадёт в список).
· Сравниваем доступное место в байтах с порогом 1 ГБ, выводим человекочитаемое сообщение.

Альтернативное исправление (с сохранением df)

Если вы по каким-то причинам хотите сохранить старый подход с df, нужно переписать условие корректно. Например:

```yaml
- name: Проверка свободного места на важных разделах
  shell: |
    df / /boot --output=target,avail | tail -n +2
  register: disk_space
  changed_when: false

- name: Вывести предупреждение при нехватке места (<1G)
  debug:
    msg: >
      ⚠️ Внимание: мало свободного места на разделах:
      {% for line in disk_space.stdout_lines %}
        {{ line }}{% if not loop.last %}, {% endif %}
      {% endfor %}
  when: >
    disk_space.stdout_lines |
    select('match', '.*\s(\d+\.?\d*)(G|M)') |
    map('regex_replace', '.*\s(\d+\.?\d*)(G|M)$', '\1 \2') |
    map('split') |
    map('first') |
    map('float') |
    min < 1.0
```

Но учтите, что этот вариант всё ещё может дать пустую последовательность, если в выводе нет значений с G/M. Тогда min упадёт с той же ошибкой. Лучше сделать проверку длины:

```yaml
  when: >
    disk_space.stdout_lines |
    select('match', '.*\s(\d+\.?\d*)(G|M)') | list | length > 0
    and
    ( ... ваш min ... )
```

И всё же рекомендую первый вариант – он стандартный, идемпотентный и не зависит от формата вывода утилит.

_______


Я доработал роль, добавив возможность сохранять сформированный отчёт не только на целевом хосте, но и на локальной машине управления Ansible. Теперь отчёты автоматически копируются в указанную локальную директорию с уникальным именем, включающим хост и временную метку.

Что изменилось

defaults/main.yml — новые переменные

```yaml
# Сохранение отчёта локально (на control node)
update_local_report: true                # включить/отключить
update_local_report_dir: "./ansible_update_reports"  # локальный каталог (можно абсолютный путь)
```

tasks/main.yml — добавлены две таски в конец файла

Фрагмент, который нужно вставить после задачи «Создать отчёт об обновлении»:

```yaml
- name: Убедиться, что локальная директория для отчётов существует
  file:
    path: "{{ update_local_report_dir }}"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true
  when:
    - update_generate_report | bool
    - update_local_report | bool

- name: Скопировать отчёт на локальный сервер
  fetch:
    src: "{{ update_report_dir }}/update_report_{{ ansible_date_time.iso8601_basic_short }}.txt"
    dest: "{{ update_local_report_dir }}/{{ inventory_hostname }}_{{ ansible_date_time.iso8601_basic_short }}.txt"
    flat: yes
  when:
    - update_generate_report | bool
    - update_local_report | bool
```

Пояснение работы

1. Создание локальной папки – file с delegate_to: localhost гарантирует существование каталога на машине управления.
2. Копирование файла – модуль fetch забирает отчёт с удалённого хоста и сохраняет под именем hostname_время.txt в локальной папке. Параметр flat: yes убирает вложенность с путём источника.
3. Обе таски управляются флагами update_generate_report и update_local_report, что позволяет гибко отключать локальное сохранение.

Полный листинг роли с учётом исправления ошибки дискового пространства и локального отчёта

Ниже представлены итоговые файлы роли system_update, включая предыдущее исправление проверки свободного места через факты ansible_mounts.

defaults/main.yml

```yaml
---
# Тип обновления: safe, full, dist
update_upgrade_type: safe

# Автоматическое удаление старых пакетов и очистка кэша
update_autoremove: true
update_autoclean: true

# Разрешить перезагрузку, если требуется (по умолчанию нет)
update_reboot_allowed: false

# Генерировать отчёт
update_generate_report: true
update_report_dir: /var/log/ansible-update

# Локальное сохранение отчёта
update_local_report: true
update_local_report_dir: "./ansible_update_reports"

# Таймауты и повторы
update_lock_timeout: 300
update_retries: 3
update_delay: 10
```

tasks/main.yml

```yaml
---
- name: Проверка совместимости
  assert:
    that:
      - ansible_distribution == 'Ubuntu'
      - ansible_distribution_version in ['22.04', '24.04']
    fail_msg: "Роль поддерживает только Ubuntu 22.04 / 24.04. Текущая ОС: {{ ansible_distribution }} {{ ansible_distribution_version }}"

- name: Убедиться, что директория для отчётов существует
  file:
    path: "{{ update_report_dir }}"
    state: directory
    mode: '0755'
  when: update_generate_report | bool

- name: Ожидание снятия блокировок APT/DPKG
  shell: |
    for lock in /var/lib/dpkg/lock /var/lib/apt/lists/lock /var/cache/apt/archives/lock; do
      while fuser "$lock" >/dev/null 2>&1; do
        echo "Ожидание освобождения $lock..."
        sleep 5
      done
    done
  changed_when: false
  register: wait_lock
  timeout: "{{ update_lock_timeout }}"
  args:
    executable: /bin/bash

- name: Проверка свободного места и предупреждение (через ansible_mounts)
  vars:
    check_mounts: ["/", "/boot"]
    min_free_bytes: "{{ 1 * 1024**3 }}"   # 1 ГБ
  debug:
    msg: >
      ⚠️ Внимание: на разделе {{ item.mount }} свободно
      {{ (item.size_available / 1024**3) | round(2) }} ГБ,
      что меньше {{ min_free_bytes / 1024**3 }} ГБ!
  loop: "{{ ansible_mounts | selectattr('mount', 'in', check_mounts) | list }}"
  when:
    - item.size_available < min_free_bytes
    - item.mount in check_mounts
  loop_control:
    label: "{{ item.mount }}"

- name: Получить список пакетов до обновления
  shell: dpkg-query -W -f='${Package} ${Version}\n' | sort > /tmp/pre-update-packages.txt
  changed_when: false

- name: Обновление кэша APT
  apt:
    update_cache: yes
    cache_valid_time: 3600
  register: apt_update
  retries: "{{ update_retries }}"
  delay: "{{ update_delay }}"
  until: apt_update is success

- name: Безопасное обновление пакетов
  block:
    - name: Выполнение upgrade
      apt:
        upgrade: "{{ update_upgrade_type }}"
        dpkg_options: 'force-confold,force-confdef'
      register: apt_upgrade
      retries: "{{ update_retries }}"
      delay: "{{ update_delay }}"
      until: apt_upgrade is success

  rescue:
    - name: Попытка исправить сломанные зависимости
      apt:
        name: '--fix-broken'
        state: present
      register: fix_broken

    - name: Повторное обновление после исправления
      apt:
        upgrade: "{{ update_upgrade_type }}"
        dpkg_options: 'force-confold,force-confdef'
      register: apt_upgrade
      retries: "{{ update_retries }}"
      delay: "{{ update_delay }}"
      until: apt_upgrade is success

- name: Автоудаление и очистка кэша
  apt:
    autoremove: "{{ update_autoremove }}"
    autoclean: "{{ update_autoclean }}"

- name: Получить список пакетов после обновления
  shell: dpkg-query -W -f='${Package} ${Version}\n' | sort > /tmp/post-update-packages.txt
  changed_when: false

- name: Сравнить версии пакетов
  shell: diff /tmp/pre-update-packages.txt /tmp/post-update-packages.txt || true
  register: diff_output
  changed_when: false

- name: Проверить необходимость перезагрузки
  stat:
    path: /var/run/reboot-required
  register: reboot_required

- name: Перезагрузить систему (если разрешено и требуется)
  reboot:
    reboot_timeout: 600
    pre_reboot_delay: 10
    post_reboot_delay: 20
    test_command: uptime
  when:
    - reboot_required.stat.exists
    - update_reboot_allowed | bool
  notify: log reboot event

- name: Создать отчёт об обновлении
  template:
    src: report.j2
    dest: "{{ update_report_dir }}/update_report_{{ ansible_date_time.iso8601_basic_short }}.txt"
    mode: '0644'
  when: update_generate_report | bool

- name: Убедиться, что локальная директория для отчётов существует
  file:
    path: "{{ update_local_report_dir }}"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true
  when:
    - update_generate_report | bool
    - update_local_report | bool

- name: Скопировать отчёт на локальный сервер
  fetch:
    src: "{{ update_report_dir }}/update_report_{{ ansible_date_time.iso8601_basic_short }}.txt"
    dest: "{{ update_local_report_dir }}/{{ inventory_hostname }}_{{ ansible_date_time.iso8601_basic_short }}.txt"
    flat: yes
  when:
    - update_generate_report | bool
    - update_local_report | bool

- name: Удалить временные файлы
  file:
    path: "{{ item }}"
    state: absent
  loop:
    - /tmp/pre-update-packages.txt
    - /tmp/post-update-packages.txt
  changed_when: false
```

Остальные файлы (handlers/main.yml и templates/report.j2) не изменились.

Пример использования с локальным сохранением

```yaml
- name: Обновление Ubuntu с сохранением отчёта локально
  hosts: ubuntu_servers
  become: yes
  roles:
    - role: system_update
      vars:
        update_local_report_dir: "/home/ansible/reports/{{ ansible_date_time.date }}"
```

После выполнения playbook в указанной локальной директории появятся файлы вида ya-testgd-001-d_20260717T150000.txt. Это позволяет централизованно хранить историю обновлений и использовать её для аудита.