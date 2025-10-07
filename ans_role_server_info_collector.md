Вот пример Ansible-роли для сбора информации о серверах и вывода отчёта в CSV-файл.

---

## 📁 Структура роли

```
roles/
└── server_info_collector/
    ├── tasks/
    │   └── main.yml
    ├── defaults/
    │   └── main.yml
    └── templates/
        └── report.csv.j2
```

---

## 📄 `defaults/main.yml`

```yaml
---
# Путь к файлу отчёта на локальной машине (control node)
server_info_report_path: "{{ playbook_dir }}/reports/server_info_report.csv"

# Список пакетов, наличие которых нужно проверить
server_info_software_filter:
  - nginx
  - docker
  - postgresql
  - python3
```

---

## 📄 `tasks/main.yml`

```yaml
---
- name: Gather facts (if not already gathered)
  setup:

- name: Get disk usage info
  shell: df -h --output=target,size | tail -n +2
  register: disk_info
  changed_when: false

- name: Format disk info as list of dicts
  set_fact:
    disk_mounts: "{{ disk_mounts | default([]) + [{'mount': item.split()[-1], 'size': item.split()[0]}] }}"
  loop: "{{ disk_info.stdout_lines }}"

- name: Check if software packages are installed
  package_facts:
    manager: auto

- name: Determine which filtered software is installed
  set_fact:
    installed_software: "{{ server_info_software_filter | select('in', ansible_facts.packages.keys()) | list }}"

- name: Ensure report directory exists (on control node)
  delegate_to: localhost
  run_once: true
  file:
    path: "{{ server_info_report_path | dirname }}"
    state: directory
    mode: '0755'

- name: Render and write CSV report (on control node)
  delegate_to: localhost
  run_once: true
  template:
    src: report.csv.j2
    dest: "{{ server_info_report_path }}"
    mode: '0644'
```

---

## 📄 `templates/report.csv.j2`

```jinja2
{%- if ansible_play_hosts_all | length == 1 -%}
hostname,os,cpu_cores,total_memory_mb,disk_mounts,installed_software
{%- endif %}
"{{ inventory_hostname }}",\
"{{ ansible_distribution }} {{ ansible_distribution_version }}",\
"{{ ansible_processor_vcpus | default(ansible_processor | length) }}",\
"{{ (ansible_memtotal_mb | int) }}",\
"{% for d in hostvars[inventory_hostname].disk_mounts | default([]) %}{{ d.mount }}({{ d.size }}){% if not loop.last %}; {% endif %}{% endfor %}",\
"{% for pkg in hostvars[inventory_hostname].installed_software | default([]) %}{{ pkg }}{% if not loop.last %}, {% endif %}{% endfor %}"
```

> 💡 **Примечание**: шаблон использует `run_once` и `delegate_to: localhost`, поэтому он собирает данные со всех хостов и записывает их в один CSV-файл. Чтобы это работало корректно, шаблон должен быть вызван **после** сбора данных со всех хостов. Для этого лучше использовать отдельный playbook с двумя plays.

---

## ✅ Рекомендуемый playbook

Создайте `playbook.yml`:

```yaml
---
- name: Gather server info
  hosts: all
  roles:
    - server_info_collector

- name: Generate final CSV report
  hosts: localhost
  gather_facts: no
  tasks:
    - name: Render final CSV with all hosts
      template:
        src: roles/server_info_collector/templates/report.csv.j2
        dest: "{{ server_info_report_path }}"
      vars:
        # Передаём все хосты из предыдущей игры
        ansible_play_hosts_all: "{{ groups['all'] }}"
```

> Но проще — немного изменить роль, чтобы она **накапливала данные**, а не перезаписывала файл каждый раз. Однако стандартный способ в Ansible — использовать `run_once` и шаблон с циклом по `hostvars`.

---

## 🔁 Альтернатива: один проход с накоплением данных

Изменим задачу в `tasks/main.yml`:

```yaml
- name: Collect host data into a list
  delegate_to: localhost
  run_once: true
  set_fact:
    all_host_data: "{{ all_host_data | default([]) + [host_data] }}"
  vars:
    host_data:
      hostname: "{{ inventory_hostname }}"
      os: "{{ ansible_distribution }} {{ ansible_distribution_version }}"
      cpu_cores: "{{ ansible_processor_vcpus | default(ansible_processor | length) }}"
      total_memory_mb: "{{ ansible_memtotal_mb }}"
      disk_mounts: "{{ disk_mounts | default([]) }}"
      installed_software: "{{ installed_software | default([]) }}"

- name: Write final CSV report
  delegate_to: localhost
  run_once: true
  template:
    src: report_full.csv.j2
    dest: "{{ server_info_report_path }}"
  when: play_hosts[-1] == inventory_hostname  # только на последнем хосте
```

И создадим новый шаблон `templates/report_full.csv.j2`:

```jinja2
hostname,os,cpu_cores,total_memory_mb,disk_mounts,installed_software
{% for host in all_host_data | default([]) %}
"{{ host.hostname }}",\
"{{ host.os }}",\
"{{ host.cpu_cores }}",\
"{{ host.total_memory_mb }}",\
"{% for d in host.disk_mounts %}{{ d.mount }}({{ d.size }}){% if not loop.last %}; {% endif %}{% endfor %}",\
"{% for pkg in host.installed_software %}{{ pkg }}{% if not loop.last %}, {% endif %}{% endfor %}"
{%- endfor %}
```

Этот подход более надёжен для многопоточного выполнения.

---

## 🧪 Запуск

```bash
ansible-playbook -i inventory.ini playbook.yml
```

Результат будет в `./reports/server_info_report.csv`.

---

## 📌 Поддерживаемые ОС

Роль использует модуль `package_facts` с `manager: auto`, поэтому работает на:
- Ubuntu/Debian (`apt`)
- CentOS/RHEL/Alma/Rocky (`yum`/`dnf`)
- Fedora (`dnf`)
- и других, поддерживаемых Ansible.

---

Если нужно — могу упаковать всё в архив или предоставить в формате `ansible-galaxy init`.