Ниже представлена улучшенная универсальная Ansible-роль для настройки chrony.
Она автоматически определяет дистрибутив, корректно обрабатывает пути к конфигурации, позволяет гибко задавать NTP-серверы через переменные, создаёт резервные копии, комментирует старые записи и проверяет работоспособность после перезапуска.

Структура роли

```
chrony/
├── defaults
│   └── main.yml          # Параметры по умолчанию (список серверов)
├── handlers
│   └── main.yml          # Перезапуск chrony
├── tasks
│   └── main.yml          # Основные задачи
├── vars
│   └── main.yml          # Внутренние переменные (пути к конфигам)
└── templates
    └── chrony.conf.j2    # Опциональный шаблон (если нужен полный контроль)
```

Содержимое файлов

defaults/main.yml

Здесь определяются серверы, которые будут добавлены. Можно переопределить в плейбуке или инвентаре.

```yaml
---
# Список NTP-серверов для добавления в конфигурацию
chrony_servers:
  - nn-ntp-001.tk.ru iburst
  - nn-ntp-002.tk.ru iburst

# Режим работы: "replace" (комментировать старые строки) или "template" (использовать шаблон)
chrony_config_mode: "replace"

# Путь к шаблону, если mode = template (по умолчанию берётся из роли)
chrony_template: "chrony.conf.j2"
```

vars/main.yml

Определяем пути к конфигурационному файлу в зависимости от семейства ОС.

```yaml
---
# Путь к конфигурационному файлу chrony
chrony_config_file: >-
  {% if ansible_os_family == 'Debian' -%}
    /etc/chrony/chrony.conf
  {%- elif ansible_os_family == 'RedHat' -%}
    /etc/chrony.conf
  {%- elif ansible_os_family == 'Suse' -%}
    /etc/chrony.conf
  {%- else -%}
    /etc/chrony.conf
  {%- endif %}

# Имя сервиса (везде chronyd, но на всякий случай)
chrony_service_name: chronyd
```

tasks/main.yml

```yaml
---
- name: Install chrony package
  package:
    name: chrony
    state: present
  when: ansible_os_family in ['RedHat', 'Debian', 'Suse']
  # На Debian/Ubuntu при установке chrony автоматически удалит конфликтующий systemd-timesyncd

- name: Ensure chrony configuration directory exists (for Debian)
  file:
    path: /etc/chrony
    state: directory
    mode: '0755'
  when: ansible_os_family == 'Debian'

- name: Backup existing configuration
  copy:
    src: "{{ chrony_config_file }}"
    dest: "{{ chrony_config_file }}.{{ ansible_date_time.date }}.backup"
    remote_src: yes
    backup: no
  ignore_errors: yes
  when: chrony_config_file is file

- name: "Modify configuration: {{ chrony_config_mode }} mode"
  include_tasks: "modify_{{ chrony_config_mode }}.yml"
```

tasks/modify_replace.yml

Режим замены строк (комментирование старых pool/server и добавление новых).

```yaml
---
- name: Comment out existing pool and server lines
  replace:
    path: "{{ chrony_config_file }}"
    regexp: '^(pool|server)\s+.*$'
    replace: '# \g<0> (commented by Ansible)'
  backup: no
  notify: restart chrony

- name: Add custom NTP servers
  blockinfile:
    path: "{{ chrony_config_file }}"
    block: |
      {% for server in chrony_servers %}
      server {{ server }}
      {% endfor %}
    marker: "# {mark} ANSIBLE MANAGED NTP SERVERS"
    insertafter: EOF
  notify: restart chrony
```

tasks/modify_template.yml

Режим полной замены конфигурации из шаблона (требуется файл templates/chrony.conf.j2).

```yaml
---
- name: Deploy chrony configuration from template
  template:
    src: "{{ chrony_template }}"
    dest: "{{ chrony_config_file }}"
    backup: yes
  notify: restart chrony
```

handlers/main.yml

```yaml
---
- name: restart chrony
  systemd:
    name: "{{ chrony_service_name }}"
    state: restarted
    daemon_reload: yes
  listen: "restart chrony"
```

Дополнительные проверки в tasks/main.yml (после handler)

```yaml
- name: Flush handlers to restart chrony if needed
  meta: flush_handlers

- name: Wait for chrony to start
  pause:
    seconds: 5

- name: Check chrony sources
  command: chronyc sources
  register: chronyc_sources
  changed_when: false

- name: Show chronyc sources
  debug:
    var: chronyc_sources.stdout_lines

- name: Check chrony tracking
  command: chronyc tracking
  register: chronyc_tracking
  changed_when: false

- name: Show chronyc tracking
  debug:
    var: chronyc_tracking.stdout_lines
```

Опциональный шаблон templates/chrony.conf.j2 (пример)

```
# This file is managed by Ansible
# NTP servers
{% for server in chrony_servers %}
server {{ server }}
{% endfor %}

# Other defaults
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
```

Универсальность и гибкость

· Дистрибутивы: автоматически определяются пути (RedHat, Debian, Suse).
· Метод настройки: можно выбрать replace (комментирование старых записей) или template (полное управление через шаблон).
· Серверы: задаются списком в chrony_servers, легко расширить или изменить.
· Резервное копирование: создаётся копия с датой перед любыми изменениями.
· Перезапуск: только при изменении конфигурации (благодаря notify).
· Проверка: автоматический вывод chronyc sources и chronyc tracking.

Использование

1. Поместите роль в директорию roles/chrony.
2. В плейбуке подключите роль и при необходимости переопределите переменные:

```yaml
- hosts: all
  become: yes
  roles:
    - role: chrony
      vars:
        chrony_servers:
          - nn-ntp-001.tk.ru iburst
          - nn-ntp-002.tk.ru iburst
        chrony_config_mode: replace   # или template
```

Роль готова к использованию в окружениях с Oracle Linux, Ubuntu и другими системами на базе RHEL/Debian.