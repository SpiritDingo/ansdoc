Вот пример Ansible роли для сбора информации о серверах и генерации CSV отчета.

Структура роли

```
roles/server-info/
├── tasks
│   └── main.yml
├── defaults
│   └── main.yml
├── templates
│   └── report.csv.j2
└── meta
    └── main.yml
```

defaults/main.yml

```yaml
---
# Директория для сохранения отчета
report_dir: "/tmp/server-reports"
# Имя файла отчета
report_filename: "server_info.csv"
# Фильтр для программного обеспечения (regex)
software_filter: ".*"
# Список пакетов для проверки (если пустой - проверяются все)
software_packages: []
```

tasks/main.yml

```yaml
---
- name: Create report directory
  ansible.builtin.file:
    path: "{{ report_dir }}"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true

- name: Gather system facts
  ansible.builtin.setup:
    gather_subset:
      - hardware
      - network
      - virtual
      - distribution

- name: Get disk information
  ansible.builtin.shell:
    cmd: df -h | grep -v tmpfs | tail -n +2
  register: disk_info
  changed_when: false

- name: Get installed software
  block:
    - name: Get all installed packages (RedHat/CentOS)
      ansible.builtin.package_facts:
        manager: auto
      when: ansible_os_family == "RedHat"
    
    - name: Get all installed packages (Debian/Ubuntu)
      ansible.builtin.package_facts:
        manager: auto
      when: ansible_os_family == "Debian"
    
    - name: Filter software packages
      ansible.builtin.set_fact:
        filtered_software: |
          {% set result = [] %}
          {% if software_packages and software_packages|length > 0 %}
            {% for pkg in software_packages %}
              {% if pkg in ansible_facts.packages %}
                {% for item in ansible_facts.packages[pkg] %}
                  {% set _ = result.append(pkg ~ ":" ~ item.version) %}
                {% endfor %}
              {% endif %}
            {% endfor %}
          {% else %}
            {% for pkg_name, pkg_list in ansible_facts.packages.items() %}
              {% if pkg_name | regex_search(software_filter) %}
                {% for pkg in pkg_list %}
                  {% set _ = result.append(pkg_name ~ ":" ~ pkg.version) %}
                {% endfor %}
              {% endif %}
            {% endfor %}
          {% endif %}
          {{ result | join(';') }}
  when: ansible_facts.packages is defined

- name: Set empty software list if not defined
  ansible.builtin.set_fact:
    filtered_software: "No package information"
  when: ansible_facts.packages is not defined

- name: Generate CSV report
  ansible.builtin.template:
    src: report.csv.j2
    dest: "{{ report_dir }}/{{ report_filename }}"
    mode: '0644'
  delegate_to: localhost
  run_once: false
```

templates/report.csv.j2

```jinja2
Hostname,OS,OS Version,CPU Model,CPU Cores,Total Memory (MB),Disk Info,Installed Software
{{ ansible_hostname }},{{ ansible_distribution }},{{ ansible_distribution_version }},"{{ ansible_processor[1] if ansible_processor[1] is defined else ansible_processor[0] }}",{{ ansible_processor_cores }},{{ ansible_memtotal_mb }},"{% for line in disk_info.stdout_lines %}{{ line }}{% if not loop.last %}; {% endif %}{% endfor %}",{{ filtered_software | replace('"', '""') }}
```

Playbook для использования роли

Создайте playbook gather_server_info.yml:

```yaml
---
- name: Gather server information and generate report
  hosts: all
  become: yes
  roles:
    - server-info
  vars:
    software_filter: "(python|java|docker|nginx|apache|mysql|postgresql)"
    # или конкретный список пакетов:
    # software_packages: ["python3", "docker", "nginx"]
```

Использование

1. Создайте inventory файл с вашими серверами:

```ini
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com
```

1. Запустите playbook:

```bash
ansible-playbook -i inventory.ini gather_server_info.yml
```

1. Отчеты будут созданы на control node в директории /tmp/server-reports/ с именами server_info_<hostname>.csv

Пример содержимого CSV файла

```csv
Hostname,OS,OS Version,CPU Model,CPU Cores,Total Memory (MB),Disk Info,Installed Software
web1,Ubuntu,20.04,Intel(R) Xeon(R) CPU E5-2680 v4,4,8192,/dev/sda1 50G 20G 29G 42% /; /dev/sda2 100G 30G 65G 32% /data,python3:3.8.5;docker:20.10.7;nginx:1.18.0
```

Дополнительные улучшения

Если хотите собрать более детальную информацию, можете добавить следующие задачи:

```yaml
- name: Get detailed CPU info
  ansible.builtin.shell:
    cmd: |
      lscpu | grep -E "(Architecture|CPU\(s\)|Thread\(s\) per core|Core\(s\) per socket|Model name|CPU MHz|Virtualization)"
  register: cpu_details
  changed_when: false
  when: ansible_os_family != "Windows"

- name: Get network interfaces
  ansible.builtin.shell:
    cmd: ip -o addr show | grep -v "127.0.0.1" | awk '{print $2 ": " $4}'
  register: network_info
  changed_when: false
```

Эта роль предоставляет гибкую систему для сбора информации о серверах и может быть легко расширена для сбора дополнительных данных.