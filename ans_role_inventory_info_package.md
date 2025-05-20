# Ansible роль для сбора информации об установленном ПО на Oracle Linux 9

Вот пример Ansible роли, которая собирает информацию об установленных пакетах на Oracle Linux 9 (и других RHEL-подобных системах).

## Структура роли

```
roles/
  collect_software_info/
    tasks/
      main.yml
    defaults/
      main.yml
    templates/
      software_report.j2
```

## Файлы роли

### `roles/collect_software_info/defaults/main.yml`

```yaml
---
# Настройки по умолчанию
output_dir: "/tmp/software_reports"
report_filename: "installed_software_{{ ansible_hostname }}.txt"
gather_dnf_info: true
gather_rpm_info: true
gather_flatpak_info: false
gather_snap_info: false
```

### `roles/collect_software_info/tasks/main.yml`

```yaml
---
- name: Создать директорию для отчетов
  ansible.builtin.file:
    path: "{{ output_dir }}"
    state: directory
    mode: '0755'

- name: Собрать информацию об установленных пакетах (dnf)
  ansible.builtin.command: dnf list installed
  register: dnf_packages
  when: gather_dnf_info
  changed_when: false

- name: Собрать информацию об установленных пакетах (rpm)
  ansible.builtin.command: rpm -qa --queryformat '%{NAME}-%{VERSION}-%{RELEASE}.%{ARCH}\n'
  register: rpm_packages
  when: gather_rpm_info
  changed_when: false

- name: Собрать информацию о модулях
  ansible.builtin.command: dnf module list
  register: dnf_modules
  when: gather_dnf_info
  changed_when: false

- name: Собрать информацию о flatpak (если установлен)
  ansible.builtin.command: flatpak list
  register: flatpak_packages
  when: gather_flatpak_info
  changed_when: false
  ignore_errors: true

- name: Собрать информацию о snap (если установлен)
  ansible.builtin.command: snap list
  register: snap_packages
  when: gather_snap_info
  changed_when: false
  ignore_errors: true

- name: Создать отчет об установленном ПО
  ansible.builtin.template:
    src: software_report.j2
    dest: "{{ output_dir }}/{{ report_filename }}"
    mode: '0644'
```

### `roles/collect_software_info/templates/software_report.j2`

```jinja2
Отчет об установленном ПО на сервере {{ ansible_hostname }}
Дата создания: {{ ansible_date_time.iso8601 }}
Дистрибутив: {{ ansible_distribution }} {{ ansible_distribution_version }}
Ядро: {{ ansible_kernel }}

{% if gather_dnf_info %}
=== Установленные пакеты (dnf) ===
{{ dnf_packages.stdout }}

=== Установленные модули (dnf) ===
{{ dnf_modules.stdout }}
{% endif %}

{% if gather_rpm_info %}
=== Установленные пакеты (rpm) ===
{{ rpm_packages.stdout }}
{% endif %}

{% if gather_flatpak_info and flatpak_packages is success %}
=== Установленные Flatpak приложения ===
{{ flatpak_packages.stdout }}
{% endif %}

{% if gather_snap_info and snap_packages is success %}
=== Установленные Snap пакеты ===
{{ snap_packages.stdout }}
{% endif %}
```

## Пример playbook для использования роли

`collect_software.yml`:

```yaml
---
- name: Сбор информации об установленном ПО
  hosts: all
  become: yes
  gather_facts: yes

  roles:
    - role: collect_software_info
      vars:
        output_dir: "/var/log/software_reports"
        gather_flatpak_info: true
        gather_snap_info: true
```

## Использование

1. Разместите роль в директории `roles/`
2. Создайте playbook (например, `collect_software.yml`)
3. Запустите playbook:

```bash
ansible-playbook -i inventory_file collect_software.yml
```

## Дополнительные возможности

Вы можете расширить эту роль, добавив:
- Сбор информации о версиях Python пакетов (`pip list`)
- Сбор информации о Docker образах
- Сбор информации о версиях ядра и доступных обновлениях
- Отправку отчетов по email или сохранение в централизованное хранилище

Для Oracle Linux 9 эта роль должна работать без изменений, так как он использует тот же менеджер пакетов (dnf), что и RHEL 9.