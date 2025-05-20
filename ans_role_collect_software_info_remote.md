# Ansible роль для сбора информации об установленном ПО на Oracle Linux 9 с сохранением на локальный сервер

Вот расширенная версия роли, которая собирает информацию об установленных пакетах на Oracle Linux 9 и сохраняет результаты на локальный Ansible-сервер.

## Структура роли

```
roles/
  collect_software_info_remote/
    tasks/
      main.yml
    defaults/
      main.yml
    templates/
      software_report.j2
    files/
      parse_software.sh
```

## Файлы роли

### `roles/collect_software_info_remote/defaults/main.yml`

```yaml
---
# Настройки по умолчанию
remote_output_dir: "/tmp/software_reports"
local_output_dir: "./software_reports"
report_filename: "installed_software_{{ ansible_hostname }}_{{ ansible_date_time.date }}.json"
report_format: "json"  # json или text

# Что собирать
gather_dnf_info: true
gather_rpm_info: true
gather_module_info: true
gather_flatpak_info: false
gather_snap_info: false
gather_pip_info: false
gather_docker_info: false

# Очистка
keep_local_copy: true
remove_remote_files: true
```

### `roles/collect_software_info_remote/tasks/main.yml`

```yaml
---
- name: Создать директорию для отчетов на удаленном сервере
  ansible.builtin.file:
    path: "{{ remote_output_dir }}"
    state: directory
    mode: '0755'

- name: Установить jq для обработки JSON (если нужно)
  ansible.builtin.package:
    name: jq
    state: present
  when: report_format == "json"

- name: Собрать базовую информацию о системе
  ansible.builtin.setup:
    gather_subset:
      - '!all'
      - '!min'
      - distribution
      - kernel
      - hostname

- name: Собрать информацию об установленных пакетах (dnf)
  ansible.builtin.command: dnf list installed --quiet
  register: dnf_packages
  when: gather_dnf_info
  changed_when: false

- name: Собрать детальную информацию о пакетах (rpm)
  ansible.builtin.command: rpm -qa --queryformat '{"name":"%{NAME}","version":"%{VERSION}","release":"%{RELEASE}","arch":"%{ARCH}","installtime":"%{INSTALLTIME:date}","vendor":"%{VENDOR}","size":"%{SIZE}","summary":"%{SUMMARY}"}\n'
  register: rpm_packages_details
  when: gather_rpm_info
  changed_when: false

- name: Собрать информацию о модулях
  ansible.builtin.command: dnf module list --quiet
  register: dnf_modules
  when: gather_module_info
  changed_when: false

- name: Собрать информацию о flatpak
  ansible.builtin.command: flatpak list --columns=application,version,branch,installation
  register: flatpak_packages
  when: gather_flatpak_info
  changed_when: false
  ignore_errors: true

- name: Собрать информацию о snap
  ansible.builtin.command: snap list --all
  register: snap_packages
  when: gather_snap_info
  changed_when: false
  ignore_errors: true

- name: Собрать информацию о pip пакетах
  ansible.builtin.command: pip3 list --format=json
  register: pip_packages
  when: gather_pip_info
  changed_when: false
  ignore_errors: true

- name: Собрать информацию о Docker образах
  ansible.builtin.command: docker images --format '{{ '{{.Repository}} {{.Tag}} {{.ID}} {{.CreatedSince}} {{.Size}}' }}'
  register: docker_images
  when: gather_docker_info
  changed_when: false
  ignore_errors: true

- name: Создать JSON отчет на удаленном сервере
  ansible.builtin.template:
    src: software_report.j2
    dest: "{{ remote_output_dir }}/{{ report_filename }}"
    mode: '0644'

- name: Создать директорию для отчетов на локальном сервере
  ansible.builtin.file:
    path: "{{ local_output_dir }}"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true

- name: Забрать отчет с удаленного сервера
  ansible.builtin.fetch:
    src: "{{ remote_output_dir }}/{{ report_filename }}"
    dest: "{{ local_output_dir }}/{{ inventory_hostname }}/"
    flat: no
    validate_checksum: yes

- name: Удалить временные файлы с удаленного сервера
  ansible.builtin.file:
    path: "{{ remote_output_dir }}/{{ report_filename }}"
    state: absent
  when: remove_remote_files
```

### `roles/collect_software_info_remote/templates/software_report.j2`

```jinja2
{% if report_format == "json" %}
{
  "metadata": {
    "hostname": "{{ ansible_hostname }}",
    "distribution": "{{ ansible_distribution }}",
    "version": "{{ ansible_distribution_version }}",
    "kernel": "{{ ansible_kernel }}",
    "report_date": "{{ ansible_date_time.iso8601 }}"
  },
  "packages": {
    {% if gather_dnf_info %}
    "dnf_packages": {{ dnf_packages.stdout_lines | to_json }},
    {% endif %}
    {% if gather_rpm_info %}
    "rpm_packages": {{ rpm_packages_details.stdout_lines | map('from_json') | list | to_json }},
    {% endif %}
    {% if gather_module_info %}
    "dnf_modules": {{ dnf_modules.stdout_lines | to_json }},
    {% endif %}
    {% if gather_flatpak_info and flatpak_packages is success %}
    "flatpak_packages": {{ flatpak_packages.stdout_lines | to_json }},
    {% endif %}
    {% if gather_snap_info and snap_packages is success %}
    "snap_packages": {{ snap_packages.stdout_lines | to_json }},
    {% endif %}
    {% if gather_pip_info and pip_packages is success %}
    "pip_packages": {{ pip_packages.stdout | from_json | to_json }},
    {% endif %}
    {% if gather_docker_info and docker_images is success %}
    "docker_images": {{ docker_images.stdout_lines | to_json }}
    {% endif %}
  }
}
{% else %}
Отчет об установленном ПО на сервере {{ ansible_hostname }}
Дата создания: {{ ansible_date_time.iso8601 }}
Дистрибутив: {{ ansible_distribution }} {{ ansible_distribution_version }}
Ядро: {{ ansible_kernel }}

{% if gather_dnf_info %}
=== Установленные пакеты (dnf) ===
{{ dnf_packages.stdout }}

{% endif %}
{% if gather_rpm_info %}
=== Установленные пакеты (rpm) ===
{% for pkg in rpm_packages_details.stdout_lines %}
{{ pkg }}
{% endfor %}

{% endif %}
{% if gather_module_info %}
=== Установленные модули (dnf) ===
{{ dnf_modules.stdout }}

{% endif %}
{% if gather_flatpak_info and flatpak_packages is success %}
=== Установленные Flatpak приложения ===
{{ flatpak_packages.stdout }}

{% endif %}
{% if gather_snap_info and snap_packages is success %}
=== Установленные Snap пакеты ===
{{ snap_packages.stdout }}

{% endif %}
{% if gather_pip_info and pip_packages is success %}
=== Установленные Python пакеты ===
{{ pip_packages.stdout }}

{% endif %}
{% if gather_docker_info and docker_images is success %}
=== Docker образы ===
{{ docker_images.stdout }}
{% endif %}
{% endif %}
```

## Пример playbook для использования роли

`collect_software_remote.yml`:

```yaml
---
- name: Сбор информации об установленном ПО с сохранением локально
  hosts: oracle_linux_servers
  become: yes
  gather_facts: yes

  vars:
    # Переопределение настроек по умолчанию
    local_output_dir: "/opt/ansible/software_reports"
    report_format: "json"
    gather_pip_info: true
    gather_docker_info: true

  roles:
    - role: collect_software_info_remote
```

## Использование

1. Разместите роль в директории `roles/`
2. Создайте playbook (например, `collect_software_remote.yml`)
3. Запустите playbook:

```bash
ansible-playbook -i inventory_file collect_software_remote.yml
```

## Дополнительные возможности

1. **Обработка отчетов**: Добавьте задачу для обработки собранных JSON файлов и создания сводного отчета.

2. **Уведомления**: Добавьте отправку email с результатами или уведомление в чат (Slack, Telegram).

3. **Хранение в БД**: Модифицируйте роль для сохранения данных в базу данных (например, PostgreSQL или Elasticsearch).

4. **Планирование**: Настройте регулярный запуск через cron для мониторинга изменений в установленном ПО.

5. **Визуализация**: Создайте скрипт для генерации HTML отчета на основе собранных данных.

Эта роль обеспечивает сбор детальной информации об установленном ПО в формате JSON или текстовом формате с сохранением результатов на локальном сервере Ansible.