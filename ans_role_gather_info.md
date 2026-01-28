Вот пример роли Ansible для сбора информации и создания CSV-отчета:

Структура роли:

```
roles/gather_info/
├── tasks/
│   └── main.yml
├── templates/
│   └── report.csv.j2
└── vars/
    └── main.yml
```

1. roles/gather_info/vars/main.yml:

```yaml
report_filename: "system_info_report.csv"
report_path: "/tmp/{{ report_filename }}"
```

2. roles/gather_info/templates/report.csv.j2:

```jinja2
"Hostname","Timestamp","Check","Result"
{% for host in ansible_play_hosts %}
{% set host_vars = hostvars[host] %}
{% if host_vars.open_ports_result is defined %}
"{{ host }}","{{ ansible_date_time.iso8601 }}","Open Ports","{{ host_vars.open_ports_result.stdout | replace('\n', ' ') | replace('"', '""') }}"
{% endif %}
{% if host_vars.iptables_result is defined %}
"{{ host }}","{{ ansible_date_time.iso8601 }}","Firewall Rules","{{ host_vars.iptables_result.stdout | replace('\n', ' ') | replace('"', '""') }}"
{% endif %}
{% if host_vars.firewall_nft_result is defined %}
"{{ host }}","{{ ansible_date_time.iso8601 }}","Firewall Rules (nft)","{{ host_vars.firewall_nft_result.stdout | replace('\n', ' ') | replace('"', '""') }}"
{% endif %}
{% if host_vars.users_result is defined %}
"{{ host }}","{{ ansible_date_time.iso8601 }}","Users","{{ host_vars.users_result.stdout | replace('\n', ' ') | replace('"', '""') }}"
{% endif %}
{% if host_vars.groups_result is defined %}
"{{ host }}","{{ ansible_date_time.iso8601 }}","Groups","{{ host_vars.groups_result.stdout | replace('\n', ' ') | replace('"', '""') }}"
{% endif %}
{% if host_vars.sudoers_result is defined %}
"{{ host }}","{{ ansible_date_time.iso8601 }}","Sudoers","{{ host_vars.sudoers_result.stdout | replace('\n', ' ') | replace('"', '""') }}"
{% endif %}
{% if host_vars.sudoers_d_result is defined %}
"{{ host }}","{{ ansible_date_time.iso8601 }}","Sudoers.d Files","{{ host_vars.sudoers_d_result.stdout | replace('\n', ' ') | replace('"', '""') }}"
{% endif %}
{% if host_vars.sssd_conf_result is defined %}
"{{ host }}","{{ ansible_date_time.iso8601 }}","SSSD Config","{{ host_vars.sssd_conf_result.stdout | replace('\n', ' ') | replace('"', '""') }}"
{% endif %}
{% if host_vars.nsswitch_result is defined %}
"{{ host }}","{{ ansible_date_time.iso8601 }}","nsswitch.conf","{{ host_vars.nsswitch_result.stdout | replace('\n', ' ') | replace('"', '""') }}"
{% endif %}
{% if host_vars.pg_hba_result is defined %}
"{{ host }}","{{ ansible_date_time.iso8601 }}","PostgreSQL pg_hba.conf","{{ host_vars.pg_hba_result.stdout | replace('\n', ' ') | replace('"', '""') }}"
{% endif %}
{% if host_vars.postgresql_conf_result is defined %}
"{{ host }}","{{ ansible_date_time.iso8601 }}","PostgreSQL postgresql.conf","{{ host_vars.postgresql_conf_result.stdout | replace('\n', ' ') | replace('"', '""') }}"
{% endif %}
{% if host_vars.services_result is defined %}
"{{ host }}","{{ ansible_date_time.iso8601 }}","Enabled Services","{{ host_vars.services_result.stdout | replace('\n', ' ') | replace('"', '""') }}"
{% endif %}
{% endfor %}
```

3. roles/gather_info/tasks/main.yml:

```yaml
---
- name: Create temporary directory for report
  tempfile:
    state: directory
    suffix: ansible_gather
  register: temp_dir
  delegate_to: localhost
  run_once: yes

- name: Gather open ports
  block:
    - name: Try using ss command
      command: ss -tulpn
      register: open_ports_result
      ignore_errors: yes
      changed_when: false
    
    - name: Try using netstat if ss fails
      command: netstat -tulpn
      register: open_ports_result
      when: open_ports_result is failed or open_ports_result is skipped
      ignore_errors: yes
      changed_when: false

- name: Gather firewall rules (iptables)
  command: iptables-save
  register: iptables_result
  ignore_errors: yes
  changed_when: false

- name: Gather firewall rules (nftables)
  command: nft list ruleset
  register: firewall_nft_result
  ignore_errors: yes
  changed_when: false

- name: Gather users information
  command: cat /etc/passwd
  register: users_result
  changed_when: false

- name: Gather groups information
  command: cat /etc/group
  register: groups_result
  changed_when: false

- name: Gather sudoers configuration
  block:
    - name: Read main sudoers file
      command: cat /etc/sudoers
      register: sudoers_result
      ignore_errors: yes
      changed_when: false
    
    - name: Read sudoers.d directory files
      shell: |
        if [ -d /etc/sudoers.d ]; then
          for file in /etc/sudoers.d/*; do
            if [ -f "$file" ]; then
              echo "=== File: $file ==="
              cat "$file"
              echo ""
            fi
          done
        fi
      register: sudoers_d_result
      ignore_errors: yes
      changed_when: false

- name: Gather SSSD configuration
  block:
    - name: Check if SSSD config exists
      stat:
        path: /etc/sssd/sssd.conf
      register: sssd_conf_stat
    
    - name: Read SSSD config if exists
      command: cat /etc/sssd/sssd.conf
      register: sssd_conf_result
      when: sssd_conf_stat.stat.exists
      changed_when: false

- name: Gather nsswitch configuration
  command: cat /etc/nsswitch.conf
  register: nsswitch_result
  changed_when: false

- name: Gather PostgreSQL configurations
  block:
    - name: Find PostgreSQL data directory
      shell: |
        # Try to find postgresql.conf in common locations
        find /var/lib/pgsql /var/lib/postgresql /etc/postgresql -name "postgresql.conf" 2>/dev/null | head -1
      register: postgresql_conf_path
      changed_when: false
      ignore_errors: yes
    
    - name: Read postgresql.conf if exists
      shell: |
        if [ -f "{{ postgresql_conf_path.stdout }}" ]; then
          cat "{{ postgresql_conf_path.stdout }}"
        fi
      register: postgresql_conf_result
      when: postgresql_conf_path.stdout != ""
      changed_when: false
      ignore_errors: yes
    
    - name: Find pg_hba.conf
      shell: |
        # Try to find pg_hba.conf in common locations
        find /var/lib/pgsql /var/lib/postgresql /etc/postgresql -name "pg_hba.conf" 2>/dev/null | head -1
      register: pg_hba_path
      changed_when: false
      ignore_errors: yes
    
    - name: Read pg_hba.conf if exists
      shell: |
        if [ -f "{{ pg_hba_path.stdout }}" ]; then
          cat "{{ pg_hba_path.stdout }}"
        fi
      register: pg_hba_result
      when: pg_hba_path.stdout != ""
      changed_when: false
      ignore_errors: yes

- name: Gather enabled services
  command: systemctl list-unit-files --type=service --state=enabled
  register: services_result
  changed_when: false

- name: Generate CSV report on control node
  template:
    src: report.csv.j2
    dest: "{{ temp_dir.path }}/{{ report_filename }}"
  delegate_to: localhost
  run_once: yes

- name: Display report location
  debug:
    msg: "Report generated at {{ temp_dir.path }}/{{ report_filename }}"
  delegate_to: localhost
  run_once: yes

- name: Download report to local machine (optional)
  fetch:
    src: "{{ temp_dir.path }}/{{ report_filename }}"
    dest: "./"
    flat: yes
  delegate_to: localhost
  run_once: yes
  when: false  # Set to true if you want to download automatically
```

4. Playbook для использования роли (gather_info_playbook.yml):

```yaml
---
- name: Gather system information and create report
  hosts: all
  become: yes
  gather_facts: yes
  
  roles:
    - gather_info
```

Использование:

1. Создайте структуру директорий как указано выше
2. Поместите файлы в соответствующие директории
3. Создайте inventory файл с вашими хостами
4. Запустите playbook:

```bash
ansible-playbook -i inventory gather_info_playbook.yml
```

Особенности:

1. Роль собирает все указанные данные
2. Создает CSV файл на control node
3. Обрабатывает ошибки (например, если команды или файлы отсутствуют)
4. Для PostgreSQL ищет конфигурационные файлы в типичных местах
5. CSV файл содержит:
   · Имя хоста
   · Метку времени
   · Тип проверки
   · Результат (данные)

Вы можете модифицировать шаблон CSV под свои нужды, добавлять дополнительные проверки или изменять формат вывода.