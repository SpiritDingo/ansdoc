# Ansible роль для сбора информации об установленных cAdvisor агентах на Linux серверах

Вот пример Ansible роли, которая собирает информацию об установленных cAdvisor агентах на Linux серверах.

## Структура роли

```
roles/cadvisor_info/
├── tasks/
│   └── main.yml
├── templates/
│   └── cadvisor_report.j2
└── vars/
    └── main.yml
```

## Файлы роли

### roles/cadvisor_info/tasks/main.yml

```yaml
---
- name: Check if cAdvisor is installed as Docker container
  shell: docker ps --filter "name=cadvisor" --format "{{ '{{' }}.Names}} {{ '{{' }}.Image}} {{ '{{' }}.Status}} {{ '{{' }}.Ports}}"
  register: cadvisor_docker
  ignore_errors: yes
  changed_when: false

- name: Check if cAdvisor is running as systemd service
  systemd:
    name: cadvisor
  register: cadvisor_systemd
  changed_when: false

- name: Check cAdvisor process
  shell: ps aux | grep '[c]advisor'
  register: cadvisor_process
  changed_when: false
  ignore_errors: yes

- name: Gather listening ports (to find cAdvisor web interface)
  shell: netstat -tulnp | grep -E '8080|8081|4194' || ss -tulnp | grep -E '8080|8081|4194'
  register: cadvisor_ports
  changed_when: false
  ignore_errors: yes

- name: Create report
  template:
    src: cadvisor_report.j2
    dest: /tmp/cadvisor_report_{{ ansible_hostname }}.txt
  delegate_to: localhost
  run_once: true
```

### roles/cadvisor_info/templates/cadvisor_report.j2

```jinja2
cAdvisor Information Report for {{ ansible_hostname }}
Generated at: {{ ansible_date_time.iso8601 }}

=== Docker Container Check ===
{% if cadvisor_docker.stdout_lines %}
cAdvisor is running in Docker:
{% for line in cadvisor_docker.stdout_lines %}
  - {{ line }}
{% endfor %}
{% else %}
No cAdvisor Docker containers found.
{% endif %}

=== Systemd Service Check ===
{% if cadvisor_systemd.status %}
cAdvisor systemd service is {{ cadvisor_systemd.status }}.
{% else %}
No cAdvisor systemd service found.
{% endif %}

=== Process Check ===
{% if cadvisor_process.stdout %}
cAdvisor processes running:
{% for line in cadvisor_process.stdout_lines %}
  - {{ line }}
{% endfor %}
{% else %}
No cAdvisor processes found.
{% endif %}

=== Port Check ===
{% if cadvisor_ports.stdout %}
Possible cAdvisor ports:
{% for line in cadvisor_ports.stdout_lines %}
  - {{ line }}
{% endfor %}
{% else %}
No typical cAdvisor ports (8080, 8081, 4194) found.
{% endif %}
```

### roles/cadvisor_info/vars/main.yml

```yaml
---
# Порт по умолчанию для cAdvisor
cadvisor_default_port: 8080
```

## Использование роли

1. Создайте playbook (например, `gather_cadvisor_info.yml`):

```yaml
---
- name: Gather cAdvisor information from servers
  hosts: all
  become: yes
  roles:
    - cadvisor_info
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini gather_cadvisor_info.yml
```

3. После выполнения отчеты будут сохранены в `/tmp/cadvisor_report_<hostname>.txt` на управляющей машине.

## Дополнительные улучшения

1. Вы можете добавить сбор версии cAdvisor:
```yaml
- name: Get cAdvisor version (if running)
  shell: curl -s http://localhost:8080/metrics | grep 'cadvisor_version_info' || true
  register: cadvisor_version
  ignore_errors: yes
```

2. Для сбора информации в JSON формате можно изменить шаблон и добавить модуль `copy` для сохранения в JSON.

3. Можно добавить проверку конфигурационных файлов cAdvisor, если они существуют.