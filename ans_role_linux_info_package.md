Вот обновленная Ansible роль с фильтрацией пакетов и поддержкой CSV и JSON форматов:

Обновленная структура роли

```
package-info-collector/
├── tasks/
│   ├── main.yml
│   └── get_specific_packages.yml
├── templates/
│   ├── report_csv.j2
│   └── report_json.j2
├── vars/
│   └── main.yml
└── files/
    └── merge_reports.py
```

tasks/main.yml (обновленный)

```yaml
---
- name: Сбор информации о системе
  setup:
    filter:
      - ansible_distribution
      - ansible_distribution_version
      - ansible_os_family
      - ansible_hostname
      - ansible_date_time
  register: system_info

- name: Определение менеджера пакетов
  set_fact:
    package_manager: >-
      {%- if ansible_os_family == "Debian" -%}apt
      {%- elif ansible_os_family == "RedHat" -%}rpm
      {%- else -%}unknown
      {%- endif -%}

- name: Определение формата отчета
  set_fact:
    report_format: "{{ report_format | default('csv') }}"
    report_extension: "{{ 'csv' if report_format == 'csv' else 'json' }}"

- name: Получение списка всех установленных пакетов (Debian/Ubuntu)
  shell: |
    dpkg-query -W -f='${Package}\t${Version}\t${Architecture}\t${Installed-Size}\t${Description}\n' | \
    grep "install ok installed" | cut -f1-5
  register: deb_packages
  when: ansible_os_family == "Debian"
  changed_when: false

- name: Получение списка всех установленных пакетов (RedHat/Oracle Linux)
  shell: |
    rpm -qa --queryformat "%{NAME}\t%{VERSION}-%{RELEASE}\t%{ARCH}\t%{SIZE}\t%{SUMMARY}\n" | sort
  register: rh_packages
  when: ansible_os_family == "RedHat"
  changed_when: false

- name: Фильтрация пакетов по имени
  set_fact:
    filtered_packages: >-
      {%- if package_filter is defined and package_filter | length > 0 -%}
        {%- set filtered = [] -%}
        {%- if ansible_os_family == "Debian" -%}
          {%- for line in deb_packages.stdout_lines -%}
            {%- set parts = line.split('\t') -%}
            {%- for filter_pattern in package_filter -%}
              {%- if parts[0] | regex_search(filter_pattern) -%}
                {%- set _ = filtered.append(line) -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endfor -%}
        {%- elif ansible_os_family == "RedHat" -%}
          {%- for line in rh_packages.stdout_lines -%}
            {%- set parts = line.split('\t') -%}
            {%- for filter_pattern in package_filter -%}
              {%- if parts[0] | regex_search(filter_pattern) -%}
                {%- set _ = filtered.append(line) -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endfor -%}
        {%- endif -%}
        {{ filtered }}
      {%- else -%}
        {{ deb_packages.stdout_lines if ansible_os_family == "Debian" else rh_packages.stdout_lines }}
      {%- endif -%}

- name: Подсчет отфильтрованных пакетов
  set_fact:
    total_filtered_packages: "{{ filtered_packages | length }}"

- name: Создание структурированных данных о пакетах (CSV формат)
  set_fact:
    package_data_csv: >-
      {%- set output = [] -%}
      {%- for line in filtered_packages -%}
        {%- set parts = line.split('\t') -%}
        {%- if ansible_os_family == "Debian" -%}
          {%- set _ = output.append(
            '"' + parts[0] + '","' + 
            parts[1] + '","' + 
            (parts[2] if parts|length > 2 else 'unknown') + '","' + 
            (parts[3] if parts|length > 3 else '0') + '","' + 
            (parts[4]|replace('"', '""') if parts|length > 4 else '') + '"'
          ) -%}
        {%- else -%}
          {%- set _ = output.append(
            '"' + parts[0] + '","' + 
            parts[1] + '","' + 
            (parts[2] if parts|length > 2 else 'unknown') + '","' + 
            (parts[3] if parts|length > 3 else '0') + '","' + 
            (parts[4]|replace('"', '""') if parts|length > 4 else '') + '"'
          ) -%}
        {%- endif -%}
      {%- endfor -%}
      {{ output }}

- name: Создание структурированных данных о пакетах (JSON формат)
  set_fact:
    package_data_json: >-
      {%- set packages = [] -%}
      {%- for line in filtered_packages -%}
        {%- set parts = line.split('\t') -%}
        {%- if ansible_os_family == "Debian" -%}
          {%- set _ = packages.append({
            'name': parts[0],
            'version': parts[1],
            'architecture': parts[2] if parts|length > 2 else 'unknown',
            'size': parts[3] if parts|length > 3 else '0',
            'description': parts[4] if parts|length > 4 else ''
          }) -%}
        {%- else -%}
          {%- set _ = packages.append({
            'name': parts[0],
            'version': parts[1],
            'architecture': parts[2] if parts|length > 2 else 'unknown',
            'size': parts[3] if parts|length > 3 else '0',
            'description': parts[4] if parts|length > 4 else ''
          }) -%}
        {%- endif -%}
      {%- endfor -%}
      {{ packages | to_json }}

- name: Получение информации о конкретных пакетах
  include_tasks: get_specific_packages.yml
  when: specific_packages is defined

- name: Сохранение отчета на целевой машине (CSV)
  template:
    src: report_csv.j2
    dest: "/tmp/package_report_{{ ansible_hostname }}.{{ report_extension }}"
    mode: '0644'
  when: report_format == 'csv'
  delegate_to: "{{ inventory_hostname }}"

- name: Сохранение отчета на целевой машине (JSON)
  template:
    src: report_json.j2
    dest: "/tmp/package_report_{{ ansible_hostname }}.{{ report_extension }}"
    mode: '0644'
  when: report_format == 'json'
  delegate_to: "{{ inventory_hostname }}"

- name: Сбор всех отчетов на контрольной машине
  fetch:
    src: "/tmp/package_report_{{ ansible_hostname }}.{{ report_extension }}"
    dest: "{{ report_directory }}/"
    flat: yes
  run_once: true
  delegate_to: "{{ inventory_hostname }}"

- name: Создание сводного отчета
  template:
    src: "summary_report_{{ report_format }}.j2"
    dest: "{{ report_directory }}/summary_report.{{ report_extension }}"
  delegate_to: localhost
  run_once: true

- name: Копирование скрипта для объединения JSON отчетов
  copy:
    src: merge_reports.py
    dest: "{{ report_directory }}/merge_reports.py"
    mode: '0755'
  delegate_to: localhost
  run_once: true
  when: report_format == 'json'

- name: Объединение всех JSON отчетов в один файл
  shell: |
    cd {{ report_directory }}
    python3 merge_reports.py
  delegate_to: localhost
  run_once: true
  when: report_format == 'json'
```

templates/report_csv.j2

```jinja2
# Отчет об установленных пакетах
# Хост: {{ ansible_hostname }}
# Дата: {{ ansible_date_time.date }} {{ ansible_date_time.time }}
# ОС: {{ ansible_distribution }} {{ ansible_distribution_version }}
# Всего пакетов (с учетом фильтра): {{ total_filtered_packages }}
# Фильтр: {{ package_filter | default('Нет фильтра') }}
"Package Name","Version","Architecture","Size (KB)","Description"
{% for line in package_data_csv -%}
{{ line }}
{% endfor %}
```

templates/report_json.j2

```jinja2
{
  "metadata": {
    "hostname": "{{ ansible_hostname }}",
    "timestamp": "{{ ansible_date_time.iso8601 }}",
    "os": {
      "distribution": "{{ ansible_distribution }}",
      "version": "{{ ansible_distribution_version }}",
      "family": "{{ ansible_os_family }}"
    },
    "package_manager": "{{ package_manager }}",
    "total_packages": {{ total_filtered_packages }},
    "filter_applied": {{ 'true' if package_filter is defined else 'false' }},
    "filter_patterns": {{ package_filter | default([]) | to_json }}
  },
  "packages": {{ package_data_json | from_json | to_nice_json(indent=2) }}
}
```

templates/summary_report_csv.j2

```jinja2
# СВОДНЫЙ ОТЧЕТ ПО ВСЕМ ХОСТАМ
# Дата создания: {{ ansible_date_time.date }} {{ ansible_date_time.time }}
# Формат: CSV

"Host","OS","Version","Total Packages","Report File"
{% for host in groups['all'] -%}
{% set host_report = hostvars[host] %}
"{{ host }}","{{ host_report.ansible_distribution }}","{{ host_report.ansible_distribution_version }}","{{ host_report.total_filtered_packages }}","package_report_{{ host }}.csv"
{% endfor %}
```

templates/summary_report_json.j2

```jinja2
{
  "metadata": {
    "generated": "{{ ansible_date_time.iso8601 }}",
    "total_hosts": {{ groups['all'] | length }},
    "format": "json"
  },
  "hosts": [
    {% for host in groups['all'] %}
    {% set host_report = hostvars[host] %}
    {
      "hostname": "{{ host }}",
      "os": {
        "distribution": "{{ host_report.ansible_distribution }}",
        "version": "{{ host_report.ansible_distribution_version }}",
        "family": "{{ host_report.ansible_os_family }}"
      },
      "package_stats": {
        "total_filtered": {{ host_report.total_filtered_packages }},
        "filter_applied": {{ 'true' if host_report.package_filter is defined else 'false' }}
      },
      "report_file": "package_report_{{ host }}.json"
    }{{ ',' if not loop.last else '' }}
    {% endfor %}
  ]
}
```

files/merge_reports.py

```python
#!/usr/bin/env python3
"""
Скрипт для объединения всех JSON отчетов в один файл
"""
import json
import glob
import os
from datetime import datetime

def merge_json_reports():
    """Объединяет все JSON отчеты в один файл"""
    
    # Получаем все JSON файлы отчетов
    report_files = glob.glob("package_report_*.json")
    
    if not report_files:
        print("Не найдено JSON отчетов")
        return
    
    merged_data = {
        "metadata": {
            "generated": datetime.now().isoformat(),
            "total_reports": len(report_files),
            "reports_included": []
        },
        "reports": []
    }
    
    for report_file in report_files:
        try:
            with open(report_file, 'r') as f:
                data = json.load(f)
                merged_data["reports"].append(data)
                merged_data["metadata"]["reports_included"].append(report_file)
        except Exception as e:
            print(f"Ошибка при чтении {report_file}: {e}")
    
    # Сохраняем объединенный отчет
    with open("all_reports_merged.json", 'w') as f:
        json.dump(merged_data, f, indent=2, ensure_ascii=False)
    
    print(f"Создан объединенный отчет: all_reports_merged.json")
    print(f"Включено отчетов: {len(report_files)}")

if __name__ == "__main__":
    merge_json_reports()
```

vars/main.yml (обновленный)

```yaml
---
# Директория для сохранения отчетов
report_directory: ./package_reports

# Формат отчета: 'csv' или 'json'
report_format: json

# Фильтр пакетов по имени (регулярные выражения)
# Примеры:
#   - "^nginx" - все пакеты, начинающиеся с nginx
#   - "python" - все пакеты, содержащие python
#   - "docker|kubectl" - пакеты docker или kubectl
package_filter:
  - "^nginx"
  - "^apache"
  - "^python3"
  - "^docker"
  - "^kubectl"

# Список конкретных пакетов для проверки
specific_packages:
  - nginx
  - docker
  - python3
  - git
  - curl
  - wget
```

Обновленный плейбук collect_packages.yml

```yaml
---
- name: Сбор информации о пакетах с фильтрацией
  hosts: all
  become: yes
  
  vars:
    # Директория для отчетов
    report_directory: /tmp/package_reports
    # Формат отчета: csv или json
    report_format: json
    # Фильтр пакетов (регулярные выражения)
    package_filter:
      - "^nginx"
      - "^apache"
      - "^mysql"
      - "^postgresql"
      - "^redis"
      - "^docker"
      - "^python"
      - "^java"
    # Список пакетов для детальной проверки
    specific_packages:
      - nginx
      - apache2
      - mysql-server
      - postgresql
      - docker-ce
      - python3
      - openjdk-11-jdk
  
  roles:
    - package-info-collector
```

Примеры использования

Запуск с CSV форматом:

```bash
ansible-playbook -i inventory.ini collect_packages.yml -e "report_format=csv"
```

Запуск с JSON форматом и фильтром:

```bash
ansible-playbook -i inventory.ini collect_packages.yml \
  -e '{
    "report_format": "json",
    "package_filter": ["^nginx", "^docker", "^kubernetes"]
  }'
```

Запуск без фильтра (все пакеты):

```bash
ansible-playbook -i inventory.ini collect_packages.yml \
  -e '{"report_format": "json", "package_filter": []}'
```

Структура выходных файлов

CSV формат:

```
package_reports/
├── package_report_ubuntu1.csv
├── package_report_oracle1.csv
├── summary_report.csv
└── merge_reports.py (для JSON)
```

JSON формат:

```
package_reports/
├── package_report_ubuntu1.json
├── package_report_oracle1.json
├── summary_report.json
├── all_reports_merged.json
└── merge_reports.py
```

Особенности обновленной версии

1. Фильтрация пакетов:
   · Поддержка регулярных выражений
   · Возможность комбинировать несколько паттернов
   · Гибкая настройка через переменные
2. Два формата отчетов:
   · CSV: удобен для импорта в Excel/Google Sheets
   · JSON: удобен для программной обработки
3. Расширенная информация о пакетах:
   · Имя пакета
   · Версия
   · Архитектура
   · Размер
   · Описание
4. Объединение JSON отчетов:
   · Автоматическое создание единого файла
   · Сохранение метаданных
   · Удобно для дальнейшего анализа
5. Метаданные в отчетах:
   · Информация о примененных фильтрах
   · Временные метки
   · Статистика по хостам

Это решение предоставляет гибкую систему сбора информации о пакетах с возможностью фильтрации и выбора формата вывода, что делает его удобным для различных сценариев использования - от простого аудита до интеграции с системами мониторинга и анализа.



Вот обновленная Ansible роль с фильтрацией пакетов и поддержкой CSV и JSON форматов:

Обновленная структура роли

```
package-info-collector/
├── tasks/
│   ├── main.yml
│   └── get_specific_packages.yml
├── templates/
│   ├── report_csv.j2
│   └── report_json.j2
├── vars/
│   └── main.yml
└── files/
    └── merge_reports.py
```

tasks/main.yml (обновленный)

```yaml
---
- name: Сбор информации о системе
  setup:
    filter:
      - ansible_distribution
      - ansible_distribution_version
      - ansible_os_family
      - ansible_hostname
      - ansible_date_time
  register: system_info

- name: Определение менеджера пакетов
  set_fact:
    package_manager: >-
      {%- if ansible_os_family == "Debian" -%}apt
      {%- elif ansible_os_family == "RedHat" -%}rpm
      {%- else -%}unknown
      {%- endif -%}

- name: Определение формата отчета
  set_fact:
    report_format: "{{ report_format | default('csv') }}"
    report_extension: "{{ 'csv' if report_format == 'csv' else 'json' }}"

- name: Получение списка всех установленных пакетов (Debian/Ubuntu)
  shell: |
    dpkg-query -W -f='${Package}\t${Version}\t${Architecture}\t${Installed-Size}\t${Description}\n' | \
    grep "install ok installed" | cut -f1-5
  register: deb_packages
  when: ansible_os_family == "Debian"
  changed_when: false

- name: Получение списка всех установленных пакетов (RedHat/Oracle Linux)
  shell: |
    rpm -qa --queryformat "%{NAME}\t%{VERSION}-%{RELEASE}\t%{ARCH}\t%{SIZE}\t%{SUMMARY}\n" | sort
  register: rh_packages
  when: ansible_os_family == "RedHat"
  changed_when: false

- name: Фильтрация пакетов по имени
  set_fact:
    filtered_packages: >-
      {%- if package_filter is defined and package_filter | length > 0 -%}
        {%- set filtered = [] -%}
        {%- if ansible_os_family == "Debian" -%}
          {%- for line in deb_packages.stdout_lines -%}
            {%- set parts = line.split('\t') -%}
            {%- for filter_pattern in package_filter -%}
              {%- if parts[0] | regex_search(filter_pattern) -%}
                {%- set _ = filtered.append(line) -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endfor -%}
        {%- elif ansible_os_family == "RedHat" -%}
          {%- for line in rh_packages.stdout_lines -%}
            {%- set parts = line.split('\t') -%}
            {%- for filter_pattern in package_filter -%}
              {%- if parts[0] | regex_search(filter_pattern) -%}
                {%- set _ = filtered.append(line) -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endfor -%}
        {%- endif -%}
        {{ filtered }}
      {%- else -%}
        {{ deb_packages.stdout_lines if ansible_os_family == "Debian" else rh_packages.stdout_lines }}
      {%- endif -%}

- name: Подсчет отфильтрованных пакетов
  set_fact:
    total_filtered_packages: "{{ filtered_packages | length }}"

- name: Создание структурированных данных о пакетах (CSV формат)
  set_fact:
    package_data_csv: >-
      {%- set output = [] -%}
      {%- for line in filtered_packages -%}
        {%- set parts = line.split('\t') -%}
        {%- if ansible_os_family == "Debian" -%}
          {%- set _ = output.append(
            '"' + parts[0] + '","' + 
            parts[1] + '","' + 
            (parts[2] if parts|length > 2 else 'unknown') + '","' + 
            (parts[3] if parts|length > 3 else '0') + '","' + 
            (parts[4]|replace('"', '""') if parts|length > 4 else '') + '"'
          ) -%}
        {%- else -%}
          {%- set _ = output.append(
            '"' + parts[0] + '","' + 
            parts[1] + '","' + 
            (parts[2] if parts|length > 2 else 'unknown') + '","' + 
            (parts[3] if parts|length > 3 else '0') + '","' + 
            (parts[4]|replace('"', '""') if parts|length > 4 else '') + '"'
          ) -%}
        {%- endif -%}
      {%- endfor -%}
      {{ output }}

- name: Создание структурированных данных о пакетах (JSON формат)
  set_fact:
    package_data_json: >-
      {%- set packages = [] -%}
      {%- for line in filtered_packages -%}
        {%- set parts = line.split('\t') -%}
        {%- if ansible_os_family == "Debian" -%}
          {%- set _ = packages.append({
            'name': parts[0],
            'version': parts[1],
            'architecture': parts[2] if parts|length > 2 else 'unknown',
            'size': parts[3] if parts|length > 3 else '0',
            'description': parts[4] if parts|length > 4 else ''
          }) -%}
        {%- else -%}
          {%- set _ = packages.append({
            'name': parts[0],
            'version': parts[1],
            'architecture': parts[2] if parts|length > 2 else 'unknown',
            'size': parts[3] if parts|length > 3 else '0',
            'description': parts[4] if parts|length > 4 else ''
          }) -%}
        {%- endif -%}
      {%- endfor -%}
      {{ packages | to_json }}

- name: Получение информации о конкретных пакетах
  include_tasks: get_specific_packages.yml
  when: specific_packages is defined

- name: Сохранение отчета на целевой машине (CSV)
  template:
    src: report_csv.j2
    dest: "/tmp/package_report_{{ ansible_hostname }}.{{ report_extension }}"
    mode: '0644'
  when: report_format == 'csv'
  delegate_to: "{{ inventory_hostname }}"

- name: Сохранение отчета на целевой машине (JSON)
  template:
    src: report_json.j2
    dest: "/tmp/package_report_{{ ansible_hostname }}.{{ report_extension }}"
    mode: '0644'
  when: report_format == 'json'
  delegate_to: "{{ inventory_hostname }}"

- name: Сбор всех отчетов на контрольной машине
  fetch:
    src: "/tmp/package_report_{{ ansible_hostname }}.{{ report_extension }}"
    dest: "{{ report_directory }}/"
    flat: yes
  run_once: true
  delegate_to: "{{ inventory_hostname }}"

- name: Создание сводного отчета
  template:
    src: "summary_report_{{ report_format }}.j2"
    dest: "{{ report_directory }}/summary_report.{{ report_extension }}"
  delegate_to: localhost
  run_once: true

- name: Копирование скрипта для объединения JSON отчетов
  copy:
    src: merge_reports.py
    dest: "{{ report_directory }}/merge_reports.py"
    mode: '0755'
  delegate_to: localhost
  run_once: true
  when: report_format == 'json'

- name: Объединение всех JSON отчетов в один файл
  shell: |
    cd {{ report_directory }}
    python3 merge_reports.py
  delegate_to: localhost
  run_once: true
  when: report_format == 'json'
```

templates/report_csv.j2

```jinja2
# Отчет об установленных пакетах
# Хост: {{ ansible_hostname }}
# Дата: {{ ansible_date_time.date }} {{ ansible_date_time.time }}
# ОС: {{ ansible_distribution }} {{ ansible_distribution_version }}
# Всего пакетов (с учетом фильтра): {{ total_filtered_packages }}
# Фильтр: {{ package_filter | default('Нет фильтра') }}
"Package Name","Version","Architecture","Size (KB)","Description"
{% for line in package_data_csv -%}
{{ line }}
{% endfor %}
```

templates/report_json.j2

```jinja2
{
  "metadata": {
    "hostname": "{{ ansible_hostname }}",
    "timestamp": "{{ ansible_date_time.iso8601 }}",
    "os": {
      "distribution": "{{ ansible_distribution }}",
      "version": "{{ ansible_distribution_version }}",
      "family": "{{ ansible_os_family }}"
    },
    "package_manager": "{{ package_manager }}",
    "total_packages": {{ total_filtered_packages }},
    "filter_applied": {{ 'true' if package_filter is defined else 'false' }},
    "filter_patterns": {{ package_filter | default([]) | to_json }}
  },
  "packages": {{ package_data_json | from_json | to_nice_json(indent=2) }}
}
```

templates/summary_report_csv.j2

```jinja2
# СВОДНЫЙ ОТЧЕТ ПО ВСЕМ ХОСТАМ
# Дата создания: {{ ansible_date_time.date }} {{ ansible_date_time.time }}
# Формат: CSV

"Host","OS","Version","Total Packages","Report File"
{% for host in groups['all'] -%}
{% set host_report = hostvars[host] %}
"{{ host }}","{{ host_report.ansible_distribution }}","{{ host_report.ansible_distribution_version }}","{{ host_report.total_filtered_packages }}","package_report_{{ host }}.csv"
{% endfor %}
```

templates/summary_report_json.j2

```jinja2
{
  "metadata": {
    "generated": "{{ ansible_date_time.iso8601 }}",
    "total_hosts": {{ groups['all'] | length }},
    "format": "json"
  },
  "hosts": [
    {% for host in groups['all'] %}
    {% set host_report = hostvars[host] %}
    {
      "hostname": "{{ host }}",
      "os": {
        "distribution": "{{ host_report.ansible_distribution }}",
        "version": "{{ host_report.ansible_distribution_version }}",
        "family": "{{ host_report.ansible_os_family }}"
      },
      "package_stats": {
        "total_filtered": {{ host_report.total_filtered_packages }},
        "filter_applied": {{ 'true' if host_report.package_filter is defined else 'false' }}
      },
      "report_file": "package_report_{{ host }}.json"
    }{{ ',' if not loop.last else '' }}
    {% endfor %}
  ]
}
```

files/merge_reports.py

```python
#!/usr/bin/env python3
"""
Скрипт для объединения всех JSON отчетов в один файл
"""
import json
import glob
import os
from datetime import datetime

def merge_json_reports():
    """Объединяет все JSON отчеты в один файл"""
    
    # Получаем все JSON файлы отчетов
    report_files = glob.glob("package_report_*.json")
    
    if not report_files:
        print("Не найдено JSON отчетов")
        return
    
    merged_data = {
        "metadata": {
            "generated": datetime.now().isoformat(),
            "total_reports": len(report_files),
            "reports_included": []
        },
        "reports": []
    }
    
    for report_file in report_files:
        try:
            with open(report_file, 'r') as f:
                data = json.load(f)
                merged_data["reports"].append(data)
                merged_data["metadata"]["reports_included"].append(report_file)
        except Exception as e:
            print(f"Ошибка при чтении {report_file}: {e}")
    
    # Сохраняем объединенный отчет
    with open("all_reports_merged.json", 'w') as f:
        json.dump(merged_data, f, indent=2, ensure_ascii=False)
    
    print(f"Создан объединенный отчет: all_reports_merged.json")
    print(f"Включено отчетов: {len(report_files)}")

if __name__ == "__main__":
    merge_json_reports()
```

vars/main.yml (обновленный)

```yaml
---
# Директория для сохранения отчетов
report_directory: ./package_reports

# Формат отчета: 'csv' или 'json'
report_format: json

# Фильтр пакетов по имени (регулярные выражения)
# Примеры:
#   - "^nginx" - все пакеты, начинающиеся с nginx
#   - "python" - все пакеты, содержащие python
#   - "docker|kubectl" - пакеты docker или kubectl
package_filter:
  - "^nginx"
  - "^apache"
  - "^python3"
  - "^docker"
  - "^kubectl"

# Список конкретных пакетов для проверки
specific_packages:
  - nginx
  - docker
  - python3
  - git
  - curl
  - wget
```

Обновленный плейбук collect_packages.yml

```yaml
---
- name: Сбор информации о пакетах с фильтрацией
  hosts: all
  become: yes
  
  vars:
    # Директория для отчетов
    report_directory: /tmp/package_reports
    # Формат отчета: csv или json
    report_format: json
    # Фильтр пакетов (регулярные выражения)
    package_filter:
      - "^nginx"
      - "^apache"
      - "^mysql"
      - "^postgresql"
      - "^redis"
      - "^docker"
      - "^python"
      - "^java"
    # Список пакетов для детальной проверки
    specific_packages:
      - nginx
      - apache2
      - mysql-server
      - postgresql
      - docker-ce
      - python3
      - openjdk-11-jdk
  
  roles:
    - package-info-collector
```

Примеры использования

Запуск с CSV форматом:

```bash
ansible-playbook -i inventory.ini collect_packages.yml -e "report_format=csv"
```

Запуск с JSON форматом и фильтром:

```bash
ansible-playbook -i inventory.ini collect_packages.yml \
  -e '{
    "report_format": "json",
    "package_filter": ["^nginx", "^docker", "^kubernetes"]
  }'
```

Запуск без фильтра (все пакеты):

```bash
ansible-playbook -i inventory.ini collect_packages.yml \
  -e '{"report_format": "json", "package_filter": []}'
```

Структура выходных файлов

CSV формат:

```
package_reports/
├── package_report_ubuntu1.csv
├── package_report_oracle1.csv
├── summary_report.csv
└── merge_reports.py (для JSON)
```

JSON формат:

```
package_reports/
├── package_report_ubuntu1.json
├── package_report_oracle1.json
├── summary_report.json
├── all_reports_merged.json
└── merge_reports.py
```

Особенности обновленной версии

1. Фильтрация пакетов:
   · Поддержка регулярных выражений
   · Возможность комбинировать несколько паттернов
   · Гибкая настройка через переменные
2. Два формата отчетов:
   · CSV: удобен для импорта в Excel/Google Sheets
   · JSON: удобен для программной обработки
3. Расширенная информация о пакетах:
   · Имя пакета
   · Версия
   · Архитектура
   · Размер
   · Описание
4. Объединение JSON отчетов:
   · Автоматическое создание единого файла
   · Сохранение метаданных
   · Удобно для дальнейшего анализа
5. Метаданные в отчетах:
   · Информация о примененных фильтрах
   · Временные метки
   · Статистика по хостам

Это решение предоставляет гибкую систему сбора информации о пакетах с возможностью фильтрации и выбора формата вывода, что делает его удобным для различных сценариев использования - от простого аудита до интеграции с системами мониторинга и анализа.