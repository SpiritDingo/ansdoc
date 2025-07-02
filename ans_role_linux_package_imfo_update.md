# Универсальная Ansible роль для сбора информации о пакетах

Эта роль собирает информацию об установленных пакетах как для Ubuntu/Debian (deb), так и для Oracle Linux 9 (rpm) с возможностью фильтрации по конкретному пакету.

## Структура роли

```
roles/
└── package_info/
    ├── defaults/
    │   └── main.yml
    ├── tasks/
    │   ├── debian.yml
    │   ├── oracle_linux.yml
    │   └── main.yml
    └── templates/
        └── package_report.j2
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Имя пакета для фильтрации (оставьте пустым для всех пакетов)
package_name: ""

# Формат вывода (json, yaml, human)
output_format: "human"

# Файл для сохранения отчета (необязательно)
output_file: ""
```

### tasks/main.yml

```yaml
---
- name: Detect OS family
  ansible.builtin.set_fact:
    os_family: "{{ ansible_facts['os_family'] }}"

- name: Include OS-specific tasks
  ansible.builtin.include_tasks: "{{ os_family | lower }}.yml"
  when: os_family in ['Debian', 'RedHat']

- name: Generate report
  ansible.builtin.template:
    src: package_report.j2
    dest: "/tmp/package_report.txt"
  when: output_format == 'human' and output_file == ""

- name: Save report to custom file
  ansible.builtin.template:
    src: package_report.j2
    dest: "{{ output_file }}"
  when: output_format == 'human' and output_file != ""

- name: Display JSON report
  ansible.builtin.debug:
    var: packages_info
  when: output_format == 'json'

- name: Display YAML report
  ansible.builtin.debug:
    var: packages_info
  when: output_format == 'yaml'
```

### tasks/debian.yml

```yaml
---
- name: Get installed packages (Debian/Ubuntu)
  ansible.builtin.shell: |
    if [ -z "{{ package_name }}" ]; then
      dpkg-query -W -f='${Package}\t${Version}\t${Architecture}\n'
    else
      dpkg-query -W -f='${Package}\t${Version}\t${Architecture}\n' "{{ package_name }}"
    fi
  register: deb_packages
  changed_when: false

- name: Parse Debian packages info
  ansible.builtin.set_fact:
    packages_info: |
      {% set packages = [] %}
      {% for line in deb_packages.stdout_lines %}
        {% set parts = line.split('\t') %}
        {% if parts | length >= 2 %}
          {% set package = {
            'name': parts[0],
            'version': parts[1],
            'architecture': parts[2] if parts | length > 2 else '',
            'manager': 'dpkg'
          } %}
          {% set _ = packages.append(package) %}
        {% endif %}
      {% endfor %}
      {{ packages }}
```

### tasks/oracle_linux.yml

```yaml
---
- name: Get installed packages (Oracle Linux 9)
  ansible.builtin.shell: |
    if [ -z "{{ package_name }}" ]; then
      rpm -qa --queryformat='%{NAME}\t%{VERSION}-%{RELEASE}\t%{ARCH}\n'
    else
      rpm -qa --queryformat='%{NAME}\t%{VERSION}-%{RELEASE}\t%{ARCH}\n' "*{{ package_name }}*"
    fi
  register: rpm_packages
  changed_when: false

- name: Parse RPM packages info
  ansible.builtin.set_fact:
    packages_info: |
      {% set packages = [] %}
      {% for line in rpm_packages.stdout_lines %}
        {% set parts = line.split('\t') %}
        {% if parts | length >= 2 %}
          {% set package = {
            'name': parts[0],
            'version': parts[1],
            'architecture': parts[2] if parts | length > 2 else '',
            'manager': 'rpm'
          } %}
          {% set _ = packages.append(package) %}
        {% endif %}
      {% endfor %}
      {{ packages }}
```

### templates/package_report.j2

```jinja2
Package Information Report
Generated on: {{ ansible_date_time.iso8601 }}
{% if package_name %}Filtered by package: {{ package_name }}{% endif %}

{% for package in packages_info %}
Package: {{ package.name }}
Version: {{ package.version }}
Architecture: {{ package.architecture }}
Package Manager: {{ package.manager }}
----------------------------------------
{% endfor %}

Total packages: {{ packages_info | length }}
```

## Примеры использования

1. Собрать информацию о всех пакетах:

```yaml
- hosts: all
  roles:
    - role: package_info
```

2. Собрать информацию о конкретном пакете (например, python):

```yaml
- hosts: all
  roles:
    - role: package_info
      vars:
        package_name: "python"
```

3. Получить вывод в JSON:

```yaml
- hosts: all
  roles:
    - role: package_info
      vars:
        output_format: "json"
```

4. Сохранить отчет в файл:

```yaml
- hosts: all
  roles:
    - role: package_info
      vars:
        output_file: "/var/log/package_report.txt"
```

## Особенности

1. Роля автоматически определяет ОС и использует соответствующий менеджер пакетов.
2. Поддерживает фильтрацию по имени пакета.
3. Предоставляет несколько форматов вывода (human-readable, JSON, YAML).
4. Может сохранять отчет в файл.

Для работы с другими дистрибутивами на основе RPM или DEB можно расширить список поддерживаемых OS families в условии include_tasks.


# Универсальная Ansible роль для управления пакетами (информация + обновление)

Расширенная роль, которая не только собирает информацию о пакетах, но и позволяет обновлять их до указанной версии.

## Обновленная структура роли

```
roles/
└── package_mgmt/
    ├── defaults/
    │   └── main.yml
    ├── tasks/
    │   ├── debian.yml
    │   ├── oracle_linux.yml
    │   ├── update.yml
    │   └── main.yml
    └── templates/
        └── package_report.j2
```

## Основные изменения

### defaults/main.yml

```yaml
---
# Режим работы: info (только информация) или update (обновление)
mode: "info"

# Имя пакета для работы
package_name: ""

# Для mode=info - формат вывода (json, yaml, human)
output_format: "human"

# Для mode=info - файл для сохранения отчета
output_file: ""

# Для mode=update - целевая версия пакета
target_version: ""

# Для mode=update - разрешить понижение версии
allow_downgrade: false
```

### tasks/main.yml

```yaml
---
- name: Validate parameters
  ansible.builtin.assert:
    that:
      - mode in ['info', 'update']
      - package_name != "" or mode == 'info'
      - (target_version != "" and mode == 'update') or mode == 'info'
    fail_msg: "Invalid parameters combination"

- name: Detect OS family
  ansible.builtin.set_fact:
    os_family: "{{ ansible_facts['os_family'] }}"

- name: Include OS-specific tasks
  ansible.builtin.include_tasks: "{{ os_family | lower }}.yml"
  when: os_family in ['Debian', 'RedHat']

- name: Include update tasks if needed
  ansible.builtin.include_tasks: update.yml
  when: mode == 'update'

- name: Generate report
  ansible.builtin.template:
    src: package_report.j2
    dest: "/tmp/package_report.txt"
  when: mode == 'info' and output_format == 'human' and output_file == ""

- name: Save report to custom file
  ansible.builtin.template:
    src: package_report.j2
    dest: "{{ output_file }}"
  when: mode == 'info' and output_format == 'human' and output_file != ""

- name: Display JSON report
  ansible.builtin.debug:
    var: packages_info
  when: mode == 'info' and output_format == 'json'

- name: Display YAML report
  ansible.builtin.debug:
    var: packages_info
  when: mode == 'info' and output_format == 'yaml'
```

### tasks/update.yml

```yaml
---
- name: Check if package is installed
  ansible.builtin.assert:
    that:
      - packages_info | length > 0
    fail_msg: "Package {{ package_name }} is not installed"

- name: Check current version vs target version
  ansible.builtin.set_fact:
    version_compare: "{{ (packages_info[0].version.split('-')[0] == target_version.split('-')[0]) | ternary('same', (packages_info[0].version.split('-')[0] | version_compare(target_version.split('-')[0]))) }}"

- name: Validate version update
  ansible.builtin.assert:
    that:
      - version_compare != 'same' or allow_downgrade
    fail_msg: |
      Package {{ package_name }} is already at version {{ packages_info[0].version }}.
      Set allow_downgrade=true if you want to downgrade.

- name: Update package (Debian)
  ansible.builtin.apt:
    name: "{{ package_name }}={{ target_version }}"
    state: present
    allow_downgrade: "{{ allow_downgrade }}"
  when: os_family == 'Debian'
  register: update_result

- name: Update package (Oracle Linux)
  ansible.builtin.yum:
    name: "{{ package_name }}-{{ target_version }}"
    state: present
    allow_downgrade: "{{ allow_downgrade }}"
  when: os_family == 'RedHat'
  register: update_result

- name: Verify updated package
  ansible.builtin.include_tasks: "{{ os_family | lower }}.yml"
  when: update_result is changed

- name: Show update result
  ansible.builtin.debug:
    msg: |
      Package {{ package_name }} updated successfully.
      Old version: {{ packages_info[0].version }}
      New version: {{ packages_info[0].version if not update_result is changed else (packages_info_new | default(packages_info))[0].version }}
```

### Обновленные OS-специфичные задачи

Для `debian.yml` и `oracle_linux.yml` нужно добавить регистрацию результатов в разные переменные при повторном запуске:

```yaml
# В конце debian.yml/oracle_linux.yml добавить:
- name: Store package info (for update verification)
  ansible.builtin.set_fact:
    packges_info_new: "{{ packages_info }}"
  when: mode == 'update' and update_result is defined and update_result is changed
```

## Примеры использования

1. Получить информацию о пакете python:

```yaml
- hosts: all
  roles:
    - role: package_mgmt
      vars:
        package_name: "python"
        mode: "info"
        output_format: "human"
```

2. Обновить пакет python до версии 3.8.10:

```yaml
- hosts: all
  roles:
    - role: package_mgmt
      vars:
        package_name: "python"
        mode: "update"
        target_version: "3.8.10"
```

3. Обновить пакет с возможностью понижения версии:

```yaml
- hosts: all
  roles:
    - role: package_mgmt
      vars:
        package_name: "nginx"
        mode: "update"
        target_version: "1.18.0"
        allow_downgrade: true
```

4. Получить JSON информацию о всех пакетах:

```yaml
- hosts: all
  roles:
    - role: package_mgmt
      vars:
        mode: "info"
        output_format: "json"
```

## Особенности реализации

1. **Безопасность**: Проверка текущей версии перед обновлением
2. **Гибкость**: Поддержка разных форматов вывода
3. **Контроль**: Возможность запрета понижения версий
4. **Валидация**: Проверка параметров перед выполнением
5. **Подтверждение**: Проверка результата после обновления

Для работы с другими дистрибутивами можно добавить соответствующие task-файлы и расширить логику в update.yml.