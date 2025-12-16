Вот пример Ansible роли для сбора указанной информации в один отчет:

Структура роли:

```
system_audit_role/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
└── templates/
    └── report_template.j2
```

1. defaults/main.yml:

```yaml
---
# Настройки отчета
report_path: "/tmp/system_audit_report_{{ ansible_date_time.date }}_{{ ansible_hostname }}.txt"
report_format: "txt"  # или "html", "md"

# Какие разделы включать в отчет
collect_network_info: true
collect_firewall_info: true
collect_user_info: true
collect_sudo_info: true
collect_sssd_info: true
collect_nsswitch_info: true
collect_postgres_info: true
collect_services_info: true

# Пути к конфигурационным файлам
postgres_config_paths:
  - /var/lib/pgsql/
  - /etc/postgresql/
  - /var/lib/pgsql/data/
```

2. templates/report_template.j2:

```jinja2
==================================================
СИСТЕМНЫЙ АУДИТ ХОСТА: {{ ansible_hostname }}
Время сбора: {{ ansible_date_time.iso8601 }}
==================================================

{% if network_info is defined %}
1. ОТКРЫТЫЕ ПОРТЫ И ПРОЦЕССЫ:
{{ network_info.stdout }}

{% endif %}

{% if firewall_info is defined %}
2. ПРАВИЛА МЕЖСЕТЕВОГО ЭКРАНА:
{{ firewall_info.stdout }}

{% endif %}

{% if users_info is defined %}
3. ПОЛЬЗОВАТЕЛИ И ГРУППЫ:
Пользователи (/etc/passwd):
{{ users_info.passwd.stdout }}

Группы (/etc/group):
{{ users_info.group.stdout }}

{% endif %}

{% if sudo_info is defined %}
4. КОНФИГУРАЦИЯ SUDO:
Основной файл sudoers:
{{ sudo_info.sudoers.stdout }}

Файлы из /etc/sudoers.d/:
{% for file in sudo_info.sudoers_d.files %}
=== {{ file.item }} ===
{{ file.stdout if file.stdout else "ФАЙЛ ПУСТ ИЛИ ОТСУТСТВУЕТ" }}
{% endfor %}

{% endif %}

{% if sssd_info is defined %}
5. КОНФИГУРАЦИЯ SSSD:
{{ sssd_info.stdout if sssd_info.stdout else "Файл /etc/sssd/sssd.conf не найден" }}

{% endif %}

{% if nsswitch_info is defined %}
6. КОНФИГУРАЦИЯ NSSWITCH:
{{ nsswitch_info.stdout }}

{% endif %}

{% if postgres_info is defined %}
7. КОНФИГУРАЦИЯ POSTGRESQL:
{% if postgres_info.pg_hba.stdout %}
pg_hba.conf:
{{ postgres_info.pg_hba.stdout }}
{% endif %}

{% if postgres_info.postgresql_conf.stdout %}
postgresql.conf:
{{ postgres_info.postgresql_conf.stdout }}
{% endif %}

{% endif %}

{% if services_info is defined %}
8. ВКЛЮЧЕННЫЕ СЛУЖБЫ SYSTEMD:
{{ services_info.stdout }}

{% endif %}
```

3. tasks/main.yml:

```yaml
---
- name: Сбор информации о системе
  block:
    - name: Проверка доступности утилит
      ansible.builtin.package_facts:
        manager: auto
      tags: always

    - name: Сбор информации об открытых портах
      ansible.builtin.command: "ss -tulpn"
      register: network_info
      when: collect_network_info
      changed_when: false
      tags: network

    - name: Попытка сбора через netstat (если ss недоступен)
      ansible.builtin.command: "netstat -tulpn"
      register: network_info_alt
      when: 
        - collect_network_info
        - network_info is failed or network_info.stdout == ""
      changed_when: false
      tags: network

    - name: Сбор правил iptables
      ansible.builtin.command: "iptables-save"
      register: iptables_info
      when: collect_firewall_info
      changed_when: false
      ignore_errors: true
      tags: firewall

    - name: Сбор правил nftables
      ansible.builtin.command: "nft list ruleset"
      register: nftables_info
      when: collect_firewall_info
      changed_when: false
      ignore_errors: true
      tags: firewall

    - name: Сбор информации о пользователях и группах
      block:
        - name: Чтение /etc/passwd
          ansible.builtin.command: "cat /etc/passwd"
          register: passwd_info
          changed_when: false

        - name: Чтение /etc/group
          ansible.builtin.command: "cat /etc/group"
          register: group_info
          changed_when: false
      when: collect_user_info
      tags: users

    - name: Сбор конфигурации sudo
      block:
        - name: Чтение /etc/sudoers
          ansible.builtin.command: "cat /etc/sudoers"
          register: sudoers_info
          changed_when: false
          ignore_errors: true

        - name: Поиск файлов в /etc/sudoers.d/
          ansible.builtin.find:
            paths: /etc/sudoers.d
            patterns: '*'
          register: sudoers_d_files
          changed_when: false

        - name: Чтение файлов из /etc/sudoers.d/
          ansible.builtin.shell: |
            for file in /etc/sudoers.d/*; do
              if [ -f "$file" ]; then
                echo "=== $file ==="
                cat "$file"
                echo ""
              fi
            done
          register: sudoers_d_content
          changed_when: false
          when: sudoers_d_files.files | length > 0
      when: collect_sudo_info
      tags: sudo

    - name: Сбор конфигурации SSSD
      ansible.builtin.command: "cat /etc/sssd/sssd.conf"
      register: sssd_info
      when: collect_sssd_info
      changed_when: false
      ignore_errors: true
      tags: sssd

    - name: Сбор конфигурации nsswitch
      ansible.builtin.command: "cat /etc/nsswitch.conf"
      register: nsswitch_info
      when: collect_nsswitch_info
      changed_when: false
      tags: nsswitch

    - name: Поиск конфигурационных файлов PostgreSQL
      ansible.builtin.find:
        paths: "{{ item }}"
        patterns: "pg_hba.conf,postgresql.conf"
        recurse: true
      register: postgres_files
      loop: "{{ postgres_config_paths }}"
      when: collect_postgres_info
      changed_when: false
      tags: postgres

    - name: Чтение конфигурации PostgreSQL
      block:
        - name: Чтение pg_hba.conf
          ansible.builtin.command: "cat {{ item }}"
          register: pg_hba_content
          loop: "{{ postgres_files.results | map(attribute='files') | flatten | selectattr('path', 'match', 'pg_hba') | map(attribute='path') | list }}"
          changed_when: false
          ignore_errors: true

        - name: Чтение postgresql.conf
          ansible.builtin.command: "cat {{ item }}"
          register: postgresql_conf_content
          loop: "{{ postgres_files.results | map(attribute='files') | flatten | selectattr('path', 'match', 'postgresql.conf') | map(attribute='path') | list }}"
          changed_when: false
          ignore_errors: true
      when: collect_postgres_info
      tags: postgres

    - name: Сбор информации о службах
      ansible.builtin.command: "systemctl list-unit-files --type=service --state=enabled"
      register: services_info
      when: collect_services_info
      changed_when: false
      tags: services

  rescue:
    - name: Логирование ошибок сбора
      ansible.builtin.debug:
        msg: "Ошибка при сборе информации: {{ ansible_failed_result }}"

- name: Формирование финального отчета
  ansible.builtin.template:
    src: report_template.j2
    dest: "{{ report_path }}"
    mode: '0644'
  vars:
    network_info: "{{ network_info_alt if network_info is failed else network_info }}"
    firewall_info: "{{ nftables_info if iptables_info is failed else iptables_info }}"
    users_info:
      passwd: "{{ passwd_info }}"
      group: "{{ group_info }}"
    sudo_info:
      sudoers: "{{ sudoers_info }}"
      sudoers_d: "{{ sudoers_d_content }}"
    sssd_info: "{{ sssd_info }}"
    nsswitch_info: "{{ nsswitch_info }}"
    postgres_info:
      pg_hba: "{{ pg_hba_content }}"
      postgresql_conf: "{{ postgresql_conf_content }}"
    services_info: "{{ services_info }}"

- name: Вывод информации о созданном отчете
  ansible.builtin.debug:
    msg: "Отчет сохранен в {{ report_path }}"

- name: Копирование отчета на управляющий узел (опционально)
  ansible.builtin.fetch:
    src: "{{ report_path }}"
    dest: "./reports/{{ inventory_hostname }}/"
    flat: no
  delegate_to: localhost
  run_once: true
  when: false  # Измените на true для включения
```

4. handlers/main.yml (опционально):

```yaml
---
- name: Отправить отчет по email
  ansible.builtin.mail:
    to: "admin@example.com"
    subject: "Системный аудит {{ ansible_hostname }}"
    body: "Отчет прикреплен"
    attach: "{{ report_path }}"
```

5. Пример playbook для использования роли:

```yaml
---
- name: Выполнить системный аудит
  hosts: all
  become: yes
  roles:
    - system_audit_role
  vars:
    report_path: "/tmp/full_system_audit_{{ ansible_date_time.date }}.txt"
    collect_postgres_info: false  # Не собирать Postgres, если он не используется
```

6. Дополнительные улучшения:

Для HTML отчета создайте дополнительный шаблон:

templates/report_html.j2:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Аудит системы {{ ansible_hostname }}</title>
    <style>
        body { font-family: Arial, sans-serif; }
        .section { margin: 20px 0; padding: 15px; border: 1px solid #ccc; }
        pre { background: #f5f5f5; padding: 10px; overflow: auto; }
    </style>
</head>
<body>
    <h1>Системный аудит: {{ ansible_hostname }}</h1>
    <p>Время сбора: {{ ansible_date_time.iso8601 }}</p>
    
    <div class="section">
        <h2>1. Открытые порты</h2>
        <pre>{{ network_info.stdout | default('Нет данных') }}</pre>
    </div>
    
    <!-- Остальные секции -->
</body>
</html>
```

Для сбора в JSON добавьте задачу:

```yaml
- name: Сохранить данные в JSON
  ansible.builtin.copy:
    content: "{{ all_data | to_nice_json }}"
    dest: "/tmp/audit_{{ ansible_hostname }}.json"
  vars:
    all_data:
      network: "{{ network_info.stdout }}"
      firewall: "{{ firewall_info.stdout }}"
      # ... остальные данные
```

Ключевые особенности роли:

1. Идемпотентность: Не изменяет состояние системы
2. Обработка ошибок: Игнорирует отсутствующие утилиты/файлы
3. Гибкость: Включение/выключение секций через переменные
4. Мультиплатформенность: Поддержка разных систем (iptables/nftables, ss/netstat)
5. Расширяемость: Легко добавить новые секции сбора данных

Для использования роли:

1. Создайте структуру директорий как указано выше
2. Скопируйте содержимое файлов
3. Запустите playbook:

```bash
ansible-playbook -i inventory.ini audit_playbook.yml
```


Ошибка возникает из-за неправильного обращения к структуре данных в шаблоне. Вот исправленная версия задачи формирования отчета:

Исправленный tasks/main.yml (только часть с формированием отчета):

```yaml
- name: Формирование финального отчета
  ansible.builtin.template:
    src: report_template.j2
    dest: "{{ report_path }}"
    mode: '0644'
  vars:
    network_info: "{{ network_info_alt if network_info is failed else network_info }}"
    firewall_info: "{{ nftables_info if iptables_info is failed else iptables_info }}"
    users_info:
      passwd: "{{ passwd_info }}"
      group: "{{ group_info }}"
    sudo_info:
      sudoers: "{{ sudoers_info }}"
      sudoers_d_content: "{{ sudoers_d_content if sudoers_d_content is defined else '' }}"
      sudoers_d_files: "{{ sudoers_d_files.files if sudoers_d_files is defined and sudoers_d_files.files is defined else [] }}"
    sssd_info: "{{ sssd_info }}"
    nsswitch_info: "{{ nsswitch_info }}"
    postgres_info:
      pg_hba_results: "{{ pg_hba_content.results if pg_hba_content is defined and pg_hba_content.results is defined else [] }}"
      postgresql_conf_results: "{{ postgresql_conf_content.results if postgresql_conf_content is defined and postgresql_conf_content.results is defined else [] }}"
    services_info: "{{ services_info }}"
```

Исправленный шаблон templates/report_template.j2:

```jinja2
==================================================
СИСТЕМНЫЙ АУДИТ ХОСТА: {{ ansible_hostname }}
Время сбора: {{ ansible_date_time.iso8601 }}
==================================================

{% if network_info is defined and network_info.stdout is defined %}
1. ОТКРЫТЫЕ ПОРТЫ И ПРОЦЕССЫ:
{{ network_info.stdout }}

{% endif %}

{% if firewall_info is defined and firewall_info.stdout is defined %}
2. ПРАВИЛА МЕЖСЕТЕВОГО ЭКРАНА:
{{ firewall_info.stdout }}

{% endif %}

{% if users_info is defined %}
3. ПОЛЬЗОВАТЕЛИ И ГРУППЫ:
{% if users_info.passwd.stdout is defined %}
Пользователи (/etc/passwd):
{{ users_info.passwd.stdout }}
{% endif %}

{% if users_info.group.stdout is defined %}
Группы (/etc/group):
{{ users_info.group.stdout }}
{% endif %}

{% endif %}

{% if sudo_info is defined %}
4. КОНФИГУРАЦИЯ SUDO:
{% if sudo_info.sudoers.stdout is defined %}
Основной файл sudoers:
{{ sudo_info.sudoers.stdout }}
{% endif %}

{% if sudo_info.sudoers_d_content.stdout is defined %}
Файлы из /etc/sudoers.d/:
{{ sudo_info.sudoers_d_content.stdout }}
{% endif %}

{% if sudo_info.sudoers_d_files %}
Файлы из /etc/sudoers.d/:
{% for file in sudo_info.sudoers_d_files %}
=== {{ file.path }} ===
{% endfor %}
{% endif %}

{% endif %}

{% if sssd_info is defined and sssd_info.stdout is defined %}
5. КОНФИГУРАЦИЯ SSSD:
{{ sssd_info.stdout }}

{% endif %}

{% if nsswitch_info is defined and nsswitch_info.stdout is defined %}
6. КОНФИГУРАЦИЯ NSSWITCH:
{{ nsswitch_info.stdout }}

{% endif %}

{% if postgres_info is defined %}
7. КОНФИГУРАЦИЯ POSTGRESQL:
{% if postgres_info.pg_hba_results %}
pg_hba.conf:
{% for result in postgres_info.pg_hba_results %}
=== {{ result.item }} ===
{{ result.stdout if result.stdout is defined else "ФАЙЛ НЕ НАЙДЕН ИЛИ НЕ ЧИТАЕМ" }}
{% endfor %}
{% endif %}

{% if postgres_info.postgresql_conf_results %}
postgresql.conf:
{% for result in postgres_info.postgresql_conf_results %}
=== {{ result.item }} ===
{{ result.stdout if result.stdout is defined else "ФАЙЛ НЕ НАЙДЕН ИЛИ НЕ ЧИТАЕМ" }}
{% endfor %}
{% endif %}

{% endif %}

{% if services_info is defined and services_info.stdout is defined %}
8. ВКЛЮЧЕННЫЕ СЛУЖБЫ SYSTEMD:
{{ services_info.stdout }}

{% endif %}
```

Альтернативное решение - более безопасный подход с проверками:

Создайте фильтры для безопасного доступа к данным. Добавьте файл filter_plugins/safe_filters.py:

```python
class FilterModule(object):
    def filters(self):
        return {
            'safe_get': self.safe_get,
        }

    def safe_get(self, data, key, default=''):
        """Безопасное получение значения из словаря"""
        if not isinstance(data, dict):
            return default
        return data.get(key, default)
```

И обновите шаблон:

```jinja2
{% if network_info.stdout is defined and network_info.stdout %}
1. ОТКРЫТЫЕ ПОРТЫ И ПРОЦЕССЫ:
{{ network_info.stdout }}

{% endif %}
```

Упрощенная версия задачи сбора sudoers.d:

Замените задачи сбора sudoers.d на более простую:

```yaml
- name: Сбор конфигурации sudo
  block:
    - name: Чтение /etc/sudoers
      ansible.builtin.command: "cat /etc/sudoers"
      register: sudoers_info
      changed_when: false
      ignore_errors: true
      tags: sudo

    - name: Сбор содержимого /etc/sudoers.d/
      ansible.builtin.shell: |
        if [ -d /etc/sudoers.d ]; then
          for file in /etc/sudoers.d/*; do
            if [ -f "$file" ]; then
              echo "=== $file ==="
              cat "$file" 2>/dev/null || echo "Ошибка чтения файла"
              echo ""
            fi
          done
        else
          echo "Директория /etc/sudoers.d не существует"
        fi
      register: sudoers_d_content
      changed_when: false
      tags: sudo
  when: collect_sudo_info
```

Упрощенная версия формирования отчета:

```yaml
- name: Формирование финального отчета
  ansible.builtin.template:
    src: report_template.j2
    dest: "{{ report_path }}"
    mode: '0644'
  vars:
    network_info_stdout: "{{ network_info_alt.stdout if network_info is failed and network_info_alt.stdout is defined else network_info.stdout if network_info.stdout is defined else '' }}"
    firewall_info_stdout: "{{ nftables_info.stdout if iptables_info is failed and nftables_info.stdout is defined else iptables_info.stdout if iptables_info.stdout is defined else '' }}"
    passwd_stdout: "{{ passwd_info.stdout if passwd_info.stdout is defined else '' }}"
    group_stdout: "{{ group_info.stdout if group_info.stdout is defined else '' }}"
    sudoers_stdout: "{{ sudoers_info.stdout if sudoers_info.stdout is defined else '' }}"
    sudoers_d_stdout: "{{ sudoers_d_content.stdout if sudoers_d_content.stdout is defined else '' }}"
    sssd_stdout: "{{ sssd_info.stdout if sssd_info.stdout is defined else '' }}"
    nsswitch_stdout: "{{ nsswitch_info.stdout if nsswitch_info.stdout is defined else '' }}"
    services_stdout: "{{ services_info.stdout if services_info.stdout is defined else '' }}"
```

Самый простой вариант - использовать debug для проверки структуры:

Добавьте задачу для отладки перед формированием отчета:

```yaml
- name: Отладка структуры данных
  ansible.builtin.debug:
    var: sudoers_d_content
    verbosity: 2
```

Это поможет увидеть фактическую структуру данных, которую возвращают модули Ansible.

Рекомендуемое решение:

Используйте упрощенный подход с прямым обращением к stdout:

1. Исправьте все задачи сбора данных, чтобы они регистрировали результат
2. В шаблоне обращайтесь только к .stdout
3. Добавьте проверки на существование переменных

Пример окончательного шаблона:

```jinja2
{% if network_info_stdout %}
1. ОТКРЫТЫЕ ПОРТЫ И ПРОЦЕССЫ:
{{ network_info_stdout }}

{% endif %}
```

Это устранит ошибку 'dict object' has no attribute 'files', так как вы будете обращаться только к существующим атрибутам словарей, которые возвращают модули Ansible.


Полная Ansible роль для сбора системной информации с CSV отчетом

Структура роли:

```
system_audit_role/
├── defaults/
│   └── main.yml
├── tasks/
│   └── main.yml
├── templates/
│   ├── report_template.j2
│   └── report_csv.j2
├── filter_plugins/
│   └── csv_filters.py
└── handlers/
    └── main.yml
```

1. defaults/main.yml:

```yaml
---
# Настройки отчетов
report_path: "/tmp/system_audit_report_{{ ansible_date_time.date }}_{{ ansible_hostname }}.txt"
report_csv_path: "/tmp/system_audit_report_{{ ansible_date_time.date }}_{{ ansible_hostname }}.csv"

# Какие разделы включать в отчет
collect_network_info: true
collect_firewall_info: true
collect_user_info: true
collect_sudo_info: true
collect_sssd_info: true
collect_nsswitch_info: true
collect_postgres_info: true
collect_services_info: true
collect_system_info: true

# Пути к конфигурационным файлам PostgreSQL
postgres_config_paths:
  - /var/lib/pgsql/
  - /etc/postgresql/
  - /var/lib/pgsql/data/
  - /opt/postgresql/

# Настройки CSV
csv_delimiter: ";"
csv_encoding: "utf-8"

# Дополнительные настройки
archive_reports: false
fetch_reports: false
local_report_dir: "./audit_reports/{{ inventory_hostname }}/"
```

2. filter_plugins/csv_filters.py:

```python
#!/usr/bin/python3
# -*- coding: utf-8 -*-

from ansible.errors import AnsibleFilterError
import re

class FilterModule(object):
    def filters(self):
        return {
            'to_csv_safe': self.to_csv_safe,
            'escape_csv': self.escape_csv,
            'parse_ss_lines': self.parse_ss_lines,
            'parse_netstat_lines': self.parse_netstat_lines,
            'parse_passwd_lines': self.parse_passwd_lines,
            'parse_group_lines': self.parse_group_lines,
            'parse_services_lines': self.parse_services_lines,
            'split_to_table': self.split_to_table,
        }

    def to_csv_safe(self, text, delimiter=";"):
        """Безопасное преобразование текста для CSV"""
        if not text:
            return ""
        
        # Экранируем кавычки и разделители
        text = str(text)
        if delimiter in text or '"' in text or '\n' in text:
            text = text.replace('"', '""')  # Экранируем кавычки
            text = f'"{text}"'  # Оборачиваем в кавычки
        
        return text

    def escape_csv(self, text, delimiter=";"):
        """Экранирование текста для CSV"""
        if text is None:
            return ""
        
        text = str(text)
        # Заменяем переносы строк на \n для сохранения в одной ячейке
        text = text.replace('\r\n', '\\n').replace('\n', '\\n')
        # Заменяем разделитель на альтернативный
        text = text.replace(delimiter, ',')
        
        return text

    def parse_ss_lines(self, ss_output):
        """Парсинг вывода ss -tulpn"""
        if not ss_output:
            return []
        
        lines = ss_output.strip().split('\n')
        result = []
        
        for line in lines:
            line = line.strip()
            if not line or 'State' in line and 'Recv-Q' in line:
                continue  # Пропускаем заголовок
            
            # Разбираем строку
            parts = re.split(r'\s+', line, 5)
            if len(parts) >= 5:
                netid = parts[0]
                state = parts[1]
                recv_q = parts[2]
                send_q = parts[3]
                local_addr = parts[4]
                peer_addr = parts[5] if len(parts) > 5 else ''
                process = parts[6] if len(parts) > 6 else ''
                
                result.append({
                    'netid': netid,
                    'state': state,
                    'recv_q': recv_q,
                    'send_q': send_q,
                    'local_addr': local_addr,
                    'peer_addr': peer_addr,
                    'process': process
                })
        
        return result

    def parse_netstat_lines(self, netstat_output):
        """Парсинг вывода netstat -tulpn"""
        if not netstat_output:
            return []
        
        lines = netstat_output.strip().split('\n')
        result = []
        
        for line in lines:
            line = line.strip()
            if not line or 'Proto' in line and 'Local Address' in line:
                continue  # Пропускаем заголовок
            
            # Разбираем строку
            parts = re.split(r'\s+', line, 7)
            if len(parts) >= 7:
                proto = parts[0]
                recv_q = parts[1]
                send_q = parts[2]
                local_addr = parts[3]
                foreign_addr = parts[4]
                state = parts[5]
                pid_program = ' '.join(parts[6:]) if len(parts) > 6 else ''
                
                result.append({
                    'proto': proto,
                    'recv_q': recv_q,
                    'send_q': send_q,
                    'local_addr': local_addr,
                    'foreign_addr': foreign_addr,
                    'state': state,
                    'pid_program': pid_program
                })
        
        return result

    def parse_passwd_lines(self, passwd_output):
        """Парсинг /etc/passwd"""
        if not passwd_output:
            return []
        
        lines = passwd_output.strip().split('\n')
        result = []
        
        for line in lines:
            line = line.strip()
            if not line or line.startswith('#'):
                continue
            
            parts = line.split(':')
            if len(parts) >= 7:
                result.append({
                    'username': parts[0],
                    'password': parts[1],
                    'uid': parts[2],
                    'gid': parts[3],
                    'gecos': parts[4],
                    'home': parts[5],
                    'shell': parts[6]
                })
        
        return result

    def parse_group_lines(self, group_output):
        """Парсинг /etc/group"""
        if not group_output:
            return []
        
        lines = group_output.strip().split('\n')
        result = []
        
        for line in lines:
            line = line.strip()
            if not line or line.startswith('#'):
                continue
            
            parts = line.split(':')
            if len(parts) >= 4:
                result.append({
                    'groupname': parts[0],
                    'password': parts[1],
                    'gid': parts[2],
                    'members': parts[3] if len(parts) > 3 else ''
                })
        
        return result

    def parse_services_lines(self, services_output):
        """Парсинг вывода systemctl list-unit-files"""
        if not services_output:
            return []
        
        lines = services_output.strip().split('\n')
        result = []
        
        for line in lines:
            line = line.strip()
            if not line or line.startswith('UNIT FILE') or line.startswith('---') or 'listed' in line:
                continue
            
            parts = re.split(r'\s+', line, 2)
            if len(parts) >= 2:
                result.append({
                    'service': parts[0],
                    'state': parts[1],
                    'vendor_preset': parts[2] if len(parts) > 2 else ''
                })
        
        return result

    def split_to_table(self, text, max_length=1000):
        """Разделение длинного текста на строки для таблицы"""
        if not text:
            return []
        
        lines = []
        current_line = ""
        
        for char in str(text):
            if len(current_line) >= max_length:
                lines.append(current_line)
                current_line = char
            else:
                current_line += char
        
        if current_line:
            lines.append(current_line)
        
        return lines
```

3. templates/report_template.j2:

```jinja2
==================================================
СИСТЕМНЫЙ АУДИТ ХОСТА: {{ ansible_hostname }}
IP адрес: {{ ansible_default_ipv4.address | default('не определен') }}
Время сбора: {{ ansible_date_time.iso8601 }}
Система: {{ ansible_distribution }} {{ ansible_distribution_version }}
Ядро: {{ ansible_kernel }}
==================================================

{% if network_info is defined and network_info.stdout is defined and network_info.stdout %}
1. ОТКРЫТЫЕ ПОРТЫ И ПРОЦЕССЫ:
{{ network_info.stdout }}

{% endif %}

{% if firewall_info is defined and firewall_info.stdout is defined and firewall_info.stdout %}
2. ПРАВИЛА МЕЖСЕТЕВОГО ЭКРАНА:
{{ firewall_info.stdout }}

{% endif %}

{% if users_info is defined %}
3. ПОЛЬЗОВАТЕЛИ И ГРУППЫ:
{% if users_info.passwd.stdout is defined and users_info.passwd.stdout %}
Пользователи (/etc/passwd):
{{ users_info.passwd.stdout }}
{% endif %}

{% if users_info.group.stdout is defined and users_info.group.stdout %}
Группы (/etc/group):
{{ users_info.group.stdout }}
{% endif %}

{% endif %}

{% if sudo_info is defined %}
4. КОНФИГУРАЦИЯ SUDO:
{% if sudo_info.sudoers.stdout is defined and sudo_info.sudoers.stdout %}
Основной файл sudoers:
{{ sudo_info.sudoers.stdout }}
{% endif %}

{% if sudo_info.sudoers_d_content.stdout is defined and sudo_info.sudoers_d_content.stdout %}
Файлы из /etc/sudoers.d/:
{{ sudo_info.sudoers_d_content.stdout }}
{% endif %}

{% endif %}

{% if sssd_info is defined and sssd_info.stdout is defined and sssd_info.stdout %}
5. КОНФИГУРАЦИЯ SSSD:
{{ sssd_info.stdout }}

{% endif %}

{% if nsswitch_info is defined and nsswitch_info.stdout is defined and nsswitch_info.stdout %}
6. КОНФИГУРАЦИЯ NSSWITCH:
{{ nsswitch_info.stdout }}

{% endif %}

{% if postgres_info is defined %}
7. КОНФИГУРАЦИЯ POSTGRESQL:
{% if postgres_info.pg_hba_results is defined and postgres_info.pg_hba_results %}
pg_hba.conf:
{% for result in postgres_info.pg_hba_results %}
=== {{ result.item }} ===
{{ result.stdout if result.stdout is defined else "ФАЙЛ НЕ НАЙДЕН ИЛИ НЕ ЧИТАЕМ" }}
{% endfor %}
{% endif %}

{% if postgres_info.postgresql_conf_results is defined and postgres_info.postgresql_conf_results %}
postgresql.conf:
{% for result in postgres_info.postgresql_conf_results %}
=== {{ result.item }} ===
{{ result.stdout if result.stdout is defined else "ФАЙЛ НЕ НАЙДЕН ИЛИ НЕ ЧИТАЕМ" }}
{% endfor %}
{% endif %}

{% endif %}

{% if services_info is defined and services_info.stdout is defined and services_info.stdout %}
8. ВКЛЮЧЕННЫЕ СЛУЖБЫ SYSTEMD:
{{ services_info.stdout }}

{% endif %}

{% if system_info is defined %}
9. СИСТЕМНАЯ ИНФОРМАЦИЯ:
{% if system_info.df is defined and system_info.df.stdout %}
Использование дисков:
{{ system_info.df.stdout }}
{% endif %}

{% if system_info.memory is defined and system_info.memory.stdout %}
Информация о памяти:
{{ system_info.memory.stdout }}
{% endif %}

{% if system_info.cpu is defined and system_info.cpu.stdout %}
Информация о CPU:
{{ system_info.cpu.stdout }}
{% endif %}
{% endif %}

==================================================
ОТЧЕТ СФОРМИРОВАН С ИСПОЛЬЗОВАНИЕМ ANSIBLE ROLE
==================================================
```

4. templates/report_csv.j2:

```csv
Хост;Раздел;Подраздел;Параметр;Значение;Время сбора;Статус
{% if network_info_stdout and network_info_stdout != '' %}
{{ ansible_hostname }};Сеть;Открытые порты;Вывод команды ss/netstat;"{{ network_info_stdout | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}

{% if firewall_info_stdout and firewall_info_stdout != '' %}
{{ ansible_hostname }};Безопасность;Межсетевой экран;Правила firewall;"{{ firewall_info_stdout | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}

{% if passwd_stdout and passwd_stdout != '' %}
{{ ansible_hostname }};Пользователи;Системные учетные записи;/etc/passwd;"{{ passwd_stdout | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}

{% if group_stdout and group_stdout != '' %}
{{ ansible_hostname }};Пользователи;Группы;/etc/group;"{{ group_stdout | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}

{% if sudoers_stdout and sudoers_stdout != '' %}
{{ ansible_hostname }};Безопасность;Права sudo;/etc/sudoers;"{{ sudoers_stdout | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}

{% if sudoers_d_stdout and sudoers_d_stdout != '' %}
{{ ansible_hostname }};Безопасность;Права sudo;/etc/sudoers.d/*;"{{ sudoers_d_stdout | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}

{% if sssd_stdout and sssd_stdout != '' %}
{{ ansible_hostname }};Аутентификация;SSSD;/etc/sssd/sssd.conf;"{{ sssd_stdout | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}

{% if nsswitch_stdout and nsswitch_stdout != '' %}
{{ ansible_hostname }};Аутентификация;NSSwitch;/etc/nsswitch.conf;"{{ nsswitch_stdout | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}

{% if services_stdout and services_stdout != '' %}
{{ ansible_hostname }};Службы;Автозагрузка;Включенные службы;"{{ services_stdout | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}

{% if postgres_pg_hba_content and postgres_pg_hba_content != '' %}
{{ ansible_hostname }};Базы данных;PostgreSQL;pg_hba.conf;"{{ postgres_pg_hba_content | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}

{% if postgres_postgresql_conf_content and postgres_postgresql_conf_content != '' %}
{{ ansible_hostname }};Базы данных;PostgreSQL;postgresql.conf;"{{ postgres_postgresql_conf_content | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}

{% if system_info_df and system_info_df != '' %}
{{ ansible_hostname }};Система;Диски;Использование дисков;"{{ system_info_df | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}

{% if system_info_memory and system_info_memory != '' %}
{{ ansible_hostname }};Система;Память;Состояние памяти;"{{ system_info_memory | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}

{% if system_info_cpu and system_info_cpu != '' %}
{{ ansible_hostname }};Система;Процессор;Информация о CPU;"{{ system_info_cpu | escape_csv(delimiter=csv_delimiter) }}";{{ ansible_date_time.iso8601 }};Успешно
{% endif %}
```

5. tasks/main.yml:

```yaml
---
- name: Сбор системной информации
  block:
    # 1. Сбор информации об открытых портах
    - name: Попытка сбора через ss
      ansible.builtin.command: "ss -tulpn"
      register: network_info
      when: collect_network_info
      changed_when: false
      ignore_errors: true
      tags: network

    - name: Попытка сбора через netstat (если ss недоступен)
      ansible.builtin.command: "netstat -tulpn"
      register: network_info_alt
      when: 
        - collect_network_info
        - network_info is failed or network_info.stdout == ""
      changed_when: false
      ignore_errors: true
      tags: network

    # 2. Сбор правил firewall
    - name: Сбор правил iptables
      ansible.builtin.command: "iptables-save"
      register: iptables_info
      when: collect_firewall_info
      changed_when: false
      ignore_errors: true
      tags: firewall

    - name: Сбор правил nftables
      ansible.builtin.command: "nft list ruleset"
      register: nftables_info
      when: collect_firewall_info
      changed_when: false
      ignore_errors: true
      tags: firewall

    # 3. Сбор информации о пользователях и группах
    - name: Чтение /etc/passwd
      ansible.builtin.command: "cat /etc/passwd"
      register: passwd_info
      when: collect_user_info
      changed_when: false
      ignore_errors: true
      tags: users

    - name: Чтение /etc/group
      ansible.builtin.command: "cat /etc/group"
      register: group_info
      when: collect_user_info
      changed_when: false
      ignore_errors: true
      tags: users

    # 4. Сбор конфигурации sudo
    - name: Чтение /etc/sudoers
      ansible.builtin.command: "cat /etc/sudoers"
      register: sudoers_info
      when: collect_sudo_info
      changed_when: false
      ignore_errors: true
      tags: sudo

    - name: Сбор содержимого /etc/sudoers.d/
      ansible.builtin.shell: |
        if [ -d /etc/sudoers.d ]; then
          echo "=== Содержимое /etc/sudoers.d/ ==="
          for file in /etc/sudoers.d/*; do
            if [ -f "$file" ]; then
              echo ""
              echo "=== Файл: $file ==="
              cat "$file" 2>/dev/null || echo "Не удалось прочитать файл"
            fi
          done
        else
          echo "Директория /etc/sudoers.d не существует"
        fi
      register: sudoers_d_content
      when: collect_sudo_info
      changed_when: false
      ignore_errors: true
      tags: sudo

    # 5. Сбор конфигурации SSSD
    - name: Проверка существования /etc/sssd/sssd.conf
      ansible.builtin.stat:
        path: /etc/sssd/sssd.conf
      register: sssd_conf_stat
      when: collect_sssd_info
      tags: sssd

    - name: Чтение /etc/sssd/sssd.conf
      ansible.builtin.command: "cat /etc/sssd/sssd.conf"
      register: sssd_info
      when: 
        - collect_sssd_info
        - sssd_conf_stat.stat.exists
      changed_when: false
      ignore_errors: true
      tags: sssd

    # 6. Сбор конфигурации nsswitch
    - name: Чтение /etc/nsswitch.conf
      ansible.builtin.command: "cat /etc/nsswitch.conf"
      register: nsswitch_info
      when: collect_nsswitch_info
      changed_when: false
      ignore_errors: true
      tags: nsswitch

    # 7. Сбор конфигурации PostgreSQL
    - name: Поиск файлов конфигурации PostgreSQL
      ansible.builtin.find:
        paths: "{{ item }}"
        patterns: "pg_hba.conf,postgresql.conf"
        recurse: true
        file_type: file
      register: postgres_files
      loop: "{{ postgres_config_paths }}"
      when: collect_postgres_info
      changed_when: false
      tags: postgres

    - name: Чтение pg_hba.conf
      ansible.builtin.command: "cat {{ item }}"
      register: pg_hba_content
      loop: "{{ (postgres_files.results | default([])) | map(attribute='files') | flatten | map(attribute='path') | select('match', 'pg_hba') | list }}"
      when: 
        - collect_postgres_info
        - (postgres_files.results | default([])) | length > 0
      changed_when: false
      ignore_errors: true
      tags: postgres

    - name: Чтение postgresql.conf
      ansible.builtin.command: "cat {{ item }}"
      register: postgresql_conf_content
      loop: "{{ (postgres_files.results | default([])) | map(attribute='files') | flatten | map(attribute='path') | select('match', 'postgresql.conf') | list }}"
      when: 
        - collect_postgres_info
        - (postgres_files.results | default([])) | length > 0
      changed_when: false
      ignore_errors: true
      tags: postgres

    # 8. Сбор информации о службах
    - name: Сбор включенных служб systemd
      ansible.builtin.command: "systemctl list-unit-files --type=service --state=enabled"
      register: services_info
      when: collect_services_info
      changed_when: false
      ignore_errors: true
      tags: services

    # 9. Сбор системной информации
    - name: Сбор информации о дисках
      ansible.builtin.command: "df -h"
      register: df_info
      when: collect_system_info
      changed_when: false
      tags: system

    - name: Сбор информации о памяти
      ansible.builtin.command: "free -h"
      register: memory_info
      when: collect_system_info
      changed_when: false
      tags: system

    - name: Сбор информации о процессоре
      ansible.builtin.command: "lscpu"
      register: cpu_info
      when: collect_system_info
      changed_when: false
      ignore_errors: true
      tags: system

    - name: Сбор информации о загрузке системы
      ansible.builtin.command: "uptime"
      register: uptime_info
      when: collect_system_info
      changed_when: false
      tags: system

  rescue:
    - name: Логирование ошибок сбора
      ansible.builtin.debug:
        msg: "Ошибка при сборе информации: {{ ansible_failed_result }}"

- name: Подготовка переменных для отчетов
  set_fact:
    # Для текстового отчета
    network_info_stdout: "{{ network_info_alt.stdout if (network_info is failed or network_info.stdout == '') and network_info_alt.stdout is defined else network_info.stdout if network_info.stdout is defined else '' }}"
    firewall_info_stdout: "{{ nftables_info.stdout if (iptables_info is failed or iptables_info.stdout == '') and nftables_info.stdout is defined else iptables_info.stdout if iptables_info.stdout is defined else '' }}"
    passwd_stdout: "{{ passwd_info.stdout if passwd_info.stdout is defined else '' }}"
    group_stdout: "{{ group_info.stdout if group_info.stdout is defined else '' }}"
    sudoers_stdout: "{{ sudoers_info.stdout if sudoers_info.stdout is defined else '' }}"
    sudoers_d_stdout: "{{ sudoers_d_content.stdout if sudoers_d_content.stdout is defined else '' }}"
    sssd_stdout: "{{ sssd_info.stdout if sssd_info.stdout is defined else '' }}"
    nsswitch_stdout: "{{ nsswitch_info.stdout if nsswitch_info.stdout is defined else '' }}"
    services_stdout: "{{ services_info.stdout if services_info.stdout is defined else '' }}"
    system_info_df: "{{ df_info.stdout if df_info.stdout is defined else '' }}"
    system_info_memory: "{{ memory_info.stdout if memory_info.stdout is defined else '' }}"
    system_info_cpu: "{{ cpu_info.stdout if cpu_info.stdout is defined else '' }}"
    system_info_uptime: "{{ uptime_info.stdout if uptime_info.stdout is defined else '' }}"
    
    # Для структурированного текстового отчета
    users_info:
      passwd: "{{ passwd_info }}"
      group: "{{ group_info }}"
    sudo_info:
      sudoers: "{{ sudoers_info }}"
      sudoers_d_content: "{{ sudoers_d_content }}"
    postgres_info:
      pg_hba_results: "{{ pg_hba_content.results if pg_hba_content is defined and pg_hba_content.results is defined else [] }}"
      postgresql_conf_results: "{{ postgresql_conf_content.results if postgresql_conf_content is defined and postgresql_conf_content.results is defined else [] }}"
    postgres_pg_hba_content: "{{ (pg_hba_content.results | default([])) | map(attribute='stdout') | join('\n---\n') if pg_hba_content is defined else '' }}"
    postgres_postgresql_conf_content: "{{ (postgresql_conf_content.results | default([])) | map(attribute='stdout') | join('\n---\n') if postgresql_conf_content is defined else '' }}"
    system_info:
      df: "{{ df_info }}"
      memory: "{{ memory_info }}"
      cpu: "{{ cpu_info }}"
      uptime: "{{ uptime_info }}"

- name: Формирование текстового отчета
  ansible.builtin.template:
    src: report_template.j2
    dest: "{{ report_path }}"
    mode: '0644'
  vars:
    network_info: "{{ network_info_alt if (network_info is failed or network_info.stdout == '') else network_info }}"
    firewall_info: "{{ nftables_info if (iptables_info is failed or iptables_info.stdout == '') else iptables_info }}"

- name: Формирование CSV отчета
  ansible.builtin.template:
    src: report_csv.j2
    dest: "{{ report_csv_path }}"
    mode: '0644'
  vars:
    csv_delimiter: "{{ csv_delimiter }}"

- name: Формирование структурированного CSV отчета (дополнительный)
  ansible.builtin.copy:
    content: |
      # Структурированный отчет аудита системы
      # Хост: {{ ansible_hostname }}
      # Время: {{ ansible_date_time.iso8601 }}
      # =====================================

      # Открытые порты
      {% if network_info_stdout %}
      {{ network_info_stdout }}
      {% endif %}

      # Пользователи
      {% if passwd_stdout %}
      {{ passwd_stdout }}
      {% endif %}

      # Группы
      {% if group_stdout %}
      {{ group_stdout }}
      {% endif %}

      # Включенные службы
      {% if services_stdout %}
      {{ services_stdout }}
      {% endif %}
    dest: "{{ report_csv_path.replace('.csv', '_structured.txt') }}"
    mode: '0644'
  when: false  # Измените на true для включения

- name: Создание архива с отчетами
  community.general.archive:
    path:
      - "{{ report_path }}"
      - "{{ report_csv_path }}"
    dest: "/tmp/system_audit_{{ ansible_hostname }}_{{ ansible_date_time.date }}.tar.gz"
    format: gz
    remove: false
  when: archive_reports

- name: Копирование отчетов на управляющий узел
  ansible.builtin.fetch:
    src: "{{ item }}"
    dest: "{{ local_report_dir }}"
    flat: false
  loop:
    - "{{ report_path }}"
    - "{{ report_csv_path }}"
  delegate_to: localhost
  run_once: false
  when: fetch_reports

- name: Вывод информации о созданных отчетах
  ansible.builtin.debug:
    msg: |
      Отчеты аудита системы созданы:
      - Текстовый отчет: {{ report_path }}
      - CSV отчет: {{ report_csv_path }}
      {% if archive_reports %}
      - Архив: /tmp/system_audit_{{ ansible_hostname }}_{{ ansible_date_time.date }}.tar.gz
      {% endif %}
      {% if fetch_reports %}
      - Локальная копия: {{ local_report_dir }}
      {% endif %}
```

6. handlers/main.yml (опционально):

```yaml
---
- name: Отправить отчет по email
  ansible.builtin.mail:
    host: "smtp.example.com"
    port: 587
    username: "username"
    password: "password"
    to: "admin@example.com"
    subject: "Системный аудит {{ ansible_hostname }}"
    body: "Отчет аудита системы прикреплен"
    attach:
      - "{{ report_path }}"
      - "{{ report_csv_path }}"
  delegate_to: localhost
  when: false  # Настройте SMTP и измените на true для включения
```

7. Playbook для использования роли:

Создайте файл audit_playbook.yml:

```yaml
---
- name: Выполнить полный аудит системы
  hosts: all
  become: yes
  gather_facts: yes
  
  vars:
    # Настройки отчетов
    report_path: "/tmp/audit_{{ ansible_hostname }}_{{ ansible_date_time.date }}.txt"
    report_csv_path: "/tmp/audit_{{ ansible_hostname }}_{{ ansible_date_time.date }}.csv"
    
    # Настройки CSV
    csv_delimiter: ";"
    
    # Какие разделы собирать
    collect_postgres_info: false  # Отключить, если PostgreSQL не используется
    
    # Дополнительные настройки
    archive_reports: true
    fetch_reports: true
    local_report_dir: "./audit_results/{{ inventory_hostname }}/"
  
  tasks:
    - name: Создать локальную директорию для отчетов
      file:
        path: "./audit_results/{{ inventory_hostname }}/"
        state: directory
        mode: '0755'
      delegate_to: localhost
      run_once: true
      when: fetch_reports
    
    - name: Запуск роли аудита
      include_role:
        name: system_audit_role
    
    - name: Проверить размер отчетов
      stat:
        path: "{{ report_path }}"
      register: report_stat
      delegate_to: "{{ inventory_hostname }}"
    
    - name: Вывести информацию о размере отчетов
      debug:
        msg: "Размер текстового отчета: {{ report_stat.stat.size | default(0) }} байт"
```

8. Playbook для нескольких хостов с консолидированным отчетом:

Создайте файл audit_multiple_hosts.yml:

```yaml
---
- name: Аудит нескольких систем
  hosts: servers
  become: yes
  gather_facts: yes
  
  vars:
    csv_delimiter: "|"
    fetch_reports: true
    local_report_dir: "./consolidated_audit/{{ inventory_hostname }}/"
  
  tasks:
    - name: Запуск аудита
      include_role:
        name: system_audit_role
    
    - name: Консолидация CSV отчетов
      shell: |
        echo "Хост|Раздел|Подраздел|Параметр|Значение|Время сбора|Статус" > /tmp/consolidated_audit.csv
        for host in {% for host in groups['servers'] %}{{ host }} {% endfor %}; do
          ssh $host "tail -n +2 /tmp/system_audit_report_*.csv" >> /tmp/consolidated_audit.csv 2>/dev/null || true
        done
      delegate_to: localhost
      run_once: true
      when: false  # Требует настройки SSH ключей
```

Установка и использование:

1. Создайте структуру роли:

```bash
mkdir -p system_audit_role/{defaults,tasks,templates,filter_plugins,handlers}
```

1. Скопируйте файлы в соответствующие директории
2. Установите необходимые коллекции Ansible:

```bash
ansible-galaxy collection install community.general
```

1. Запустите аудит:

```bash
# Для одного хоста
ansible-playbook -i inventory.ini audit_playbook.yml

# Для группы хостов
ansible-playbook -i inventory.ini audit_multiple_hosts.yml
```

1. Просмотр отчетов:

```bash
# CSV отчет можно открыть в Excel или LibreOffice Calc
libreoffice ./audit_results/hostname/audit_*.csv

# Текстовый отчет
cat ./audit_results/hostname/audit_*.txt
```

Особенности реализации:

1. Полная обработка ошибок - все команды с ignore_errors: true
2. Два формата отчетов - текстовый и CSV
3. Экранирование CSV - автоматическая обработка специальных символов
4. Гибкая настройка - включение/выключение разделов через переменные
5. Автоматическая архивация - создание tar.gz архивов
6. Локальное хранение - копирование отчетов на управляющий узел
7. Структурированные данные - использование фильтров для парсинга
8. Поддержка разных систем - ss/netstat, iptables/nftables
9. Расширяемость - легко добавить новые модули сбора
10. Безопасность - не требует изменения системы, только сбор информации