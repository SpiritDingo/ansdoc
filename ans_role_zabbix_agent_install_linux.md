Вот пример Ansible роли для установки и настройки Zabbix агента на Linux (Ubuntu 22.04, 24.04 и Oracle Linux 9):

Структура роли

```
zabbix-agent/
├── tasks/
│   ├── main.yml
│   ├── ubuntu.yml
│   └── oraclelinux.yml
├── templates/
│   └── zabbix_agent2.conf.j2
├── vars/
│   ├── ubuntu.yml
│   ├── ubuntu_24.yml
│   └── oraclelinux_9.yml
├── defaults/
│   └── main.yml
└── handlers/
    └── main.yml
```

defaults/main.yml

```yaml
# Основные настройки Zabbix агента
zabbix_version: "6.0"
zabbix_agent_package: "zabbix-agent2"
zabbix_agent_service: "zabbix-agent2"

# Настройки подключения
zabbix_server: "zabbix-server.example.com"
zabbix_server_active: "zabbix-server.example.com"
zabbix_agent_hostname: "{{ ansible_hostname }}"

# Настройки агента
zabbix_agent_listenport: "10050"
zabbix_agent_listenip: "0.0.0.0"
zabbix_agent_timeout: "30"
zabbix_agent_buffer_send: "5"
zabbix_agent_buffer_size: "100"
zabbix_agent_debug_level: "3"
zabbix_agent_enable_remote_commands: "0"
zabbix_agent_log_remote_commands: "0"

# TLS настройки (опционально)
zabbix_agent_tls_connect: "unencrypted"
zabbix_agent_tls_accept: "unencrypted"
zabbix_agent_tls_psk_identity: ""
zabbix_agent_tls_psk_file: ""

# Пользовательские параметры
zabbix_user_parameters: []
```

vars/ubuntu.yml (для Ubuntu 22.04)

```yaml
zabbix_repo_url: "https://repo.zabbix.com/zabbix/{{ zabbix_version }}/ubuntu/pool/main/z/zabbix-release"
zabbix_repo_package: "zabbix-release_{{ zabbix_version }}-1+ubuntu22.04_all.deb"
zabbix_repo_gpg_key: "https://repo.zabbix.com/zabbix-official-repo.key"
```

vars/ubuntu_24.yml (для Ubuntu 24.04)

```yaml
zabbix_repo_url: "https://repo.zabbix.com/zabbix/{{ zabbix_version }}/ubuntu/pool/main/z/zabbix-release"
zabbix_repo_package: "zabbix-release_{{ zabbix_version }}-1+ubuntu24.04_all.deb"
zabbix_repo_gpg_key: "https://repo.zabbix.com/zabbix-official-repo.key"
```

vars/oraclelinux_9.yml

```yaml
zabbix_repo_url: "https://repo.zabbix.com/zabbix/{{ zabbix_version }}/rhel/9/x86_64"
zabbix_repo_package: "zabbix-release-{{ zabbix_version }}-1.el9.noarch.rpm"
zabbix_repo_gpg_key: "https://repo.zabbix.com/zabbix-official-repo.key"
```

tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ item }}"
  with_first_found:
    - "{{ ansible_distribution }}_{{ ansible_distribution_major_version }}.yml"
    - "{{ ansible_distribution }}.yml"
    - "{{ ansible_os_family }}.yml"
  tags:
    - vars

- name: Include distribution-specific tasks
  include_tasks: "{{ ansible_distribution | lower }}.yml"
  tags:
    - installation

- name: Configure Zabbix agent
  template:
    src: zabbix_agent2.conf.j2
    dest: /etc/zabbix/zabbix_agent2.conf
    owner: root
    group: zabbix
    mode: '0640'
  notify:
    - restart zabbix agent
  tags:
    - configuration

- name: Create directory for user parameters
  file:
    path: /etc/zabbix/zabbix_agent2.d
    state: directory
    owner: root
    group: zabbix
    mode: '0750'
  tags:
    - configuration

- name: Add user parameters
  copy:
    content: "{{ item }}"
    dest: "/etc/zabbix/zabbix_agent2.d/user_parameter_{{ loop.index }}.conf"
    owner: root
    group: zabbix
    mode: '0640'
  loop: "{{ zabbix_user_parameters }}"
  when: zabbix_user_parameters | length > 0
  notify:
    - restart zabbix agent
  tags:
    - userparameters

- name: Enable and start Zabbix agent service
  systemd:
    name: "{{ zabbix_agent_service }}"
    state: started
    enabled: true
    daemon_reload: true
  tags:
    - service
```

tasks/ubuntu.yml

```yaml
---
- name: Install prerequisites (Ubuntu)
  apt:
    name:
      - wget
      - gnupg
      - lsb-release
    state: present
    update_cache: true
  tags:
    - prerequisites

- name: Download and install Zabbix repository (Ubuntu)
  apt:
    deb: "{{ zabbix_repo_url }}/{{ zabbix_repo_package }}"
    state: present
  tags:
    - repository

- name: Install Zabbix agent (Ubuntu)
  apt:
    name:
      - "{{ zabbix_agent_package }}"
    state: present
    update_cache: true
  tags:
    - installation
```

tasks/oraclelinux.yml

```yaml
---
- name: Install prerequisites (Oracle Linux)
  dnf:
    name:
      - wget
      - yum-utils
    state: present
  tags:
    - prerequisites

- name: Download and install Zabbix repository (Oracle Linux)
  rpm:
    name: "{{ zabbix_repo_url }}/{{ zabbix_repo_package }}"
    state: present
  tags:
    - repository

- name: Clean DNF cache (Oracle Linux)
  command: dnf clean all
  args:
    warn: false
  tags:
    - repository

- name: Install Zabbix agent (Oracle Linux)
  dnf:
    name:
      - "{{ zabbix_agent_package }}"
    state: present
    enablerepo: "zabbix,zabbix-frontend"
  tags:
    - installation

- name: Configure SELinux for Zabbix agent (Oracle Linux)
  selinux:
    policy: targeted
    state: permissive
  when: ansible_selinux.status == "enabled"
  tags:
    - selinux
```

templates/zabbix_agent2.conf.j2

```jinja2
# This file is managed by Ansible
# Zabbix agent configuration file

# Basic settings
PidFile=/run/zabbix/zabbix_agent2.pid
LogFile=/var/log/zabbix/zabbix_agent2.log
LogFileSize=0
DebugLevel={{ zabbix_agent_debug_level }}

# Connection settings
Server={{ zabbix_server }}
ServerActive={{ zabbix_server_active }}
Hostname={{ zabbix_agent_hostname }}

# Network settings
ListenPort={{ zabbix_agent_listenport }}
ListenIP={{ zabbix_agent_listenip }}
SourceIP=

# Timeouts
Timeout={{ zabbix_agent_timeout }}

# Buffers
BufferSend={{ zabbix_agent_buffer_send }}
BufferSize={{ zabbix_agent_buffer_size }}

# Remote commands
EnableRemoteCommands={{ zabbix_agent_enable_remote_commands }}
LogRemoteCommands={{ zabbix_agent_log_remote_commands }}

# TLS settings
TLSConnect={{ zabbix_agent_tls_connect }}
TLSAccept={{ zabbix_agent_tls_accept }}
TLSPSKIdentity={{ zabbix_agent_tls_psk_identity }}
TLSPSKFile={{ zabbix_agent_tls_psk_file }}

# Include directories
Include=/etc/zabbix/zabbix_agent2.d/*.conf
Include=/etc/zabbix/zabbix_agent2.d/plugins.d/*.conf

# Plugins
LoadModulePath=/usr/lib/zabbix-agent2/plugins
```

handlers/main.yml

```yaml
---
- name: restart zabbix agent
  systemd:
    name: "{{ zabbix_agent_service }}"
    state: restarted
    daemon_reload: true
```

Пример playbook

```yaml
---
- name: Install and configure Zabbix Agent
  hosts: all
  become: true
  vars:
    zabbix_server: "192.168.1.100"
    zabbix_server_active: "192.168.1.100"
    zabbix_agent_hostname: "{{ inventory_hostname }}"
    
    # Пример пользовательских параметров
    zabbix_user_parameters:
      - "UserParameter=app.version,cat /opt/app/VERSION"
      - "UserParameter=app.process.count,ps aux | grep -c appname"
      
  roles:
    - zabbix-agent
```

Использование

1. Создайте структуру роли:

```bash
mkdir -p zabbix-agent/{tasks,templates,vars,defaults,handlers}
```

1. Разместите файлы в соответствующих директориях
2. Используйте в playbook:

```yaml
- hosts: servers
  roles:
    - role: zabbix-agent
      vars:
        zabbix_server: "your-zabbix-server.com"
        zabbix_agent_hostname: "{{ ansible_fqdn }}"
```

Особенности для разных ОС

Ubuntu 22.04/24.04:

· Использует APT для управления пакетами
· Автоматически устанавливает репозиторий Zabbix

Oracle Linux 9:

· Использует DNF для управления пакетами
· Настраивает SELinux в permissive mode
· Устанавливает репозиторий из RPM пакета

Дополнительные возможности

Для расширения функциональности можно добавить:

1. Настройку TLS/PSK шифрования
2. Мониторинг специфичных сервисов
3. Автоматическую регистрацию на Zabbix сервере
4. Кастомные проверки и скрипты