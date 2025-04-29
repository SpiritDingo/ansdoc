# Ansible роль для настройки SOC инфраструктуры

Роль автоматизирует базовую настройку SOC (Security Operations Center) на Linux серверах, включая создание специализированных пользователей, настройку auditd, rsyslog и интеграцию с SIEM системой.

## Структура роли

```
roles/soc_setup/
├── defaults/
│   └── main.yml          # Параметры по умолчанию
├── tasks/
│   ├── main.yml          # Основной плейбук
│   ├── users.yml         # Создание пользователей SOC
│   ├── auditd.yml        # Настройка auditd
│   ├── audispd.yml       # Настройка audispd
│   ├── rsyslog.yml       # Настройка rsyslog
│   └── hardening.yml     # Дополнительное укрепление
├── templates/
│   ├── auditd.conf.j2    # Шаблон auditd
│   ├── audispd.conf.j2   # Шаблон audispd
│   ├── rsyslog.conf.j2   # Шаблон rsyslog
│   └── audit.rules.j2    # Шаблон правил аудита
└── handlers/
    └── main.yml          # Обработчики сервисов
```

## Основные файлы роли

### defaults/main.yml

```yaml
---
# Пользователи SOC
soc_users:
  - name: "soc_analyst"
    groups: ["sudo", "adm"]
    ssh_key: "ssh-rsa AAAAB3NzaC1yc2E..."
    shell: "/bin/bash"
  
  - name: "soc_investigator"
    groups: ["sudo", "adm"]
    ssh_key: "ssh-rsa AAAAB3NzaC2E..."
    shell: "/bin/bash"

# Настройки auditd
auditd:
  log_file: "/var/log/audit/audit.log"
  log_format: "RAW"
  log_group: "root"
  priority_boost: 4
  flush: "INCREMENTAL"
  freq: 20
  max_log_file: 50
  num_logs: 5
  dispatcher: "/sbin/audispd"
  name_format: "NONE"
  admin_space_left: 75
  space_left: 25
  action_mail_acct: "root"
  disk_full_action: "SUSPEND"
  disk_error_action: "SUSPEND"

# Настройки audispd
audispd:
  remote_server: "siem.example.com"
  remote_port: 514
  protocol: "tcp"
  format: "ascii"
  network_failure_action: "stop"

# Настройки rsyslog
rsyslog:
  enable_tls: true
  tls_ca_file: "/etc/ssl/certs/siem-ca.pem"
  tls_cert_file: "/etc/ssl/certs/client-cert.pem"
  tls_key_file: "/etc/ssl/private/client-key.pem"
  remote_server: "siem.example.com"
  remote_port: 6514
  log_files:
    - "/var/log/auth.log"
    - "/var/log/syslog"
    - "/var/log/kern.log"
```

### tasks/main.yml

```yaml
---
- name: Install required packages
  ansible.builtin.package:
    name:
      - auditd
      - audispd-plugins
      - rsyslog-gnutls
      - openssl
    state: present

- name: Setup SOC users
  ansible.builtin.include_tasks: users.yml
  tags: users

- name: Configure auditd
  ansible.builtin.include_tasks: auditd.yml
  tags: auditd

- name: Configure audispd
  ansible.builtin.include_tasks: audispd.yml
  tags: audispd

- name: Configure rsyslog
  ansible.builtin.include_tasks: rsyslog.yml
  tags: rsyslog

- name: Apply additional hardening
  ansible.builtin.include_tasks: hardening.yml
  tags: hardening
```

### tasks/users.yml

```yaml
---
- name: Create SOC user groups
  ansible.builtin.group:
    name: "soc_team"
    state: present

- name: Create SOC users
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups | join(',') }},soc_team"
    append: yes
    shell: "{{ item.shell }}"
    password: "{{ item.password | default(omit) }}"
    generate_ssh_key: "{{ item.ssh_key is not defined }}"
  loop: "{{ soc_users }}"
  register: users_created

- name: Deploy SSH keys for SOC users
  ansible.builtin.authorized_key:
    user: "{{ item.0.name }}"
    key: "{{ item.1 }}"
    state: present
  with_subelements:
    - "{{ soc_users }}"
    - ssh_key
    - skip_missing: true
  when: item.1 is defined

- name: Set up sudo access for SOC team
  ansible.builtin.lineinfile:
    path: /etc/sudoers
    line: "%soc_team ALL=(ALL) NOPASSWD: ALL"
    validate: "visudo -cf %s"
```

### tasks/auditd.yml

```yaml
---
- name: Configure auditd main settings
  ansible.builtin.template:
    src: "auditd.conf.j2"
    dest: "/etc/audit/auditd.conf"
    owner: root
    group: root
    mode: "0640"
  notify: Restart auditd

- name: Deploy audit rules
  ansible.builtin.template:
    src: "audit.rules.j2"
    dest: "/etc/audit/rules.d/audit.rules"
    owner: root
    group: root
    mode: "0640"
  notify: Restart auditd

- name: Ensure auditd is enabled and running
  ansible.builtin.service:
    name: auditd
    state: started
    enabled: yes
```

### tasks/audispd.yml

```yaml
---
- name: Configure audispd
  ansible.builtin.template:
    src: "audispd.conf.j2"
    dest: "/etc/audisp/audispd.conf"
    owner: root
    group: root
    mode: "0640"
  notify: Restart auditd

- name: Configure audispd remote logging
  ansible.builtin.template:
    src: "audisp-remote.conf.j2"
    dest: "/etc/audisp/plugins.d/au-remote.conf"
    owner: root
    group: root
    mode: "0640"
  notify: Restart auditd

- name: Install audispd remote plugin
  ansible.builtin.package:
    name: audispd-plugins
    state: present
```

### tasks/rsyslog.yml

```yaml
---
- name: Configure rsyslog
  ansible.builtin.template:
    src: "rsyslog.conf.j2"
    dest: "/etc/rsyslog.conf"
    owner: root
    group: root
    mode: "0644"
  notify: Restart rsyslog

- name: Configure TLS for rsyslog
  ansible.builtin.copy:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    owner: root
    group: root
    mode: "{{ item.mode }}"
  loop:
    - { src: "files/siem-ca.pem", dest: "{{ rsyslog.tls_ca_file }}", mode: "0644" }
    - { src: "files/client-cert.pem", dest: "{{ rsyslog.tls_cert_file }}", mode: "0644" }
    - { src: "files/client-key.pem", dest: "{{ rsyslog.tls_key_file }}", mode: "0600" }
  when: rsyslog.enable_tls

- name: Ensure rsyslog is enabled and running
  ansible.builtin.service:
    name: rsyslog
    state: started
    enabled: yes
```

### handlers/main.yml

```yaml
---
- name: Restart auditd
  ansible.builtin.service:
    name: auditd
    state: restarted

- name: Restart rsyslog
  ansible.builtin.service:
    name: rsyslog
    state: restarted
```

## Пример использования роли

1. Создайте playbook `setup_soc.yml`:

```yaml
---
- hosts: security_servers
  become: yes
  vars:
    soc_users:
      - name: "soc_analyst1"
        groups: ["sudo", "adm"]
        ssh_key: "ssh-rsa AAAAB3NzaC1yc2E..."
      
      - name: "soc_analyst2"
        groups: ["sudo", "adm"]
        ssh_key: "ssh-rsa AAAAB3NzaC2E..."

    audispd:
      remote_server: "siem.internal.company.com"
      remote_port: 601
      protocol: "tcp"

    rsyslog:
      remote_server: "siem.internal.company.com"
      remote_port: 6514
      enable_tls: true

  roles:
    - soc_setup
```

2. Подготовьте TLS сертификаты в директории `files/`:
   - siem-ca.pem
   - client-cert.pem
   - client-key.pem

3. Запустите playbook:

```bash
ansible-playbook -i inventory.ini setup_soc.yml
```

## Дополнительные возможности

1. **Интеграция с SIEM**:
   - Поддержка различных форматов логов (CEF, LEEF)
   - Настройка фильтрации событий перед отправкой

2. **Расширенный аудит**:
   - Правила для мониторинга критичных файлов
   - Аудит выполнения привилегированных команд

3. **Мониторинг целостности**:
   - Настройка AIDE для обнаружения изменений
   - Интеграция с системой мониторинга

4. **Управление ключами**:
   - Ротация SSH ключей
   - Централизованное управление доступом

5. **Оповещения**:
   - Настройка email-уведомлений о критичных событиях
   - Интеграция с системами ticketing

Эта роль предоставляет комплексное решение для базовой настройки SOC инфраструктуры с фокусом на сборе и передаче аудит-логов в централизованную систему мониторинга безопасности.