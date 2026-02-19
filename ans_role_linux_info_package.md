Вот решение для создания Ansible роли, которая собирает информацию об установленных пакетах на Ubuntu и Oracle Linux и объединяет её в один отчет.

Структура роли

```
package-info-collector/
├── tasks/
│   └── main.yml
├── templates/
│   └── report.j2
└── vars/
    └── main.yml
```

Содержимое файлов

tasks/main.yml

```yaml
---
- name: Сбор информации о системе
  setup:
    filter:
      - ansible_distribution
      - ansible_distribution_version
      - ansible_os_family
      - ansible_hostname
  register: system_info

- name: Определение менеджера пакетов
  set_fact:
    package_manager: >-
      {%- if ansible_os_family == "Debian" -%}apt
      {%- elif ansible_os_family == "RedHat" -%}rpm
      {%- else -%}unknown
      {%- endif -%}

- name: Получение списка всех установленных пакетов (Debian/Ubuntu)
  shell: |
    dpkg-query -W -f='${Package}\t${Version}\t${Status}\n' | grep "install ok installed" | cut -f1,2
  register: deb_packages
  when: ansible_os_family == "Debian"
  changed_when: false

- name: Получение списка всех установленных пакетов (RedHat/Oracle Linux)
  shell: |
    rpm -qa --queryformat "%{NAME}\t%{VERSION}-%{RELEASE}\n" | sort
  register: rh_packages
  when: ansible_os_family == "RedHat"
  changed_when: false

- name: Получение информации о конкретных пакетах (по списку)
  include_tasks: get_specific_packages.yml
  when: specific_packages is defined

- name: Подсчет общего количества пакетов
  set_fact:
    total_packages: >-
      {%- if ansible_os_family == "Debian" -%}
        {{ deb_packages.stdout_lines | length }}
      {%- elif ansible_os_family == "RedHat" -%}
        {{ rh_packages.stdout_lines | length }}
      {%- else -%}
        0
      {%- endif -%}

- name: Создание структурированных данных о пакетах
  set_fact:
    package_data: >-
      {%- if ansible_os_family == "Debian" -%}
        {%- set packages = [] -%}
        {%- for line in deb_packages.stdout_lines -%}
          {%- set parts = line.split('\t') -%}
          {%- set _ = packages.append({'name': parts[0], 'version': parts[1]}) -%}
        {%- endfor -%}
        {{ packages }}
      {%- elif ansible_os_family == "RedHat" -%}
        {%- set packages = [] -%}
        {%- for line in rh_packages.stdout_lines -%}
          {%- set parts = line.split('\t') -%}
          {%- set _ = packages.append({'name': parts[0], 'version': parts[1]}) -%}
        {%- endfor -%}
        {{ packages }}
      {%- else -%}
        []
      {%- endif -%}

- name: Сохранение отчета на целевой машине
  template:
    src: report.j2
    dest: /tmp/package_report_{{ ansible_hostname }}.txt
    mode: '0644'
  delegate_to: "{{ inventory_hostname }}"

- name: Сбор всех отчетов на контрольной машине
  fetch:
    src: /tmp/package_report_{{ ansible_hostname }}.txt
    dest: "{{ report_directory }}/"
    flat: yes
  run_once: true
  delegate_to: "{{ inventory_hostname }}"

- name: Создание сводного отчета на контрольной машине
  template:
    src: summary_report.j2
    dest: "{{ report_directory }}/summary_report.txt"
  delegate_to: localhost
  run_once: true
```

tasks/get_specific_packages.yml

```yaml
---
- name: Получение информации о конкретных пакетах (Ubuntu)
  shell: |
    {% for package in specific_packages %}
    dpkg-query -W -f='${Package}\t${Version}\t${Status}\n' {{ package }} 2>/dev/null || echo "{{ package }}\tnot installed\tnot installed"
    {% endfor %}
  register: specific_deb_packages
  when: ansible_os_family == "Debian"
  changed_when: false

- name: Получение информации о конкретных пакетах (Oracle Linux)
  shell: |
    {% for package in specific_packages %}
    rpm -q --queryformat "%{NAME}\t%{VERSION}-%{RELEASE}\n" {{ package }} 2>/dev/null || echo "{{ package }\tnot installed"
    {% endfor %}
  register: specific_rh_packages
  when: ansible_os_family == "RedHat"
  changed_when: false

- name: Сохранение информации о конкретных пакетах
  set_fact:
    specific_package_info: >-
      {%- if ansible_os_family == "Debian" -%}
        {{ specific_deb_packages.stdout_lines }}
      {%- elif ansible_os_family == "RedHat" -%}
        {{ specific_rh_packages.stdout_lines }}
      {%- endif -%}
```

templates/report.j2

```jinja2
==================================================
ОТЧЕТ ОБ УСТАНОВЛЕННЫХ ПАКЕТАХ
==================================================
Хост: {{ ansible_hostname }}
Дата: {{ ansible_date_time.date }} {{ ansible_date_time.time }}
ОС: {{ ansible_distribution }} {{ ansible_distribution_version }}
Семейство ОС: {{ ansible_os_family }}
Менеджер пакетов: {{ package_manager }}
Всего пакетов: {{ total_packages }}

==================================================
СПИСОК ВСЕХ УСТАНОВЛЕННЫХ ПАКЕТОВ
==================================================
{% for package in package_data %}
{{ package.name }} : {{ package.version }}
{% endfor %}

{% if specific_package_info is defined %}
==================================================
ИНФОРМАЦИЯ О КОНКРЕТНЫХ ПАКЕТАХ
==================================================
{% for line in specific_package_info %}
{{ line }}
{% endfor %}
{% endif %}
```

templates/summary_report.j2

```jinja2
==================================================
СВОДНЫЙ ОТЧЕТ ПО ВСЕМ ХОСТАМ
==================================================
Дата создания: {{ ansible_date_time.date }} {{ ansible_date_time.time }}

{% for host in groups['all'] %}
--------------------------------------------------
Хост: {{ host }}
--------------------------------------------------
{% endwith %}
{% endfor %}

==================================================
ДЕТАЛЬНЫЕ ОТЧЕТЫ ПО КАЖДОМУ ХОСТУ
==================================================
Для просмотра детальной информации по каждому хосту,
смотрите файлы package_report_*.txt в директории {{ report_directory }}
```

vars/main.yml

```yaml
---
# Директория для сохранения отчетов
report_directory: ./package_reports

# Список конкретных пакетов для проверки (можно переопределить в плейбуке)
# specific_packages:
#   - nginx
#   - docker-ce
#   - python3
```

Пример плейбука для использования роли

collect_packages.yml

```yaml
---
- name: Сбор информации о пакетах с Ubuntu и Oracle Linux
  hosts: all
  become: yes
  
  vars:
    # Директория для отчетов
    report_directory: /tmp/package_reports
    # Список пакетов для детальной проверки
    specific_packages:
      - nginx
      - docker
      - python3
      - git
      - curl
  
  roles:
    - package-info-collector
```

Инвентарный файл

inventory.ini

```ini
[ubuntu_servers]
ubuntu1 ansible_host=192.168.1.10 ansible_user=ubuntu
ubuntu2 ansible_host=192.168.1.11 ansible_user=ubuntu

[oracle_servers]
oracle1 ansible_host=192.168.1.20 ansible_user=opc
oracle2 ansible_host=192.168.1.21 ansible_user=opc

[all:children]
ubuntu_servers
oracle_servers
```

Запуск

```bash
# Запуск сбора информации
ansible-playbook -i inventory.ini collect_packages.yml

# Результат будет в директории /tmp/package_reports/
# - summary_report.txt - сводный отчет
# - package_report_*.txt - отчеты по каждому хосту
```

Особенности решения

1. Кроссплатформенность: Работает на Ubuntu (Debian) и Oracle Linux (RedHat)
2. Два уровня отчетов:
   · Индивидуальные отчеты по каждому серверу
   · Сводный отчет по всем серверам
3. Гибкость: Можно задать список конкретных пакетов для проверки
4. Структурированные данные: Информация сохраняется в удобном для анализа формате
5. Автоматический сбор: Все отчеты собираются на контрольной машине

Это решение предоставляет полную информацию об установленных пакетах и может быть легко расширено для добавления дополнительных функций, таких как сравнение версий или поиск уязвимостей.



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



Ошибка возникает из-за синтаксической проблемы в шаблоне для конкретных пакетов (Oracle Linux). В файле get_specific_packages.yml в строке echo пропущена закрывающая фигурная скобка для переменной package. Также аналогичная проблема может быть в Ubuntu-ветке.

Исправление ошибки

1. Исправленный файл tasks/get_specific_packages.yml

```yaml
---
- name: Получение информации о конкретных пакетах (Ubuntu)
  shell: |
    {% for pkg in specific_packages %}
    dpkg-query -W -f='${Package}\t${Version}\t${Status}\n' {{ pkg }} 2>/dev/null || echo "{{ pkg }}\tnot installed\tnot installed"
    {% endfor %}
  register: specific_deb_packages
  when: ansible_os_family == "Debian"
  changed_when: false

- name: Получение информации о конкретных пакетах (Oracle Linux)
  shell: |
    {% for pkg in specific_packages %}
    rpm -q --queryformat "%{NAME}\t%{VERSION}-%{RELEASE}\n" {{ pkg }} 2>/dev/null || echo "{{ pkg }}\tnot installed"
    {% endfor %}
  register: specific_rh_packages
  when: ansible_os_family == "RedHat"
  changed_when: false

- name: Сохранение информации о конкретных пакетах
  set_fact:
    specific_package_info: >-
      {%- if ansible_os_family == "Debian" -%}
        {{ specific_deb_packages.stdout_lines }}
      {%- elif ansible_os_family == "RedHat" -%}
        {{ specific_rh_packages.stdout_lines }}
      {%- endif -%}
```

Что было исправлено:

· В строках echo для обеих веток теперь правильно закрыта переменная pkg: "{{ pkg }}\tnot installed" (было "{{ package }\tnot installed" – пропущена вторая фигурная скобка).
· Везде используется переменная pkg вместо package для единообразия (в цикле for pkg in specific_packages).

2. Дополнительная проверка в tasks/main.yml

Чтобы избежать выполнения задачи, если список specific_packages пуст или не определён, добавьте условие:

```yaml
- name: Получение информации о конкретных пакетах
  include_tasks: get_specific_packages.yml
  when: specific_packages is defined and specific_packages | length > 0
```

3. Полный исправленный файл tasks/get_specific_packages.yml

(см. выше пункт 1)

Причина ошибки

Ansible пытается интерпретировать содержимое shell как шаблон Jinja2. Внутри этого шаблона использовалась конструкция {{ package }, которая не была закрыта, из-за чего парсер Jinja2 не смог корректно разобрать выражение и выдал ошибку.

Проверка

После исправления запустите плейбук снова:

```bash
ansible-playbook -i inventory.ini collect_packages.yml
```

Убедитесь, что задача Получение информации о конкретных пакетах выполняется без ошибок. Если список specific_packages пуст, задача будет пропущена благодаря добавленному условию.

Дополнительные рекомендации

· Для избежания проблем с интерпретацией символа табуляции \t в echo можно использовать printf (более переносимо), например:
  ```bash
  printf "%s\t%s\n" "{{ pkg }}" "not installed"
  ```
  Но текущий вариант с echo также будет работать в большинстве случаев, так как \t в двойных кавычках обычно преобразуется в табуляцию.
· Убедитесь, что переменная specific_packages в вашем плейбуке или инвентаре определена корректно как список. Пример:
  ```yaml
  specific_packages:
    - nginx
    - docker
    - python3
  ```



Ошибка возникает из-за того, что в момент создания сводного отчета на localhost не все факты (переменные ansible_distribution и др.) доступны в hostvars для некоторых хостов. Это может случиться, если для каких-то хостов не были собраны факты (например, из-за проблем с подключением или если gather_facts: no). Чтобы избежать ошибки, нужно использовать значения по умолчанию в шаблоне.

Исправление шаблонов сводного отчета

1. Исправленный templates/summary_report_csv.j2

```jinja2
# СВОДНЫЙ ОТЧЕТ ПО ВСЕМ ХОСТАМ
# Дата создания: {{ ansible_date_time.date }} {{ ansible_date_time.time }}
# Формат: CSV

"Host","OS","Version","Total Packages","Report File"
{% for host in groups['all'] -%}
{% set h = hostvars[host] %}
"{{ host }}","{{ h['ansible_distribution'] | default('N/A') }}","{{ h['ansible_distribution_version'] | default('N/A') }}","{{ h['total_filtered_packages'] | default(0) }}","package_report_{{ host }}.csv"
{% endfor %}
```

2. Исправленный templates/summary_report_json.j2

```jinja2
{
  "metadata": {
    "generated": "{{ ansible_date_time.iso8601 }}",
    "total_hosts": {{ groups['all'] | length }},
    "format": "json"
  },
  "hosts": [
    {% for host in groups['all'] %}
    {% set h = hostvars[host] %}
    {
      "hostname": "{{ host }}",
      "os": {
        "distribution": "{{ h['ansible_distribution'] | default('N/A') }}",
        "version": "{{ h['ansible_distribution_version'] | default('N/A') }}",
        "family": "{{ h['ansible_os_family'] | default('N/A') }}"
      },
      "package_stats": {
        "total_filtered": {{ h['total_filtered_packages'] | default(0) }},
        "filter_applied": {{ 'true' if h['package_filter'] is defined else 'false' }}
      },
      "report_file": "package_report_{{ host }}.json"
    }{{ ',' if not loop.last else '' }}
    {% endfor %}
  ]
}
```

Дополнительные рекомендации

· Убедитесь, что в плейбуке включён сбор фактов (gather_facts: yes). Если он отключён, добавьте задачу setup перед использованием фактов.
· Если некоторые хосты недоступны, они будут отмечены как N/A в отчёте, но ошибка больше не возникнет.

После замены шаблонов запустите плейбук заново:

```bash
ansible-playbook -i inventory.ini collect_packages.yml
```

Теперь задача создания сводного отчёта должна выполняться без ошибок.


Мы исправили роль, чтобы все данные об установленных пакетах собирались в один единый отчёт и сохранялись в локальную папку.
Теперь после выполнения роли в папке {{ report_directory }} появляется файл unified_report.csv или unified_report.json, который содержит все пакеты со всех хостов в плоской структуре (с указанием хоста для каждой записи).
Промежуточные отчёты по каждому хосту также сохраняются, но главный результат — один объединённый файл.

Что изменено/добавлено

1. Новый скрипт create_unified_report.py
      Копируется на control machine и запускается после сбора всех индивидуальных отчётов.
   · Для CSV: читает все package_report_*.csv, объединяет их, добавляет колонку Host и сохраняет как unified_report.csv.
   · Для JSON: читает все package_report_*.json, извлекает список пакетов, добавляет к каждому пакету поле host и сохраняет как unified_report.json (массив объектов).
2. Обновлён tasks/main.yml – добавлены задачи для копирования и запуска скрипта.
3. Исправлены ошибки
   · Синтаксическая ошибка в get_specific_packages.yml (пропущенная скобка).
   · Проблема с hostvars в шаблонах сводного отчёта – теперь используются значения по умолчанию (default('N/A')).

Файлы для добавления/изменения

1. files/create_unified_report.py

```python
#!/usr/bin/env python3
"""
Скрипт для создания единого отчёта из всех индивидуальных отчётов.
Поддерживает форматы CSV и JSON.
"""
import argparse
import csv
import json
import glob
import os
import sys

def create_unified_csv():
    """Объединяет все CSV отчёты в один с добавлением колонки Host."""
    csv_files = glob.glob("package_report_*.csv")
    if not csv_files:
        print("CSV отчёты не найдены")
        return

    with open("unified_report.csv", "w", newline='', encoding='utf-8') as outfile:
        writer = None
        first_file = True

        for filename in sorted(csv_files):
            hostname = filename.replace("package_report_", "").replace(".csv", "")
            with open(filename, 'r', encoding='utf-8') as infile:
                reader = csv.reader(infile)
                headers = next(reader)  # пропускаем строки комментариев до заголовка
                # Ищем строку с заголовками столбцов
                while headers and not headers[0].startswith('"Package Name"'):
                    headers = next(reader, None)
                    if headers is None:
                        break
                if headers is None:
                    continue

                if first_file:
                    # Записываем заголовок с добавлением Host
                    writer = csv.writer(outfile)
                    writer.writerow(['Host'] + headers)
                    first_file = False

                # Записываем данные, добавляя в начало название хоста
                for row in reader:
                    writer.writerow([hostname] + row)
    print(f"Создан unified_report.csv, включено файлов: {len(csv_files)}")

def create_unified_json():
    """Объединяет все JSON отчёты в один массив объектов с полем host."""
    json_files = glob.glob("package_report_*.json")
    if not json_files:
        print("JSON отчёты не найдены")
        return

    unified = []
    for filename in sorted(json_files):
        hostname = filename.replace("package_report_", "").replace(".json", "")
        with open(filename, 'r', encoding='utf-8') as f:
            data = json.load(f)
            packages = data.get('packages', [])
            for pkg in packages:
                pkg['host'] = hostname
                unified.append(pkg)

    with open("unified_report.json", "w", encoding='utf-8') as f:
        json.dump(unified, f, indent=2, ensure_ascii=False)
    print(f"Создан unified_report.json, всего записей: {len(unified)}")

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--format', choices=['csv', 'json'], default='csv')
    args = parser.parse_args()

    if args.format == 'csv':
        create_unified_csv()
    else:
        create_unified_json()

if __name__ == "__main__":
    main()
```

2. Добавить в tasks/main.yml (после блока fetch)

```yaml
- name: Копирование скрипта для создания единого отчета
  copy:
    src: create_unified_report.py
    dest: "{{ report_directory }}/create_unified_report.py"
    mode: '0755'
  delegate_to: localhost
  run_once: true

- name: Создание единого отчета
  shell: |
    cd {{ report_directory }}
    python3 create_unified_report.py --format {{ report_format }}
  delegate_to: localhost
  run_once: true
  register: unified_result
  changed_when: unified_result.rc == 0
```

3. Убедиться, что в vars/main.yml определён report_directory и report_format

```yaml
report_directory: ./package_reports
report_format: json   # или csv
```

Результат

После запуска плейбука в папке ./package_reports появятся:

· Индивидуальные отчёты: package_report_<host>.csv (или .json)
· Сводный отчёт по хостам: summary_report.csv (или .json)
· Единый отчёт со всеми пакетами: unified_report.csv (или .json)

Пример содержимого unified_report.csv:

```
Host,Package Name,Version,Architecture,Size,Description
ubuntu1,nginx,1.18.0-0ubuntu1,amd64,1234,high performance web server
oracle1,nginx,1.20.1-1.el7,x86_64,5678,nginx web server
...
```

Пример unified_report.json:

```json
[
  {
    "name": "nginx",
    "version": "1.18.0-0ubuntu1",
    "architecture": "amd64",
    "size": "1234",
    "description": "high performance web server",
    "host": "ubuntu1"
  },
  ...
]
```

Проверка

Запустите плейбук:

```bash
ansible-playbook -i inventory.ini collect_packages.yml
```

Убедитесь, что задача создания единого отчёта выполняется без ошибок и файл unified_report создан.

Если нужна только плоская структура без промежуточных файлов, можно удалить задачи создания индивидуальных отчётов, но мы рекомендуем оставить их для отладки.




Проблема в том, что на Ubuntu (включая версии 22.04 и 24.04) команда для сбора всех пакетов не включала проверку статуса установки. Мы запрашивали поля имени, версии, архитектуры, размера и описания, но не добавляли поле статуса, а затем пытались отфильтровать по нему с помощью grep. В результате фильтрация не срабатывала, и список пакетов оставался пустым.

Исправление

Замените задачу получения списка пакетов для Debian/Ubuntu в файле tasks/main.yml на следующий код, который корректно учитывает статус пакета:

```yaml
- name: Получение списка всех установленных пакетов (Debian/Ubuntu)
  shell: |
    dpkg-query -W -f='${Package}\t${Version}\t${Architecture}\t${Installed-Size}\t${Description}\t${Status}\n' | \
    awk -F'\t' '$6 ~ /install ok installed/ { print $1"\t"$2"\t"$3"\t"$4"\t"$5 }'
  register: deb_packages
  when: ansible_os_family == "Debian"
  changed_when: false
```

Что изменилось:

· В формат вывода добавлено поле ${Status} (шестое поле).
· Вместо grep и cut используется awk, который надёжно фильтрует строки по шестому полю (статус должен содержать "install ok installed") и выводит только первые пять полей (имя, версия, архитектура, размер, описание).
· Такой подход гарантирует, что в deb_packages попадут только действительно установленные пакеты.

Для Oracle Linux (RedHat) изменений не требуется, так как rpm -qa по умолчанию выводит только установленные пакеты.

Проверка

После исправления запустите плейбук заново:

```bash
ansible-playbook -i inventory.ini collect_packages.yml
```

Убедитесь, что в отчётах появились пакеты с хостов Ubuntu 22.04 и 24.04.

Если проблема сохраняется, проверьте:

· Доступность хостов и корректность подключения (ansible_user, права sudo).
· Что на целевых хостах установлен dpkg (обычно есть всегда).
· Что в выводе dpkg-query -W действительно присутствует статус (можно выполнить команду вручную на одном из хостов).

Данное исправление также автоматически повлияет на формирование JSON и CSV отчётов, так как данные для них берутся из deb_packages.