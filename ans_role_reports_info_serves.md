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