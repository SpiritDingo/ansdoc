Исправляем ошибку с неправильным синтаксисом debug задачи. Вот исправленная версия:

Исправленный tasks/main.yml

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
    safe_processor_count: "{{ ansible_processor_count | default('N/A') }}"
    safe_processor_threads_per_core: "{{ ansible_processor_threads_per_core | default('N/A') }}"
    safe_processor_vcpus: "{{ ansible_processor_vcpus | default('N/A') }}"

- name: Get detailed CPU information
  set_fact:
    safe_cpu_model: |
      {%- if ansible_processor is defined -%}
        {%- if ansible_processor[2] is defined -%}
          {{ ansible_processor[2] }}
        {%- elif ansible_processor[1] is defined and ansible_processor[0] == '0' -%}
          {{ ansible_processor[1] }}
        {%- elif ansible_processor[0] is defined -%}
          {{ ansible_processor[0] }}
        {%- else -%}
          "N/A"
        {%- endif -%}
      {%- else -%}
        "N/A"
      {%- endif -%}

- name: Get CPU model name from lscpu (more accurate)
  block:
    - name: Get CPU model using lscpu
      shell: |
        lscpu | grep "Model name" | cut -d: -f2 | sed 's/^[ \t]*//'
      register: lscpu_model
      changed_when: false
      ignore_errors: yes

    - name: Set CPU model from lscpu if available
      set_fact:
        safe_cpu_model: "{{ lscpu_model.stdout | default(safe_cpu_model) | trim }}"
      when: lscpu_model is defined and lscpu_model.stdout != ""

  rescue:
    - name: Debug lscpu failure
      debug:
        msg: "Failed to get CPU model from lscpu, using default method"

- name: Get detailed CPU architecture information
  block:
    - name: Get CPU architecture details
      shell: |
        lscpu | grep -E "(Architecture|CPU op-mode|Byte Order)" | awk -F: '{print $2}' | sed 's/^[ \t]*//' | tr '\n' ';'
      register: cpu_arch_details
      changed_when: false
      ignore_errors: yes

    - name: Set CPU architecture details
      set_fact:
        safe_cpu_arch_details: "{{ cpu_arch_details.stdout | default('N/A') }}"

    - name: Get CPU sockets and cores details
      shell: |
        echo "Sockets:$(lscpu | grep 'Socket(s)' | awk '{print $2}');Cores per socket:$(lscpu | grep 'Core(s) per socket' | awk '{print $4}');Threads per core:$(lscpu | grep 'Thread(s) per core' | awk '{print $4}')"
      register: cpu_topology
      changed_when: false
      ignore_errors: yes

    - name: Set CPU topology information
      set_fact:
        safe_cpu_topology: "{{ cpu_topology.stdout | default('N/A') }}"

  rescue:
    - name: Set default CPU architecture details
      set_fact:
        safe_cpu_arch_details: "N/A"
        safe_cpu_topology: "N/A"

- name: Collect IP addresses information
  set_fact:
    ip_addresses: |
      {%- set ips = [] -%}
      {%- if ansible_default_ipv4 is defined and ansible_default_ipv4.address is defined -%}
        {%- if ips.append(ansible_default_ipv4.address) -%}{%- endif -%}
      {%- endif -%}
      {%- if include_all_ips and ansible_all_ipv4_addresses is defined -%}
        {%- for ip in ansible_all_ipv4_addresses -%}
          {%- if ip != ansible_default_ipv4.address and ip not in ips -%}
            {%- if ips.append(ip) -%}{%- endif -%}
          {%- endif -%}
        {%- endfor -%}
      {%- endif -%}
      {{ ips }}

- name: Get primary IP address
  set_fact:
    primary_ip: "{{ ansible_default_ipv4.address | default('N/A') }}"

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

- name: Collect service information
  block:
    - name: Gather service facts
      service_facts:

    - name: Filter running services
      set_fact:
        filtered_services: |
          {%- set services_list = [] -%}
          {%- if ansible_facts.services is defined -%}
            {%- for service_name, service_info in ansible_facts.services.items() -%}
              {%- set service_status = service_info.state | default('unknown') -%}
              {%- if service_status == 'running' -%}
                {%- for pattern in service_filter -%}
                  {%- if pattern | lower in service_name | lower -%}
                    {%- if services_list.append(service_name) -%}{%- endif -%}
                  {%- endif -%}
                {%- endfor -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ services_list | unique | sort }}

  rescue:
    - name: Use systemctl to get running services
      shell: |
        systemctl list-units --type=service --state=running --no-legend 2>/dev/null | awk '{print $1}' | while read service; do
          for pattern in {{ service_filter | join(' ') }}; do
            if echo "$service" | grep -qi "$pattern"; then
              echo "$service"
              break
            fi
          done
        done | sort | uniq
      register: systemd_services
      changed_when: false
      ignore_errors: yes

    - name: Set services from systemctl
      set_fact:
        filtered_services: "{{ systemd_services.stdout_lines }}"
      when: systemd_services is defined and systemd_services.stdout_lines is defined

    - name: Set empty services list as final fallback
      set_fact:
        filtered_services: []
      when: filtered_services is not defined

- name: Ensure filtered_services is defined
  set_fact:
    filtered_services: []
  when: filtered_services is not defined

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

- name: Add CSV header (only once)
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: "Hostname,Primary IP,All IPs,Distribution,Version,Architecture,CPU Model,CPU Cores,CPU Sockets,CPU Threads,CPU Topology,CPU Architecture,Memory (MB),Swap (MB),Virtualization,Disks,Running Services,Installed Software"
    create: yes
    insertbefore: BOF
  delegate_to: localhost
  run_once: true
  when: inventory_hostname == play_hosts[0]

- name: Add host data to consolidated report
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: |
      "{{ safe_hostname }}","{{ primary_ip }}","{{ ip_addresses | join('; ') }}","{{ safe_distribution }}","{{ safe_distribution_version }}","{{ safe_architecture }}","{{ safe_cpu_model }}","{{ safe_processor_cores }}","{{ safe_processor_count }}","{{ safe_processor_vcpus }}","{{ safe_cpu_topology }}","{{ safe_cpu_arch_details }}","{{ safe_memory }}","{{ safe_swap }}","{{ safe_virtualization_type }}/{{ safe_virtualization_role }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in filtered_services %}{{ service }}{% if not loop.last %}; {% endif %}{% endfor %}","{% for software in filtered_software %}{{ software.name }}-{{ software.version }}{% if not loop.last %}; {% endif %}{% endfor %}"
    create: yes
  delegate_to: localhost
  run_once: false

- name: Generate checksum for unencrypted report
  shell: |
    cd "{{ report_path }}" && 
    sha256sum "{{ output_filename }}.csv" > "{{ output_filename }}.csv.sha256"
  delegate_to: localhost
  run_once: true
  when: not encrypt_report

- name: Check if report exists before encryption
  stat:
    path: "{{ report_path }}/{{ output_filename }}.csv"
  register: report_stat
  delegate_to: localhost
  run_once: true
  when: encrypt_report

- name: Encrypt report with Ansible Vault using command
  command: >
    ansible-vault encrypt
    {% if vault_password_file %}--vault-password-file {{ vault_password_file }}{% endif %}
    --output "{{ report_path }}/{{ output_filename }}_encrypted.csv"
    "{{ report_path }}/{{ output_filename }}.csv"
  delegate_to: localhost
  run_once: true
  when:
    - encrypt_report
    - report_stat.stat.exists

- name: Generate checksum for encrypted report
  shell: |
    cd "{{ report_path }}" && 
    sha256sum "{{ output_filename }}_encrypted.csv" > "{{ output_filename }}_encrypted.csv.sha256"
  delegate_to: localhost
  run_once: true
  when: 
    - encrypt_report
    - report_stat.stat.exists

- name: Remove unencrypted report after encryption
  file:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    state: absent
  delegate_to: localhost
  run_once: true
  when:
    - encrypt_report
    - remove_unencrypted
    - report_stat.stat.exists

- name: Display report location
  debug:
    msg: |
      {% if encrypt_report and report_stat.stat.exists %}
      Encrypted report generated: {{ report_path }}/{{ output_filename }}_encrypted.csv
      Use 'ansible-vault view {{ report_path }}/{{ output_filename }}_encrypted.csv' to view the report
      {% if vault_password_file != "" %}
      Vault password file: {{ vault_password_file }}
      {% endif %}
      Checksum file: {{ report_path }}/{{ output_filename }}_encrypted.csv.sha256
      {% if remove_unencrypted %}
      Note: Unencrypted version was removed
      {% endif %}
      {% elif encrypt_report and not report_stat.stat.exists %}
      ERROR: Report file not found for encryption
      {% else %}
      Consolidated report generated: {{ report_path }}/{{ output_filename }}.csv
      Checksum file: {{ report_path }}/{{ output_filename }}.csv.sha256
      {% endif %}
  delegate_to: localhost
  run_once: true

- name: Warn about missing vault password file
  debug:
    msg: |
      WARNING: encrypt_report is true but vault_password_file is not specified.
      You will be prompted for the vault password interactively.
  delegate_to: localhost
  run_once: true
  when:
    - encrypt_report
    - vault_password_file == ""
```

Основные исправления:

1. Исправил синтаксис debug задачи - убрал неправильное использование verbosity как параметра задачи
2. Добавил правильное имя задачи для debug в rescue блоке
3. Упростил debug задачу - убрал verbosity, так как это сообщение важно всегда показывать

Улучшенная версия с условным выводом debug

Если вы хотите сохранить verbosity для debug сообщений, используйте правильный синтаксис:

```yaml
- name: Debug lscpu failure with verbosity
  debug:
    msg: "Failed to get CPU model from lscpu, using default method"
    verbosity: 1
```

Но в данном случае я убрал verbosity, так как информация о сбое получения модели CPU важна для диагностики.

Теперь роль должна работать без синтаксических ошибок и собирать полную информацию о CPU, включая:

· Модель процессора
· Количество ядер
· Количество сокетов
· Количество потоков
· Топологию CPU
· Детали архитектуры

Добавим сбор описаний запущенных служб в Ansible роль. Вот модифицированная версия:

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

# Службы для проверки
service_filter:
  - "nginx"
  - "apache2"
  - "httpd"
  - "mysql"
  - "postgresql"
  - "docker"
  - "ssh"
  - "redis"
  - "mongod"
  - "elasticsearch"

# Путь для сохранения отчета
report_path: "/tmp/server_report"
output_filename: "servers_consolidated_report"

# Шифрование отчета
encrypt_report: false
vault_password_file: ""
remove_unencrypted: false

# Настройки сбора IP-адресов
preferred_interface: "eth0"  # Предпочтительный интерфейс для основного IP
include_all_ips: true         # Включать все IP-адреса

# Сбор описаний служб
collect_service_descriptions: true

# Формат времени для отчета
timestamp_format: "%Y-%m-%d_%H-%M-%S"
```

Обновленный tasks/main.yml

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
    safe_processor_count: "{{ ansible_processor_count | default('N/A') }}"
    safe_processor_threads_per_core: "{{ ansible_processor_threads_per_core | default('N/A') }}"
    safe_processor_vcpus: "{{ ansible_processor_vcpus | default('N/A') }}"

- name: Get detailed CPU information
  set_fact:
    safe_cpu_model: |
      {%- if ansible_processor is defined -%}
        {%- if ansible_processor[2] is defined -%}
          {{ ansible_processor[2] }}
        {%- elif ansible_processor[1] is defined and ansible_processor[0] == '0' -%}
          {{ ansible_processor[1] }}
        {%- elif ansible_processor[0] is defined -%}
          {{ ansible_processor[0] }}
        {%- else -%}
          "N/A"
        {%- endif -%}
      {%- else -%}
        "N/A"
      {%- endif -%}

- name: Get CPU model name from lscpu (more accurate)
  block:
    - name: Get CPU model using lscpu
      shell: |
        lscpu | grep "Model name" | cut -d: -f2 | sed 's/^[ \t]*//'
      register: lscpu_model
      changed_when: false
      ignore_errors: yes

    - name: Set CPU model from lscpu if available
      set_fact:
        safe_cpu_model: "{{ lscpu_model.stdout | default(safe_cpu_model) | trim }}"
      when: lscpu_model is defined and lscpu_model.stdout != ""

  rescue:
    - name: Debug lscpu failure
      debug:
        msg: "Failed to get CPU model from lscpu, using default method"

- name: Get detailed CPU architecture information
  block:
    - name: Get CPU architecture details
      shell: |
        lscpu | grep -E "(Architecture|CPU op-mode|Byte Order)" | awk -F: '{print $2}' | sed 's/^[ \t]*//' | tr '\n' ';'
      register: cpu_arch_details
      changed_when: false
      ignore_errors: yes

    - name: Set CPU architecture details
      set_fact:
        safe_cpu_arch_details: "{{ cpu_arch_details.stdout | default('N/A') }}"

    - name: Get CPU sockets and cores details
      shell: |
        echo "Sockets:$(lscpu | grep 'Socket(s)' | awk '{print $2}');Cores per socket:$(lscpu | grep 'Core(s) per socket' | awk '{print $4}');Threads per core:$(lscpu | grep 'Thread(s) per core' | awk '{print $4}')"
      register: cpu_topology
      changed_when: false
      ignore_errors: yes

    - name: Set CPU topology information
      set_fact:
        safe_cpu_topology: "{{ cpu_topology.stdout | default('N/A') }}"

  rescue:
    - name: Set default CPU architecture details
      set_fact:
        safe_cpu_arch_details: "N/A"
        safe_cpu_topology: "N/A"

- name: Collect IP addresses information
  set_fact:
    ip_addresses: |
      {%- set ips = [] -%}
      {%- if ansible_default_ipv4 is defined and ansible_default_ipv4.address is defined -%}
        {%- if ips.append(ansible_default_ipv4.address) -%}{%- endif -%}
      {%- endif -%}
      {%- if include_all_ips and ansible_all_ipv4_addresses is defined -%}
        {%- for ip in ansible_all_ipv4_addresses -%}
          {%- if ip != ansible_default_ipv4.address and ip not in ips -%}
            {%- if ips.append(ip) -%}{%- endif -%}
          {%- endif -%}
        {%- endfor -%}
      {%- endif -%}
      {{ ips }}

- name: Get primary IP address
  set_fact:
    primary_ip: "{{ ansible_default_ipv4.address | default('N/A') }}"

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

- name: Collect service information
  block:
    - name: Gather service facts
      service_facts:

    - name: Filter running services
      set_fact:
        filtered_services: |
          {%- set services_list = [] -%}
          {%- if ansible_facts.services is defined -%}
            {%- for service_name, service_info in ansible_facts.services.items() -%}
              {%- set service_status = service_info.state | default('unknown') -%}
              {%- if service_status == 'running' -%}
                {%- for pattern in service_filter -%}
                  {%- if pattern | lower in service_name | lower -%}
                    {%- if services_list.append({"name": service_name, "status": service_status}) -%}{%- endif -%}
                  {%- endif -%}
                {%- endfor -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ services_list | unique | sort(attribute='name') }}

  rescue:
    - name: Use systemctl to get running services
      shell: |
        systemctl list-units --type=service --state=running --no-legend 2>/dev/null | awk '{print $1}' | while read service; do
          for pattern in {{ service_filter | join(' ') }}; do
            if echo "$service" | grep -qi "$pattern"; then
              echo "$service"
              break
            fi
          done
        done | sort | uniq
      register: systemd_services
      changed_when: false
      ignore_errors: yes

    - name: Set services from systemctl
      set_fact:
        filtered_services: |
          {%- set services_list = [] -%}
          {%- if systemd_services is defined and systemd_services.stdout_lines is defined -%}
            {%- for service_name in systemd_services.stdout_lines -%}
              {%- if services_list.append({"name": service_name, "status": "running"}) -%}{%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ services_list }}

    - name: Set empty services list as final fallback
      set_fact:
        filtered_services: []
      when: filtered_services is not defined

- name: Collect service descriptions
  block:
    - name: Get service descriptions using systemctl
      shell: |
        systemctl show "{{ item.name }}" --property=Description --no-pager 2>/dev/null | grep Description= | cut -d= -f2-
      register: service_descriptions
      loop: "{{ filtered_services }}"
      when: 
        - collect_service_descriptions
        - ansible_service_mgr == "systemd"
      changed_when: false
      ignore_errors: yes

    - name: Combine services with descriptions
      set_fact:
        services_with_descriptions: |
          {%- set services_with_desc = [] -%}
          {%- for service in filtered_services -%}
            {%- set description = '' -%}
            {%- for desc_result in service_descriptions.results -%}
              {%- if desc_result.item.name == service.name and desc_result.stdout != '' -%}
                {%- set description = desc_result.stdout | trim -%}
              {%- endif -%}
            {%- endfor -%}
            {%- if description != '' -%}
              {%- if services_with_desc.append({"name": service.name, "status": service.status, "description": description}) -%}{%- endif -%}
            {%- else -%}
              {%- if services_with_desc.append({"name": service.name, "status": service.status, "description": "No description"}) -%}{%- endif -%}
            {%- endif -%}
          {%- endfor -%}
          {{ services_with_desc }}
      when: collect_service_descriptions and ansible_service_mgr == "systemd"

  rescue:
    - name: Set services without descriptions on error
      set_fact:
        services_with_descriptions: "{{ filtered_services }}"

- name: Ensure services_with_descriptions is defined
  set_fact:
    services_with_descriptions: "{{ filtered_services }}"
  when: services_with_descriptions is not defined

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

- name: Add CSV header (only once)
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: "Hostname,Primary IP,All IPs,Distribution,Version,Architecture,CPU Model,CPU Cores,CPU Sockets,CPU Threads,CPU Topology,CPU Architecture,Memory (MB),Swap (MB),Virtualization,Disks,Running Services,Service Descriptions,Installed Software"
    create: yes
    insertbefore: BOF
  delegate_to: localhost
  run_once: true
  when: inventory_hostname == play_hosts[0]

- name: Add host data to consolidated report
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: |
      "{{ safe_hostname }}","{{ primary_ip }}","{{ ip_addresses | join('; ') }}","{{ safe_distribution }}","{{ safe_distribution_version }}","{{ safe_architecture }}","{{ safe_cpu_model }}","{{ safe_processor_cores }}","{{ safe_processor_count }}","{{ safe_processor_vcpus }}","{{ safe_cpu_topology }}","{{ safe_cpu_arch_details }}","{{ safe_memory }}","{{ safe_swap }}","{{ safe_virtualization_type }}/{{ safe_virtualization_role }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_descriptions %}{{ service.name }}{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_descriptions %}{% if service.description is defined %}{{ service.name }}: {{ service.description }}{% else %}{{ service.name }}{% endif %}{% if not loop.last %}; {% endif %}{% endfor %}","{% for software in filtered_software %}{{ software.name }}-{{ software.version }}{% if not loop.last %}; {% endif %}{% endfor %}"
    create: yes
  delegate_to: localhost
  run_once: false

- name: Generate checksum for unencrypted report
  shell: |
    cd "{{ report_path }}" && 
    sha256sum "{{ output_filename }}.csv" > "{{ output_filename }}.csv.sha256"
  delegate_to: localhost
  run_once: true
  when: not encrypt_report

- name: Check if report exists before encryption
  stat:
    path: "{{ report_path }}/{{ output_filename }}.csv"
  register: report_stat
  delegate_to: localhost
  run_once: true
  when: encrypt_report

- name: Encrypt report with Ansible Vault using command
  command: >
    ansible-vault encrypt
    {% if vault_password_file %}--vault-password-file {{ vault_password_file }}{% endif %}
    --output "{{ report_path }}/{{ output_filename }}_encrypted.csv"
    "{{ report_path }}/{{ output_filename }}.csv"
  delegate_to: localhost
  run_once: true
  when:
    - encrypt_report
    - report_stat.stat.exists

- name: Generate checksum for encrypted report
  shell: |
    cd "{{ report_path }}" && 
    sha256sum "{{ output_filename }}_encrypted.csv" > "{{ output_filename }}_encrypted.csv.sha256"
  delegate_to: localhost
  run_once: true
  when: 
    - encrypt_report
    - report_stat.stat.exists

- name: Remove unencrypted report after encryption
  file:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    state: absent
  delegate_to: localhost
  run_once: true
  when:
    - encrypt_report
    - remove_unencrypted
    - report_stat.stat.exists

- name: Display report location
  debug:
    msg: |
      {% if encrypt_report and report_stat.stat.exists %}
      Encrypted report generated: {{ report_path }}/{{ output_filename }}_encrypted.csv
      Use 'ansible-vault view {{ report_path }}/{{ output_filename }}_encrypted.csv' to view the report
      {% if vault_password_file != "" %}
      Vault password file: {{ vault_password_file }}
      {% endif %}
      Checksum file: {{ report_path }}/{{ output_filename }}_encrypted.csv.sha256
      {% if remove_unencrypted %}
      Note: Unencrypted version was removed
      {% endif %}
      {% elif encrypt_report and not report_stat.stat.exists %}
      ERROR: Report file not found for encryption
      {% else %}
      Consolidated report generated: {{ report_path }}/{{ output_filename }}.csv
      Checksum file: {{ report_path }}/{{ output_filename }}.csv.sha256
      {% endif %}
  delegate_to: localhost
  run_once: true

- name: Warn about missing vault password file
  debug:
    msg: |
      WARNING: encrypt_report is true but vault_password_file is not specified.
      You will be prompted for the vault password interactively.
  delegate_to: localhost
  run_once: true
  when:
    - encrypt_report
    - vault_password_file == ""
```

Основные добавления:

1. Сбор описаний служб

· Добавлена новая переменная collect_service_descriptions для управления сбором описаний
· Используется команда systemctl show --property=Description для получения описаний служб
· Обработка ошибок если описания недоступны

2. Новая структура данных для служб

· filtered_services теперь содержит словари с полями name, status, description
· services_with_descriptions объединяет имена служб с их описаниями

3. Дополнительные столбцы в CSV

· Running Services - список имен запущенных служб
· Service Descriptions - подробные описания служб в формате "имя: описание"

4. Улучшенная обработка служб

· Поддержка как systemd, так и других систем инициализации
· Автоматический fallback если описания недоступны
· Обработка случаев когда служба не имеет описания

Пример вывода в CSV:

```
Hostname,Primary IP,All IPs,...,Running Services,Service Descriptions,...
"server1","192.168.1.10","192.168.1.10;10.0.0.10",...,"nginx;docker;ssh","nginx: A high performance web server; docker: Docker Application Container Engine; ssh: OpenBSD Secure Shell server",...
```

Playbook для использования:

```yaml
---
- name: Collect comprehensive server information with service descriptions
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
    service_filter:
      - "nginx"
      - "apache"
      - "mysql"
      - "docker"
      - "ssh"
    report_path: "/tmp/server_reports"
    output_filename: "all_servers_detailed_inventory"
    collect_service_descriptions: true
    encrypt_report: false
```

Использование:

```bash
# Сбор информации с описаниями служб
ansible-playbook -i hosts.ini playbook.yml

# Без сбора описаний служб (для ускорения)
ansible-playbook -i hosts.ini playbook.yml -e "collect_service_descriptions=false"
```

Теперь отчет будет содержать не только список запущенных служб, но и их подробные описания, что упростит понимание назначения каждой службы на сервере.


Ошибка возникает потому, что переменная ansible_service_mgr не определена. Это происходит из-за того, что мы отключили сбор фактов в плейбуке (gather_facts: no), и факт о менеджере служб не собирается. Давайте исправим это:

Исправленный tasks/main.yml

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

- name: Detect service manager
  block:
    - name: Check for systemd
      shell: |
        if systemctl --version >/dev/null 2>&1; then
          echo "systemd"
        elif initctl --version >/dev/null 2>&1; then
          echo "upstart"
        else
          echo "sysvinit"
        fi
      register: service_mgr_check
      changed_when: false
      ignore_errors: yes

    - name: Set service manager fact
      set_fact:
        service_manager: "{{ service_mgr_check.stdout }}"

  rescue:
    - name: Set default service manager
      set_fact:
        service_manager: "unknown"

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
    safe_processor_count: "{{ ansible_processor_count | default('N/A') }}"
    safe_processor_threads_per_core: "{{ ansible_processor_threads_per_core | default('N/A') }}"
    safe_processor_vcpus: "{{ ansible_processor_vcpus | default('N/A') }}"
    safe_service_manager: "{{ service_manager }}"

- name: Get detailed CPU information
  set_fact:
    safe_cpu_model: |
      {%- if ansible_processor is defined -%}
        {%- if ansible_processor[2] is defined -%}
          {{ ansible_processor[2] }}
        {%- elif ansible_processor[1] is defined and ansible_processor[0] == '0' -%}
          {{ ansible_processor[1] }}
        {%- elif ansible_processor[0] is defined -%}
          {{ ansible_processor[0] }}
        {%- else -%}
          "N/A"
        {%- endif -%}
      {%- else -%}
        "N/A"
      {%- endif -%}

- name: Get CPU model name from lscpu (more accurate)
  block:
    - name: Get CPU model using lscpu
      shell: |
        lscpu | grep "Model name" | cut -d: -f2 | sed 's/^[ \t]*//'
      register: lscpu_model
      changed_when: false
      ignore_errors: yes

    - name: Set CPU model from lscpu if available
      set_fact:
        safe_cpu_model: "{{ lscpu_model.stdout | default(safe_cpu_model) | trim }}"
      when: lscpu_model is defined and lscpu_model.stdout != ""

  rescue:
    - name: Debug lscpu failure
      debug:
        msg: "Failed to get CPU model from lscpu, using default method"

- name: Get detailed CPU architecture information
  block:
    - name: Get CPU architecture details
      shell: |
        lscpu | grep -E "(Architecture|CPU op-mode|Byte Order)" | awk -F: '{print $2}' | sed 's/^[ \t]*//' | tr '\n' ';'
      register: cpu_arch_details
      changed_when: false
      ignore_errors: yes

    - name: Set CPU architecture details
      set_fact:
        safe_cpu_arch_details: "{{ cpu_arch_details.stdout | default('N/A') }}"

    - name: Get CPU sockets and cores details
      shell: |
        echo "Sockets:$(lscpu | grep 'Socket(s)' | awk '{print $2}');Cores per socket:$(lscpu | grep 'Core(s) per socket' | awk '{print $4}');Threads per core:$(lscpu | grep 'Thread(s) per core' | awk '{print $4}')"
      register: cpu_topology
      changed_when: false
      ignore_errors: yes

    - name: Set CPU topology information
      set_fact:
        safe_cpu_topology: "{{ cpu_topology.stdout | default('N/A') }}"

  rescue:
    - name: Set default CPU architecture details
      set_fact:
        safe_cpu_arch_details: "N/A"
        safe_cpu_topology: "N/A"

- name: Collect IP addresses information
  set_fact:
    ip_addresses: |
      {%- set ips = [] -%}
      {%- if ansible_default_ipv4 is defined and ansible_default_ipv4.address is defined -%}
        {%- if ips.append(ansible_default_ipv4.address) -%}{%- endif -%}
      {%- endif -%}
      {%- if include_all_ips and ansible_all_ipv4_addresses is defined -%}
        {%- for ip in ansible_all_ipv4_addresses -%}
          {%- if ip != ansible_default_ipv4.address and ip not in ips -%}
            {%- if ips.append(ip) -%}{%- endif -%}
          {%- endif -%}
        {%- endfor -%}
      {%- endif -%}
      {{ ips }}

- name: Get primary IP address
  set_fact:
    primary_ip: "{{ ansible_default_ipv4.address | default('N/A') }}"

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

- name: Collect service information
  block:
    - name: Gather service facts
      service_facts:

    - name: Filter running services
      set_fact:
        filtered_services: |
          {%- set services_list = [] -%}
          {%- if ansible_facts.services is defined -%}
            {%- for service_name, service_info in ansible_facts.services.items() -%}
              {%- set service_status = service_info.state | default('unknown') -%}
              {%- if service_status == 'running' -%}
                {%- for pattern in service_filter -%}
                  {%- if pattern | lower in service_name | lower -%}
                    {%- if services_list.append({"name": service_name, "status": service_status}) -%}{%- endif -%}
                  {%- endif -%}
                {%- endfor -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ services_list | unique | sort(attribute='name') }}

  rescue:
    - name: Use systemctl to get running services
      shell: |
        systemctl list-units --type=service --state=running --no-legend 2>/dev/null | awk '{print $1}' | while read service; do
          for pattern in {{ service_filter | join(' ') }}; do
            if echo "$service" | grep -qi "$pattern"; then
              echo "$service"
              break
            fi
          done
        done | sort | uniq
      register: systemd_services
      changed_when: false
      ignore_errors: yes
      when: safe_service_manager == "systemd"

    - name: Use service command for SysV init
      shell: |
        service --status-all 2>/dev/null | grep -iE "{{ service_filter | join('|') }}" | awk '{print $4}' || true
      register: sysv_services
      changed_when: false
      ignore_errors: yes
      when: safe_service_manager == "sysvinit"

    - name: Set services from systemctl
      set_fact:
        filtered_services: |
          {%- set services_list = [] -%}
          {%- if systemd_services is defined and systemd_services.stdout_lines is defined -%}
            {%- for service_name in systemd_services.stdout_lines -%}
              {%- if services_list.append({"name": service_name, "status": "running"}) -%}{%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ services_list }}
      when: safe_service_manager == "systemd"

    - name: Set services from SysV
      set_fact:
        filtered_services: |
          {%- set services_list = [] -%}
          {%- if sysv_services is defined and sysv_services.stdout_lines is defined -%}
            {%- for service_name in sysv_services.stdout_lines -%}
              {%- if services_list.append({"name": service_name, "status": "running"}) -%}{%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ services_list }}
      when: safe_service_manager == "sysvinit"

    - name: Set empty services list as final fallback
      set_fact:
        filtered_services: []
      when: filtered_services is not defined

- name: Collect service descriptions
  block:
    - name: Get service descriptions using systemctl
      shell: |
        systemctl show "{{ item.name }}" --property=Description --no-pager 2>/dev/null | grep Description= | cut -d= -f2-
      register: service_descriptions
      loop: "{{ filtered_services }}"
      when: 
        - collect_service_descriptions
        - safe_service_manager == "systemd"
      changed_when: false
      ignore_errors: yes

    - name: Get service descriptions using init scripts
      shell: |
        if [ -f "/etc/init.d/{{ item.name }}" ]; then
          head -n 10 "/etc/init.d/{{ item.name }}" | grep -i "description" | head -n 1 | sed 's/.*://' || echo "No description available"
        else
          echo "No description available"
        fi
      register: sysv_service_descriptions
      loop: "{{ filtered_services }}"
      when: 
        - collect_service_descriptions
        - safe_service_manager == "sysvinit"
      changed_when: false
      ignore_errors: yes

    - name: Combine services with descriptions (systemd)
      set_fact:
        services_with_descriptions: |
          {%- set services_with_desc = [] -%}
          {%- for service in filtered_services -%}
            {%- set description = '' -%}
            {%- for desc_result in service_descriptions.results -%}
              {%- if desc_result.item.name == service.name and desc_result.stdout != '' -%}
                {%- set description = desc_result.stdout | trim -%}
              {%- endif -%}
            {%- endfor -%}
            {%- if description != '' -%}
              {%- if services_with_desc.append({"name": service.name, "status": service.status, "description": description}) -%}{%- endif -%}
            {%- else -%}
              {%- if services_with_desc.append({"name": service.name, "status": service.status, "description": "No description"}) -%}{%- endif -%}
            {%- endif -%}
          {%- endfor -%}
          {{ services_with_desc }}
      when: collect_service_descriptions and safe_service_manager == "systemd"

    - name: Combine services with descriptions (SysV)
      set_fact:
        services_with_descriptions: |
          {%- set services_with_desc = [] -%}
          {%- for service in filtered_services -%}
            {%- set description = '' -%}
            {%- for desc_result in sysv_service_descriptions.results -%}
              {%- if desc_result.item.name == service.name and desc_result.stdout != '' -%}
                {%- set description = desc_result.stdout | trim -%}
              {%- endif -%}
            {%- endfor -%}
            {%- if description != '' -%}
              {%- if services_with_desc.append({"name": service.name, "status": service.status, "description": description}) -%}{%- endif -%}
            {%- else -%}
              {%- if services_with_desc.append({"name": service.name, "status": service.status, "description": "No description"}) -%}{%- endif -%}
            {%- endif -%}
          {%- endfor -%}
          {{ services_with_desc }}
      when: collect_service_descriptions and safe_service_manager == "sysvinit"

  rescue:
    - name: Set services without descriptions on error
      set_fact:
        services_with_descriptions: "{{ filtered_services }}"

- name: Ensure services_with_descriptions is defined
  set_fact:
    services_with_descriptions: "{{ filtered_services }}"
  when: services_with_descriptions is not defined

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

- name: Add CSV header (only once)
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: "Hostname,Primary IP,All IPs,Distribution,Version,Architecture,CPU Model,CPU Cores,CPU Sockets,CPU Threads,CPU Topology,CPU Architecture,Memory (MB),Swap (MB),Virtualization,Disks,Running Services,Service Descriptions,Installed Software"
    create: yes
    insertbefore: BOF
  delegate_to: localhost
  run_once: true
  when: inventory_hostname == play_hosts[0]

- name: Add host data to consolidated report
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: |
      "{{ safe_hostname }}","{{ primary_ip }}","{{ ip_addresses | join('; ') }}","{{ safe_distribution }}","{{ safe_distribution_version }}","{{ safe_architecture }}","{{ safe_cpu_model }}","{{ safe_processor_cores }}","{{ safe_processor_count }}","{{ safe_processor_vcpus }}","{{ safe_cpu_topology }}","{{ safe_cpu_arch_details }}","{{ safe_memory }}","{{ safe_swap }}","{{ safe_virtualization_type }}/{{ safe_virtualization_role }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_descriptions %}{{ service.name }}{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_descriptions %}{% if service.description is defined %}{{ service.name }}: {{ service.description }}{% else %}{{ service.name }}{% endif %}{% if not loop.last %}; {% endif %}{% endfor %}","{% for software in filtered_software %}{{ software.name }}-{{ software.version }}{% if not loop.last %}; {% endif %}{% endfor %}"
    create: yes
  delegate_to: localhost
  run_once: false

- name: Generate checksum for unencrypted report
  shell: |
    cd "{{ report_path }}" && 
    sha256sum "{{ output_filename }}.csv" > "{{ output_filename }}.csv.sha256"
  delegate_to: localhost
  run_once: true
  when: not encrypt_report

- name: Check if report exists before encryption
  stat:
    path: "{{ report_path }}/{{ output_filename }}.csv"
  register: report_stat
  delegate_to: localhost
  run_once: true
  when: encrypt_report

- name: Encrypt report with Ansible Vault using command
  command: >
    ansible-vault encrypt
    {% if vault_password_file %}--vault-password-file {{ vault_password_file }}{% endif %}
    --output "{{ report_path }}/{{ output_filename }}_encrypted.csv"
    "{{ report_path }}/{{ output_filename }}.csv"
  delegate_to: localhost
  run_once: true
  when:
    - encrypt_report
    - report_stat.stat.exists

- name: Generate checksum for encrypted report
  shell: |
    cd "{{ report_path }}" && 
    sha256sum "{{ output_filename }}_encrypted.csv" > "{{ output_filename }}_encrypted.csv.sha256"
  delegate_to: localhost
  run_once: true
  when: 
    - encrypt_report
    - report_stat.stat.exists

- name: Remove unencrypted report after encryption
  file:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    state: absent
  delegate_to: localhost
  run_once: true
  when:
    - encrypt_report
    - remove_unencrypted
    - report_stat.stat.exists

- name: Display report location
  debug:
    msg: |
      {% if encrypt_report and report_stat.stat.exists %}
      Encrypted report generated: {{ report_path }}/{{ output_filename }}_encrypted.csv
      Use 'ansible-vault view {{ report_path }}/{{ output_filename }}_encrypted.csv' to view the report
      {% if vault_password_file != "" %}
      Vault password file: {{ vault_password_file }}
      {% endif %}
      Checksum file: {{ report_path }}/{{ output_filename }}_encrypted.csv.sha256
      {% if remove_unencrypted %}
      Note: Unencrypted version was removed
      {% endif %}
      {% elif encrypt_report and not report_stat.stat.exists %}
      ERROR: Report file not found for encryption
      {% else %}
      Consolidated report generated: {{ report_path }}/{{ output_filename }}.csv
      Checksum file: {{ report_path }}/{{ output_filename }}.csv.sha256
      {% endif %}
  delegate_to: localhost
  run_once: true

- name: Warn about missing vault password file
  debug:
    msg: |
      WARNING: encrypt_report is true but vault_password_file is not specified.
      You will be prompted for the vault password interactively.
  delegate_to: localhost
  run_once: true
  when:
    - encrypt_report
    - vault_password_file == ""
```

Основные исправления:

1. Автоматическое определение менеджера служб

· Добавлена задача Detect service manager которая определяет тип менеджера служб (systemd, upstart, sysvinit)
· Используются команды systemctl --version, initctl --version для определения
· Результат сохраняется в переменную service_manager

2. Безопасная переменная менеджера служб

· Добавлена safe_service_manager в задачу Set safe facts with defaults
· Используется во всех условиях вместо ansible_service_mgr

3. Поддержка разных систем инициализации

· Для systemd: используется systemctl show для получения описаний
· Для sysvinit: используется чтение init скриптов для получения описаний
· Раздельная обработка для каждого типа менеджера

4. Улучшенная обработка ошибок

· Все задачи имеют ignore_errors: yes и changed_when: false
· Резервные методы если основной способ не работает
· Гарантированное определение всех переменных

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

# Службы для проверки
service_filter:
  - "nginx"
  - "apache2"
  - "httpd"
  - "mysql"
  - "postgresql"
  - "docker"
  - "ssh"
  - "redis"
  - "mongod"
  - "elasticsearch"

# Путь для сохранения отчета
report_path: "/tmp/server_report"
output_filename: "servers_consolidated_report"

# Шифрование отчета
encrypt_report: false
vault_password_file: ""
remove_unencrypted: false

# Настройки сбора IP-адресов
preferred_interface: "eth0"  # Предпочтительный интерфейс для основного IP
include_all_ips: true         # Включать все IP-адреса

# Сбор описаний служб
collect_service_descriptions: true

# Формат времени для отчета
timestamp_format: "%Y-%m-%d_%H-%M-%S"
```

Теперь роль должна работать корректно на системах с разными менеджерами служб и собирать описания запущенных служб независимо от типа инициализации системы.