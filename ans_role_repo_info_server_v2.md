Вот пример Ansible роли для сбора информации с серверов в CSV-отчет.

Структура роли:

```
collect_info_role/
├── tasks/
│   └── main.yml
├── templates/
│   └── report.csv.j2
├── files/
│   └── parsers.py
└── defaults/
    └── main.yml
```

1. defaults/main.yml

```yaml
---
# Каталог для сохранения отчетов
report_dir: "/tmp/ansible_reports"
# Имя файла отчета
report_filename: "system_info_report.csv"
# Пути к конфигурационным файлам
postgresql_paths:
  - /var/lib/pgsql
  - /etc/postgresql
  - /opt/postgresql
```

2. tasks/main.yml

```yaml
---
- name: Создать директорию для отчетов
  delegate_to: localhost
  run_once: yes
  file:
    path: "{{ report_dir }}"
    state: directory

- name: Получить информацию об открытых портах
  shell: |
    ss -tulpn 2>/dev/null || netstat -tulpn 2>/dev/null
  register: open_ports
  changed_when: false

- name: Получить конфигурацию firewall
  shell: |
    iptables-save 2>/dev/null || nft list ruleset 2>/dev/null || echo "Firewall not found"
  register: firewall_config
  changed_when: false

- name: Получить список пользователей
  shell: "cat /etc/passwd"
  register: users
  changed_when: false

- name: Получить список групп
  shell: "cat /etc/group"
  register: groups
  changed_when: false

- name: Получить конфигурацию sudo
  shell: |
    {
      echo "=== /etc/sudoers ==="
      cat /etc/sudoers 2>/dev/null || echo "File not found"
      echo ""
      echo "=== /etc/sudoers.d/ files ==="
      for file in /etc/sudoers.d/*; do
        if [ -f "$file" ]; then
          echo "--- $file ---"
          cat "$file"
          echo ""
        fi
      done
    }
  register: sudo_config
  changed_when: false

- name: Получить конфигурацию SSSD
  shell: "cat /etc/sssd/sssd.conf 2>/dev/null || echo 'File not found'"
  register: sssd_config
  changed_when: false

- name: Получить конфигурацию nsswitch
  shell: "cat /etc/nsswitch.conf"
  register: nsswitch_config
  changed_when: false

- name: Найти конфигурационные файлы PostgreSQL
  find:
    paths: "{{ postgresql_paths }}"
    patterns: "pg_hba.conf,postgresql.conf"
    recurse: yes
  register: postgresql_files
  changed_when: false

- name: Получить содержимое PostgreSQL конфигураций
  shell: |
    {
      echo "=== PostgreSQL Configuration Files ==="
      {% for file in postgresql_files.files %}
      echo "--- {{ file.path }} ---"
      cat "{{ file.path }}" 2>/dev/null || echo "Cannot read file"
      echo ""
      {% endfor %}
    }
  register: postgresql_config
  changed_when: false
  when: postgresql_files.files | length > 0

- name: Получить список включенных служб
  shell: "systemctl list-unit-files --type=service --state=enabled"
  register: enabled_services
  changed_when: false

- name: Собрать всю информацию в одну переменную
  set_fact:
    collected_info: |
      {
        "hostname": "{{ ansible_hostname }}",
        "ip": "{{ ansible_default_ipv4.address }}",
        "open_ports": {{ open_ports.stdout_lines | to_json }},
        "firewall_config": {{ firewall_config.stdout | to_json }},
        "users_count": {{ users.stdout_lines | length }},
        "groups_count": {{ groups.stdout_lines | length }},
        "sudo_config": {{ sudo_config.stdout | to_json }},
        "sssd_config": {{ sssd_config.stdout | to_json }},
        "nsswitch_config": {{ nsswitch_config.stdout | to_json }},
        "postgresql_config": {{ postgresql_config.stdout | default('Not found') | to_json }},
        "enabled_services": {{ enabled_services.stdout_lines | to_json }},
        "timestamp": "{{ ansible_date_time.iso8601 }}"
      }

- name: Сохранить данные в CSV на control node
  delegate_to: localhost
  run_once: yes
  template:
    src: report.csv.j2
    dest: "{{ report_dir }}/{{ report_filename }}"
  when: inventory_hostname == play_hosts[0]
```

3. templates/report.csv.j2

```jinja2
hostname;ip;timestamp;open_ports;firewall_config;users_count;groups_count;sudo_config;sssd_config;nsswitch_config;postgresql_config;enabled_services
{% for host in play_hosts %}
{{ hostvars[host].collected_info.hostname }};{{ hostvars[host].collected_info.ip }};{{ hostvars[host].collected_info.timestamp }};"{{ hostvars[host].collected_info.open_ports | join(' | ') }}";"{{ hostvars[host].collected_info.firewall_config }}";{{ hostvars[host].collected_info.users_count }};{{ hostvars[host].collected_info.groups_count }};"{{ hostvars[host].collected_info.sudo_config }}";"{{ hostvars[host].collected_info.sssd_config }}";"{{ hostvars[host].collected_info.nsswitch_config }}";"{{ hostvars[host].collected_info.postgresql_config }}";"{{ hostvars[host].collected_info.enabled_services | join(' | ') }}"
{% endfor %}
```

4. Плейбук для использования роли

```yaml
---
- name: Сбор информации с серверов
  hosts: all
  gather_facts: yes
  tasks:
    - name: Включить роль сбора информации
      include_role:
        name: collect_info_role
```

5. Альтернативный вариант с модулем assemble (для объединения файлов)

Для очень большого количества серверов можно использовать такой подход:

```yaml
---
- name: Сбор информации с серверов
  hosts: all
  gather_facts: yes
  tasks:
    - name: Создать временную директорию
      tempfile:
        state: directory
        suffix: ansible_collect
      register: temp_dir
      delegate_to: localhost
      run_once: yes

    - name: Собрать информацию (все задачи из роли)

    - name: Сохранить данные в отдельный файл для каждого хоста
      delegate_to: localhost
      copy:
        content: |
          hostname: {{ ansible_hostname }}
          ip: {{ ansible_default_ipv4.address }}
          open_ports: |
            {{ open_ports.stdout }}
          firewall: |
            {{ firewall_config.stdout }}
          users_count: {{ users.stdout_lines | length }}
          groups_count: {{ groups.stdout_lines | length }}
          enabled_services: |
            {{ enabled_services.stdout }}
        dest: "{{ temp_dir.path }}/{{ inventory_hostname }}.txt"

- name: Объединить все файлы в CSV
  hosts: localhost
  gather_facts: no
  tasks:
    - name: Создать CSV заголовок
      copy:
        content: "hostname;ip;open_ports;firewall;users_count;groups_count;enabled_services\n"
        dest: "{{ report_dir }}/final_report.csv"

    - name: Добавить данные каждого хоста в CSV
      shell: |
        for file in {{ temp_dir.path }}/*.txt; do
          awk '
            BEGIN { FS=": "; OFS=";" }
            /^hostname:/ { hostname=$2 }
            /^ip:/ { ip=$2 }
            /^users_count:/ { users=$2 }
            /^groups_count:/ { groups=$2 }
            /^open_ports:/ {
              getline;
              ports="";
              while ($0 !~ /^[a-zA-Z_]+:/ && length($0) > 0) {
                ports=ports $0 "\\n";
                getline;
              }
            }
            /^enabled_services:/ {
              services="";
              while (getline && length($0) > 0) {
                services=services $0 " ";
              }
            }
            END {
              print "\"" hostname "\"", "\"" ip "\"", "\"" ports "\"", "\"" users "\"", "\"" groups "\"", "\"" services "\""
            }
          ' "$file" >> {{ report_dir }}/final_report.csv
        done
      args:
        executable: /bin/bash
```

Рекомендации:

1. Для очень большого количества серверов используйте стратегию free и ограничивайте параллельное выполнение:

```yaml
- name: Сбор информации
  hosts: all
  strategy: free
  serial: 50  # Обрабатывать по 50 серверов одновременно
```

1. Добавьте обработку ошибок в задачи с block/rescue:

```yaml
- name: Попытка получения данных
  block:
    - name: Получить информацию
      shell: команда
      register: result
  rescue:
    - name: Записать ошибку
      set_fact:
        error: "Failed to collect data"
```

1. Используйте фильтры для CSV:

```yaml
# В шаблоне
"{{ value | replace('"', '""') | replace('\n', ' ') }}"
```

1. Оптимизируйте производительность:

· Отключите gather_facts если не нужны все факты Ansible
· Используйте async для длительных операций
· Настройте timeout для команд

Эта роль создаст CSV файл с разделителем точка с запятой, где каждая строка соответствует одному серверу, а поля содержат всю запрошенную информацию.