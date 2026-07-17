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