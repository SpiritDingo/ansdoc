Вот пример роли Ansible для сбора указанной информации с генерацией JSON-отчетов:

Структура ролей:

```
gather_system_info/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
└── meta/
    └── main.yml
```

Файлы роли:

defaults/main.yml:

```yaml
---
# Директория для сохранения отчетов
report_dir: "/tmp/system_reports"
```

tasks/main.yml:

```yaml
---
- name: Создать директорию для отчетов на control node
  delegate_to: localhost
  run_once: yes
  file:
    path: "{{ report_dir }}"
    state: directory
    mode: '0755'

- name: Получить hostname и IP адреса
  block:
    - name: Получить hostname
      command: hostname -s
      register: hostname_short
      changed_when: false
      
    - name: Получить полное имя хоста
      command: hostname -f
      register: hostname_full
      changed_when: false
      
    - name: Получить IP адреса
      shell: |
        ip -o -4 addr show | awk '{print $2": "$4}' | sed 's/\/.*//'
      register: ip_addresses
      changed_when: false
  rescue:
    - name: Установить значения по умолчанию при ошибке
      set_fact:
        hostname_short: "ERROR"
        hostname_full: "ERROR"
        ip_addresses: "ERROR"

- name: Получить информацию об ОС
  block:
    - name: Определить дистрибутив
      shell: |
        if [ -f /etc/os-release ]; then
          source /etc/os-release
          echo "$NAME $VERSION"
        elif [ -f /etc/redhat-release ]; then
          cat /etc/redhat-release
        else
          uname -o
        fi
      register: os_info
      changed_when: false
  rescue:
    - set_fact:
        os_info: "ERROR"

- name: Получить список открытых портов
  block:
    - name: Проверить наличие ss
      stat:
        path: /usr/sbin/ss
      register: ss_exists
      
    - name: Получить открытые порты через ss
      command: ss -tulpn
      when: ss_exists.stat.exists
      register: open_ports
      changed_when: false
      
    - name: Получить открытые порты через netstat
      command: netstat -tulpn
      when: not ss_exists.stat.exists
      register: open_ports
      changed_when: false
  rescue:
    - set_fact:
        open_ports: "ERROR"

- name: Проверить конфигурацию firewall
  block:
    - name: Проверить наличие iptables
      stat:
        path: /usr/sbin/iptables
      register: iptables_exists
      
    - name: Проверить наличие nft
      stat:
        path: /usr/sbin/nft
      register: nft_exists
      
    - name: Получить правила iptables
      command: iptables-save
      when: iptables_exists.stat.exists
      register: firewall_config
      changed_when: false
      
    - name: Получить правила nftables
      command: nft list ruleset
      when: nft_exists.stat.exists and not iptables_exists.stat.exists
      register: firewall_config
      changed_when: false
      
    - name: Установить статус false если фаервол отключен
      set_fact:
        firewall_config: "false"
      when: not iptables_exists.stat.exists and not nft_exists.stat.exists
  rescue:
    - set_fact:
        firewall_config: "ERROR"

- name: Получить список пользователей и групп
  block:
    - name: Получить список пользователей
      shell: |
        cat /etc/passwd | cut -d: -f1,3,4,6,7
      register: users_list
      changed_when: false
      
    - name: Получить список групп
      shell: |
        cat /etc/group | cut -d: -f1,2,3,4
      register: groups_list
      changed_when: false
  rescue:
    - set_fact:
        users_list: "ERROR"
        groups_list: "ERROR"

- name: Получить конфигурацию sudo
  block:
    - name: Проверить наличие /etc/sudoers
      stat:
        path: /etc/sudoers
      register: sudoers_exists
      
    - name: Прочитать /etc/sudoers
      shell: |
        cat /etc/sudoers
      when: sudoers_exists.stat.exists
      register: sudoers_config
      changed_when: false
      
    - name: Прочитать файлы в /etc/sudoers.d/
      find:
        paths: /etc/sudoers.d
        patterns: '*'
        file_type: file
      register: sudoers_d_files
      changed_when: false
      
    - name: Собрать содержимое файлов sudoers.d
      shell: |
        for file in /etc/sudoers.d/*; do
          echo "=== $file ==="
          [ -f "$file" ] && cat "$file" || echo "File not found"
          echo ""
        done
      when: sudoers_d_files.files | length > 0
      register: sudoers_d_content
      changed_when: false
  rescue:
    - set_fact:
        sudoers_config: "ERROR"
        sudoers_d_content: "ERROR"

- name: Получить конфигурацию SSSD
  block:
    - name: Проверить наличие SSSD конфига
      stat:
        path: /etc/sssd/sssd.conf
      register: sssd_exists
      
    - name: Прочитать конфиг SSSD
      shell: |
        cat /etc/sssd/sssd.conf
      when: sssd_exists.stat.exists
      register: sssd_config
      changed_when: false
  rescue:
    - set_fact:
        sssd_config: "false"

- name: Получить конфигурацию nsswitch
  block:
    - name: Прочитать nsswitch.conf
      shell: |
        cat /etc/nsswitch.conf
      register: nsswitch_config
      changed_when: false
  rescue:
    - set_fact:
        nsswitch_config: "ERROR"

- name: Получить конфигурацию PostgreSQL
  block:
    - name: Найти файлы PostgreSQL
      find:
        paths: 
          - /var/lib/pgsql
          - /etc/postgresql
          - /var/lib/postgresql
        patterns: 
          - "pg_hba.conf"
          - "postgresql.conf"
        recurse: yes
      register: postgres_files
      changed_when: false
      
    - name: Прочитать pg_hba.conf если найден
      shell: |
        cat {{ item }}
      loop: "{{ postgres_files.files | map(attribute='path') | list | select('search', 'pg_hba') }}"
      register: pg_hba_config
      changed_when: false
      
    - name: Прочитать postgresql.conf если найден
      shell: |
        cat {{ item }}
      loop: "{{ postgres_files.files | map(attribute='path') | list | select('search', 'postgresql.conf') }}"
      register: postgresql_config
      changed_when: false
      
    - name: Установить false если файлы не найдены
      set_fact:
        pg_hba_config: "false"
        postgresql_config: "false"
      when: postgres_files.files | length == 0
  rescue:
    - set_fact:
        pg_hba_config: "ERROR"
        postgresql_config: "ERROR"

- name: Получить список включенных сервисов
  block:
    - name: Получить enabled сервисы
      command: systemctl list-unit-files --type=service --state=enabled
      register: enabled_services
      changed_when: false
  rescue:
    - set_fact:
        enabled_services: "ERROR"

- name: Собрать все данные в переменную
  set_fact:
    system_info: |
      {
        "hostname": {
          "short": "{{ hostname_short.stdout | default('ERROR') }}",
          "full": "{{ hostname_full.stdout | default('ERROR') }}"
        },
        "ip_addresses": "{{ ip_addresses.stdout_lines | default('ERROR') }}",
        "os_info": "{{ os_info.stdout | default('ERROR') }}",
        "open_ports": "{{ open_ports.stdout | default('ERROR') }}",
        "firewall_config": "{{ firewall_config.stdout | default(firewall_config) | string }}",
        "users": "{{ users_list.stdout | default('ERROR') }}",
        "groups": "{{ groups_list.stdout | default('ERROR') }}",
        "sudoers": {
          "main": "{{ sudoers_config.stdout | default('ERROR') }}",
          "sudoers_d": "{{ sudoers_d_content.stdout | default('ERROR') }}"
        },
        "sssd_config": "{{ sssd_config.stdout | default(sssd_config) | string }}",
        "nsswitch_config": "{{ nsswitch_config.stdout | default('ERROR') }}",
        "postgresql": {
          "pg_hba.conf": "{{ pg_hba_config.results[0].stdout if pg_hba_config is defined and 'results' in pg_hba_config else pg_hba_config }}",
          "postgresql.conf": "{{ postgresql_config.results[0].stdout if postgresql_config is defined and 'results' in postgresql_config else postgresql_config }}"
        },
        "enabled_services": "{{ enabled_services.stdout | default('ERROR') }}",
        "collection_time": "{{ ansible_date_time.iso8601 }}"
      }

- name: Сохранить отчет в JSON файл
  delegate_to: localhost
  copy:
    content: "{{ system_info | to_nice_json }}"
    dest: "{{ report_dir }}/{{ inventory_hostname }}_report.json"
    mode: '0644'
```

Playbook для использования роли:

gather_info_playbook.yml:

```yaml
---
- name: Сбор системной информации
  hosts: all
  become: yes
  gather_facts: yes
  
  roles:
    - gather_system_info
```

Пример использования:

1. Установите структуру ролей:

```bash
mkdir -p roles/gather_system_info/{tasks,defaults,meta}
touch roles/gather_system_info/{tasks,defaults,meta}/main.yml
```

1. Создайте inventory файл (hosts.ini):

```ini
[servers]
server1 ansible_host=192.168.1.10
server2 ansible_host=192.168.1.11

[servers:vars]
ansible_user=your_user
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

1. Запустите playbook:

```bash
ansible-playbook -i hosts.ini gather_info_playbook.yml
```

1. Результаты будут сохранены в директории /tmp/system_reports/ на control node:

```
/tmp/system_reports/
├── server1_report.json
└── server2_report.json
```

Дополнительные улучшения:

1. Шифрование чувствительных данных (если нужно):

```yaml
- name: Зашифровать отчеты
  delegate_to: localhost
  community.crypto.openssl_encrypt:
    src: "{{ report_dir }}/{{ inventory_hostname }}_report.json"
    dest: "{{ report_dir }}/{{ inventory_hostname }}_report.json.enc"
    cipher: aes256
    passphrase: "your_encryption_key"
```

1. Архивация отчетов:

```yaml
- name: Архивировать все отчеты
  delegate_to: localhost
  run_once: yes
  archive:
    path: "{{ report_dir }}"
    dest: "/tmp/system_reports_archive.zip"
    format: zip
```

1. Отправка отчетов (например, по SCP):

```yaml
- name: Отправить отчеты на удаленный сервер
  delegate_to: localhost
  run_once: yes
  community.general.scp:
    src: "{{ report_dir }}/"
    dest: "user@backup.server:/backups/"
    recursive: yes
```

Эта роль собирает всю запрошенную информацию и сохраняет ее в JSON-файлы для каждого сервера отдельно, как и требовалось.