# Ansible роль для сбора информации о системе

Эта роль собирает следующую информацию:
- Версия ОС и дистрибутива
- Установленные пакеты
- Запущенные сервисы и службы
- Информация о системе

## Структура роли

```
os_info_collector/
├── tasks/
│   └── main.yml
├── templates/
│   └── report.j2
└── defaults/
    └── main.yml
```

## Файлы роли

### defaults/main.yml
```yaml
---
# Настройки вывода отчета
report_filename: "system_report_{{ ansible_hostname }}_{{ ansible_date_time.date }}.txt"
report_path: "/tmp"
```

### tasks/main.yml
```yaml
---
- name: Gather system facts
  setup:
    gather_subset:
      - '!hardware'
      - '!virtual'
      - '!all'

- name: Get OS distribution info
  block:
    - name: Get OS info (RedHat)
      command: cat /etc/redhat-release
      register: os_info
      when: ansible_os_family == "RedHat"
      ignore_errors: yes

    - name: Get OS info (Debian)
      command: cat /etc/os-release
      register: os_info
      when: ansible_os_family == "Debian"
      ignore_errors: yes

    - name: Get OS info (SUSE)
      command: cat /etc/SuSE-release
      register: os_info
      when: ansible_os_family == "Suse"
      ignore_errors: yes

- name: Get installed packages (RPM)
  command: rpm -qa --queryformat '%{NAME}\t%{VERSION}\t%{RELEASE}\t%{INSTALLTIME:date}\n' | sort
  register: installed_packages_rpm
  when: ansible_pkg_mgr == "rpm"
  changed_when: false

- name: Get installed packages (APT)
  command: dpkg-query -W -f='${Package}\t${Version}\t${Status}\n' | sort
  register: installed_packages_deb
  when: ansible_pkg_mgr == "apt"
  changed_when: false

- name: Get running services (systemd)
  command: systemctl list-units --type=service --state=running --no-pager --no-legend
  register: running_services_systemd
  when: ansible_service_mgr == "systemd"
  changed_when: false

- name: Get running services (init.d)
  command: service --status-all | grep running
  register: running_services_initd
  when: ansible_service_mgr != "systemd"
  changed_when: false

- name: Generate system report
  template:
    src: report.j2
    dest: "{{ report_path }}/{{ report_filename }}"
  delegate_to: localhost
```

### templates/report.j2
```jinja2
System Information Report
=========================
Generated at: {{ ansible_date_time.iso8601 }}
Hostname: {{ ansible_hostname }}

OS Information
--------------
Distribution: {{ ansible_distribution }}
Version: {{ ansible_distribution_version }}
OS Family: {{ ansible_os_family }}
Kernel: {{ ansible_kernel }}
Architecture: {{ ansible_architecture }}

{% if os_info is defined and os_info.stdout %}
Additional OS Info:
{{ os_info.stdout }}
{% endif %}

Installed Packages
------------------
{% if installed_packages_rpm is defined and installed_packages_rpm.stdout %}
RPM Packages:
{{ installed_packages_rpm.stdout }}
{% endif %}

{% if installed_packages_deb is defined and installed_packages_deb.stdout %}
DEB Packages:
{{ installed_packages_deb.stdout }}
{% endif %}

Running Services
---------------
{% if running_services_systemd is defined and running_services_systemd.stdout %}
Systemd Services:
{{ running_services_systemd.stdout }}
{% endif %}

{% if running_services_initd is defined and running_services_initd.stdout %}
Init.d Services:
{{ running_services_initd.stdout }}
{% endif %}

System Facts
------------
Memory: {{ ansible_memtotal_mb }} MB
Processors: {{ ansible_processor_vcpus }}
IP Addresses: {{ ansible_all_ipv4_addresses | join(', ') }}
FQDN: {{ ansible_fqdn }}
```

## Использование роли

1. Создайте playbook для использования роли:

```yaml
- hosts: all
  become: yes
  roles:
    - os_info_collector
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini gather_info.yml
```

3. Отчеты будут сохранены в `/tmp` на управляющей машине с именами в формате `system_report_<hostname>_<date>.txt`

## Дополнительные возможности

Вы можете расширить эту роль, добавив сбор:
- Информации о дисках
- Сетевых интерфейсах
- Пользователях
- Открытых портах
- Конфигурации критичных сервисов

Для этого просто добавьте соответствующие задачи в `tasks/main.yml` и шаблоны вывода в `report.j2`.