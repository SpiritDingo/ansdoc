Ansible роль: Установка и настройка auditd и системного журнала (syslog/rsyslog/syslog-ng) с отправкой событий аудита через audispd

Данная Ansible роль предназначена для автоматизации установки и настройки auditd (системы аудита Linux) и одного из демонов системного журнала (syslog, rsyslog, syslog-ng), а также для настройки перенаправления событий аудита в syslog через audispd (плагин auditd).

Возможности роли

· Установка пакетов auditd и выбранного демона syslog.
· Генерация конфигурационных файлов auditd и audispd.
· Включение и настройка плагина audispd-syslog для отправки событий аудита в syslog.
· Настройка соответствующего syslog-демона для приёма событий (например, создание отдельного файла лога или отправка на удалённый сервер).
· Поддержка различных дистрибутивов Linux (RedHat-подобные и Debian-подобные).
· Возможность выбора типа syslog через переменную.

Структура роли

```
audit-syslog-role/
├── defaults
│   └── main.yml          # переменные по умолчанию
├── vars
│   └── main.yml          # переменные для конкретных ОС (необязательно)
├── tasks
│   └── main.yml          # основные задачи
├── handlers
│   └── main.yml          # обработчики перезапуска сервисов
├── templates
│   ├── auditd.conf.j2    # шаблон конфига auditd
│   ├── audispd.conf.j2   # шаблон конфига audispd
│   ├── syslog_plugin.conf.j2  # шаблон для /etc/audisp/plugins.d/syslog.conf
│   ├── rsyslog.conf.j2   # шаблон для rsyslog (если выбран rsyslog)
│   ├── syslog-ng.conf.j2 # шаблон для syslog-ng (если выбран syslog-ng)
│   └── syslog.conf.j2    # шаблон для классического syslogd
└── meta
    └── main.yml          # зависимости (если есть)
```

Переменные (defaults/main.yml)

```yaml
---
# Выбор демона syslog: "rsyslog", "syslog-ng" или "syslogd"
syslog_daemon: rsyslog

# Настройки auditd
auditd:
  log_file: /var/log/audit/audit.log
  log_format: RAW
  log_group: root
  priority_boost: 4
  flush: INCREMENTAL
  freq: 20
  num_logs: 5
  disp_qos: lossy
  dispatcher: /sbin/audispd
  name_format: NONE
  max_log_file: 8
  max_log_file_action: ROTATE
  space_left: 75
  space_left_action: SYSLOG
  action_mail_acct: root
  admin_space_left: 50
  admin_space_left_action: SUSPEND
  disk_full_action: SUSPEND
  disk_error_action: SUSPEND

# Настройки audispd (общие)
audispd:
  q_depth: 80
  overflow_action: SYSLOG
  priority_boost: 4
  max_restarts: 10
  name_format: HOSTNAME

# Включение плагина syslog для audispd
audisp_syslog_plugin:
  active: yes
  direction: out
  path: /sbin/audisp-syslog
  type: builtin
  args: ""          # аргументы для плагина, например "LOG_LOCAL6"
  format: string    # или "managed"

# Настройки для syslog (общие для всех демонов)
syslog:
  # Файл, куда будут писаться события аудита (если локально)
  audit_log_file: /var/log/audit-forwarded.log
  # Удалённый сервер syslog (опционально)
  remote_server: ""
  remote_port: 514
  remote_protocol: udp  # или tcp
  # Facility и severity для событий аудита (по умолчанию LOG_LOCAL6, LOG_INFO)
  audit_facility: local6
  audit_severity: info

# Для rsyslog могут быть дополнительные параметры
rsyslog_config:
  modules:
    - imuxsock
    - imjournal
  work_directory: /var/spool/rsyslog

# Для syslog-ng
syslog_ng_config:
  version: 3.38   # версия конфига
  include: /etc/syslog-ng/conf.d/*.conf
```

Задачи (tasks/main.yml)

Основные шаги:

1. Установка необходимых пакетов в зависимости от выбранного syslog_daemon и дистрибутива.
2. Копирование шаблонов конфигурации auditd и audispd.
3. Настройка плагина syslog для audispd.
4. Настройка демона syslog (создание шаблона и, при необходимости, настройка удалённой отправки).
5. Запуск и включение сервисов.

Пример (сокращённый) tasks/main.yml:

```yaml
---
- name: Установка auditd
  package:
    name: audit
    state: present
  when: ansible_os_family == "RedHat"

- name: Установка auditd (Debian)
  package:
    name: auditd
    state: present
  when: ansible_os_family == "Debian"

- name: Установка выбранного syslog-демона
  package:
    name: "{{ syslog_daemon }}"
    state: present

- name: Создание конфигурации auditd
  template:
    src: auditd.conf.j2
    dest: /etc/audit/auditd.conf
    owner: root
    group: root
    mode: 0640
  notify: restart auditd

- name: Создание конфигурации audispd
  template:
    src: audispd.conf.j2
    dest: /etc/audisp/audispd.conf
    owner: root
    group: root
    mode: 0640
  notify: restart auditd

- name: Настройка плагина syslog для audispd
  template:
    src: syslog_plugin.conf.j2
    dest: /etc/audisp/plugins.d/syslog.conf
    owner: root
    group: root
    mode: 0640
  notify: restart auditd

- name: Настройка syslog-демона (rsyslog)
  template:
    src: rsyslog.conf.j2
    dest: /etc/rsyslog.conf
    owner: root
    group: root
    mode: 0644
  notify: restart rsyslog
  when: syslog_daemon == "rsyslog"

- name: Настройка syslog-демона (syslog-ng)
  template:
    src: syslog-ng.conf.j2
    dest: /etc/syslog-ng/syslog-ng.conf
    owner: root
    group: root
    mode: 0644
  notify: restart syslog-ng
  when: syslog_daemon == "syslog-ng"

- name: Настройка syslog-демона (классический syslogd)
  template:
    src: syslog.conf.j2
    dest: /etc/syslog.conf
    owner: root
    group: root
    mode: 0644
  notify: restart syslogd
  when: syslog_daemon == "syslogd"

- name: Запуск и включение auditd
  service:
    name: auditd
    state: started
    enabled: yes

- name: Запуск и включение syslog-демона
  service:
    name: "{{ syslog_daemon }}"
    state: started
    enabled: yes
```

Шаблоны (примеры)

templates/auditd.conf.j2

```jinja2
#
# Этот файл управляется Ansible
#
log_file = {{ auditd.log_file }}
log_format = {{ auditd.log_format }}
log_group = {{ auditd.log_group }}
priority_boost = {{ auditd.priority_boost }}
flush = {{ auditd.flush }}
freq = {{ auditd.freq }}
num_logs = {{ auditd.num_logs }}
disp_qos = {{ auditd.disp_qos }}
dispatcher = {{ auditd.dispatcher }}
name_format = {{ auditd.name_format }}
max_log_file = {{ auditd.max_log_file }}
max_log_file_action = {{ auditd.max_log_file_action }}
space_left = {{ auditd.space_left }}
space_left_action = {{ auditd.space_left_action }}
action_mail_acct = {{ auditd.action_mail_acct }}
admin_space_left = {{ auditd.admin_space_left }}
admin_space_left_action = {{ auditd.admin_space_left_action }}
disk_full_action = {{ auditd.disk_full_action }}
disk_error_action = {{ auditd.disk_error_action }}
```

templates/audispd.conf.j2

```jinja2
#
q_depth = {{ audispd.q_depth }}
overflow_action = {{ audispd.overflow_action }}
priority_boost = {{ audispd.priority_boost }}
max_restarts = {{ audispd.max_restarts }}
name_format = {{ audispd.name_format }}
```

templates/syslog_plugin.conf.j2

```jinja2
active = {{ 'yes' if audisp_syslog_plugin.active else 'no' }}
direction = {{ audisp_syslog_plugin.direction }}
path = {{ audisp_syslog_plugin.path }}
type = {{ audisp_syslog_plugin.type }}
args = {{ audisp_syslog_plugin.args }}
format = {{ audisp_syslog_plugin.format }}
```

templates/rsyslog.conf.j2 (фрагмент для приёма событий от auditd)

```jinja2
#### GLOBAL DIRECTIVES ####
module(load="imuxsock")
module(load="imjournal")
module(load="imklog")

#### RULES ####
# Сообщения от auditd (facility local6) пишем в отдельный файл
local6.*    {{ syslog.audit_log_file }}

# Отправка на удалённый сервер, если задан
{% if syslog.remote_server %}
local6.*    @{{ syslog.remote_server }}:{{ syslog.remote_port }}
{% endif %}
```

templates/syslog-ng.conf.j2 (фрагмент)

```jinja2
@version: {{ syslog_ng_config.version }}
@include "scl.conf"

source s_sys {
    system();
    internal();
};

destination d_audit {
    file("{{ syslog.audit_log_file }}");
};

{% if syslog.remote_server %}
destination d_remote {
    {{ 'udp' if syslog.remote_protocol == 'udp' else 'tcp' }}("{{ syslog.remote_server }}" port({{ syslog.remote_port }}));
};
{% endif %}

log {
    source(s_sys);
    filter(f_facility_local6);
    destination(d_audit);
    {% if syslog.remote_server %}destination(d_remote);{% endif %}
};

filter f_facility_local6 {
    facility(local6);
};
```

templates/syslog.conf.j2 (для классического syslogd)

```jinja2
#  События local6 в отдельный файл
local6.*    {{ syslog.audit_log_file }}

{% if syslog.remote_server %}
# Отправка на удалённый сервер
local6.*    @{{ syslog.remote_server }}
{% endif %}
```

Обработчики (handlers/main.yml)

```yaml
---
- name: restart auditd
  service:
    name: auditd
    state: restarted

- name: restart rsyslog
  service:
    name: rsyslog
    state: restarted

- name: restart syslog-ng
  service:
    name: syslog-ng
    state: restarted

- name: restart syslogd
  service:
    name: syslogd
    state: restarted
```

Переменные для разных ОС (vars/main.yml)

Можно определить специфичные имена пакетов:

```yaml
---
audit_package_redhat: audit
audit_package_debian: auditd
syslog_package_rsyslog_redhat: rsyslog
syslog_package_rsyslog_debian: rsyslog
# и т.д.
```

Пример использования роли в плейбуке

```yaml
- hosts: all
  become: yes
  vars:
    syslog_daemon: rsyslog
    syslog:
      remote_server: "192.168.1.100"
      remote_port: 514
      remote_protocol: udp
      audit_log_file: /var/log/audit/remote.log
    audisp_syslog_plugin:
      active: yes
      args: "LOG_LOCAL6"
  roles:
    - audit-syslog-role
```

Примечания

· audispd – это диспетчер событий auditd, который может передавать события внешним приложениям. Плагин audisp-syslog отправляет каждое событие аудита в системный журнал с выбранным facility (по умолчанию LOG_AUTHPRIV, но можно изменить через аргумент args).
· Для корректной работы убедитесь, что в auditd.conf параметр dispatcher указывает на /sbin/audispd.
· В зависимости от дистрибутива пути к конфигурационным файлам могут отличаться (например, /etc/audit/audispd.conf может быть в /etc/audisp/audispd.conf). В примерах использованы стандартные пути.
· Классический syslogd (обычно из пакета inetutils-syslogd или sysklogd) поддерживает только UDP и простую конфигурацию. Если нужны расширенные возможности, лучше выбрать rsyslog или syslog-ng.
· Для применения правил аудита (например, добавление своих watch) роль можно расширить задачами, которые копируют файлы правил в /etc/audit/rules.d/.

Данная роль даёт гибкость в выборе syslog-демона и полностью настраивает отправку событий аудита в системный журнал, что удобно для централизованного сбора логов безопасности.




------------------------------




Ошибка возникает из-за того, что в шаблоне syslog_plugin.conf.j2 используется переменная audisp_syslog_plugin.direction, но в словаре audisp_syslog_plugin нет ключа direction. Это может быть вызвано тем, что переменная не полностью определена в defaults/main.yml или переопределена в другом месте без этого ключа.

Решение

1. Добавить недостающий ключ в defaults/main.yml

Убедитесь, что структура audisp_syslog_plugin содержит все необходимые поля, включая direction:

```yaml
audisp_syslog_plugin:
  active: yes
  direction: out          # обязательно
  path: /sbin/audisp-syslog
  type: builtin
  args: ""                # опционально
  format: string          # опционально
```

2. Использовать в шаблоне фильтр default() для защиты от отсутствующих ключей

Измените templates/syslog_plugin.conf.j2:

```jinja2
active = {{ 'yes' if audisp_syslog_plugin.active else 'no' }}
direction = {{ audisp_syslog_plugin.direction | default('out') }}
path = {{ audisp_syslog_plugin.path }}
type = {{ audisp_syslog_plugin.type }}
args = {{ audisp_syslog_plugin.args | default('') }}
format = {{ audisp_syslog_plugin.format | default('string') }}
```

3. Проверьте переопределение переменных в плейбуке

Если вы задаёте audisp_syslog_plugin в плейбуке или инвентори, убедитесь, что указаны все ключи, например:

```yaml
- hosts: all
  vars:
    audisp_syslog_plugin:
      active: yes
      direction: out
      path: /sbin/audisp-syslog
      type: builtin
      args: "LOG_LOCAL6"
      format: string
  roles:
    - audit-syslog-role
```

Почему это произошло?

· В defaults/main.yml либо отсутствовал ключ direction, либо он был случайно удалён при редактировании.
· Или вы переопределили audisp_syslog_plugin в другом месте, но не включили direction.
· Ansible ожидает, что атрибут существует, и выдает ошибку, если его нет.

Рекомендация

Всегда задавайте полные структуры переменных в defaults и используйте фильтр default() в шаблонах для критически важных атрибутов. Это повышает устойчивость роли к частичным переопределениям.


------------------------------------
------------------------------------


Полная исправленная роль Ansible: audit_syslog

Структура роли

```
audit_syslog/
├── defaults
│   └── main.yml
├── vars
│   └── main.yml
├── tasks
│   └── main.yml
├── handlers
│   └── main.yml
├── templates
│   ├── auditd.conf.j2
│   ├── audispd.conf.j2
│   ├── syslog_plugin.conf.j2
│   ├── rsyslog.conf.j2
│   ├── syslog-ng.conf.j2
│   └── syslog.conf.j2
└── meta
    └── main.yml
```

---

1. defaults/main.yml

```yaml
---
# Выбор демона syslog: "rsyslog", "syslog-ng" или "syslogd"
syslog_daemon: rsyslog

# Настройки auditd
auditd:
  log_file: /var/log/audit/audit.log
  log_format: RAW
  log_group: root
  priority_boost: 4
  flush: INCREMENTAL
  freq: 20
  num_logs: 5
  disp_qos: lossy
  dispatcher: /sbin/audispd
  name_format: NONE
  max_log_file: 8
  max_log_file_action: ROTATE
  space_left: 75
  space_left_action: SYSLOG
  action_mail_acct: root
  admin_space_left: 50
  admin_space_left_action: SUSPEND
  disk_full_action: SUSPEND
  disk_error_action: SUSPEND

# Настройки audispd (общие)
audispd:
  q_depth: 80
  overflow_action: SYSLOG
  priority_boost: 4
  max_restarts: 10
  name_format: HOSTNAME

# Конфигурация плагина syslog для audispd
audisp_syslog_plugin:
  active: yes
  direction: out
  path: /sbin/audisp-syslog
  type: builtin
  args: ""                # можно указать facility, например "LOG_LOCAL6"
  format: string

# Общие настройки syslog (для всех демонов)
syslog:
  # Файл, куда будут писаться события аудита (локально)
  audit_log_file: /var/log/audit-forwarded.log
  # Удалённый сервер (опционально)
  remote_server: ""
  remote_port: 514
  remote_protocol: udp   # udp или tcp
  # Facility и severity для событий аудита (используется в правилах)
  audit_facility: local6
  audit_severity: info

# Специфичные настройки для rsyslog (дополнительно)
rsyslog_config:
  modules:
    - imuxsock
    - imjournal
  work_directory: /var/spool/rsyslog

# Специфичные настройки для syslog-ng
syslog_ng_config:
  version: 3.38
  include: /etc/syslog-ng/conf.d/*.conf
```

---

2. vars/main.yml (определения для разных ОС)

```yaml
---
# Имена пакетов и пути в зависимости от дистрибутива
audit_package:
  RedHat: audit
  Debian: auditd

syslog_package:
  rsyslog:
    RedHat: rsyslog
    Debian: rsyslog
  syslog-ng:
    RedHat: syslog-ng
    Debian: syslog-ng
  syslogd:
    RedHat: rsyslog   # в RHEL классического syslogd нет, используем rsyslog как замену
    Debian: inetutils-syslogd   # или sysklogd

# Пути к конфигурационным файлам
auditd_conf_path: /etc/audit/auditd.conf
audispd_conf_path: /etc/audit/audispd.conf
audisp_plugin_dir: /etc/audit/plugins.d
syslog_plugin_conf_path: "{{ audisp_plugin_dir }}/syslog.conf"

# Пути к конфигам syslog-демонов
rsyslog_conf_path: /etc/rsyslog.conf
syslog_ng_conf_path: /etc/syslog-ng/syslog-ng.conf
syslogd_conf_path: /etc/syslog.conf

# Имена сервисов
auditd_service: auditd
rsyslog_service: rsyslog
syslog_ng_service: syslog-ng
syslogd_service: syslogd
```

---

3. tasks/main.yml

```yaml
---
- name: Установка auditd
  package:
    name: "{{ audit_package[ansible_os_family] }}"
    state: present
  when: ansible_os_family in ['RedHat', 'Debian']

- name: Установка выбранного syslog-демона
  package:
    name: "{{ syslog_package[syslog_daemon][ansible_os_family] }}"
    state: present
  when: ansible_os_family in ['RedHat', 'Debian']

- name: Создание директории для плагинов audisp (если не существует)
  file:
    path: "{{ audisp_plugin_dir }}"
    state: directory
    owner: root
    group: root
    mode: 0755
  when: audisp_plugin_dir != ''

- name: Копирование конфигурации auditd
  template:
    src: auditd.conf.j2
    dest: "{{ auditd_conf_path }}"
    owner: root
    group: root
    mode: 0640
  notify: restart auditd

- name: Копирование конфигурации audispd
  template:
    src: audispd.conf.j2
    dest: "{{ audispd_conf_path }}"
    owner: root
    group: root
    mode: 0640
  notify: restart auditd

- name: Копирование конфигурации плагина syslog для audispd
  template:
    src: syslog_plugin.conf.j2
    dest: "{{ syslog_plugin_conf_path }}"
    owner: root
    group: root
    mode: 0640
  notify: restart auditd

- name: Настройка rsyslog
  block:
    - name: Копирование конфигурации rsyslog
      template:
        src: rsyslog.conf.j2
        dest: "{{ rsyslog_conf_path }}"
        owner: root
        group: root
        mode: 0644
      notify: restart rsyslog
  when: syslog_daemon == "rsyslog"

- name: Настройка syslog-ng
  block:
    - name: Копирование конфигурации syslog-ng
      template:
        src: syslog-ng.conf.j2
        dest: "{{ syslog_ng_conf_path }}"
        owner: root
        group: root
        mode: 0644
      notify: restart syslog-ng
  when: syslog_daemon == "syslog-ng"

- name: Настройка классического syslogd
  block:
    - name: Копирование конфигурации syslogd
      template:
        src: syslog.conf.j2
        dest: "{{ syslogd_conf_path }}"
        owner: root
        group: root
        mode: 0644
      notify: restart syslogd
  when: syslog_daemon == "syslogd"

- name: Запуск и включение auditd
  service:
    name: "{{ auditd_service }}"
    state: started
    enabled: yes

- name: Запуск и включение syslog-демона
  service:
    name: "{{ syslog_daemon }}"
    state: started
    enabled: yes
```

---

4. handlers/main.yml

```yaml
---
- name: restart auditd
  service:
    name: "{{ auditd_service }}"
    state: restarted

- name: restart rsyslog
  service:
    name: "{{ rsyslog_service }}"
    state: restarted

- name: restart syslog-ng
  service:
    name: "{{ syslog_ng_service }}"
    state: restarted

- name: restart syslogd
  service:
    name: "{{ syslogd_service }}"
    state: restarted
```

---

5. Шаблоны

templates/auditd.conf.j2

```jinja2
#
# Этот файл управляется Ansible
#
log_file = {{ auditd.log_file }}
log_format = {{ auditd.log_format }}
log_group = {{ auditd.log_group }}
priority_boost = {{ auditd.priority_boost }}
flush = {{ auditd.flush }}
freq = {{ auditd.freq }}
num_logs = {{ auditd.num_logs }}
disp_qos = {{ auditd.disp_qos }}
dispatcher = {{ auditd.dispatcher }}
name_format = {{ auditd.name_format }}
max_log_file = {{ auditd.max_log_file }}
max_log_file_action = {{ auditd.max_log_file_action }}
space_left = {{ auditd.space_left }}
space_left_action = {{ auditd.space_left_action }}
action_mail_acct = {{ auditd.action_mail_acct }}
admin_space_left = {{ auditd.admin_space_left }}
admin_space_left_action = {{ auditd.admin_space_left_action }}
disk_full_action = {{ auditd.disk_full_action }}
disk_error_action = {{ auditd.disk_error_action }}
```

templates/audispd.conf.j2

```jinja2
#
q_depth = {{ audispd.q_depth }}
overflow_action = {{ audispd.overflow_action }}
priority_boost = {{ audispd.priority_boost }}
max_restarts = {{ audispd.max_restarts }}
name_format = {{ audispd.name_format }}
```

templates/syslog_plugin.conf.j2

```jinja2
active = {{ 'yes' if audisp_syslog_plugin.active else 'no' }}
direction = {{ audisp_syslog_plugin.direction | default('out') }}
path = {{ audisp_syslog_plugin.path }}
type = {{ audisp_syslog_plugin.type }}
args = {{ audisp_syslog_plugin.args | default('') }}
format = {{ audisp_syslog_plugin.format | default('string') }}
```

templates/rsyslog.conf.j2

```jinja2
#### GLOBAL DIRECTIVES ####
module(load="imuxsock")
module(load="imjournal")
module(load="imklog")

#### RULES ####
# Сообщения от auditd (facility local6) пишем в отдельный файл
{{ syslog.audit_facility }}.{{ syslog.audit_severity }}    {{ syslog.audit_log_file }}

# Отправка на удалённый сервер, если задан
{% if syslog.remote_server %}
{{ syslog.audit_facility }}.{{ syslog.audit_severity }}    @{{ syslog.remote_server }}:{{ syslog.remote_port }}
{% endif %}

# Можно добавить другие правила, если нужно
```

templates/syslog-ng.conf.j2

```jinja2
@version: {{ syslog_ng_config.version }}
@include "scl.conf"

source s_sys {
    system();
    internal();
};

destination d_audit {
    file("{{ syslog.audit_log_file }}");
};

{% if syslog.remote_server %}
destination d_remote {
    {{ 'udp' if syslog.remote_protocol == 'udp' else 'tcp' }}("{{ syslog.remote_server }}" port({{ syslog.remote_port }}));
};
{% endif %}

log {
    source(s_sys);
    filter(f_audit);
    destination(d_audit);
    {% if syslog.remote_server %}destination(d_remote);{% endif %}
};

filter f_audit {
    facility({{ syslog.audit_facility }});
};
```

templates/syslog.conf.j2 (для классического syslogd)

```jinja2
#  События local6 в отдельный файл
{{ syslog.audit_facility }}.{{ syslog.audit_severity }}    {{ syslog.audit_log_file }}

{% if syslog.remote_server %}
# Отправка на удалённый сервер
{{ syslog.audit_facility }}.{{ syslog.audit_severity }}    @{{ syslog.remote_server }}
{% endif %}
```

---

6. meta/main.yml (опционально)

```yaml
---
galaxy_info:
  author: your_name
  description: Установка и настройка auditd и syslog с отправкой через audispd
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: EL
      versions:
        - 7
        - 8
        - 9
    - name: Ubuntu
      versions:
        - focal
        - jammy
    - name: Debian
      versions:
        - bullseye
        - bookworm
  galaxy_tags:
    - auditd
    - syslog
    - rsyslog
    - syslog-ng
    - audispd
    - security
dependencies: []
```

---

Пример использования в плейбуке

```yaml
- hosts: all
  become: yes
  vars:
    syslog_daemon: rsyslog
    syslog:
      remote_server: "192.168.1.100"
      remote_port: 514
      remote_protocol: udp
      audit_log_file: /var/log/audit-forwarded.log
      audit_facility: local6
    audisp_syslog_plugin:
      active: yes
      args: "LOG_LOCAL6"   # установка facility local6
  roles:
    - audit_syslog
```

Проверка

· После выполнения роли убедитесь, что файлы конфигурации созданы по правильным путям.
· Проверьте, что сервисы запущены и события аудита попадают в файл /var/log/audit-forwarded.log (или указанный).
· Если задан remote_server, проверьте отправку на удалённый syslog-сервер.

Эта роль теперь корректно обрабатывает все предыдущие ошибки и готова к использованию.

----------------------fix------------------

Обновлённая роль с параметрами write_log

Добавлены переменные write_local_log и write_remote_log в секцию syslog для гибкого управления записью событий аудита в локальный файл и на удалённый сервер. Это позволяет отдельно включать/отключать каждый канал логирования.

Обновлённый defaults/main.yml

```yaml
---
# Выбор демона syslog: "rsyslog", "syslog-ng" или "syslogd"
syslog_daemon: rsyslog

# Настройки auditd
auditd:
  log_file: /var/log/audit/audit.log
  log_format: RAW
  log_group: root
  priority_boost: 4
  flush: INCREMENTAL
  freq: 20
  num_logs: 5
  disp_qos: lossy
  dispatcher: /sbin/audispd
  name_format: NONE
  max_log_file: 8
  max_log_file_action: ROTATE
  space_left: 75
  space_left_action: SYSLOG
  action_mail_acct: root
  admin_space_left: 50
  admin_space_left_action: SUSPEND
  disk_full_action: SUSPEND
  disk_error_action: SUSPEND

# Настройки audispd (общие)
audispd:
  q_depth: 80
  overflow_action: SYSLOG
  priority_boost: 4
  max_restarts: 10
  name_format: HOSTNAME

# Конфигурация плагина syslog для audispd
audisp_syslog_plugin:
  active: yes
  direction: out
  path: /sbin/audisp-syslog
  type: builtin
  args: ""                # можно указать facility, например "LOG_LOCAL6"
  format: string

# Общие настройки syslog (для всех демонов)
syslog:
  # Файл, куда будут писаться события аудита (локально)
  audit_log_file: /var/log/audit-forwarded.log
  # Удалённый сервер (опционально)
  remote_server: ""
  remote_port: 514
  remote_protocol: udp   # udp или tcp
  # Facility и severity для событий аудита (используется в правилах)
  audit_facility: local6
  audit_severity: info
  # Флаги управления записью
  write_local_log: yes    # записывать ли в локальный файл
  write_remote_log: yes   # отправлять ли на удалённый сервер (если задан)

# Специфичные настройки для rsyslog (дополнительно)
rsyslog_config:
  modules:
    - imuxsock
    - imjournal
  work_directory: /var/spool/rsyslog

# Специфичные настройки для syslog-ng
syslog_ng_config:
  version: 3.38
  include: /etc/syslog-ng/conf.d/*.conf
```

Обновлённые шаблоны syslog-демонов

templates/rsyslog.conf.j2

```jinja2
#### GLOBAL DIRECTIVES ####
module(load="imuxsock")
module(load="imjournal")
module(load="imklog")

#### RULES ####
{% if syslog.write_local_log %}
# Сообщения от auditd (facility local6) пишем в отдельный файл
{{ syslog.audit_facility }}.{{ syslog.audit_severity }}    {{ syslog.audit_log_file }}
{% endif %}

{% if syslog.write_remote_log and syslog.remote_server %}
# Отправка на удалённый сервер
{{ syslog.audit_facility }}.{{ syslog.audit_severity }}    @{{ syslog.remote_server }}:{{ syslog.remote_port }}
{% endif %}
```

templates/syslog-ng.conf.j2

```jinja2
@version: {{ syslog_ng_config.version }}
@include "scl.conf"

source s_sys {
    system();
    internal();
};

{% if syslog.write_local_log %}
destination d_audit {
    file("{{ syslog.audit_log_file }}");
};
{% endif %}

{% if syslog.write_remote_log and syslog.remote_server %}
destination d_remote {
    {{ 'udp' if syslog.remote_protocol == 'udp' else 'tcp' }}("{{ syslog.remote_server }}" port({{ syslog.remote_port }}));
};
{% endif %}

log {
    source(s_sys);
    filter(f_audit);
    {% if syslog.write_local_log %}destination(d_audit);{% endif %}
    {% if syslog.write_remote_log and syslog.remote_server %}destination(d_remote);{% endif %}
};

filter f_audit {
    facility({{ syslog.audit_facility }});
};
```

templates/syslog.conf.j2 (для классического syslogd)

```jinja2
{% if syslog.write_local_log %}
# События local6 в отдельный файл
{{ syslog.audit_facility }}.{{ syslog.audit_severity }}    {{ syslog.audit_log_file }}
{% endif %}

{% if syslog.write_remote_log and syslog.remote_server %}
# Отправка на удалённый сервер
{{ syslog.audit_facility }}.{{ syslog.audit_severity }}    @{{ syslog.remote_server }}
{% endif %}
```

Дополнительные пояснения

· write_local_log: если yes (по умолчанию), то в конфигурации syslog-демона создаётся правило записи в локальный файл, указанный в audit_log_file.
· write_remote_log: если yes (по умолчанию) и задан remote_server, то в конфигурацию добавляется правило отправки на удалённый сервер.
· Оба флага работают независимо, позволяя, например, писать локально, но не отправлять на удалённый сервер, и наоборот.

Пример использования

```yaml
- hosts: all
  vars:
    syslog_daemon: rsyslog
    syslog:
      remote_server: "192.168.1.100"
      write_local_log: no        # не писать локально
      write_remote_log: yes      # только отправлять на сервер
  roles:
    - audit_syslog
```

Теперь роль позволяет гибко управлять записью логов через параметры write_local_log и write_remote_log.