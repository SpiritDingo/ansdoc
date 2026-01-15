Вот Ansible роль для сбора информации с серверов и сохранения в CSV файлы:

Структура роли:

```
server_audit/
├── tasks/
│   └── main.yml
├── templates/
│   └── report_template.j2
└── defaults/
    └── main.yml
```

1. defaults/main.yml:

```yaml
---
# Директория для сохранения отчетов
report_dir: "/tmp/server_audit_reports"
# Формат даты для имен файлов
date_format: "%Y-%m-%d_%H-%M-%S"
```

2. templates/report_template.j2:

```jinja2
Сервер: {{ inventory_hostname }}
Дата сбора: {{ ansible_date_time.iso8601 }}

1. ОТКРЫТЫЕ ПОРТЫ И ПРОЦЕССЫ:
{{ open_ports.stdout }}

2. ПРАВИЛА МЭ (IPTABLES/NFTABLES):
{{ firewall_config.stdout }}

3. ПОЛЬЗОВАТЕЛИ И ГРУППЫ:
=== /etc/passwd ===
{{ etc_passwd.stdout }}
=== /etc/group ===
{{ etc_group.stdout }}

4. КОНФИГУРАЦИЯ SUDO:
=== /etc/sudoers ===
{{ sudoers.stdout }}
{% if sudoers_d.stat.exists %}
=== /etc/sudoers.d/ ===
{{ sudoers_d.stdout }}
{% endif %}

5. КОНФИГУРАЦИЯ SSSD:
{% if sssd_conf.stat.exists %}
{{ sssd_conf.stdout }}
{% else %}
Файл /etc/sssd/sssd.conf не найден
{% endif %}

6. КОНФИГУРАЦИЯ NSSWITCH:
{{ nsswitch.stdout }}

7. КОНФИГУРАЦИЯ POSTGRESQL:
{% if pg_hba.stat.exists %}
=== pg_hba.conf ===
{{ pg_hba.stdout }}
{% else %}
Файл pg_hba.conf не найден
{% endif %}

{% if postgresql_conf.stat.exists %}
=== postgresql.conf ===
{{ postgresql_conf.stdout }}
{% else %}
Файл postgresql.conf не найден
{% endif %}

8. ВКЛЮЧЕННЫЕ СЕРВИСЫ:
{{ enabled_services.stdout }}
```

3. tasks/main.yml:

```yaml
---
- name: Создать директорию для отчетов на control node
  delegate_to: localhost
  run_once: yes
  file:
    path: "{{ report_dir }}"
    state: directory
    mode: '0755'

- name: Проверить наличие ss или netstat
  block:
    - name: Проверить наличие ss
      command: which ss
      register: ss_check
      changed_when: false
      failed_when: false
    
    - name: Установить команду для проверки портов
      set_fact:
        ports_command: "ss -tulpn"
      when: ss_check.rc == 0
    
    - name: Установить команду netstat если ss не найден
      set_fact:
        ports_command: "netstat -tulpn"
      when: ss_check.rc != 0
  always:
    - debug:
        msg: "Будет использована команда: {{ ports_command }}"

- name: Собрать открытые порты
  command: "{{ ports_command }}"
  register: open_ports
  changed_when: false

- name: Проверить тип firewall (iptables/nftables)
  block:
    - name: Проверить наличие nft
      command: which nft
      register: nft_check
      changed_when: false
      failed_when: false
    
    - name: Проверить наличие iptables-save
      command: which iptables-save
      register: iptables_check
      changed_when: false
      failed_when: false
    
    - name: Установить команду для firewall
      set_fact:
        firewall_command: "nft list ruleset"
      when: nft_check.rc == 0
    
    - name: Установить команду iptables если nft не найден
      set_fact:
        firewall_command: "iptables-save"
      when: nft_check.rc != 0 and iptables_check.rc == 0
  always:
    - debug:
        msg: "Будет использована команда firewall: {{ firewall_command }}"

- name: Собрать конфигурацию firewall
  command: "{{ firewall_command }}"
  register: firewall_config
  changed_when: false
  ignore_errors: yes

- name: Собрать список пользователей
  command: cat /etc/passwd
  register: etc_passwd
  changed_when: false

- name: Собрать список групп
  command: cat /etc/group
  register: etc_group
  changed_when: false

- name: Собрать конфигурацию sudoers
  command: cat /etc/sudoers
  register: sudoers
  changed_when: false
  ignore_errors: yes

- name: Собрать файлы из sudoers.d
  shell: |
    if [ -d /etc/sudoers.d ]; then
      for file in /etc/sudoers.d/*; do
        if [ -f "$file" ]; then
          echo "=== $file ==="
          cat "$file"
          echo ""
        fi
      done
    fi
  register: sudoers_d
  changed_when: false

- name: Проверить наличие файла sssd.conf
  stat:
    path: /etc/sssd/sssd.conf
  register: sssd_conf_stat

- name: Собрать конфигурацию SSSD
  command: cat /etc/sssd/sssd.conf
  register: sssd_conf
  changed_when: false
  when: sssd_conf_stat.stat.exists

- name: Собрать конфигурацию nsswitch
  command: cat /etc/nsswitch.conf
  register: nsswitch
  changed_when: false

- name: Найти файлы PostgreSQL
  block:
    - name: Найти pg_hba.conf
      shell: |
        find / -name "pg_hba.conf" 2>/dev/null | head -1
      register: pg_hba_path
      changed_when: false
    
    - name: Найти postgresql.conf
      shell: |
        find / -name "postgresql.conf" 2>/dev/null | head -1
      register: postgresql_conf_path
      changed_when: false
  always:
    - debug:
        msg: |
          pg_hba.conf: {{ pg_hba_path.stdout | default('Не найден') }}
          postgresql.conf: {{ postgresql_conf_path.stdout | default('Не найден') }}

- name: Собрать pg_hba.conf если найден
  command: cat "{{ pg_hba_path.stdout }}"
  register: pg_hba
  changed_when: false
  when: pg_hba_path.stdout != ""

- name: Собрать postgresql.conf если найден
  command: cat "{{ postgresql_conf_path.stdout }}"
  register: postgresql_conf
  changed_when: false
  when: postgresql_conf_path.stdout != ""

- name: Собрать включенные сервисы
  command: systemctl list-unit-files --type=service --state=enabled
  register: enabled_services
  changed_when: false
  ignore_errors: yes

- name: Создать отчет на удаленном хосте
  template:
    src: report_template.j2
    dest: "/tmp/server_audit_{{ inventory_hostname }}.txt"
    mode: '0644'

- name: Собрать отчеты на control node
  fetch:
    src: "/tmp/server_audit_{{ inventory_hostname }}.txt"
    dest: "{{ report_dir }}/"
    flat: no

- name: Конвертировать в CSV (простая версия)
  delegate_to: localhost
  run_once: yes
  shell: |
    for file in {{ report_dir }}/*/tmp/server_audit_*.txt; do
      if [ -f "$file" ]; then
        csv_file="${file%.txt}.csv"
        echo '"Секция","Содержимое"' > "$csv_file"
        
        # Извлекаем имя сервера
        server_name=$(basename "$file" | sed 's/server_audit_//' | sed 's/.txt//')
        echo "\"Сервер\",\"$server_name\"" >> "$csv_file"
        
        # Добавляем остальные секции (упрощенно)
        # В реальности нужно более сложное преобразование
        grep -A1000 "1. ОТКРЫТЫЕ ПОРТЫ" "$file" | head -200 | while IFS= read -r line; do
          echo "\"Открытые порты\",\"$line\"" >> "$csv_file"
        done
      fi
    done
  changed_when: false

- name: Очистить временные файлы на хостах
  file:
    path: "/tmp/server_audit_{{ inventory_hostname }}.txt"
    state: absent

- name: Показать информацию о созданных отчетах
  delegate_to: localhost
  run_once: yes
  debug:
    msg: |
      Отчеты сохранены в: {{ report_dir }}
      Для каждого сервера созданы файлы:
        - .txt (полный отчет)
        - .csv (CSV версия)
```

4. Пример плейбука для использования роли:

```yaml
---
- name: Сбор аудиторской информации с серверов
  hosts: all
  become: yes
  gather_facts: yes
  
  vars:
    report_dir: "/opt/audit_reports/{{ ansible_date_time.date }}"
  
  roles:
    - server_audit
  
  tasks:
    - name: Архивировать отчеты
      delegate_to: localhost
      run_once: yes
      archive:
        path: "{{ report_dir }}"
        dest: "/opt/audit_reports/server_audit_{{ ansible_date_time.date }}.tar.gz"
        format: gz
```

5. Улучшенная версия CSV генератора:

Создайте отдельный скрипт для преобразования в CSV:

```jinja2
{# templates/generate_csv.j2 #}
#!/bin/bash
REPORT_DIR="{{ report_dir }}"

for file in $REPORT_DIR/*/tmp/server_audit_*.txt; do
    if [ -f "$file" ]; then
        SERVER_NAME=$(basename "$file" | sed 's/server_audit_//' | sed 's/.txt//')
        CSV_FILE="${file%.txt}.csv"
        
        echo '"Раздел","Подраздел","Содержимое"' > "$CSV_FILE"
        
        # Извлекаем каждую секцию
        awk -v server="$SERVER_NAME" '
        BEGIN {
            print "\"Сервер\",\"" server "\",\"\""
            section = ""
            subsection = ""
        }
        /^1\. ОТКРЫТЫЕ ПОРТЫ/ { section = "Открытые порты"; subsection = ""; next }
        /^2\. ПРАВИЛА МЭ/ { section = "Firewall"; subsection = ""; next }
        /^3\. ПОЛЬЗОВАТЕЛИ И ГРУППЫ/ { section = "Пользователи"; subsection = ""; next }
        /^=== \/etc\/passwd ===/ { subsection = "/etc/passwd"; next }
        /^=== \/etc\/group ===/ { subsection = "/etc/group"; next }
        /^4\. КОНФИГУРАЦИЯ SUDO/ { section = "Sudo"; subsection = ""; next }
        /^5\. КОНФИГУРАЦИЯ SSSD/ { section = "SSSD"; subsection = ""; next }
        /^6\. КОНФИГУРАЦИЯ NSSWITCH/ { section = "NSSwitch"; subsection = ""; next }
        /^7\. КОНФИГУРАЦИЯ POSTGRESQL/ { section = "PostgreSQL"; subsection = ""; next }
        /^8\. ВКЛЮЧЕННЫЕ СЕРВИСЫ/ { section = "Сервисы"; subsection = ""; next }
        
        /^=== .* ===$/ {
            if (match($0, /^=== (.*) ===$/, arr)) {
                subsection = arr[1]
            }
            next
        }
        
        section != "" && $0 !~ /^Сервер:/ && $0 !~ /^Дата сбора:/ {
            gsub(/"/, "\"\"", $0)
            printf "\"%s\",\"%s\",\"%s\"\n", section, subsection, $0 >> "'"$CSV_FILE"'"
        }
        ' "$file"
        
        echo "Создан CSV: $CSV_FILE"
    fi
done
```

Использование:

1. Создайте структуру роли как показано выше
2. Создайте плейбук для запуска роли
3. Запустите плейбук:

```bash
ansible-playbook -i inventory.ini audit_servers.yml
```

Примечания:

1. Роль использует become: yes для доступа к системным файлам
2. Отчеты сохраняются на control node в указанной директории
3. Для каждого сервера создается отдельный файл
4. CSV файлы содержат структурированные данные по разделам

Для более сложного CSV преобразования рекомендуется использовать Python скрипт или специализированные модули Ansible для обработки данных.