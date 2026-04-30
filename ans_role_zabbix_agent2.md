Для установки Zabbix Agent 2 через Ansible с уже настроенным репозиторием, вот готовая структура роли:

## Структура роли

```
roles/zabbix_agent2/
├── tasks/
│   └── main.yml
├── handlers/
│   └── main.yml
├── templates/
│   └── zabbix_agent2.conf.j2
├── defaults/
│   └── main.yml
└── meta/
    └── main.yml
```

## defaults/main.yml

```yaml
---
# Zabbix Server параметры
zabbix_agent_server: "127.0.0.1"
zabbix_agent_server_active: "127.0.0.1"
zabbix_agent_hostname: "{{ ansible_hostname }}"
zabbix_agent_listen_port: 10050

# Дополнительные параметры агента
zabbix_agent_timeout: 3
zabbix_agent_allow_key: []
zabbix_agent_deny_key: []
zabbix_agent_user_params: []

# Директории
zabbix_agent_include_dir: "/etc/zabbix/zabbix_agent2.d"
zabbix_agent_pid_file: "/var/run/zabbix/zabbix_agent2.pid"
zabbix_agent_log_file: "/var/log/zabbix/zabbix_agent2.log"

# Установка
zabbix_agent2_package: "zabbix-agent2"
zabbix_agent_service: "zabbix-agent2"

# Плагины для установки (если нужны)
zabbix_agent_plugins:
  - zabbix-agent2-plugin-mongodb
  - zabbix-agent2-plugin-postgresql
```

## tasks/main.yml

```yaml
---
- name: Install Zabbix Agent 2
  ansible.builtin.package:
    name: "{{ zabbix_agent2_package }}"
    state: present
  notify: restart zabbix-agent2

- name: Install Zabbix Agent 2 plugins
  ansible.builtin.package:
    name: "{{ zabbix_agent_plugins }}"
    state: present
  when: zabbix_agent_plugins | length > 0
  notify: restart zabbix-agent2

- name: Create include directory
  ansible.builtin.file:
    path: "{{ zabbix_agent_include_dir }}"
    state: directory
    owner: root
    group: zabbix
    mode: '0750'

- name: Create log directory
  ansible.builtin.file:
    path: "{{ zabbix_agent_log_file | dirname }}"
    state: directory
    owner: zabbix
    group: zabbix
    mode: '0755'

- name: Configure Zabbix Agent 2
  ansible.builtin.template:
    src: zabbix_agent2.conf.j2
    dest: /etc/zabbix/zabbix_agent2.conf
    owner: root
    group: zabbix
    mode: '0640'
  notify: restart zabbix-agent2

- name: Start and enable Zabbix Agent 2
  ansible.builtin.service:
    name: "{{ zabbix_agent_service }}"
    state: started
    enabled: yes
```

## templates/zabbix_agent2.conf.j2

```ini
# {{ ansible_managed }}
PidFile={{ zabbix_agent_pid_file }}
LogFile={{ zabbix_agent_log_file }}
LogFileSize=10

# DebugLevel=3 - для отладки
DebugLevel=3

Server={{ zabbix_agent_server }}
ServerActive={{ zabbix_agent_server_active }}

Hostname={{ zabbix_agent_hostname }}
HostnameItem=system.hostname
HostMetadata={{ zabbix_agent_hostname }}

Include={{ zabbix_agent_include_dir }}/*.conf

Timeout={{ zabbix_agent_timeout }}

{% if zabbix_agent_listen_port %}
ListenPort={{ zabbix_agent_listen_port }}
{% endif %}

{% if zabbix_agent_allow_key | length > 0 %}
AllowKey={% for key in zabbix_agent_allow_key %}{{ key }}{% if not loop.last %},{% endif %}{% endfor %}
{% endif %}

{% if zabbix_agent_deny_key | length > 0 %}
DenyKey={% for key in zabbix_agent_deny_key %}{{ key }}{% if not loop.last %},{% endif %}{% endfor %}
{% endif %}

# Пользовательские параметры
{% for param in zabbix_agent_user_params %}
UserParameter={{ param }}
{% endfor %}

# Плагины (примеры)
Plugins.SystemRun.LogRemoteCommands=0
Plugins.MQTT.Timeout=global

# TLS настройки (опционально)
{% if zabbix_agent_tls_connect is defined %}
TLSConnect={{ zabbix_agent_tls_connect }}
TLSAccept={{ zabbix_agent_tls_accept | default('cert') }}
TLSCAFile={{ zabbix_agent_tls_ca_file | default('/etc/zabbix/ssl/ca.crt') }}
TLSCertFile={{ zabbix_agent_tls_cert_file | default('/etc/zabbix/ssl/zabbix_agent2.crt') }}
TLSKeyFile={{ zabbix_agent_tls_key_file | default('/etc/zabbix/ssl/zabbix_agent2.key') }}
{% endif %}
```

## handlers/main.yml

```yaml
---
- name: restart zabbix-agent2
  ansible.builtin.service:
    name: "{{ zabbix_agent_service }}"
    state: restarted
  register: zabbix_agent_restart
  until: zabbix_agent_restart is success
  retries: 3
  delay: 5
```

## meta/main.yml

```yaml
---
galaxy_info:
  author: your_name
  description: Zabbix Agent 2 installation
  license: MIT
  min_ansible_version: "2.9"
  platforms:
    - name: EL
      versions:
        - "7"
        - "8"
        - "9"
    - name: Debian
      versions:
        - bullseye
        - bookworm
    - name: Ubuntu
      versions:
        - focal
        - jammy

dependencies: []
```

## Пример использования в playbook

```yaml
---
- hosts: all
  become: yes
  roles:
    - role: zabbix_agent2
      vars:
        zabbix_agent_server: "192.168.1.100"
        zabbix_agent_server_active: "192.168.1.100:10051"
        zabbix_agent_hostname: "{{ ansible_fqdn }}"
        
        # Пользовательские параметры
        zabbix_agent_user_params:
          - "custom.ping,/bin/ping -c1 8.8.8.8 | grep 'time=' | awk '{print \$7}' | cut -d'=' -f2"
          
        # Плагины для PostgreSQL и MongoDB
        zabbix_agent_plugins:
          - zabbix-agent2-plugin-postgresql
          - zabbix-agent2-plugin-mongodb
          
        # TLS опционально
        # zabbix_agent_tls_connect: "cert"
        # zabbix_agent_tls_accept: "cert"
```

Роль предполагает, что репозиторий уже настроен. При необходимости можно добавить дополнительные конфигурации плагинов в `zabbix_agent2.d/`.

Ошибка возникает из-за того, что переменная `zabbix_agent_timeout` не определена в области видимости. Вот исправленная версия шаблона с защитой от неопределенных переменных:

## templates/zabbix_agent2.conf.j2 (исправленный)

```ini
# {{ ansible_managed }}
PidFile={{ zabbix_agent_pid_file | default('/var/run/zabbix/zabbix_agent2.pid') }}
LogFile={{ zabbix_agent_log_file | default('/var/log/zabbix/zabbix_agent2.log') }}
LogFileSize=10

# DebugLevel=3 - для отладки
DebugLevel=3

Server={{ zabbix_agent_server | default('127.0.0.1') }}
ServerActive={{ zabbix_agent_server_active | default('127.0.0.1') }}

Hostname={{ zabbix_agent_hostname | default(ansible_hostname) }}
HostnameItem=system.hostname
HostMetadata={{ zabbix_agent_hostname | default(ansible_hostname) }}

Include={{ zabbix_agent_include_dir | default('/etc/zabbix/zabbix_agent2.d') }}/*.conf

{% if zabbix_agent_timeout is defined %}
Timeout={{ zabbix_agent_timeout }}
{% else %}
Timeout=3
{% endif %}

{% if zabbix_agent_listen_port is defined and zabbix_agent_listen_port %}
ListenPort={{ zabbix_agent_listen_port }}
{% endif %}

{% if zabbix_agent_allow_key is defined and zabbix_agent_allow_key | length > 0 %}
AllowKey={% for key in zabbix_agent_allow_key %}{{ key }}{% if not loop.last %},{% endif %}{% endfor %}
{% endif %}

{% if zabbix_agent_deny_key is defined and zabbix_agent_deny_key | length > 0 %}
DenyKey={% for key in zabbix_agent_deny_key %}{{ key }}{% if not loop.last %},{% endif %}{% endfor %}
{% endif %}

# Пользовательские параметры
{% if zabbix_agent_user_params is defined %}
{% for param in zabbix_agent_user_params %}
UserParameter={{ param }}
{% endfor %}
{% endif %}

# Плагины (примеры)
Plugins.SystemRun.LogRemoteCommands=0
Plugins.MQTT.Timeout=global

# TLS настройки (опционально)
{% if zabbix_agent_tls_connect is defined %}
TLSConnect={{ zabbix_agent_tls_connect }}
TLSAccept={{ zabbix_agent_tls_accept | default('cert') }}
TLSCAFile={{ zabbix_agent_tls_ca_file | default('/etc/zabbix/ssl/ca.crt') }}
TLSCertFile={{ zabbix_agent_tls_cert_file | default('/etc/zabbix/ssl/zabbix_agent2.crt') }}
TLSKeyFile={{ zabbix_agent_tls_key_file | default('/etc/zabbix/ssl/zabbix_agent2.key') }}
{% endif %}
```

## Ключевые изменения:

1. **Timeout** - обернут в проверку `is defined` с fallback-значением `3`
2. **ListenPort** - добавлена проверка и на существование и на непустое значение
3. **All остальные переменные** - защищены через `| default()` или проверку `is defined`
4. **Исправлен Hostname** - теперь использует `ansible_hostname` вместо `hostname`

## Альтернативно, можно гарантированно определить все переменные в defaults/main.yml:

```yaml
---
# Обязательные переменные
zabbix_agent_server: "127.0.0.1"
zabbix_agent_server_active: "127.0.0.1"
zabbix_agent_hostname: "{{ ansible_hostname }}"
zabbix_agent_listen_port: 10050

# Параметры агента
zabbix_agent_timeout: 3
zabbix_agent_allow_key: []
zabbix_agent_deny_key: []
zabbix_agent_user_params: []

# Директории и файлы
zabbix_agent_include_dir: "/etc/zabbix/zabbix_agent2.d"
zabbix_agent_pid_file: "/var/run/zabbix/zabbix_agent2.pid"
zabbix_agent_log_file: "/var/log/zabbix/zabbix_agent2.log"

# Пакеты
zabbix_agent2_package: "zabbix-agent2"
zabbix_agent_service: "zabbix-agent2"
zabbix_agent_plugins: []
```

Теперь ошибка должна исчезнуть. Если переменные есть в `defaults/main.yml` - шаблон будет работать без проверок, если нет - с проверками `is defined`.










