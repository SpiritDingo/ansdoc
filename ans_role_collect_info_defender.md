# Ansible Role: MS Defender Info Collector (Path-Independent)

Эта роль собирает информацию о Microsoft Defender на Ubuntu и Oracle Linux 9 без явного указания пути к бинарнику.

## Структура роли

```
roles/
└── msdefender_universal/
    ├── tasks/
    │   └── main.yml
    ├── templates/
    │   └── report.j2
    └── defaults/
        └── main.yml
```

## Реализация

### tasks/main.yml

```yaml
---
- name: Check if Microsoft Defender is installed
  shell: |
    if command -v mdatp >/dev/null 2>&1; then
      echo "INSTALLED"
    else
      echo "NOT_INSTALLED"
    fi
  register: defender_check
  changed_when: false

- name: Get Defender status (if installed)
  shell: |
    mdatp health --output json
  register: defender_status
  when: defender_check.stdout == "INSTALLED"
  ignore_errors: yes
  changed_when: false

- name: Get organization ID (if installed)
  shell: |
    mdatp health --field org_id
  register: org_id
  when: defender_check.stdout == "INSTALLED"
  ignore_errors: yes
  changed_when: false

- name: Set facts for reporting
  set_fact:
    defender_installed: "{{ defender_check.stdout == 'INSTALLED' }}"
    defender_status_json: "{{ defender_status.stdout | default('{}') | from_json }}"
    org_id_value: "{{ org_id.stdout | default('N/A') }}"
    os_info: "{{ ansible_distribution }} {{ ansible_distribution_version }}"

- name: Generate CSV report
  template:
    src: report.j2
    dest: "{{ report_output_path }}"
  delegate_to: localhost
  run_once: true
```

### templates/report.j2

```jinja2
Host,OS,Defender Installed,Defender Status,Org ID
{% for host in ansible_play_hosts %}
{{ hostvars[host].inventory_hostname }},
{{ hostvars[host].os_info }},
{% if hostvars[host].defender_installed %}YES{% else %}NO{% endif %},
{% if hostvars[host].defender_installed %}{{ hostvars[host].defender_status_json | to_json | replace('\n', ' ') | replace(',', ';') }}{% else %}N/A{% endif %},
{{ hostvars[host].org_id_value }}
{% endfor %}
```

### defaults/main.yml

```yaml
---
report_output_path: "/tmp/defender_report_{{ ansible_date_time.date }}.csv"
```

## Использование

1. Создайте playbook `collect_defender.yml`:

```yaml
---
- name: Collect Microsoft Defender information
  hosts: linux_servers
  become: yes
  gather_facts: yes

  vars:
    report_output_path: "/var/tmp/defender_status_report.csv"

  roles:
    - msdefender_universal
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini collect_defender.yml
```

## Особенности

1. **Не зависит от пути установки** - использует системный PATH для поиска mdatp
2. **Автоматически определяет** установлен ли Defender
3. **Поддерживает** Ubuntu и Oracle Linux 9
4. **Генерирует отчет** в CSV формате
5. **Обрабатывает ошибки** при отсутствии Defender

## Пример вывода

```
Host,OS,Defender Installed,Defender Status,Org ID
server1,Ubuntu 22.04,YES,"{""healthy"":true,""version"":""101.92.19""}",ORG12345
server2,OracleLinux 9.2,NO,N/A,N/A
server3,Ubuntu 20.04,YES,"{""healthy"":false,""version"":""101.90.15""}",ORG67890
```

## Требования

- Ansible 2.9+
- Доступ к системному PATH на целевых хостах
- Python для работы с JSON (на управляющей машине)