Вот пример Ansible роли для сбора информации о серверах и генерации CSV отчета с хеш-файлом.

Структура роли

```
server-info-collector/
├── tasks/
│   └── main.yml
├── templates/
│   └── report.j2
├── defaults/
│   └── main.yml
└── meta/
    └── main.yml
```

defaults/main.yml

```yaml
---
# Пакеты для проверки (регулярные выражения)
software_filter:
  - "python3"
  - "docker"
  - "nginx"
  - "apache2"
  - "httpd"
  - "mysql"
  - "postgresql"
  - "node"
  - "java"
  - "ruby"

# Путь для сохранения отчета
report_path: "/tmp/server_report"
output_filename: "server_inventory"

# Формат времени для отчета
timestamp_format: "%Y-%m-%d_%H-%M-%S"
```

tasks/main.yml

```yaml
---
- name: Gather system facts
  setup:
    gather_subset:
      - hardware
      - network
      - virtual
      - os
    filter:
      - "ansible_hostname"
      - "ansible_distribution"
      - "ansible_distribution_version"
      - "ansible_architecture"
      - "ansible_processor*"
      - "ansible_memtotal_mb"
      - "ansible_swaptotal_mb"
      - "ansible_devices"
      - "ansible_mounts"

- name: Collect disk information
  set_fact:
    disk_info: |
      {% set disks = [] %}
      {% for mount in ansible_mounts %}
      {% if disks.append({"mount": mount.mount, "size_gb": (mount.size_total // (1024**3)) | round(2)}) %}{% endif %}
      {% endfor %}
      {{ disks }}

- name: Collect installed software
  block:
    - name: Get installed packages (Debian/Ubuntu)
      package_facts:
        manager: auto
      when: ansible_pkg_mgr == "apt"

    - name: Get installed packages (RHEL/CentOS)
      package_facts:
        manager: auto
      when: ansible_pkg_mgr == "yum"

    - name: Filter software based on patterns
      set_fact:
        filtered_software: |
          {% set software_list = [] %}
          {% for pattern in software_filter %}
            {% for pkg_name, pkg_info in ansible_facts.packages.items() %}
              {% if pattern | lower in pkg_name | lower or pkg_name | lower in pattern | lower %}
                {% if software_list.append({"name": pkg_name, "version": pkg_info[0].version}) %}{% endif %}
              {% endif %}
            {% endfor %}
          {% endfor %}
          {{ software_list | unique }}

  rescue:
    - name: Fallback software collection using shell
      shell: |
        {% if ansible_pkg_mgr == "apt" %}
        dpkg-query -W -f='${Package} ${Version}\n' | grep -iE "{{ software_filter | join('|') }}" || true
        {% elif ansible_pkg_mgr == "yum" %}
        rpm -qa --queryformat '%{NAME} %{VERSION}\n' | grep -iE "{{ software_filter | join('|') }}" || true
        {% else %}
        echo "Unknown package manager"
        {% endif %}
      register: software_output
      changed_when: false

    - name: Process fallback software data
      set_fact:
        filtered_software: |
          {% set software_list = [] %}
          {% for line in software_output.stdout_lines %}
            {% if line.strip() %}
              {% set parts = line.split() %}
              {% if parts | length >= 2 %}
                {% set pkg_name = parts[0] %}
                {% set pkg_version = parts[1] %}
                {% if software_list.append({"name": pkg_name, "version": pkg_version}) %}{% endif %}
              {% endif %}
            {% endfor %}
          {{ software_list }}

- name: Create report directory
  file:
    path: "{{ report_path }}"
    state: directory
    mode: '0755'
  delegate_to: localhost

- name: Generate CSV report
  template:
    src: report.j2
    dest: "{{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv"
  delegate_to: localhost

- name: Generate checksum file
  shell: |
    cd {{ report_path }} && 
    sha256sum {{ output_filename }}_{{ ansible_date_time.epoch }}.csv > {{ output_filename }}_{{ ansible_date_time.epoch }}.csv.sha256
  delegate_to: localhost

- name: Display report location
  debug:
    msg: |
      Report generated: {{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv
      Checksum file: {{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv.sha256
```

templates/report.j2

```csv
Hostname,Distribution,Version,Architecture,CPU Model,CPU Cores,Total Memory (MB),Total Swap (MB),Disks,Software
"{{ ansible_hostname }}","{{ ansible_distribution }}","{{ ansible_distribution_version }}","{{ ansible_architecture }}","{{ ansible_processor[1] if ansible_processor[0] == '0' else ansible_processor[0] }}","{{ ansible_processor_cores }}","{{ ansible_memtotal_mb }}","{{ ansible_swaptotal_mb }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for software in filtered_software %}{{ software.name }}-{{ software.version }}{% if not loop.last %}; {% endif %}{% endfor %}"
```

meta/main.yml

```yaml
---
galaxy_info:
  author: Your Name
  description: Server information collection role
  company: Your Company
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: EL
      versions:
        - 7
        - 8
        - 9
    - name: Ubuntu
      versions:
        - bionic
        - focal
        - jammy
    - name: Debian
      versions:
        - buster
        - bullseye
        - bookworm
  galaxy_tags:
    - system
    - inventory
    - reporting

dependencies: []
```

Playbook для использования роли

Создайте playbook.yml:

```yaml
---
- name: Collect server information
  hosts: all
  become: yes
  roles:
    - server-info-collector
  vars:
    software_filter:
      - "python"
      - "docker"
      - "nginx"
      - "mysql"
      - "postgres"
      - "java"
      - "node"
      - "ruby"
```

Использование

1. Создайте инвентарь hosts.ini:

```ini
[servers]
server1.example.com
server2.example.com
server3.example.com

[servers:vars]
ansible_user=your_username
ansible_ssh_private_key_file=~/.ssh/your_key
```

1. Запустите плейбук:

```bash
ansible-playbook -i hosts.ini playbook.yml
```

Результат

После выполнения будут созданы файлы:

· /tmp/server_report/server_inventory_[timestamp].csv - CSV отчет
· /tmp/server_report/server_inventory_[timestamp].csv.sha256 - хеш-файл

Пример содержимого CSV

```csv
Hostname,Distribution,Version,Architecture,CPU Model,CPU Cores,Total Memory (MB),Total Swap (MB),Disks,Software
"webserver01","Ubuntu","22.04","x86_64","Intel(R) Xeon(R) CPU E5-2680 v4",8,16384,2048,"/:50.0GB; /data:100.0GB","python3-3.10.6; docker-20.10.12; nginx-1.18.0"
```

Эта роль предоставляет гибкую систему для сбора информации о серверах с возможностью настройки фильтров для ПО и генерации проверяемых отчетов.



Вот исправленная Ansible роль с корректными значениями для gather_subset:

Структура роли

```
server-info-collector/
├── tasks/
│   └── main.yml
├── templates/
│   └── report.j2
├── defaults/
│   └── main.yml
└── meta/
    └── main.yml
```

defaults/main.yml

```yaml
---
# Пакеты для проверки (регулярные выражения)
software_filter:
  - "python3"
  - "docker"
  - "nginx"
  - "apache2"
  - "httpd"
  - "mysql"
  - "postgresql"
  - "node"
  - "java"
  - "ruby"

# Путь для сохранения отчета
report_path: "/tmp/server_report"
output_filename: "server_inventory"

# Формат времени для отчета
timestamp_format: "%Y-%m-%d_%H-%M-%S"
```

tasks/main.yml

```yaml
---
- name: Gather system facts
  setup:
    gather_subset:
      - "!all"           # Исключаем сбор всех фактов
      - "!min"           # Исключаем минимальный набор
      - hardware
      - network
      - virtual
      - distribution
      - distribution_version
      - architecture
      - processor
      - devices
      - mounts
    filter:
      - "ansible_hostname"
      - "ansible_distribution"
      - "ansible_distribution_version"
      - "ansible_architecture"
      - "ansible_processor*"
      - "ansible_memtotal_mb"
      - "ansible_swaptotal_mb"
      - "ansible_devices"
      - "ansible_mounts"
      - "ansible_virtualization_type"
      - "ansible_virtualization_role"

- name: Debug gathered facts
  debug:
    var: ansible_facts
    verbosity: 1

- name: Collect disk information
  set_fact:
    disk_info: |
      {% set disks = [] %}
      {% for mount in ansible_mounts %}
        {% if disks.append({"mount": mount.mount, "size_gb": (mount.size_total // (1024**3)) | round(2)}) %}{% endif %}
      {% endfor %}
      {{ disks }}

- name: Get CPU model information
  set_fact:
    cpu_model: "{{ ansible_processor[1] if ansible_processor[0] == '0' else ansible_processor[0] }}"

- name: Collect installed software
  block:
    - name: Get installed packages (Debian/Ubuntu)
      package_facts:
        manager: auto
      when: ansible_pkg_mgr == "apt"

    - name: Get installed packages (RHEL/CentOS)
      package_facts:
        manager: auto
      when: ansible_pkg_mgr == "yum"

    - name: Filter software based on patterns
      set_fact:
        filtered_software: |
          {% set software_list = [] %}
          {% if ansible_facts.packages is defined %}
            {% for pattern in software_filter %}
              {% for pkg_name, pkg_info in ansible_facts.packages.items() %}
                {% if pattern | lower in pkg_name | lower or pkg_name | lower in pattern | lower %}
                  {% if software_list.append({"name": pkg_name, "version": pkg_info[0].version}) %}{% endif %}
                {% endif %}
              {% endfor %}
            {% endfor %}
          {% endif %}
          {{ software_list | unique }}

  rescue:
    - name: Fallback software collection using shell
      shell: |
        {% if ansible_pkg_mgr == "apt" %}
        dpkg-query -W -f='${Package} ${Version}\n' 2>/dev/null | grep -iE "{{ software_filter | join('|') }}" || true
        {% elif ansible_pkg_mgr == "yum" %}
        rpm -qa --queryformat '%{NAME} %{VERSION}\n' 2>/dev/null | grep -iE "{{ software_filter | join('|') }}" || true
        {% else %}
        echo "Unknown package manager"
        {% endif %}
      register: software_output
      changed_when: false

    - name: Process fallback software data
      set_fact:
        filtered_software: |
          {% set software_list = [] %}
          {% for line in software_output.stdout_lines %}
            {% if line.strip() %}
              {% set parts = line.split() %}
              {% if parts | length >= 2 %}
                {% set pkg_name = parts[0] %}
                {% set pkg_version = parts[1] %}
                {% if software_list.append({"name": pkg_name, "version": pkg_version}) %}{% endif %}
              {% endif %}
            {% endif %}
          {% endfor %}
          {{ software_list }}

- name: Ensure filtered_software is defined
  set_fact:
    filtered_software: []
  when: filtered_software is not defined

- name: Create report directory on control node
  file:
    path: "{{ report_path }}"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true

- name: Generate CSV report
  template:
    src: report.j2
    dest: "{{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv"
  delegate_to: localhost
  run_once: false

- name: Generate checksum file
  shell: |
    cd "{{ report_path }}" && 
    sha256sum "{{ output_filename }}_{{ ansible_date_time.epoch }}.csv" > "{{ output_filename }}_{{ ansible_date_time.epoch }}.csv.sha256"
  delegate_to: localhost
  run_once: true

- name: Display report location
  debug:
    msg: |
      Report generated: {{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv
      Checksum file: {{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv.sha256
  run_once: true
```

templates/report.j2

```csv
Hostname,Distribution,Version,Architecture,CPU Model,CPU Cores,Total Memory (MB),Total Swap (MB),Virtualization,Disks,Software
"{{ ansible_hostname }}","{{ ansible_distribution }}","{{ ansible_distribution_version }}","{{ ansible_architecture }}","{{ cpu_model | default('N/A') }}","{{ ansible_processor_cores | default(ansible_processor_count) | default('N/A') }}","{{ ansible_memtotal_mb | default('N/A') }}","{{ ansible_swaptotal_mb | default('N/A') }}","{{ ansible_virtualization_type | default('physical') }}/{{ ansible_virtualization_role | default('host') }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for software in filtered_software %}{{ software.name }}-{{ software.version }}{% if not loop.last %}; {% endif %}{% endfor %}"
```

meta/main.yml

```yaml
---
galaxy_info:
  author: Your Name
  description: Server information collection role
  company: Your Company
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: EL
      versions:
        - 7
        - 8
        - 9
    - name: Ubuntu
      versions:
        - bionic
        - focal
        - jammy
    - name: Debian
      versions:
        - buster
        - bullseye
        - bookworm
  galaxy_tags:
    - system
    - inventory
    - reporting

dependencies: []
```

Playbook для использования роли

Создайте playbook.yml:

```yaml
---
- name: Collect server information
  hosts: all
  become: yes
  gather_facts: no  # Мы собираем факты в роли
  roles:
    - server-info-collector
  vars:
    software_filter:
      - "python"
      - "docker"
      - "nginx"
      - "mysql"
      - "postgres"
      - "java"
      - "node"
      - "ruby"
    report_path: "/tmp/server_reports"
    output_filename: "server_inventory_report"
```

Использование

1. Создайте инвентарь hosts.ini:

```ini
[servers]
server1.example.com
server2.example.com

[servers:vars]
ansible_user=your_username
ansible_ssh_private_key_file=~/.ssh/your_key
```

1. Запустите плейбук:

```bash
ansible-playbook -i hosts.ini playbook.yml
```

Основные исправления:

1. Заменил os на корректные значения gather_subset:
   · distribution
   · distribution_version
   · architecture
   · processor
2. Добавил обработку ошибок для случая, когда пакеты не найдены
3. Добавил проверки на существование переменных чтобы избежать ошибок
4. Улучшил шаблон CSV с дополнительными полями и обработкой отсутствующих значений
5. Добавил отладочную задачу для просмотра собранных фактов

Роль теперь должна работать без ошибок и собирать всю необходимую информацию о серверах.





Ошибка возникает из-за проблемы с переменной ansible_mounts. Давайте исправим роль, улучшив обработку данных о дисках и добавив проверки:

Исправленная tasks/main.yml

```yaml
---
- name: Gather system facts
  setup:
    gather_subset:
      - "!all"
      - "!min"
      - hardware
      - network
      - virtual
      - distribution
      - distribution_version
      - architecture
      - processor
      - devices
      - mounts
    filter:
      - "ansible_hostname"
      - "ansible_distribution"
      - "ansible_distribution_version"
      - "ansible_architecture"
      - "ansible_processor*"
      - "ansible_memtotal_mb"
      - "ansible_swaptotal_mb"
      - "ansible_devices"
      - "ansible_mounts"
      - "ansible_virtualization_type"
      - "ansible_virtualization_role"

- name: Check if mounts facts are available
  debug:
    msg: "Mounts data type: {{ ansible_mounts | type_debug }}, value: {{ ansible_mounts }}"
  verbosity: 1

- name: Collect disk information safely
  set_fact:
    disk_info: |
      {%- set disks = [] -%}
      {%- if ansible_mounts is defined and ansible_mounts -%}
        {%- for mount in ansible_mounts -%}
          {%- if mount is mapping and mount.mount is defined and mount.size_total is defined -%}
            {%- set size_gb = (mount.size_total | int // (1024**3)) | round(2) -%}
            {%- if disks.append({"mount": mount.mount, "size_gb": size_gb}) -%}{%- endif -%}
          {%- endif -%}
        {%- endfor -%}
      {%- endif -%}
      {{ disks }}

- name: Get CPU model information safely
  set_fact:
    cpu_model: |
      {%- if ansible_processor is defined -%}
        {%- if ansible_processor[1] is defined and ansible_processor[0] == '0' -%}
          {{ ansible_processor[1] }}
        {%- else -%}
          {{ ansible_processor[0] }}
        {%- endif -%}
      {%- else -%}
        "N/A"
      {%- endif -%}

- name: Get CPU cores count safely
  set_fact:
    cpu_cores: "{{ ansible_processor_cores | default(ansible_processor_count | default('N/A')) }}"

- name: Collect installed software
  block:
    - name: Get installed packages
      package_facts:
        manager: auto

    - name: Filter software based on patterns
      set_fact:
        filtered_software: |
          {%- set software_list = [] -%}
          {%- if ansible_facts.packages is defined -%}
            {%- for pattern in software_filter -%}
              {%- for pkg_name, pkg_info in ansible_facts.packages.items() -%}
                {%- if pattern | lower in pkg_name | lower or pkg_name | lower in pattern | lower -%}
                  {%- if software_list.append({"name": pkg_name, "version": pkg_info[0].version}) -%}{%- endif -%}
                {%- endif -%}
              {%- endfor -%}
            {%- endfor -%}
          {%- endif -%}
          {{ software_list | unique }}

  rescue:
    - name: Fallback software collection using shell
      shell: |
        {%- if ansible_pkg_mgr == "apt" -%}
        dpkg-query -W -f='${Package} ${Version}\n' 2>/dev/null | grep -iE "{{ software_filter | join('|') }}" || true
        {%- elif ansible_pkg_mgr == "yum" -%}
        rpm -qa --queryformat '%{NAME} %{VERSION}\n' 2>/dev/null | grep -iE "{{ software_filter | join('|') }}" || true
        {%- else -%}
        echo "Unknown package manager"
        {%- endif -%}
      register: software_output
      changed_when: false

    - name: Process fallback software data
      set_fact:
        filtered_software: |
          {%- set software_list = [] -%}
          {%- for line in software_output.stdout_lines -%}
            {%- if line.strip() -%}
              {%- set parts = line.strip().split() -%}
              {%- if parts | length >= 2 -%}
                {%- set pkg_name = parts[0] -%}
                {%- set pkg_version = parts[1] -%}
                {%- if software_list.append({"name": pkg_name, "version": pkg_version}) -%}{%- endif -%}
              {%- endif -%}
            {%- endif -%}
          {%- endfor -%}
          {{ software_list }}

- name: Ensure filtered_software is defined
  set_fact:
    filtered_software: []
  when: filtered_software is not defined

- name: Set default values for missing facts
  set_fact:
    safe_hostname: "{{ ansible_hostname | default('Unknown') }}"
    safe_distribution: "{{ ansible_distribution | default('Unknown') }}"
    safe_distribution_version: "{{ ansible_distribution_version | default('Unknown') }}"
    safe_architecture: "{{ ansible_architecture | default('Unknown') }}"
    safe_memory: "{{ ansible_memtotal_mb | default(0) }}"
    safe_swap: "{{ ansible_swaptotal_mb | default(0) }}"
    safe_virtualization_type: "{{ ansible_virtualization_type | default('physical') }}"
    safe_virtualization_role: "{{ ansible_virtualization_role | default('host') }}"

- name: Create report directory on control node
  file:
    path: "{{ report_path }}"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true

- name: Generate CSV report
  template:
    src: report.j2
    dest: "{{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv"
  delegate_to: localhost
  run_once: false

- name: Generate checksum file
  shell: |
    cd "{{ report_path }}" && 
    sha256sum "{{ output_filename }}_{{ ansible_date_time.epoch }}.csv" > "{{ output_filename }}_{{ ansible_date_time.epoch }}.csv.sha256"
  delegate_to: localhost
  run_once: true

- name: Display report location
  debug:
    msg: |
      Report generated: {{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv
      Checksum file: {{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv.sha256
  run_once: true
```

Исправленный templates/report.j2

```csv
Hostname,Distribution,Version,Architecture,CPU Model,CPU Cores,Total Memory (MB),Total Swap (MB),Virtualization,Disks,Software
"{{ safe_hostname }}","{{ safe_distribution }}","{{ safe_distribution_version }}","{{ safe_architecture }}","{{ cpu_model }}","{{ cpu_cores }}","{{ safe_memory }}","{{ safe_swap }}","{{ safe_virtualization_type }}/{{ safe_virtualization_role }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for software in filtered_software %}{{ software.name }}-{{ software.version }}{% if not loop.last %}; {% endif %}{% endfor %}"
```

Альтернативная версия для проблемных систем

Если проблема с ansible_mounts persists, добавьте эту задачу как альтернативу:

```yaml
- name: Collect disk information using shell command (fallback)
  block:
    - name: Get disk info via df command
      shell: df -h --output=target,size | grep -v 'Mounted on' | awk '{print $1 ":" $2}'
      register: disk_output
      changed_when: false
    
    - name: Process disk output
      set_fact:
        disk_info_fallback: |
          {%- set disks = [] -%}
          {%- for line in disk_output.stdout_lines -%}
            {%- if ':' in line -%}
              {%- set parts = line.split(':') -%}
              {%- if parts | length >= 2 -%}
                {%- if disks.append({"mount": parts[0], "size_gb": parts[1]}) -%}{%- endif -%}
              {%- endif -%}
            {%- endif -%}
          {%- endfor -%}
          {{ disks }}
  
  rescue:
    - name: Set empty disk info on error
      set_fact:
        disk_info_fallback: []
```

И используйте disk_info_fallback в шаблоне, если основной метод не работает.

Основные исправления:

1. Добавлены проверки типа данных для ansible_mounts
2. Улучшена обработка ошибок с помощью is mapping и проверок атрибутов
3. Добавлены безопасные переменные с значениями по умолчанию
4. Улучшена отладка с проверкой типа данных mounts
5. Добавлены альтернативные методы сбора информации о дисках

Теперь роль должна корректно обрабатывать случаи, когда факты о монтированиях недоступны или имеют неожиданный формат.




Ошибка возникает потому что переменная disk_info не определена. Давайте полностью перепишем роль с более надежной обработкой всех возможных ошибок:

Исправленная tasks/main.yml

```yaml
---
- name: Gather system facts
  setup:
    gather_subset:
      - "!all"
      - "!min"
      - hardware
      - network
      - virtual
      - distribution
      - distribution_version
      - architecture
      - processor
      - devices
      - mounts
  register: facts_result
  ignore_errors: yes

- name: Set safe facts with defaults
  set_fact:
    safe_hostname: "{{ ansible_hostname | default('Unknown') }}"
    safe_distribution: "{{ ansible_distribution | default('Unknown') }}"
    safe_distribution_version: "{{ ansible_distribution_version | default('Unknown') }}"
    safe_architecture: "{{ ansible_architecture | default('Unknown') }}"
    safe_memory: "{{ ansible_memtotal_mb | default(0) }}"
    safe_swap: "{{ ansible_swaptotal_mb | default(0) }}"
    safe_virtualization_type: "{{ ansible_virtualization_type | default('physical') }}"
    safe_virtualization_role: "{{ ansible_virtualization_role | default('host') }}"
    safe_processor_cores: "{{ ansible_processor_cores | default(ansible_processor_count | default('N/A')) }}"

- name: Get CPU model safely
  set_fact:
    safe_cpu_model: |
      {%- if ansible_processor is defined -%}
        {%- if ansible_processor[1] is defined and ansible_processor[0] == '0' -%}
          {{ ansible_processor[1] }}
        {%- elif ansible_processor[0] is defined -%}
          {{ ansible_processor[0] }}
        {%- else -%}
          "N/A"
        {%- endif -%}
      {%- else -%}
        "N/A"
      {%- endif -%}

- name: Collect disk information with multiple fallbacks
  block:
    - name: Try to get disk info from mounts facts
      set_fact:
        disk_info: |
          {%- set disks = [] -%}
          {%- if ansible_mounts is defined and ansible_mounts -%}
            {%- for mount in ansible_mounts -%}
              {%- if mount is mapping and mount.mount is defined and mount.size_total is defined -%}
                {%- set size_gb = (mount.size_total | int // (1024**3)) | round(2) -%}
                {%- if disks.append({"mount": mount.mount, "size_gb": size_gb}) -%}{%- endif -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ disks }}
      when: ansible_mounts is defined

    - name: Debug disk info from mounts
      debug:
        var: disk_info
      when: disk_info is defined
      verbosity: 1

  rescue:
    - name: Set empty disk info on error
      set_fact:
        disk_info: []

- name: Fallback disk info using shell command
  block:
    - name: Get disk info via df command
      shell: |
        df -h --output=target,size | tail -n +2 | awk '{gsub(/M|G|T/,"",$2); print $1 ":" $2}'
      register: disk_df
      changed_when: false
      ignore_errors: yes

    - name: Process disk output from df
      set_fact:
        disk_info_fallback: |
          {%- set disks = [] -%}
          {%- if disk_df is defined and disk_df.stdout_lines is defined -%}
            {%- for line in disk_df.stdout_lines -%}
              {%- if ':' in line -%}
                {%- set parts = line.split(':') -%}
                {%- if parts|length >= 2 and parts[0] != '' -%}
                  {%- if disks.append({"mount": parts[0], "size_gb": parts[1] | trim}) -%}{%- endif -%}
                {%- endif -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ disks }}
      when: disk_df is defined and disk_df.stdout is defined

  when: disk_info is not defined or disk_info | length == 0

- name: Ensure disk_info is always defined
  set_fact:
    disk_info: "{{ disk_info_fallback | default([]) }}"
  when: disk_info is not defined

- name: Final disk_info fallback
  set_fact:
    disk_info: []
  when: disk_info is not defined

- name: Debug final disk_info
  debug:
    msg: "Final disk_info: {{ disk_info }}"
  verbosity: 1

- name: Collect installed software
  block:
    - name: Get installed packages
      package_facts:
        manager: auto

    - name: Filter software based on patterns
      set_fact:
        filtered_software: |
          {%- set software_list = [] -%}
          {%- if ansible_facts.packages is defined -%}
            {%- for pattern in software_filter -%}
              {%- for pkg_name, pkg_info in ansible_facts.packages.items() -%}
                {%- if pattern | lower in pkg_name | lower or pkg_name | lower in pattern | lower -%}
                  {%- if software_list.append({"name": pkg_name, "version": pkg_info[0].version}) -%}{%- endif -%}
                {%- endif -%}
              {%- endfor -%}
            {%- endfor -%}
          {%- endif -%}
          {{ software_list | unique }}

  rescue:
    - name: Fallback software collection using shell
      block:
        - name: Get software via apt
          shell: |
            dpkg-query -W -f='${Package} ${Version}\n' 2>/dev/null | grep -iE "{{ software_filter | join('|') }}" || true
          register: software_apt
          when: ansible_pkg_mgr == "apt"

        - name: Get software via yum
          shell: |
            rpm -qa --queryformat '%{NAME} %{VERSION}-%{RELEASE}\n' 2>/dev/null | grep -iE "{{ software_filter | join('|') }}" || true
          register: software_yum
          when: ansible_pkg_mgr == "yum"

        - name: Set software output based on package manager
          set_fact:
            software_output: "{{ software_apt if ansible_pkg_mgr == 'apt' else software_yum }}"

        - name: Process fallback software data
          set_fact:
            filtered_software: |
              {%- set software_list = [] -%}
              {%- if software_output is defined and software_output.stdout_lines is defined -%}
                {%- for line in software_output.stdout_lines -%}
                  {%- if line.strip() -%}
                    {%- set parts = line.strip().split() -%}
                    {%- if parts|length >= 2 -%}
                      {%- set pkg_name = parts[0] -%}
                      {%- set pkg_version = parts[1] -%}
                      {%- if software_list.append({"name": pkg_name, "version": pkg_version}) -%}{%- endif -%}
                    {%- endif -%}
                  {%- endif -%}
                {%- endfor -%}
              {%- endif -%}
              {{ software_list }}

      rescue:
        - name: Set empty software list on complete failure
          set_fact:
            filtered_software: []

- name: Ensure filtered_software is defined
  set_fact:
    filtered_software: []
  when: filtered_software is not defined

- name: Create report directory on control node
  file:
    path: "{{ report_path }}"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true

- name: Generate CSV report
  template:
    src: report.j2
    dest: "{{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv"
  delegate_to: localhost

- name: Generate checksum file
  shell: |
    cd "{{ report_path }}" && 
    sha256sum "{{ output_filename }}_{{ ansible_date_time.epoch }}.csv" > "{{ output_filename }}_{{ ansible_date_time.epoch }}.csv.sha256"
  delegate_to: localhost
  run_once: true

- name: Display report location
  debug:
    msg: |
      Report generated: {{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv
      Checksum file: {{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv.sha256
  run_once: true
```

Исправленный templates/report.j2

```csv
Hostname,Distribution,Version,Architecture,CPU Model,CPU Cores,Total Memory (MB),Total Swap (MB),Virtualization,Disks,Software
"{{ safe_hostname }}","{{ safe_distribution }}","{{ safe_distribution_version }}","{{ safe_architecture }}","{{ safe_cpu_model }}","{{ safe_processor_cores }}","{{ safe_memory }}","{{ safe_swap }}","{{ safe_virtualization_type }}/{{ safe_virtualization_role }}","{% if disk_info is defined %}{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}{% if not disk.size_gb is regex_search('[MGTP]B$') %}GB{% endif %}{% if not loop.last %}; {% endif %}{% endfor %}{% else %}N/A{% endif %}","{% for software in filtered_software %}{{ software.name }}-{{ software.version }}{% if not loop.last %}; {% endif %}{% endfor %}"
```

Альтернативный упрощенный шаблон (если вышеописанный не работает)

Создайте templates/report_simple.j2:

```csv
Hostname,Distribution,Version,Architecture,CPU Model,CPU Cores,Memory (MB),Swap (MB),Virtualization,Disks,Software
"{{ safe_hostname }}","{{ safe_distribution }}","{{ safe_distribution_version }}","{{ safe_architecture }}","{{ safe_cpu_model }}","{{ safe_processor_cores }}","{{ safe_memory }}","{{ safe_swap }}","{{ safe_virtualization_type }}/{{ safe_virtualization_role }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for software in filtered_software %}{{ software.name }}-{{ software.version }}{% if not loop.last %}; {% endif %}{% endfor %}"
```

И в задаче генерации отчета используйте:

```yaml
- name: Generate CSV report
  template:
    src: report_simple.j2
    dest: "{{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv"
  delegate_to: localhost
```

Основные исправления:

1. Добавлена регистрация результатов сбора фактов с ignore_errors: yes
2. Множественные fallback'и для disk_info с разными методами сбора
3. Гарантированное определение всех переменных перед использованием в шаблоне
4. Улучшенная обработка ошибок для сбора информации о ПО
5. Безопасный шаблон с проверками на существование переменных

Эта версия роли должна надежно работать даже при проблемах со сбором отдельных фактов и гарантированно генерировать CSV отчет.

Ошибка возникает из-за неправильного синтаксиса в задаче debug. Параметр verbosity должен быть внутри модуля debug, а не как отдельный параметр. Давайте исправим роль:

Исправленная tasks/main.yml

```yaml
---
- name: Gather system facts
  setup:
    gather_subset:
      - "!all"
      - "!min"
      - hardware
      - network
      - virtual
      - distribution
      - distribution_version
      - architecture
      - processor
      - devices
      - mounts
  register: facts_result
  ignore_errors: yes

- name: Set safe facts with defaults
  set_fact:
    safe_hostname: "{{ ansible_hostname | default('Unknown') }}"
    safe_distribution: "{{ ansible_distribution | default('Unknown') }}"
    safe_distribution_version: "{{ ansible_distribution_version | default('Unknown') }}"
    safe_architecture: "{{ ansible_architecture | default('Unknown') }}"
    safe_memory: "{{ ansible_memtotal_mb | default(0) }}"
    safe_swap: "{{ ansible_swaptotal_mb | default(0) }}"
    safe_virtualization_type: "{{ ansible_virtualization_type | default('physical') }}"
    safe_virtualization_role: "{{ ansible_virtualization_role | default('host') }}"
    safe_processor_cores: "{{ ansible_processor_cores | default(ansible_processor_count | default('N/A')) }}"

- name: Get CPU model safely
  set_fact:
    safe_cpu_model: |
      {%- if ansible_processor is defined -%}
        {%- if ansible_processor[1] is defined and ansible_processor[0] == '0' -%}
          {{ ansible_processor[1] }}
        {%- elif ansible_processor[0] is defined -%}
          {{ ansible_processor[0] }}
        {%- else -%}
          "N/A"
        {%- endif -%}
      {%- else -%}
        "N/A"
      {%- endif -%}

- name: Collect disk information with multiple fallbacks
  block:
    - name: Try to get disk info from mounts facts
      set_fact:
        disk_info: |
          {%- set disks = [] -%}
          {%- if ansible_mounts is defined and ansible_mounts -%}
            {%- for mount in ansible_mounts -%}
              {%- if mount is mapping and mount.mount is defined and mount.size_total is defined -%}
                {%- set size_gb = (mount.size_total | int // (1024**3)) | round(2) -%}
                {%- if disks.append({"mount": mount.mount, "size_gb": size_gb}) -%}{%- endif -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ disks }}
      when: ansible_mounts is defined

    - name: Debug disk info from mounts
      debug:
        var: disk_info
        verbosity: 1
      when: disk_info is defined

  rescue:
    - name: Set empty disk info on error
      set_fact:
        disk_info: []

- name: Fallback disk info using shell command
  block:
    - name: Get disk info via df command
      shell: |
        df -h --output=target,size | tail -n +2 | awk '{gsub(/M|G|T/,"",$2); print $1 ":" $2}'
      register: disk_df
      changed_when: false
      ignore_errors: yes

    - name: Process disk output from df
      set_fact:
        disk_info_fallback: |
          {%- set disks = [] -%}
          {%- if disk_df is defined and disk_df.stdout_lines is defined -%}
            {%- for line in disk_df.stdout_lines -%}
              {%- if ':' in line -%}
                {%- set parts = line.split(':') -%}
                {%- if parts|length >= 2 and parts[0] != '' -%}
                  {%- if disks.append({"mount": parts[0], "size_gb": parts[1] | trim}) -%}{%- endif -%}
                {%- endif -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ disks }}
      when: disk_df is defined and disk_df.stdout is defined

  when: disk_info is not defined or disk_info | length == 0

- name: Ensure disk_info is always defined
  set_fact:
    disk_info: "{{ disk_info_fallback | default([]) }}"
  when: disk_info is not defined

- name: Final disk_info fallback
  set_fact:
    disk_info: []
  when: disk_info is not defined

- name: Debug final disk_info
  debug:
    msg: "Final disk_info: {{ disk_info }}"
    verbosity: 1

- name: Collect installed software
  block:
    - name: Get installed packages
      package_facts:
        manager: auto

    - name: Filter software based on patterns
      set_fact:
        filtered_software: |
          {%- set software_list = [] -%}
          {%- if ansible_facts.packages is defined -%}
            {%- for pattern in software_filter -%}
              {%- for pkg_name, pkg_info in ansible_facts.packages.items() -%}
                {%- if pattern | lower in pkg_name | lower or pkg_name | lower in pattern | lower -%}
                  {%- if software_list.append({"name": pkg_name, "version": pkg_info[0].version}) -%}{%- endif -%}
                {%- endif -%}
              {%- endfor -%}
            {%- endfor -%}
          {%- endif -%}
          {{ software_list | unique }}

  rescue:
    - name: Fallback software collection using shell
      block:
        - name: Get software via apt
          shell: |
            dpkg-query -W -f='${Package} ${Version}\n' 2>/dev/null | grep -iE "{{ software_filter | join('|') }}" || true
          register: software_apt
          changed_when: false
          ignore_errors: yes
          when: ansible_pkg_mgr == "apt"

        - name: Get software via yum
          shell: |
            rpm -qa --queryformat '%{NAME} %{VERSION}-%{RELEASE}\n' 2>/dev/null | grep -iE "{{ software_filter | join('|') }}" || true
          register: software_yum
          changed_when: false
          ignore_errors: yes
          when: ansible_pkg_mgr == "yum"

        - name: Set software output based on package manager
          set_fact:
            software_output: "{{ software_apt if ansible_pkg_mgr == 'apt' else software_yum }}"
          when: ansible_pkg_mgr in ['apt', 'yum']

        - name: Process fallback software data
          set_fact:
            filtered_software: |
              {%- set software_list = [] -%}
              {%- if software_output is defined and software_output.stdout_lines is defined -%}
                {%- for line in software_output.stdout_lines -%}
                  {%- if line.strip() -%}
                    {%- set parts = line.strip().split() -%}
                    {%- if parts|length >= 2 -%}
                      {%- set pkg_name = parts[0] -%}
                      {%- set pkg_version = parts[1] -%}
                      {%- if software_list.append({"name": pkg_name, "version": pkg_version}) -%}{%- endif -%}
                    {%- endif -%}
                  {%- endif -%}
                {%- endfor -%}
              {%- endif -%}
              {{ software_list }}

      rescue:
        - name: Set empty software list on complete failure
          set_fact:
            filtered_software: []

- name: Ensure filtered_software is defined
  set_fact:
    filtered_software: []
  when: filtered_software is not defined

- name: Create report directory on control node
  file:
    path: "{{ report_path }}"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true

- name: Generate CSV report
  template:
    src: report.j2
    dest: "{{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv"
  delegate_to: localhost

- name: Generate checksum file
  shell: |
    cd "{{ report_path }}" && 
    sha256sum "{{ output_filename }}_{{ ansible_date_time.epoch }}.csv" > "{{ output_filename }}_{{ ansible_date_time.epoch }}.csv.sha256"
  delegate_to: localhost
  run_once: true

- name: Display report location
  debug:
    msg: |
      Report generated: {{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv
      Checksum file: {{ report_path }}/{{ output_filename }}_{{ ansible_date_time.epoch }}.csv.sha256
  run_once: true
```

Упрощенный templates/report.j2

```csv
Hostname,Distribution,Version,Architecture,CPU Model,CPU Cores,Memory (MB),Swap (MB),Virtualization,Disks,Software
"{{ safe_hostname }}","{{ safe_distribution }}","{{ safe_distribution_version }}","{{ safe_architecture }}","{{ safe_cpu_model }}","{{ safe_processor_cores }}","{{ safe_memory }}","{{ safe_swap }}","{{ safe_virtualization_type }}/{{ safe_virtualization_role }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for software in filtered_software %}{{ software.name }}-{{ software.version }}{% if not loop.last %}; {% endif %}{% endfor %}"
```

Основные исправления:

1. Исправил синтаксис debug задач - переместил verbosity внутрь модуля debug
2. Добавил changed_when: false для задач shell чтобы избежать ложных срабатываний изменений
3. Добавил ignore_errors: yes в критические задачи чтобы роль продолжала выполняться
4. Упростил шаблон чтобы избежать сложных условных конструкций

Для быстрого тестирования также можно временно убрать все debug задачи:

```yaml
# Закомментируйте или удалите эти задачи для тестирования:
# - name: Debug disk info from mounts
#   debug:
#     var: disk_info
#     verbosity: 1
#   when: disk_info is defined

# - name: Debug final disk_info
#   debug:
#     msg: "Final disk_info: {{ disk_info }}"
#     verbosity: 1
```

Теперь роль должна работать без синтаксических ошибок и надежно собирать информацию даже при частичных сбоях.




Модифицируем роль для сбора отчета в один общий файл. Вот исправленная версия:

Исправленная tasks/main.yml

```yaml
---
- name: Gather system facts
  setup:
    gather_subset:
      - "!all"
      - "!min"
      - hardware
      - network
      - virtual
      - distribution
      - distribution_version
      - architecture
      - processor
      - devices
      - mounts
  register: facts_result
  ignore_errors: yes

- name: Set safe facts with defaults
  set_fact:
    safe_hostname: "{{ ansible_hostname | default('Unknown') }}"
    safe_distribution: "{{ ansible_distribution | default('Unknown') }}"
    safe_distribution_version: "{{ ansible_distribution_version | default('Unknown') }}"
    safe_architecture: "{{ ansible_architecture | default('Unknown') }}"
    safe_memory: "{{ ansible_memtotal_mb | default(0) }}"
    safe_swap: "{{ ansible_swaptotal_mb | default(0) }}"
    safe_virtualization_type: "{{ ansible_virtualization_type | default('physical') }}"
    safe_virtualization_role: "{{ ansible_virtualization_role | default('host') }}"
    safe_processor_cores: "{{ ansible_processor_cores | default(ansible_processor_count | default('N/A')) }}"

- name: Get CPU model safely
  set_fact:
    safe_cpu_model: |
      {%- if ansible_processor is defined -%}
        {%- if ansible_processor[1] is defined and ansible_processor[0] == '0' -%}
          {{ ansible_processor[1] }}
        {%- elif ansible_processor[0] is defined -%}
          {{ ansible_processor[0] }}
        {%- else -%}
          "N/A"
        {%- endif -%}
      {%- else -%}
        "N/A"
      {%- endif -%}

- name: Collect disk information with multiple fallbacks
  block:
    - name: Try to get disk info from mounts facts
      set_fact:
        disk_info: |
          {%- set disks = [] -%}
          {%- if ansible_mounts is defined and ansible_mounts -%}
            {%- for mount in ansible_mounts -%}
              {%- if mount is mapping and mount.mount is defined and mount.size_total is defined -%}
                {%- set size_gb = (mount.size_total | int // (1024**3)) | round(2) -%}
                {%- if disks.append({"mount": mount.mount, "size_gb": size_gb}) -%}{%- endif -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ disks }}
      when: ansible_mounts is defined

  rescue:
    - name: Set empty disk info on error
      set_fact:
        disk_info: []

- name: Fallback disk info using shell command
  block:
    - name: Get disk info via df command
      shell: |
        df -h --output=target,size | tail -n +2 | awk '{gsub(/M|G|T/,"",$2); print $1 ":" $2}'
      register: disk_df
      changed_when: false
      ignore_errors: yes

    - name: Process disk output from df
      set_fact:
        disk_info_fallback: |
          {%- set disks = [] -%}
          {%- if disk_df is defined and disk_df.stdout_lines is defined -%}
            {%- for line in disk_df.stdout_lines -%}
              {%- if ':' in line -%}
                {%- set parts = line.split(':') -%}
                {%- if parts|length >= 2 and parts[0] != '' -%}
                  {%- if disks.append({"mount": parts[0], "size_gb": parts[1] | trim}) -%}{%- endif -%}
                {%- endif -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ disks }}
      when: disk_df is defined and disk_df.stdout is defined

  when: disk_info is not defined or disk_info | length == 0

- name: Ensure disk_info is always defined
  set_fact:
    disk_info: "{{ disk_info_fallback | default([]) }}"
  when: disk_info is not defined

- name: Final disk_info fallback
  set_fact:
    disk_info: []
  when: disk_info is not defined

- name: Collect installed software
  block:
    - name: Get installed packages
      package_facts:
        manager: auto

    - name: Filter software based on patterns
      set_fact:
        filtered_software: |
          {%- set software_list = [] -%}
          {%- if ansible_facts.packages is defined -%}
            {%- for pattern in software_filter -%}
              {%- for pkg_name, pkg_info in ansible_facts.packages.items() -%}
                {%- if pattern | lower in pkg_name | lower or pkg_name | lower in pattern | lower -%}
                  {%- if software_list.append({"name": pkg_name, "version": pkg_info[0].version}) -%}{%- endif -%}
                {%- endif -%}
              {%- endfor -%}
            {%- endfor -%}
          {%- endif -%}
          {{ software_list | unique }}

  rescue:
    - name: Fallback software collection using shell
      block:
        - name: Get software via apt
          shell: |
            dpkg-query -W -f='${Package} ${Version}\n' 2>/dev/null | grep -iE "{{ software_filter | join('|') }}" || true
          register: software_apt
          changed_when: false
          ignore_errors: yes
          when: ansible_pkg_mgr == "apt"

        - name: Get software via yum
          shell: |
            rpm -qa --queryformat '%{NAME} %{VERSION}-%{RELEASE}\n' 2>/dev/null | grep -iE "{{ software_filter | join('|') }}" || true
          register: software_yum
          changed_when: false
          ignore_errors: yes
          when: ansible_pkg_mgr == "yum"

        - name: Set software output based on package manager
          set_fact:
            software_output: "{{ software_apt if ansible_pkg_mgr == 'apt' else software_yum }}"
          when: ansible_pkg_mgr in ['apt', 'yum']

        - name: Process fallback software data
          set_fact:
            filtered_software: |
              {%- set software_list = [] -%}
              {%- if software_output is defined and software_output.stdout_lines is defined -%}
                {%- for line in software_output.stdout_lines -%}
                  {%- if line.strip() -%}
                    {%- set parts = line.strip().split() -%}
                    {%- if parts|length >= 2 -%}
                      {%- set pkg_name = parts[0] -%}
                      {%- set pkg_version = parts[1] -%}
                      {%- if software_list.append({"name": pkg_name, "version": pkg_version}) -%}{%- endif -%}
                    {%- endif -%}
                  {%- endif -%}
                {%- endfor -%}
              {%- endif -%}
              {{ software_list }}

      rescue:
        - name: Set empty software list on complete failure
          set_fact:
            filtered_software: []

- name: Ensure filtered_software is defined
  set_fact:
    filtered_software: []
  when: filtered_software is not defined

- name: Create report directory on control node
  file:
    path: "{{ report_path }}"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true

- name: Add host data to consolidated report
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: |
      "{{ safe_hostname }}","{{ safe_distribution }}","{{ safe_distribution_version }}","{{ safe_architecture }}","{{ safe_cpu_model }}","{{ safe_processor_cores }}","{{ safe_memory }}","{{ safe_swap }}","{{ safe_virtualization_type }}/{{ safe_virtualization_role }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for software in filtered_software %}{{ software.name }}-{{ software.version }}{% if not loop.last %}; {% endif %}{% endfor %}"
    create: yes
    insertbefore: BOF
  delegate_to: localhost
  run_once: false

- name: Add CSV header (only once)
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: "Hostname,Distribution,Version,Architecture,CPU Model,CPU Cores,Memory (MB),Swap (MB),Virtualization,Disks,Software"
    create: yes
    insertbefore: BOF
  delegate_to: localhost
  run_once: true
  when: inventory_hostname == play_hosts[0]

- name: Generate checksum file for consolidated report
  shell: |
    cd "{{ report_path }}" && 
    sha256sum "{{ output_filename }}.csv" > "{{ output_filename }}.csv.sha256"
  delegate_to: localhost
  run_once: true

- name: Display consolidated report location
  debug:
    msg: |
      Consolidated report generated: {{ report_path }}/{{ output_filename }}.csv
      Checksum file: {{ report_path }}/{{ output_filename }}.csv.sha256
  delegate_to: localhost
  run_once: true
```

Обновленный defaults/main.yml

```yaml
---
# Пакеты для проверки (регулярные выражения)
software_filter:
  - "python3"
  - "docker"
  - "nginx"
  - "apache2"
  - "httpd"
  - "mysql"
  - "postgresql"
  - "node"
  - "java"
  - "ruby"

# Путь для сохранения отчета
report_path: "/tmp/server_report"
output_filename: "servers_consolidated_report"

# Формат времени для отчета
timestamp_format: "%Y-%m-%d_%H-%M-%S"
```

Playbook для использования роли

```yaml
---
- name: Collect consolidated server information
  hosts: all
  become: yes
  gather_facts: no
  roles:
    - server-info-collector
  vars:
    software_filter:
      - "python"
      - "docker"
      - "nginx"
      - "mysql"
      - "postgres"
      - "java"
      - "node"
      - "ruby"
    report_path: "/tmp/server_reports"
    output_filename: "all_servers_inventory"
```

Основные изменения:

1. Замена шаблона на lineinfile - вместо генерации отдельных файлов через шаблон, используем lineinfile для добавления данных каждого сервера в один файл
2. Добавление заголовка только один раз - задача добавляет CSV заголовок только при первом выполнении на первом хосте
3. Консолидированный файл - все данные собираются в один файл servers_consolidated_report.csv
4. Удалены отладочные задачи - убраны задачи debug чтобы избежать синтаксических ошибок
5. Один checksum файл - генерируется один хеш-файл для консолидированного отчета

Как это работает:

1. При первом запуске на первом хосте создается CSV файл с заголовком
2. Каждый хост добавляет свою строку с данными в начало файла (используется insertbefore: BOF)
3. В результате получается один файл со всеми серверами
4. Генерируется один checksum файл для всего отчета

Использование:

```bash
ansible-playbook -i hosts.ini playbook.yml
```

После выполнения все данные будут в одном файле: /tmp/server_reports/all_servers_inventory.csv