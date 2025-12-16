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







Вот более подробная и оптимизированная Ansible роль для сбора информации с большого количества серверов:

Структура роли

```
collect_server_info/
├── tasks/
│   ├── main.yml
│   ├── collect_data.yml
│   ├── process_data.yml
│   └── generate_report.yml
├── templates/
│   ├── report.csv.j2
│   └── summary.md.j2
├── filter_plugins/
│   └── csv_filters.py
├── library/
│   └── custom_module.py
├── defaults/
│   └── main.yml
├── vars/
│   └── main.yml
└── meta/
    └── main.yml
```

1. defaults/main.yml

```yaml
---
# Основные настройки
collect_info_dir: "/tmp/ansible_collect"
report_dir: "{{ playbook_dir }}/reports"
report_filename: "server_info_{{ ansible_date_time.date }}.csv"
max_workers: 50
timeout_seconds: 300

# Настройки сбора данных
collect_open_ports: true
collect_firewall: true
collect_users: true
collect_sudo: true
collect_sssd: true
collect_nsswitch: true
collect_postgresql: true
collect_services: true

# Пути к файлам PostgreSQL
postgresql_paths:
  - /var/lib/pgsql
  - /etc/postgresql
  - /opt/postgresql
  - /usr/local/pgsql

# Исключения
exclude_hosts: []
exclude_ports: [22]  # SSH порт
```

2. vars/main.yml

```yaml
---
# Команды для сбора данных
commands:
  open_ports: "ss -tulpn 2>/dev/null || netstat -tulpn 2>/dev/null"
  firewall: |
    if command -v iptables-save >/dev/null 2>&1; then
      iptables-save
    elif command -v nft >/dev/null 2>&1; then
      nft list ruleset
    else
      echo "No firewall tool found"
    fi
  users: "getent passwd"
  groups: "getent group"
  sudo: "cat /etc/sudoers 2>/dev/null | grep -v '^#' && find /etc/sudoers.d/ -type f -exec cat {} \\; 2>/dev/null"
  sssd: "cat /etc/sssd/sssd.conf 2>/dev/null || echo 'NOT_CONFIGURED'"
  nsswitch: "cat /etc/nsswitch.conf"
  services: "systemctl list-unit-files --type=service --state=enabled --no-legend"
```

3. tasks/main.yml

```yaml
---
- name: Валидация параметров
  include_tasks: validate_params.yml
  when: validate_params | default(true)

- name: Подготовка окружения
  include_tasks: setup_environment.yml

- name: Сбор данных с серверов
  include_tasks: collect_data.yml

- name: Обработка и анализ данных
  include_tasks: process_data.yml

- name: Генерация отчетов
  include_tasks: generate_report.yml
```

4. tasks/collect_data.yml

```yaml
---
- name: Создать временные директории на управляющем хосте
  delegate_to: localhost
  run_once: yes
  file:
    path: "{{ collect_info_dir }}/raw/{{ inventory_hostname }}"
    state: directory

- name: Собрать информацию об открытых портах
  when: collect_open_ports
  block:
    - name: Получить открытые порты
      shell: "{{ commands.open_ports }}"
      register: open_ports_result
      changed_when: false
      failed_when: false
      async: 45
      poll: 0
      register: async_ports

    - name: Дождаться завершения сбора портов
      async_status:
        jid: "{{ async_ports.ansible_job_id }}"
      register: ports_job_result
      until: ports_job_result.finished
      retries: 30
      delay: 2
      
    - name: Сохранить результат портов
      set_fact:
        open_ports_data: "{{ ports_job_result.stdout if ports_job_result.stdout else 'ERROR' }}"
        
    - name: Парсить открытые порты
      set_fact:
        parsed_ports: "{{ open_ports_data | parse_ss_output }}"
      when: "'ERROR' not in open_ports_data"

- name: Собрать конфигурацию firewall
  when: collect_firewall
  block:
    - name: Получить правила firewall
      shell: "{{ commands.firewall }}"
      register: firewall_result
      changed_when: false
      failed_when: false
      
    - name: Сохранить конфигурацию firewall
      set_fact:
        firewall_config: "{{ firewall_result.stdout }}"

- name: Собрать информацию о пользователях и группах
  when: collect_users
  block:
    - name: Получить список пользователей
      shell: "{{ commands.users }}"
      register: users_result
      changed_when: false
      
    - name: Получить список групп
      shell: "{{ commands.groups }}"
      register: groups_result
      changed_when: false
      
    - name: Анализ пользователей
      set_fact:
        users_analysis: "{{ users_result.stdout_lines | analyze_users }}"
        
    - name: Сохранить статистику
      set_fact:
        user_stats:
          total: "{{ users_result.stdout_lines | length }}"
          system: "{{ users_analysis.system_users }}"
          interactive: "{{ users_analysis.interactive_users }}"

- name: Собрать конфигурацию sudo
  when: collect_sudo
  block:
    - name: Получить правила sudo
      shell: "{{ commands.sudo }}"
      register: sudo_result
      changed_when: false
      failed_when: false
      
    - name: Сохранить конфигурацию sudo
      set_fact:
        sudo_config: "{{ sudo_result.stdout | regex_replace('\\s+', ' ') }}"

- name: Собрать конфигурацию SSSD
  when: collect_sssd
  block:
    - name: Проверить наличие SSSD
      stat:
        path: /etc/sssd/sssd.conf
      register: sssd_conf
      
    - name: Получить конфигурацию SSSD
      shell: "{{ commands.sssd }}"
      when: sssd_conf.stat.exists
      register: sssd_result
      changed_when: false
      
    - name: Сохранить конфигурацию SSSD
      set_fact:
        sssd_config: "{{ sssd_result.stdout if sssd_result is defined else 'NOT_INSTALLED' }}"

- name: Собрать конфигурацию nsswitch
  when: collect_nsswitch
  block:
    - name: Получить nsswitch.conf
      shell: "{{ commands.nsswitch }}"
      register: nsswitch_result
      changed_when: false
      
    - name: Сохранить конфигурацию nsswitch
      set_fact:
        nsswitch_config: "{{ nsswitch_result.stdout }}"

- name: Собрать конфигурацию PostgreSQL
  when: collect_postgresql
  block:
    - name: Найти файлы конфигурации PostgreSQL
      find:
        paths: "{{ postgresql_paths }}"
        patterns: "pg_hba.conf,postgresql.conf"
        recurse: yes
        file_type: file
      register: pg_files
      changed_when: false
      
    - name: Получить содержимое файлов PostgreSQL
      block:
        - name: Читать каждый файл конфигурации
          slurp:
            src: "{{ item.path }}"
          register: pg_content
          with_items: "{{ pg_files.files }}"
          
        - name: Объединить содержимое файлов
          set_fact:
            postgresql_config: |
              {% for item in pg_content.results %}
              === {{ item.item.path }} ===
              {{ item.content | b64decode }}
              
              {% endfor %}
      when: pg_files.files | length > 0
      
    - name: Установить значение по умолчанию
      set_fact:
        postgresql_config: "POSTGRESQL_NOT_FOUND"
      when: pg_files.files | length == 0

- name: Собрать информацию о службах
  when: collect_services
  block:
    - name: Получить включенные службы
      shell: "{{ commands.services }}"
      register: services_result
      changed_when: false
      
    - name: Парсить службы
      set_fact:
        enabled_services: "{{ services_result.stdout_lines | map('regex_replace', '\\s+', ';') | list }}"

- name: Собрать системную информацию
  block:
    - name: Получить информацию об ОС
      setup:
        filter: ansible_distribution*
      register: os_info
      
    - name: Получить информацию о памяти
      setup:
        filter: ansible_mem*
      register: mem_info
        
    - name: Получить информацию о CPU
      setup:
        filter: ansible_processor*
      register: cpu_info

- name: Сохранить все данные в структурированный формат
  set_fact:
    host_collected_data:
      hostname: "{{ ansible_hostname }}"
      fqdn: "{{ ansible_fqdn }}"
      ip_address: "{{ ansible_default_ipv4.address }}"
      os: "{{ os_info.ansible_facts.ansible_distribution }} {{ os_info.ansible_facts.ansible_distribution_version }}"
      kernel: "{{ ansible_kernel }}"
      timestamp: "{{ ansible_date_time.iso8601 }}"
      open_ports: "{{ parsed_ports | default([]) }}"
      firewall_rules: "{{ firewall_config | default('') | csv_escape }}"
      users_count: "{{ user_stats.total | default(0) }}"
      system_users: "{{ user_stats.system | default(0) }}"
      sudo_config: "{{ sudo_config | default('') | csv_escape }}"
      sssd_configured: "{{ 'YES' if sssd_config != 'NOT_INSTALLED' and sssd_config != 'NOT_CONFIGURED' else 'NO' }}"
      nsswitch_config: "{{ nsswitch_config | default('') | csv_escape }}"
      postgresql_found: "{{ 'YES' if postgresql_config != 'POSTGRESQL_NOT_FOUND' else 'NO' }}"
      enabled_services_count: "{{ enabled_services | default([]) | length }}"
      enabled_services_list: "{{ enabled_services | default([]) | join('|') }}"
      memory_mb: "{{ mem_info.ansible_facts.ansible_memtotal_mb | default(0) }}"
      cpu_count: "{{ cpu_info.ansible_facts.ansible_processor_count | default(0) }}"
      cpu_cores: "{{ cpu_info.ansible_facts.ansible_processor_cores | default(0) }}"

- name: Сохранить данные в файл на управляющем хосте
  delegate_to: localhost
  run_once: false
  copy:
    content: "{{ host_collected_data | to_nice_json }}"
    dest: "{{ collect_info_dir }}/raw/{{ inventory_hostname }}/data.json"
```

5. tasks/process_data.yml

```yaml
---
- name: Ожидание завершения сбора данных со всех хостов
  delegate_to: localhost
  run_once: yes
  wait_for:
    timeout: "{{ timeout_seconds }}"

- name: Собрать все JSON файлы в один каталог
  delegate_to: localhost
  run_once: yes
  find:
    paths: "{{ collect_info_dir }}/raw"
    patterns: "*.json"
    recurse: yes
  register: collected_files

- name: Загрузить данные из всех файлов
  delegate_to: localhost
  run_once: yes
  set_fact:
    all_hosts_data: |
      {% set data = [] %}
      {% for file in collected_files.files %}
      {% set hostname = file.path.split('/')[-2] %}
      {% set content = lookup('file', file.path) | from_json %}
      {% set _ = data.append(content) %}
      {% endfor %}
      {{ data }}
```

6. tasks/generate_report.yml

```yaml
---
- name: Создать CSV отчет
  delegate_to: localhost
  run_once: yes
  template:
    src: report.csv.j2
    dest: "{{ report_dir }}/{{ report_filename }}"

- name: Создать сводный отчет в Markdown
  delegate_to: localhost
  run_once: yes
  template:
    src: summary.md.j2
    dest: "{{ report_dir }}/summary_{{ ansible_date_time.date }}.md"

- name: Создать архив с отчетами
  delegate_to: localhost
  run_once: yes
  archive:
    path: "{{ report_dir }}"
    dest: "{{ report_dir }}/server_reports_{{ ansible_date_time.date }}.tar.gz"
    format: gz

- name: Вывести статистику сбора
  delegate_to: localhost
  run_once: yes
  debug:
    msg: |
      Сбор информации завершен!
      ==========================
      Обработано хостов: {{ all_hosts_data | length }}
      CSV отчет: {{ report_dir }}/{{ report_filename }}
      Сводный отчет: {{ report_dir }}/summary_{{ ansible_date_time.date }}.md
      Архив: {{ report_dir }}/server_reports_{{ ansible_date_time.date }}.tar.gz
```

7. templates/report.csv.j2

```jinja2
hostname;fqdn;ip_address;os;kernel;timestamp;open_ports_count;open_ports_list;firewall_rules;users_count;system_users;sudo_config;sssd_configured;nsswitch_config;postgresql_found;enabled_services_count;enabled_services_list;memory_mb;cpu_count;cpu_cores
{% for host in all_hosts_data %}
{{ host.hostname }};{{ host.fqdn }};{{ host.ip_address }};{{ host.os }};{{ host.kernel }};{{ host.timestamp }};{{ host.open_ports | length }};"{{ host.open_ports | map(attribute='port') | join(',') }}";"{{ host.firewall_rules }}";{{ host.users_count }};{{ host.system_users }};"{{ host.sudo_config }}";{{ host.sssd_configured }};"{{ host.nsswitch_config }}";{{ host.postgresql_found }};{{ host.enabled_services_count }};"{{ host.enabled_services_list }}";{{ host.memory_mb }};{{ host.cpu_count }};{{ host.cpu_cores }}
{% endfor %}
```

8. filter_plugins/csv_filters.py

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

class FilterModule(object):
    def filters(self):
        return {
            'csv_escape': self.csv_escape,
            'parse_ss_output': self.parse_ss_output,
            'analyze_users': self.analyze_users,
        }
    
    def csv_escape(self, text):
        """Экранирование текста для CSV"""
        if not text:
            return ''
        # Заменяем кавычки на двойные кавычки
        text = str(text).replace('"', '""')
        # Удаляем переносы строк
        text = text.replace('\n', ' ').replace('\r', ' ')
        # Удаляем лишние пробелы
        text = ' '.join(text.split())
        return f'"{text}"'
    
    def parse_ss_output(self, ss_output):
        """Парсинг вывода ss/netstat"""
        import re
        
        if not ss_output or 'ERROR' in ss_output:
            return []
        
        ports = []
        lines = ss_output.split('\n')
        
        for line in lines[1:]:  # Пропускаем заголовок
            if not line.strip():
                continue
            
            # Регулярное выражение для парсинга ss/netstat
            match = re.search(r'(\S+)\s+(\S+)\s+(\S+)\s+(\S+)\s+(\S+)\s+(.+)', line)
            if match:
                protocol, state, recv_q, send_q, local_addr, peer_addr = match.groups()
                
                # Извлекаем порт из local address
                port_match = re.search(r':(\d+)$', local_addr.split()[0])
                if port_match:
                    port = port_match.group(1)
                    ports.append({
                        'protocol': protocol,
                        'port': port,
                        'state': state,
                        'local_address': local_addr,
                        'process': peer_addr if 'users' in peer_addr else 'N/A'
                    })
        
        return ports
    
    def analyze_users(self, users_list):
        """Анализ списка пользователей"""
        system_users = 0
        interactive_users = 0
        
        for user_line in users_list:
            if not user_line:
                continue
            
            parts = user_line.split(':')
            if len(parts) >= 7:
                uid = int(parts[2]) if parts[2].isdigit() else 0
                shell = parts[6]
                
                if uid < 1000 or uid == 65534:  # system users
                    system_users += 1
                elif shell not in ['/bin/false', '/usr/sbin/nologin', '/sbin/nologin']:
                    interactive_users += 1
        
        return {
            'system_users': system_users,
            'interactive_users': interactive_users,
            'total': len(users_list)
        }
```

9. Плейбук для использования

```yaml
---
- name: Сбор информации с серверов
  hosts: all
  
  # Для большого количества серверов используем стратегию free
  strategy: free
  serial: "{{ max_workers | default(50) }}"
  
  # Отключаем факты по умолчанию, собираем только нужные
  gather_facts: partial
  
  vars:
    # Переопределяем настройки при необходимости
    max_workers: 100
    timeout_seconds: 600
    
  pre_tasks:
    - name: Проверить доступность хостов
      wait_for_connection:
        timeout: 30
      become: no
      
    - name: Собрать базовые факты
      setup:
        gather_subset:
          - '!all'
          - '!min'
          - network
          - hardware
          - virtual
          
  roles:
    - role: collect_server_info
      
  post_tasks:
    - name: Очистить временные файлы
      file:
        path: "{{ collect_info_dir }}"
        state: absent
      delegate_to: localhost
      run_once: yes
      when: cleanup_temp | default(true)
```

10. Расширенный плейбук с мониторингом

```yaml
---
- name: Мониторинг сбора информации
  hosts: localhost
  gather_facts: no
  vars:
    progress_file: "/tmp/ansible_progress.log"
    
  tasks:
    - name: Запуск сбора информации
      include_role:
        name: collect_server_info
      vars:
        collect_info_dir: "/tmp/ansible_collect_{{ ansible_date_time.epoch }}"
      run_once: yes
    
    - name: Отслеживание прогресса
      shell: |
        tail -f {{ collect_info_dir }}/progress.log 2>/dev/null || echo "Progress file not found"
      async: 3600
      poll: 0
      register: progress_monitor
      
    - name: Ожидание завершения
      async_status:
        jid: "{{ progress_monitor.ansible_job_id }}"
      register: job_result
      until: job_result.finished
      retries: 3600  # 1 час
      delay: 1
```

Основные улучшения:

1. Асинхронное выполнение - команды выполняются асинхронно для ускорения
2. Стратегия free - позволяет обрабатывать хосты независимо
3. Сериализация - контроль количества одновременных подключений
4. Фильтры Jinja2 - кастомная обработка данных
5. Структурированный JSON - данные сохраняются в формате JSON
6. Обработка ошибок - блоки failed_when: false и проверки
7. Парсинг вывода - автоматический парсинг ss/netstat
8. Анализ данных - статистика по пользователям, портам и т.д.
9. Многопоточность - настраиваемое количество воркеров
10. Архивация - автоматическое создание архивов с отчетами

Запуск:

```bash
# Базовый запуск
ansible-playbook -i inventory.ini collect_info.yml

# С увеличенным количеством потоков
ansible-playbook -i inventory.ini collect_info.yml -e "max_workers=200"

# Только определенные проверки
ansible-playbook -i inventory.ini collect_info.yml \
  -e "collect_postgresql=false collect_sssd=false"

# С сохранением временных файлов для отладки
ansible-playbook -i inventory.ini collect_info.yml \
  -e "cleanup_temp=false"
```

Эта роль оптимизирована для работы с тысячами серверов и включает все необходимые механизмы для надежного сбора информации.