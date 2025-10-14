Исправим роль для корректного отображения размера оперативной памяти. Проблема likely связана с неправильным преобразованием единиц измерения. Вот исправленная версия:

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

- name: Debug memory facts
  debug:
    msg: |
      Memory facts:
      - ansible_memtotal_mb: {{ ansible_memtotal_mb | default('undefined') }}
      - ansible_swaptotal_mb: {{ ansible_swaptotal_mb | default('undefined') }}
      - ansible_memory_mb: {{ ansible_memory_mb | default('undefined') }}
  when: false  # Установите в true для отладки

- name: Calculate memory in GB correctly
  set_fact:
    memory_gb: "{{ (ansible_memtotal_mb | default(0) | int / 1024) | round(2) }}"
    swap_gb: "{{ (ansible_swaptotal_mb | default(0) | int / 1024) | round(2) }}"

- name: Alternative memory calculation from bytes
  block:
    - name: Get memory info from free command
      shell: |
        free -b | grep -E "^(Mem|Swap):" | awk '{print $2}'
      register: memory_bytes
      changed_when: false
      ignore_errors: yes

    - name: Calculate memory from bytes if facts are wrong
      set_fact:
        memory_gb: "{{ (memory_bytes.stdout_lines[0] | default(0) | int / (1024*1024*1024)) | round(2) }}"
        swap_gb: "{{ (memory_bytes.stdout_lines[1] | default(0) | int / (1024*1024*1024)) | round(2) }}"
      when: 
        - memory_bytes is defined
        - memory_bytes.stdout_lines is defined
        - memory_bytes.stdout_lines | length >= 2

  when: memory_gb | int < 1  # Используем альтернативный метод если память меньше 1GB

- name: Set safe facts with defaults
  set_fact:
    safe_hostname: "{{ ansible_hostname | default('Unknown') }}"
    safe_distribution: "{{ ansible_distribution | default('Unknown') }}"
    safe_distribution_version: "{{ ansible_distribution_version | default('Unknown') }}"
    safe_architecture: "{{ ansible_architecture | default('Unknown') }}"
    safe_memory: "{{ memory_gb | default(0) }}"
    safe_swap: "{{ swap_gb | default(0) }}"
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

- name: Collect detailed service information (systemd)
  block:
    - name: Get detailed service information using systemctl
      shell: |
        systemctl show "{{ item.name }}" --property=Id,Description,LoadState,ActiveState,SubState,MainPID,MemoryCurrent,ExecMainStartTimestamp,FragmentPath --no-pager 2>/dev/null
      register: detailed_service_info
      loop: "{{ filtered_services }}"
      when: 
        - collect_detailed_service_info
        - safe_service_manager == "systemd"
      changed_when: false
      ignore_errors: yes

    - name: Parse detailed service information
      set_fact:
        services_with_details: |
          {%- set services_with_detail = [] -%}
          {%- for service in filtered_services -%}
            {%- set service_detail = {'name': service.name, 'status': service.status} -%}
            {%- for info_result in detailed_service_info.results -%}
              {%- if info_result.item.name == service.name and info_result.stdout != '' -%}
                {%- set properties = {} -%}
                {%- for line in info_result.stdout.split('\n') -%}
                  {%- if '=' in line -%}
                    {%- set key_value = line.split('=', 1) -%}
                    {%- if key_value.0 and key_value.1 -%}
                      {%- if properties.update({key_value.0: key_value.1}) -%}{%- endif -%}
                    {%- endif -%}
                  {%- endif -%}
                {%- endfor -%}
                {%- set memory_bytes = properties.MemoryCurrent | default(0) | int -%}
                {%- set memory_gb = (memory_bytes / (1024*1024*1024)) | round(2) -%}
                {%- if service_detail.update({
                  'description': properties.Description | default('No description'),
                  'load_state': properties.LoadState | default('unknown'),
                  'active_state': properties.ActiveState | default('unknown'),
                  'sub_state': properties.SubState | default('unknown'),
                  'main_pid': properties.MainPID | default('N/A'),
                  'memory_usage': memory_gb | string + ' GB',
                  'start_time': properties.ExecMainStartTimestamp | default('N/A'),
                  'config_path': properties.FragmentPath | default('N/A')
                }) -%}{%- endif -%}
              {%- endif -%}
            {%- endfor -%}
            {%- if services_with_detail.append(service_detail) -%}{%- endif -%}
          {%- endfor -%}
          {{ services_with_detail }}
      when: collect_detailed_service_info and safe_service_manager == "systemd"

  rescue:
    - name: Set basic service information on error
      set_fact:
        services_with_details: "{{ filtered_services }}"

- name: Collect basic service descriptions (SysV)
  block:
    - name: Get service descriptions using init scripts
      shell: |
        if [ -f "/etc/init.d/{{ item.name }}" ]; then
          # Try to extract description from init script
          DESCRIPTION=$(grep -i "description" "/etc/init.d/{{ item.name }}" | head -n 1 | sed 's/.*[:=]\s*//' | sed 's/^[\"'\'']//' | sed 's/[\"'\'']$//' | tr -d '\n' || echo "No description")
          if [ "$DESCRIPTION" = "No description" ]; then
            # Try to get short description from comments
            DESCRIPTION=$(head -n 20 "/etc/init.d/{{ item.name }}" | grep -i "#.*description" | head -n 1 | sed 's/.*[:=]\s*//' | sed 's/^[\"'\'']//' | sed 's/[\"'\'']$//' | tr -d '\n' || echo "No description")
          fi
          echo "$DESCRIPTION"
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

    - name: Combine services with basic descriptions (SysV)
      set_fact:
        services_with_details: |
          {%- set services_with_detail = [] -%}
          {%- for service in filtered_services -%}
            {%- set description = 'No description' -%}
            {%- for desc_result in sysv_service_descriptions.results -%}
              {%- if desc_result.item.name == service.name and desc_result.stdout != '' and desc_result.stdout != 'No description available' -%}
                {%- set description = desc_result.stdout | trim -%}
              {%- endif -%}
            {%- endfor -%}
            {%- if services_with_detail.append({
              'name': service.name,
              'status': service.status,
              'description': description,
              'load_state': 'N/A',
              'active_state': service.status,
              'sub_state': 'N/A',
              'main_pid': 'N/A',
              'memory_usage': 'N/A',
              'start_time': 'N/A',
              'config_path': 'N/A'
            }) -%}{%- endif -%}
          {%- endfor -%}
          {{ services_with_detail }}
      when: collect_service_descriptions and safe_service_manager == "sysvinit"

  rescue:
    - name: Set basic services on error (SysV)
      set_fact:
        services_with_details: "{{ filtered_services }}"

- name: Ensure services_with_details is defined
  set_fact:
    services_with_details: |
      {%- set services_with_detail = [] -%}
      {%- for service in filtered_services -%}
        {%- if services_with_detail.append({
          'name': service.name,
          'status': service.status,
          'description': 'No description available',
          'load_state': 'N/A',
          'active_state': service.status,
          'sub_state': 'N/A',
          'main_pid': 'N/A',
          'memory_usage': 'N/A',
          'start_time': 'N/A',
          'config_path': 'N/A'
        }) -%}{%- endif -%}
      {%- endfor -%}
      {{ services_with_detail }}
  when: services_with_details is not defined

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
    line: "Hostname,Primary IP,All IPs,Distribution,Version,Architecture,CPU Model,CPU Cores,CPU Sockets,CPU Threads,CPU Topology,CPU Architecture,Memory (GB),Swap (GB),Virtualization,Disks,Running Services,Service Details"
    create: yes
    insertbefore: BOF
  delegate_to: localhost
  run_once: true
  when: inventory_hostname == play_hosts[0]

- name: Add host data to consolidated report
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: |
      "{{ safe_hostname }}","{{ primary_ip }}","{{ ip_addresses | join('; ') }}","{{ safe_distribution }}","{{ safe_distribution_version }}","{{ safe_architecture }}","{{ safe_cpu_model }}","{{ safe_processor_cores }}","{{ safe_processor_count }}","{{ safe_processor_vcpus }}","{{ safe_cpu_topology }}","{{ safe_cpu_arch_details }}","{{ safe_memory }}","{{ safe_swap }}","{{ safe_virtualization_type }}/{{ safe_virtualization_role }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_details %}{{ service.name }}{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_details %}{% if service.description is defined %}Name: {{ service.name }}; Description: {{ service.description }}; Status: {{ service.active_state }}; PID: {{ service.main_pid }}; Memory: {{ service.memory_usage }}; Config: {{ service.config_path }}{% else %}{{ service.name }}{% endif %}{% if not loop.last %}| {% endif %}{% endfor %}"
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

Основные исправления для памяти:

1. Правильное преобразование единиц измерения

· MB → GB: (ansible_memtotal_mb | default(0) | int / 1024) | round(2)
· Байты → GB (альтернативный метод): (memory_bytes / (1024*1024*1024)) | round(2)

2. Альтернативный метод сбора информации о памяти

· Добавлена задача Alternative memory calculation from bytes которая использует команду free -b для получения точных данных в байтах
· Этот метод используется если основной метод показывает память меньше 1GB

3. Отладочная задача

· Добавлена задача Debug memory facts (закомментирована) для диагностики проблем с памятью
· Чтобы включить отладку, установите when: true

4. Исправление памяти служб

· Обновлен расчет памяти для служб: (memory_bytes / (1024*1024*1024)) | round(2)

Включение отладки памяти:

Если проблема persists, раскомментируйте задачу отладки:

```yaml
- name: Debug memory facts
  debug:
    msg: |
      Memory facts:
      - ansible_memtotal_mb: {{ ansible_memtotal_mb | default('undefined') }}
      - ansible_swaptotal_mb: {{ ansible_swaptotal_mb | default('undefined') }}
      - ansible_memory_mb: {{ ansible_memory_mb | default('undefined') }}
  when: true  # Установите в true для отладки
```

Проверка правильности отображения памяти:

После выполнения роли проверьте значения в CSV отчете. Они должны отображаться корректно в гигабайтах:

· 16 GB RAM = 16.0
· 8 GB RAM = 8.0
· 32 GB RAM = 32.0

Если память все еще отображается неправильно, возможно проблема в исходных данных Ansible facts. В этом случае альтернативный метод с free -b должен решить проблему.

Добавим сбор информации о тактовой частоте CPU в MHz. Вот модифицированная версия роли:

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

- name: Debug memory facts
  debug:
    msg: |
      Memory facts:
      - ansible_memtotal_mb: {{ ansible_memtotal_mb | default('undefined') }}
      - ansible_swaptotal_mb: {{ ansible_swaptotal_mb | default('undefined') }}
      - ansible_memory_mb: {{ ansible_memory_mb | default('undefined') }}
  when: false  # Установите в true для отладки

- name: Calculate memory in GB correctly
  set_fact:
    memory_gb: "{{ (ansible_memtotal_mb | default(0) | int / 1024) | round(2) }}"
    swap_gb: "{{ (ansible_swaptotal_mb | default(0) | int / 1024) | round(2) }}"

- name: Alternative memory calculation from bytes
  block:
    - name: Get memory info from free command
      shell: |
        free -b | grep -E "^(Mem|Swap):" | awk '{print $2}'
      register: memory_bytes
      changed_when: false
      ignore_errors: yes

    - name: Calculate memory from bytes if facts are wrong
      set_fact:
        memory_gb: "{{ (memory_bytes.stdout_lines[0] | default(0) | int / (1024*1024*1024)) | round(2) }}"
        swap_gb: "{{ (memory_bytes.stdout_lines[1] | default(0) | int / (1024*1024*1024)) | round(2) }}"
      when: 
        - memory_bytes is defined
        - memory_bytes.stdout_lines is defined
        - memory_bytes.stdout_lines | length >= 2

  when: memory_gb | int < 1  # Используем альтернативный метод если память меньше 1GB

- name: Get CPU frequency information
  block:
    - name: Get CPU frequency from lscpu
      shell: |
        lscpu | grep -E "(CPU MHz|CPU max MHz|CPU min MHz)" | awk -F: '{print $2}' | sed 's/^[ \t]*//' | head -n1
      register: cpu_freq_mhz
      changed_when: false
      ignore_errors: yes

    - name: Get CPU frequency from /proc/cpuinfo
      shell: |
        grep -m1 "cpu MHz" /proc/cpuinfo | awk -F: '{print $2}' | sed 's/^[ \t]*//'
      register: cpu_freq_proc
      changed_when: false
      ignore_errors: yes
      when: cpu_freq_mhz.stdout == ""

    - name: Get CPU frequency using dmidecode
      shell: |
        dmidecode -t processor 2>/dev/null | grep "Current Speed" | head -n1 | awk -F: '{print $2}' | sed 's/^[ \t]*//; s/ MHz//' || echo "N/A"
      register: cpu_freq_dmi
      changed_when: false
      ignore_errors: yes
      when: cpu_freq_mhz.stdout == "" and cpu_freq_proc.stdout == ""

    - name: Set CPU frequency fact
      set_fact:
        safe_cpu_frequency_mhz: |
          {%- if cpu_freq_mhz.stdout != "" -%}
            {{ cpu_freq_mhz.stdout | float | round | int }}
          {%- elif cpu_freq_proc.stdout != "" -%}
            {{ cpu_freq_proc.stdout | float | round | int }}
          {%- elif cpu_freq_dmi.stdout != "" and cpu_freq_dmi.stdout != "N/A" -%}
            {{ cpu_freq_dmi.stdout | float | round | int }}
          {%- else -%}
            "N/A"
          {%- endif -%}

  rescue:
    - name: Set default CPU frequency
      set_fact:
        safe_cpu_frequency_mhz: "N/A"

- name: Get CPU cache information
  block:
    - name: Get CPU cache sizes
      shell: |
        lscpu | grep -E "cache" | awk -F: '{print $2}' | sed 's/^[ \t]*//' | tr '\n' ';'
      register: cpu_cache
      changed_when: false
      ignore_errors: yes

    - name: Set CPU cache information
      set_fact:
        safe_cpu_cache: "{{ cpu_cache.stdout | default('N/A') }}"

  rescue:
    - name: Set default CPU cache
      set_fact:
        safe_cpu_cache: "N/A"

- name: Set safe facts with defaults
  set_fact:
    safe_hostname: "{{ ansible_hostname | default('Unknown') }}"
    safe_distribution: "{{ ansible_distribution | default('Unknown') }}"
    safe_distribution_version: "{{ ansible_distribution_version | default('Unknown') }}"
    safe_architecture: "{{ ansible_architecture | default('Unknown') }}"
    safe_memory: "{{ memory_gb | default(0) }}"
    safe_swap: "{{ swap_gb | default(0) }}"
    safe_virtualization_type: "{{ ansible_virtualization_type | default('physical') }}"
    safe_virtualization_role: "{{ ansible_virtualization_role | default('host') }}"
    safe_processor_cores: "{{ ansible_processor_cores | default(ansible_processor_count | default('N/A')) }}"
    safe_processor_count: "{{ ansible_processor_count | default('N/A') }}"
    safe_processor_threads_per_core: "{{ ansible_processor_threads_per_core | default('N/A') }}"
    safe_processor_vcpus: "{{ ansible_processor_vcpus | default('N/A') }}"
    safe_service_manager: "{{ service_manager }}"
    safe_cpu_frequency_mhz: "{{ safe_cpu_frequency_mhz | default('N/A') }}"
    safe_cpu_cache: "{{ safe_cpu_cache | default('N/A') }}"

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

- name: Collect detailed service information (systemd)
  block:
    - name: Get detailed service information using systemctl
      shell: |
        systemctl show "{{ item.name }}" --property=Id,Description,LoadState,ActiveState,SubState,MainPID,MemoryCurrent,ExecMainStartTimestamp,FragmentPath --no-pager 2>/dev/null
      register: detailed_service_info
      loop: "{{ filtered_services }}"
      when: 
        - collect_detailed_service_info
        - safe_service_manager == "systemd"
      changed_when: false
      ignore_errors: yes

    - name: Parse detailed service information
      set_fact:
        services_with_details: |
          {%- set services_with_detail = [] -%}
          {%- for service in filtered_services -%}
            {%- set service_detail = {'name': service.name, 'status': service.status} -%}
            {%- for info_result in detailed_service_info.results -%}
              {%- if info_result.item.name == service.name and info_result.stdout != '' -%}
                {%- set properties = {} -%}
                {%- for line in info_result.stdout.split('\n') -%}
                  {%- if '=' in line -%}
                    {%- set key_value = line.split('=', 1) -%}
                    {%- if key_value.0 and key_value.1 -%}
                      {%- if properties.update({key_value.0: key_value.1}) -%}{%- endif -%}
                    {%- endif -%}
                  {%- endif -%}
                {%- endfor -%}
                {%- set memory_bytes = properties.MemoryCurrent | default(0) | int -%}
                {%- set memory_gb = (memory_bytes / (1024*1024*1024)) | round(2) -%}
                {%- if service_detail.update({
                  'description': properties.Description | default('No description'),
                  'load_state': properties.LoadState | default('unknown'),
                  'active_state': properties.ActiveState | default('unknown'),
                  'sub_state': properties.SubState | default('unknown'),
                  'main_pid': properties.MainPID | default('N/A'),
                  'memory_usage': memory_gb | string + ' GB',
                  'start_time': properties.ExecMainStartTimestamp | default('N/A'),
                  'config_path': properties.FragmentPath | default('N/A')
                }) -%}{%- endif -%}
              {%- endif -%}
            {%- endfor -%}
            {%- if services_with_detail.append(service_detail) -%}{%- endif -%}
          {%- endfor -%}
          {{ services_with_detail }}
      when: collect_detailed_service_info and safe_service_manager == "systemd"

  rescue:
    - name: Set basic service information on error
      set_fact:
        services_with_details: "{{ filtered_services }}"

- name: Collect basic service descriptions (SysV)
  block:
    - name: Get service descriptions using init scripts
      shell: |
        if [ -f "/etc/init.d/{{ item.name }}" ]; then
          # Try to extract description from init script
          DESCRIPTION=$(grep -i "description" "/etc/init.d/{{ item.name }}" | head -n 1 | sed 's/.*[:=]\s*//' | sed 's/^[\"'\'']//' | sed 's/[\"'\'']$//' | tr -d '\n' || echo "No description")
          if [ "$DESCRIPTION" = "No description" ]; then
            # Try to get short description from comments
            DESCRIPTION=$(head -n 20 "/etc/init.d/{{ item.name }}" | grep -i "#.*description" | head -n 1 | sed 's/.*[:=]\s*//' | sed 's/^[\"'\'']//' | sed 's/[\"'\'']$//' | tr -d '\n' || echo "No description")
          fi
          echo "$DESCRIPTION"
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

    - name: Combine services with basic descriptions (SysV)
      set_fact:
        services_with_details: |
          {%- set services_with_detail = [] -%}
          {%- for service in filtered_services -%}
            {%- set description = 'No description' -%}
            {%- for desc_result in sysv_service_descriptions.results -%}
              {%- if desc_result.item.name == service.name and desc_result.stdout != '' and desc_result.stdout != 'No description available' -%}
                {%- set description = desc_result.stdout | trim -%}
              {%- endif -%}
            {%- endfor -%}
            {%- if services_with_detail.append({
              'name': service.name,
              'status': service.status,
              'description': description,
              'load_state': 'N/A',
              'active_state': service.status,
              'sub_state': 'N/A',
              'main_pid': 'N/A',
              'memory_usage': 'N/A',
              'start_time': 'N/A',
              'config_path': 'N/A'
            }) -%}{%- endif -%}
          {%- endfor -%}
          {{ services_with_detail }}
      when: collect_service_descriptions and safe_service_manager == "sysvinit"

  rescue:
    - name: Set basic services on error (SysV)
      set_fact:
        services_with_details: "{{ filtered_services }}"

- name: Ensure services_with_details is defined
  set_fact:
    services_with_details: |
      {%- set services_with_detail = [] -%}
      {%- for service in filtered_services -%}
        {%- if services_with_detail.append({
          'name': service.name,
          'status': service.status,
          'description': 'No description available',
          'load_state': 'N/A',
          'active_state': service.status,
          'sub_state': 'N/A',
          'main_pid': 'N/A',
          'memory_usage': 'N/A',
          'start_time': 'N/A',
          'config_path': 'N/A'
        }) -%}{%- endif -%}
      {%- endfor -%}
      {{ services_with_detail }}
  when: services_with_details is not defined

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
    line: "Hostname,Primary IP,All IPs,Distribution,Version,Architecture,CPU Model,CPU Frequency (MHz),CPU Cores,CPU Sockets,CPU Threads,CPU Cache,CPU Topology,CPU Architecture,Memory (GB),Swap (GB),Virtualization,Disks,Running Services,Service Details"
    create: yes
    insertbefore: BOF
  delegate_to: localhost
  run_once: true
  when: inventory_hostname == play_hosts[0]

- name: Add host data to consolidated report
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: |
      "{{ safe_hostname }}","{{ primary_ip }}","{{ ip_addresses | join('; ') }}","{{ safe_distribution }}","{{ safe_distribution_version }}","{{ safe_architecture }}","{{ safe_cpu_model }}","{{ safe_cpu_frequency_mhz }}","{{ safe_processor_cores }}","{{ safe_processor_count }}","{{ safe_processor_vcpus }}","{{ safe_cpu_cache }}","{{ safe_cpu_topology }}","{{ safe_cpu_arch_details }}","{{ safe_memory }}","{{ safe_swap }}","{{ safe_virtualization_type }}/{{ safe_virtualization_role }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_details %}{{ service.name }}{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_details %}{% if service.description is defined %}Name: {{ service.name }}; Description: {{ service.description }}; Status: {{ service.active_state }}; PID: {{ service.main_pid }}; Memory: {{ service.memory_usage }}; Config: {{ service.config_path }}{% else %}{{ service.name }}{% endif %}{% if not loop.last %}| {% endif %}{% endfor %}"
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

1. Сбор тактовой частоты CPU

· Добавлен блок Get CPU frequency information с несколькими методами получения частоты:
  · lscpu - основной метод (поле CPU MHz, CPU max MHz, CPU min MHz)
  · /proc/cpuinfo - резервный метод
  · dmidecode - альтернативный метод для получения номинальной частоты

2. Сбор информации о кэше CPU

· Добавлен блок Get CPU cache information для сбора информации о размерах кэша L1, L2, L3

3. Новые переменные в отчете

· safe_cpu_frequency_mhz - тактовая частота CPU в MHz
· safe_cpu_cache - информация о кэше CPU

4. Обновленный CSV формат

Новые столбцы в отчете:

· CPU Frequency (MHz) - тактовая частота процессора
· CPU Cache - информация о кэше процессора

5. Улучшенная обработка ошибок

· Множественные fallback методы для получения частоты
· Обработка случаев когда частота недоступна
· Автоматическое округление частоты до целых MHz

Пример вывода в CSV:

```
Hostname,Primary IP,...,CPU Model,CPU Frequency (MHz),CPU Cores,...,CPU Cache,...
"server1","192.168.1.10",...,"Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz","2400","8",...,"L1d: 32K; L1i: 32K; L2: 256K; L3: 30720K",...
```

Методы определения частоты:

1. lscpu - основной метод, использует команду lscpu | grep -E "(CPU MHz|CPU max MHz|CPU min MHz)"
2. /proc/cpuinfo - резервный метод, использует grep -m1 "cpu MHz" /proc/cpuinfo
3. dmidecode - альтернативный метод для получения номинальной частоты

Особенности:

· Частота округляется до целых MHz для удобства чтения
· Если частота недоступна, выводится "N/A"
· Информация о кэше собирается из вывода lscpu и объединяется в одну строку

Теперь отчет будет содержать полную информацию о тактовой частоте процессора и размерах кэша, что дает более детальное представление о производительности CPU на каждом сервере.


Модифицируем роль для отключения отображения внешних (примонтированных) дисков. Добавим фильтрацию по типам файловых систем и точкам монтирования.

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
collect_detailed_service_info: true  # Сбор расширенной информации о службах

# Фильтрация дисков
exclude_mount_types:
  - "tmpfs"
  - "devtmpfs"
  - "squashfs"
  - "overlay"
  - "cgroup"
  - "mqueue"
  - "devpts"
  - "proc"
  - "sysfs"
  - "fusectl"
  - "debugfs"
  - "securityfs"
  - "configfs"
  - "pstore"
  - "autofs"
  - "hugetlbfs"
  - "fuse.gvfsd-fuse"

exclude_mount_points:
  - "/proc"
  - "/sys"
  - "/dev"
  - "/run"
  - "/snap"
  - "/var/lib/docker"
  - "/sys/fs/cgroup"
  - "/sys/fs/fuse/connections"
  - "/sys/kernel/security"
  - "/dev/shm"
  - "/run/lock"
  - "/run/user"

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

- name: Debug memory facts
  debug:
    msg: |
      Memory facts:
      - ansible_memtotal_mb: {{ ansible_memtotal_mb | default('undefined') }}
      - ansible_swaptotal_mb: {{ ansible_swaptotal_mb | default('undefined') }}
      - ansible_memory_mb: {{ ansible_memory_mb | default('undefined') }}
  when: false  # Установите в true для отладки

- name: Calculate memory in GB correctly
  set_fact:
    memory_gb: "{{ (ansible_memtotal_mb | default(0) | int / 1024) | round(2) }}"
    swap_gb: "{{ (ansible_swaptotal_mb | default(0) | int / 1024) | round(2) }}"

- name: Alternative memory calculation from bytes
  block:
    - name: Get memory info from free command
      shell: |
        free -b | grep -E "^(Mem|Swap):" | awk '{print $2}'
      register: memory_bytes
      changed_when: false
      ignore_errors: yes

    - name: Calculate memory from bytes if facts are wrong
      set_fact:
        memory_gb: "{{ (memory_bytes.stdout_lines[0] | default(0) | int / (1024*1024*1024)) | round(2) }}"
        swap_gb: "{{ (memory_bytes.stdout_lines[1] | default(0) | int / (1024*1024*1024)) | round(2) }}"
      when: 
        - memory_bytes is defined
        - memory_bytes.stdout_lines is defined
        - memory_bytes.stdout_lines | length >= 2

  when: memory_gb | int < 1  # Используем альтернативный метод если память меньше 1GB

- name: Get CPU frequency information
  block:
    - name: Get CPU frequency from lscpu
      shell: |
        lscpu | grep -E "(CPU MHz|CPU max MHz|CPU min MHz)" | awk -F: '{print $2}' | sed 's/^[ \t]*//' | head -n1
      register: cpu_freq_mhz
      changed_when: false
      ignore_errors: yes

    - name: Get CPU frequency from /proc/cpuinfo
      shell: |
        grep -m1 "cpu MHz" /proc/cpuinfo | awk -F: '{print $2}' | sed 's/^[ \t]*//'
      register: cpu_freq_proc
      changed_when: false
      ignore_errors: yes
      when: cpu_freq_mhz.stdout == ""

    - name: Get CPU frequency using dmidecode
      shell: |
        dmidecode -t processor 2>/dev/null | grep "Current Speed" | head -n1 | awk -F: '{print $2}' | sed 's/^[ \t]*//; s/ MHz//' || echo "N/A"
      register: cpu_freq_dmi
      changed_when: false
      ignore_errors: yes
      when: cpu_freq_mhz.stdout == "" and cpu_freq_proc.stdout == ""

    - name: Set CPU frequency fact
      set_fact:
        safe_cpu_frequency_mhz: |
          {%- if cpu_freq_mhz.stdout != "" -%}
            {{ cpu_freq_mhz.stdout | float | round | int }}
          {%- elif cpu_freq_proc.stdout != "" -%}
            {{ cpu_freq_proc.stdout | float | round | int }}
          {%- elif cpu_freq_dmi.stdout != "" and cpu_freq_dmi.stdout != "N/A" -%}
            {{ cpu_freq_dmi.stdout | float | round | int }}
          {%- else -%}
            "N/A"
          {%- endif -%}

  rescue:
    - name: Set default CPU frequency
      set_fact:
        safe_cpu_frequency_mhz: "N/A"

- name: Get CPU cache information
  block:
    - name: Get CPU cache sizes
      shell: |
        lscpu | grep -E "cache" | awk -F: '{print $2}' | sed 's/^[ \t]*//' | tr '\n' ';'
      register: cpu_cache
      changed_when: false
      ignore_errors: yes

    - name: Set CPU cache information
      set_fact:
        safe_cpu_cache: "{{ cpu_cache.stdout | default('N/A') }}"

  rescue:
    - name: Set default CPU cache
      set_fact:
        safe_cpu_cache: "N/A"

- name: Set safe facts with defaults
  set_fact:
    safe_hostname: "{{ ansible_hostname | default('Unknown') }}"
    safe_distribution: "{{ ansible_distribution | default('Unknown') }}"
    safe_distribution_version: "{{ ansible_distribution_version | default('Unknown') }}"
    safe_architecture: "{{ ansible_architecture | default('Unknown') }}"
    safe_memory: "{{ memory_gb | default(0) }}"
    safe_swap: "{{ swap_gb | default(0) }}"
    safe_virtualization_type: "{{ ansible_virtualization_type | default('physical') }}"
    safe_virtualization_role: "{{ ansible_virtualization_role | default('host') }}"
    safe_processor_cores: "{{ ansible_processor_cores | default(ansible_processor_count | default('N/A')) }}"
    safe_processor_count: "{{ ansible_processor_count | default('N/A') }}"
    safe_processor_threads_per_core: "{{ ansible_processor_threads_per_core | default('N/A') }}"
    safe_processor_vcpus: "{{ ansible_processor_vcpus | default('N/A') }}"
    safe_service_manager: "{{ service_manager }}"
    safe_cpu_frequency_mhz: "{{ safe_cpu_frequency_mhz | default('N/A') }}"
    safe_cpu_cache: "{{ safe_cpu_cache | default('N/A') }}"

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
    - name: Try to get disk info from mounts facts with filtering
      set_fact:
        disk_info: |
          {%- set disks = [] -%}
          {%- if ansible_mounts is defined and ansible_mounts -%}
            {%- for mount in ansible_mounts -%}
              {%- if mount is mapping and mount.mount is defined and mount.size_total is defined and mount.fstype is defined -%}
                {%- set skip_disk = false -%}
                
                {# Проверяем тип файловой системы #}
                {%- for exclude_type in exclude_mount_types -%}
                  {%- if mount.fstype == exclude_type -%}
                    {%- set skip_disk = true -%}
                  {%- endif -%}
                {%- endfor -%}
                
                {# Проверяем точку монтирования #}
                {%- for exclude_point in exclude_mount_points -%}
                  {%- if mount.mount.startswith(exclude_point) -%}
                    {%- set skip_disk = true -%}
                  {%- endif -%}
                {%- endfor -%}
                
                {# Пропускаем маленькие файловые системы (< 1GB) #}
                {%- set size_gb = (mount.size_total | int // (1024**3)) | round(2) -%}
                {%- if size_gb < 1 -%}
                  {%- set skip_disk = true -%}
                {%- endif -%}
                
                {%- if not skip_disk -%}
                  {%- if disks.append({"mount": mount.mount, "size_gb": size_gb, "fstype": mount.fstype}) -%}{%- endif -%}
                {%- endif -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ disks }}
      when: ansible_mounts is defined

  rescue:
    - name: Set empty disk info on error
      set_fact:
        disk_info: []

- name: Fallback disk info using shell command with filtering
  block:
    - name: Get disk info via df command
      shell: |
        df -h --output=target,size,fstype | tail -n +2
      register: disk_df
      changed_when: false
      ignore_errors: yes

    - name: Process disk output from df with filtering
      set_fact:
        disk_info_fallback: |
          {%- set disks = [] -%}
          {%- if disk_df is defined and disk_df.stdout_lines is defined -%}
            {%- for line in disk_df.stdout_lines -%}
              {%- set parts = line.split() -%}
              {%- if parts|length >= 3 -%}
                {%- set mount_point = parts[0] -%}
                {%- set size_str = parts[1] -%}
                {%- set fstype = parts[2] -%}
                
                {# Конвертируем размер в GB #}
                {%- set size_gb = 0 -%}
                {%- if 'G' in size_str -%}
                  {%- set size_gb = size_str | replace('G', '') | float -%}
                {%- elif 'T' in size_str -%}
                  {%- set size_gb = (size_str | replace('T', '') | float * 1024) | round(2) -%}
                {%- elif 'M' in size_str -%}
                  {%- set size_gb = (size_str | replace('M', '') | float / 1024) | round(2) -%}
                {%- endif -%}
                
                {%- set skip_disk = false -%}
                
                {# Проверяем тип файловой системы #}
                {%- for exclude_type in exclude_mount_types -%}
                  {%- if fstype == exclude_type -%}
                    {%- set skip_disk = true -%}
                  {%- endif -%}
                {%- endfor -%}
                
                {# Проверяем точку монтирования #}
                {%- for exclude_point in exclude_mount_points -%}
                  {%- if mount_point.startswith(exclude_point) -%}
                    {%- set skip_disk = true -%}
                  {%- endif -%}
                {%- endfor -%}
                
                {# Пропускаем маленькие файловые системы (< 1GB) #}
                {%- if size_gb < 1 -%}
                  {%- set skip_disk = true -%}
                {%- endif -%}
                
                {%- if not skip_disk and size_gb > 0 -%}
                  {%- if disks.append({"mount": mount_point, "size_gb": size_gb, "fstype": fstype}) -%}{%- endif -%}
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

- name: Collect detailed service information (systemd)
  block:
    - name: Get detailed service information using systemctl
      shell: |
        systemctl show "{{ item.name }}" --property=Id,Description,LoadState,ActiveState,SubState,MainPID,MemoryCurrent,ExecMainStartTimestamp,FragmentPath --no-pager 2>/dev/null
      register: detailed_service_info
      loop: "{{ filtered_services }}"
      when: 
        - collect_detailed_service_info
        - safe_service_manager == "systemd"
      changed_when: false
      ignore_errors: yes

    - name: Parse detailed service information
      set_fact:
        services_with_details: |
          {%- set services_with_detail = [] -%}
          {%- for service in filtered_services -%}
            {%- set service_detail = {'name': service.name, 'status': service.status} -%}
            {%- for info_result in detailed_service_info.results -%}
              {%- if info_result.item.name == service.name and info_result.stdout != '' -%}
                {%- set properties = {} -%}
                {%- for line in info_result.stdout.split('\n') -%}
                  {%- if '=' in line -%}
                    {%- set key_value = line.split('=', 1) -%}
                    {%- if key_value.0 and key_value.1 -%}
                      {%- if properties.update({key_value.0: key_value.1}) -%}{%- endif -%}
                    {%- endif -%}
                  {%- endif -%}
                {%- endfor -%}
                {%- set memory_bytes = properties.MemoryCurrent | default(0) | int -%}
                {%- set memory_gb = (memory_bytes / (1024*1024*1024)) | round(2) -%}
                {%- if service_detail.update({
                  'description': properties.Description | default('No description'),
                  'load_state': properties.LoadState | default('unknown'),
                  'active_state': properties.ActiveState | default('unknown'),
                  'sub_state': properties.SubState | default('unknown'),
                  'main_pid': properties.MainPID | default('N/A'),
                  'memory_usage': memory_gb | string + ' GB',
                  'start_time': properties.ExecMainStartTimestamp | default('N/A'),
                  'config_path': properties.FragmentPath | default('N/A')
                }) -%}{%- endif -%}
              {%- endif -%}
            {%- endfor -%}
            {%- if services_with_detail.append(service_detail) -%}{%- endif -%}
          {%- endfor -%}
          {{ services_with_detail }}
      when: collect_detailed_service_info and safe_service_manager == "systemd"

  rescue:
    - name: Set basic service information on error
      set_fact:
        services_with_details: "{{ filtered_services }}"

- name: Collect basic service descriptions (SysV)
  block:
    - name: Get service descriptions using init scripts
      shell: |
        if [ -f "/etc/init.d/{{ item.name }}" ]; then
          # Try to extract description from init script
          DESCRIPTION=$(grep -i "description" "/etc/init.d/{{ item.name }}" | head -n 1 | sed 's/.*[:=]\s*//' | sed 's/^[\"'\'']//' | sed 's/[\"'\'']$//' | tr -d '\n' || echo "No description")
          if [ "$DESCRIPTION" = "No description" ]; then
            # Try to get short description from comments
            DESCRIPTION=$(head -n 20 "/etc/init.d/{{ item.name }}" | grep -i "#.*description" | head -n 1 | sed 's/.*[:=]\s*//' | sed 's/^[\"'\'']//' | sed 's/[\"'\'']$//' | tr -d '\n' || echo "No description")
          fi
          echo "$DESCRIPTION"
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

    - name: Combine services with basic descriptions (SysV)
      set_fact:
        services_with_details: |
          {%- set services_with_detail = [] -%}
          {%- for service in filtered_services -%}
            {%- set description = 'No description' -%}
            {%- for desc_result in sysv_service_descriptions.results -%}
              {%- if desc_result.item.name == service.name and desc_result.stdout != '' and desc_result.stdout != 'No description available' -%}
                {%- set description = desc_result.stdout | trim -%}
              {%- endif -%}
            {%- endfor -%}
            {%- if services_with_detail.append({
              'name': service.name,
              'status': service.status,
              'description': description,
              'load_state': 'N/A',
              'active_state': service.status,
              'sub_state': 'N/A',
              'main_pid': 'N/A',
              'memory_usage': 'N/A',
              'start_time': 'N/A',
              'config_path': 'N/A'
            }) -%}{%- endif -%}
          {%- endfor -%}
          {{ services_with_detail }}
      when: collect_service_descriptions and safe_service_manager == "sysvinit"

  rescue:
    - name: Set basic services on error (SysV)
      set_fact:
        services_with_details: "{{ filtered_services }}"

- name: Ensure services_with_details is defined
  set_fact:
    services_with_details: |
      {%- set services_with_detail = [] -%}
      {%- for service in filtered_services -%}
        {%- if services_with_detail.append({
          'name': service.name,
          'status': service.status,
          'description': 'No description available',
          'load_state': 'N/A',
          'active_state': service.status,
          'sub_state': 'N/A',
          'main_pid': 'N/A',
          'memory_usage': 'N/A',
          'start_time': 'N/A',
          'config_path': 'N/A'
        }) -%}{%- endif -%}
      {%- endfor -%}
      {{ services_with_detail }}
  when: services_with_details is not defined

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
    line: "Hostname,Primary IP,All IPs,Distribution,Version,Architecture,CPU Model,CPU Frequency (MHz),CPU Cores,CPU Sockets,CPU Threads,CPU Cache,CPU Topology,CPU Architecture,Memory (GB),Swap (GB),Virtualization,Disks,Running Services,Service Details"
    create: yes
    insertbefore: BOF
  delegate_to: localhost
  run_once: true
  when: inventory_hostname == play_hosts[0]

- name: Add host data to consolidated report
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: |
      "{{ safe_hostname }}","{{ primary_ip }}","{{ ip_addresses | join('; ') }}","{{ safe_distribution }}","{{ safe_distribution_version }}","{{ safe_architecture }}","{{ safe_cpu_model }}","{{ safe_cpu_frequency_mhz }}","{{ safe_processor_cores }}","{{ safe_processor_count }}","{{ safe_processor_vcpus }}","{{ safe_cpu_cache }}","{{ safe_cpu_topology }}","{{ safe_cpu_arch_details }}","{{ safe_memory }}","{{ safe_swap }}","{{ safe_virtualization_type }}/{{ safe_virtualization_role }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_details %}{{ service.name }}{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_details %}{% if service.description is defined %}Name: {{ service.name }}; Description: {{ service.description }}; Status: {{ service.active_state }}; PID: {{ service.main_pid }}; Memory: {{ service.memory_usage }}; Config: {{ service.config_path }}{% else %}{{ service.name }}{% endif %}{% if not loop.last %}| {% endif %}{% endfor %}"
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

Основные изменения для фильтрации дисков:

1. Новые переменные фильтрации

· exclude_mount_types - список типов файловых систем для исключения
· exclude_mount_points - список точек монтирования для исключения

2. Улучшенная фильтрация в основном методе

· Проверка типа файловой системы через mount.fstype
· Проверка точки монтирования через mount.mount.startswith()
· Исключение маленьких файловых систем (< 1GB)

3. Фильтрация в fallback методе

· Использование команды df с выводом типа файловой системы
· Аналогичная логика фильтрации по типам и точкам монтирования
· Конвертация размеров из разных единиц измерения в GB

4. Типы файловых систем для исключения по умолчанию

· Временные файловые системы: tmpfs, devtmpfs, squashfs
· Виртуальные файловые системы: proc, sysfs, cgroup
· Специальные файловые системы: devpts, mqueue, hugetlbfs
· Файловые системы контейнеров: overlay

5. Точки монтирования для исключения по умолчанию

· Системные каталоги: /proc, /sys, /dev, /run
· Временные каталоги: /tmp, /var/run, /var/tmp
· Каталоги контейнеров: /var/lib/docker
· Каталоги snap пакетов: /snap

Пример использования с кастомной фильтрацией:

```yaml
- name: Collect server information with custom disk filtering
  hosts: all
  become: yes
  gather_facts: no
  roles:
    - server-info-collector
  vars:
    exclude_mount_types:
      - "tmpfs"
      - "devtmpfs" 
      - "proc"
      - "sysfs"
      - "nfs"  # Добавляем NFS
      - "cifs" # Добавляем CIFS/SMB
    exclude_mount_points:
      - "/proc"
      - "/sys"
      - "/dev"
      - "/run"
      - "/mnt/backup"  # Исключаем точку монтирования бэкапов
```

Теперь отчет будет показывать только основные физические и сетевые диски, исключая временные, виртуальные и системные файловые системы, что делает отчет более чистым и информативным.



Добавим сбор информации о максимально физически выделенной оперативной памяти в роль. Вот модифицированная версия:

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
collect_detailed_service_info: true  # Сбор расширенной информации о службах

# Фильтрация дисков
exclude_mount_types:
  - "tmpfs"
  - "devtmpfs"
  - "squashfs"
  - "overlay"
  - "cgroup"
  - "mqueue"
  - "devpts"
  - "proc"
  - "sysfs"
  - "fusectl"
  - "debugfs"
  - "securityfs"
  - "configfs"
  - "pstore"
  - "autofs"
  - "hugetlbfs"
  - "fuse.gvfsd-fuse"

exclude_mount_points:
  - "/proc"
  - "/sys"
  - "/dev"
  - "/run"
  - "/snap"
  - "/var/lib/docker"
  - "/sys/fs/cgroup"
  - "/sys/fs/fuse/connections"
  - "/sys/kernel/security"
  - "/dev/shm"
  - "/run/lock"
  - "/run/user"

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

- name: Debug memory facts
  debug:
    msg: |
      Memory facts:
      - ansible_memtotal_mb: {{ ansible_memtotal_mb | default('undefined') }}
      - ansible_swaptotal_mb: {{ ansible_swaptotal_mb | default('undefined') }}
      - ansible_memory_mb: {{ ansible_memory_mb | default('undefined') }}
  when: false  # Установите в true для отладки

- name: Calculate memory in GB correctly
  set_fact:
    memory_gb: "{{ (ansible_memtotal_mb | default(0) | int / 1024) | round(2) }}"
    swap_gb: "{{ (ansible_swaptotal_mb | default(0) | int / 1024) | round(2) }}"

- name: Get maximum physical memory information
  block:
    - name: Get memory information from dmidecode
      shell: |
        # Получаем информацию о максимальной поддерживаемой памяти
        if command -v dmidecode >/dev/null 2>&1; then
          # Получаем информацию о максимальном объеме памяти из раздела system information
          dmidecode -t 16 2>/dev/null | grep -i "Maximum Capacity" | head -1 | awk -F: '{print $2}' | sed 's/^[ \t]*//' || echo "N/A"
        else
          echo "N/A"
        fi
      register: max_memory_dmi
      changed_when: false
      ignore_errors: yes

    - name: Get memory slots information
      shell: |
        # Получаем информацию о слотах памяти
        if command -v dmidecode >/dev/null 2>&1; then
          dmidecode -t memory 2>/dev/null | grep -E "(Size:|Type:|Speed:|Manufacturer:)" | head -20 || echo "N/A"
        else
          echo "N/A"
        fi
      register: memory_slots_info
      changed_when: false
      ignore_errors: yes

    - name: Get memory information from lshw
      shell: |
        # Альтернативный метод через lshw
        if command -v lshw >/dev/null 2>&1; then
          lshw -class memory 2>/dev/null | grep -A10 "System Memory" | grep -E "(size:|clock:)" | head -5 || echo "N/A"
        else
          echo "N/A"
        fi
      register: memory_lshw_info
      changed_when: false
      ignore_errors: yes

    - name: Parse maximum memory capacity
      set_fact:
        max_physical_memory: |
          {%- if max_memory_dmi.stdout != "N/A" and max_memory_dmi.stdout != "" -%}
            {{ max_memory_dmi.stdout | trim }}
          {%- else -%}
            "N/A"
          {%- endif -%}

    - name: Parse memory slots information
      set_fact:
        memory_slots_details: |
          {%- if memory_slots_info.stdout != "N/A" and memory_slots_info.stdout != "" -%}
            {%- set details = [] -%}
            {%- for line in memory_slots_info.stdout_lines -%}
              {%- if "Size:" in line or "Type:" in line or "Speed:" in line or "Manufacturer:" in line -%}
                {%- set clean_line = line | regex_replace('^[^:]*:', '') | trim -%}
                {%- if clean_line != "No Module Installed" and clean_line != "Unknown" -%}
                  {%- if details.append(clean_line) -%}{%- endif -%}
                {%- endif -%}
              {%- endif -%}
            {%- endfor -%}
            {{ details | join('; ') }}
          {%- else -%}
            "N/A"
          {%- endif -%}

  rescue:
    - name: Set default maximum memory values
      set_fact:
        max_physical_memory: "N/A"
        memory_slots_details: "N/A"

- name: Alternative memory calculation from bytes
  block:
    - name: Get memory info from free command
      shell: |
        free -b | grep -E "^(Mem|Swap):" | awk '{print $2}'
      register: memory_bytes
      changed_when: false
      ignore_errors: yes

    - name: Calculate memory from bytes if facts are wrong
      set_fact:
        memory_gb: "{{ (memory_bytes.stdout_lines[0] | default(0) | int / (1024*1024*1024)) | round(2) }}"
        swap_gb: "{{ (memory_bytes.stdout_lines[1] | default(0) | int / (1024*1024*1024)) | round(2) }}"
      when: 
        - memory_bytes is defined
        - memory_bytes.stdout_lines is defined
        - memory_bytes.stdout_lines | length >= 2

  when: memory_gb | int < 1  # Используем альтернативный метод если память меньше 1GB

- name: Get CPU frequency information
  block:
    - name: Get CPU frequency from lscpu
      shell: |
        lscpu | grep -E "(CPU MHz|CPU max MHz|CPU min MHz)" | awk -F: '{print $2}' | sed 's/^[ \t]*//' | head -n1
      register: cpu_freq_mhz
      changed_when: false
      ignore_errors: yes

    - name: Get CPU frequency from /proc/cpuinfo
      shell: |
        grep -m1 "cpu MHz" /proc/cpuinfo | awk -F: '{print $2}' | sed 's/^[ \t]*//'
      register: cpu_freq_proc
      changed_when: false
      ignore_errors: yes
      when: cpu_freq_mhz.stdout == ""

    - name: Get CPU frequency using dmidecode
      shell: |
        dmidecode -t processor 2>/dev/null | grep "Current Speed" | head -n1 | awk -F: '{print $2}' | sed 's/^[ \t]*//; s/ MHz//' || echo "N/A"
      register: cpu_freq_dmi
      changed_when: false
      ignore_errors: yes
      when: cpu_freq_mhz.stdout == "" and cpu_freq_proc.stdout == ""

    - name: Set CPU frequency fact
      set_fact:
        safe_cpu_frequency_mhz: |
          {%- if cpu_freq_mhz.stdout != "" -%}
            {{ cpu_freq_mhz.stdout | float | round | int }}
          {%- elif cpu_freq_proc.stdout != "" -%}
            {{ cpu_freq_proc.stdout | float | round | int }}
          {%- elif cpu_freq_dmi.stdout != "" and cpu_freq_dmi.stdout != "N/A" -%}
            {{ cpu_freq_dmi.stdout | float | round | int }}
          {%- else -%}
            "N/A"
          {%- endif -%}

  rescue:
    - name: Set default CPU frequency
      set_fact:
        safe_cpu_frequency_mhz: "N/A"

- name: Get CPU cache information
  block:
    - name: Get CPU cache sizes
      shell: |
        lscpu | grep -E "cache" | awk -F: '{print $2}' | sed 's/^[ \t]*//' | tr '\n' ';'
      register: cpu_cache
      changed_when: false
      ignore_errors: yes

    - name: Set CPU cache information
      set_fact:
        safe_cpu_cache: "{{ cpu_cache.stdout | default('N/A') }}"

  rescue:
    - name: Set default CPU cache
      set_fact:
        safe_cpu_cache: "N/A"

- name: Set safe facts with defaults
  set_fact:
    safe_hostname: "{{ ansible_hostname | default('Unknown') }}"
    safe_distribution: "{{ ansible_distribution | default('Unknown') }}"
    safe_distribution_version: "{{ ansible_distribution_version | default('Unknown') }}"
    safe_architecture: "{{ ansible_architecture | default('Unknown') }}"
    safe_memory: "{{ memory_gb | default(0) }}"
    safe_swap: "{{ swap_gb | default(0) }}"
    safe_max_physical_memory: "{{ max_physical_memory | default('N/A') }}"
    safe_memory_slots_details: "{{ memory_slots_details | default('N/A') }}"
    safe_virtualization_type: "{{ ansible_virtualization_type | default('physical') }}"
    safe_virtualization_role: "{{ ansible_virtualization_role | default('host') }}"
    safe_processor_cores: "{{ ansible_processor_cores | default(ansible_processor_count | default('N/A')) }}"
    safe_processor_count: "{{ ansible_processor_count | default('N/A') }}"
    safe_processor_threads_per_core: "{{ ansible_processor_threads_per_core | default('N/A') }}"
    safe_processor_vcpus: "{{ ansible_processor_vcpus | default('N/A') }}"
    safe_service_manager: "{{ service_manager }}"
    safe_cpu_frequency_mhz: "{{ safe_cpu_frequency_mhz | default('N/A') }}"
    safe_cpu_cache: "{{ safe_cpu_cache | default('N/A') }}"

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
    - name: Try to get disk info from mounts facts with filtering
      set_fact:
        disk_info: |
          {%- set disks = [] -%}
          {%- if ansible_mounts is defined and ansible_mounts -%}
            {%- for mount in ansible_mounts -%}
              {%- if mount is mapping and mount.mount is defined and mount.size_total is defined and mount.fstype is defined -%}
                {%- set skip_disk = false -%}
                
                {# Проверяем тип файловой системы #}
                {%- for exclude_type in exclude_mount_types -%}
                  {%- if mount.fstype == exclude_type -%}
                    {%- set skip_disk = true -%}
                  {%- endif -%}
                {%- endfor -%}
                
                {# Проверяем точку монтирования #}
                {%- for exclude_point in exclude_mount_points -%}
                  {%- if mount.mount.startswith(exclude_point) -%}
                    {%- set skip_disk = true -%}
                  {%- endif -%}
                {%- endfor -%}
                
                {# Пропускаем маленькие файловые системы (< 1GB) #}
                {%- set size_gb = (mount.size_total | int // (1024**3)) | round(2) -%}
                {%- if size_gb < 1 -%}
                  {%- set skip_disk = true -%}
                {%- endif -%}
                
                {%- if not skip_disk -%}
                  {%- if disks.append({"mount": mount.mount, "size_gb": size_gb, "fstype": mount.fstype}) -%}{%- endif -%}
                {%- endif -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ disks }}
      when: ansible_mounts is defined

  rescue:
    - name: Set empty disk info on error
      set_fact:
        disk_info: []

- name: Fallback disk info using shell command with filtering
  block:
    - name: Get disk info via df command
      shell: |
        df -h --output=target,size,fstype | tail -n +2
      register: disk_df
      changed_when: false
      ignore_errors: yes

    - name: Process disk output from df with filtering
      set_fact:
        disk_info_fallback: |
          {%- set disks = [] -%}
          {%- if disk_df is defined and disk_df.stdout_lines is defined -%}
            {%- for line in disk_df.stdout_lines -%}
              {%- set parts = line.split() -%}
              {%- if parts|length >= 3 -%}
                {%- set mount_point = parts[0] -%}
                {%- set size_str = parts[1] -%}
                {%- set fstype = parts[2] -%}
                
                {# Конвертируем размер в GB #}
                {%- set size_gb = 0 -%}
                {%- if 'G' in size_str -%}
                  {%- set size_gb = size_str | replace('G', '') | float -%}
                {%- elif 'T' in size_str -%}
                  {%- set size_gb = (size_str | replace('T', '') | float * 1024) | round(2) -%}
                {%- elif 'M' in size_str -%}
                  {%- set size_gb = (size_str | replace('M', '') | float / 1024) | round(2) -%}
                {%- endif -%}
                
                {%- set skip_disk = false -%}
                
                {# Проверяем тип файловой системы #}
                {%- for exclude_type in exclude_mount_types -%}
                  {%- if fstype == exclude_type -%}
                    {%- set skip_disk = true -%}
                  {%- endif -%}
                {%- endfor -%}
                
                {# Проверяем точку монтирования #}
                {%- for exclude_point in exclude_mount_points -%}
                  {%- if mount_point.startswith(exclude_point) -%}
                    {%- set skip_disk = true -%}
                  {%- endif -%}
                {%- endfor -%}
                
                {# Пропускаем маленькие файловые системы (< 1GB) #}
                {%- if size_gb < 1 -%}
                  {%- set skip_disk = true -%}
                {%- endif -%}
                
                {%- if not skip_disk and size_gb > 0 -%}
                  {%- if disks.append({"mount": mount_point, "size_gb": size_gb, "fstype": fstype}) -%}{%- endif -%}
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

- name: Collect detailed service information (systemd)
  block:
    - name: Get detailed service information using systemctl
      shell: |
        systemctl show "{{ item.name }}" --property=Id,Description,LoadState,ActiveState,SubState,MainPID,MemoryCurrent,ExecMainStartTimestamp,FragmentPath --no-pager 2>/dev/null
      register: detailed_service_info
      loop: "{{ filtered_services }}"
      when: 
        - collect_detailed_service_info
        - safe_service_manager == "systemd"
      changed_when: false
      ignore_errors: yes

    - name: Parse detailed service information
      set_fact:
        services_with_details: |
          {%- set services_with_detail = [] -%}
          {%- for service in filtered_services -%}
            {%- set service_detail = {'name': service.name, 'status': service.status} -%}
            {%- for info_result in detailed_service_info.results -%}
              {%- if info_result.item.name == service.name and info_result.stdout != '' -%}
                {%- set properties = {} -%}
                {%- for line in info_result.stdout.split('\n') -%}
                  {%- if '=' in line -%}
                    {%- set key_value = line.split('=', 1) -%}
                    {%- if key_value.0 and key_value.1 -%}
                      {%- if properties.update({key_value.0: key_value.1}) -%}{%- endif -%}
                    {%- endif -%}
                  {%- endif -%}
                {%- endfor -%}
                {%- set memory_bytes = properties.MemoryCurrent | default(0) | int -%}
                {%- set memory_gb = (memory_bytes / (1024*1024*1024)) | round(2) -%}
                {%- if service_detail.update({
                  'description': properties.Description | default('No description'),
                  'load_state': properties.LoadState | default('unknown'),
                  'active_state': properties.ActiveState | default('unknown'),
                  'sub_state': properties.SubState | default('unknown'),
                  'main_pid': properties.MainPID | default('N/A'),
                  'memory_usage': memory_gb | string + ' GB',
                  'start_time': properties.ExecMainStartTimestamp | default('N/A'),
                  'config_path': properties.FragmentPath | default('N/A')
                }) -%}{%- endif -%}
              {%- endif -%}
            {%- endfor -%}
            {%- if services_with_detail.append(service_detail) -%}{%- endif -%}
          {%- endfor -%}
          {{ services_with_detail }}
      when: collect_detailed_service_info and safe_service_manager == "systemd"

  rescue:
    - name: Set basic service information on error
      set_fact:
        services_with_details: "{{ filtered_services }}"

- name: Collect basic service descriptions (SysV)
  block:
    - name: Get service descriptions using init scripts
      shell: |
        if [ -f "/etc/init.d/{{ item.name }}" ]; then
          # Try to extract description from init script
          DESCRIPTION=$(grep -i "description" "/etc/init.d/{{ item.name }}" | head -n 1 | sed 's/.*[:=]\s*//' | sed 's/^[\"'\'']//' | sed 's/[\"'\'']$//' | tr -d '\n' || echo "No description")
          if [ "$DESCRIPTION" = "No description" ]; then
            # Try to get short description from comments
            DESCRIPTION=$(head -n 20 "/etc/init.d/{{ item.name }}" | grep -i "#.*description" | head -n 1 | sed 's/.*[:=]\s*//' | sed 's/^[\"'\'']//' | sed 's/[\"'\'']$//' | tr -d '\n' || echo "No description")
          fi
          echo "$DESCRIPTION"
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

    - name: Combine services with basic descriptions (SysV)
      set_fact:
        services_with_details: |
          {%- set services_with_detail = [] -%}
          {%- for service in filtered_services -%}
            {%- set description = 'No description' -%}
            {%- for desc_result in sysv_service_descriptions.results -%}
              {%- if desc_result.item.name == service.name and desc_result.stdout != '' and desc_result.stdout != 'No description available' -%}
                {%- set description = desc_result.stdout | trim -%}
              {%- endif -%}
            {%- endfor -%}
            {%- if services_with_detail.append({
              'name': service.name,
              'status': service.status,
              'description': description,
              'load_state': 'N/A',
              'active_state': service.status,
              'sub_state': 'N/A',
              'main_pid': 'N/A',
              'memory_usage': 'N/A',
              'start_time': 'N/A',
              'config_path': 'N/A'
            }) -%}{%- endif -%}
          {%- endfor -%}
          {{ services_with_detail }}
      when: collect_service_descriptions and safe_service_manager == "sysvinit"

  rescue:
    - name: Set basic services on error (SysV)
      set_fact:
        services_with_details: "{{ filtered_services }}"

- name: Ensure services_with_details is defined
  set_fact:
    services_with_details: |
      {%- set services_with_detail = [] -%}
      {%- for service in filtered_services -%}
        {%- if services_with_detail.append({
          'name': service.name,
          'status': service.status,
          'description': 'No description available',
          'load_state': 'N/A',
          'active_state': service.status,
          'sub_state': 'N/A',
          'main_pid': 'N/A',
          'memory_usage': 'N/A',
          'start_time': 'N/A',
          'config_path': 'N/A'
        }) -%}{%- endif -%}
      {%- endfor -%}
      {{ services_with_detail }}
  when: services_with_details is not defined

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
    line: "Hostname,Primary IP,All IPs,Distribution,Version,Architecture,CPU Model,CPU Frequency (MHz),CPU Cores,CPU Sockets,CPU Threads,CPU Cache,CPU Topology,CPU Architecture,Current Memory (GB),Max Physical Memory,Memory Details,Swap (GB),Virtualization,Disks,Running Services,Service Details"
    create: yes
    insertbefore: BOF
  delegate_to: localhost
  run_once: true
  when: inventory_hostname == play_hosts[0]

- name: Add host data to consolidated report
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: |
      "{{ safe_hostname }}","{{ primary_ip }}","{{ ip_addresses | join('; ') }}","{{ safe_distribution }}","{{ safe_distribution_version }}","{{ safe_architecture }}","{{ safe_cpu_model }}","{{ safe_cpu_frequency_mhz }}","{{ safe_processor_cores }}","{{ safe_processor_count }}","{{ safe_processor_vcpus }}","{{ safe_cpu_cache }}","{{ safe_cpu_topology }}","{{ safe_cpu_arch_details }}","{{ safe_memory }}","{{ safe_max_physical_memory }}","{{ safe_memory_slots_details }}","{{ safe_swap }}","{{ safe_virtualization_type }}/{{ safe_virtualization_role }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_details %}{{ service.name }}{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_details %}{% if service.description is defined %}Name: {{ service.name }}; Description: {{ service.description }}; Status: {{ service.active_state }}; PID: {{ service.main_pid }}; Memory: {{ service.memory_usage }}; Config: {{ service.config_path }}{% else %}{{ service.name }}{% endif %}{% if not loop.last %}| {% endif %}{% endfor %}"
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

Основные добавления для информации о максимальной физической памяти:

1. Новый блок сбора информации о максимальной памяти

· Добавлен блок Get maximum physical memory information с несколькими методами:
  · dmidecode -t 16 - получение информации о максимальной поддерживаемой памяти из DMI
  · dmidecode -t memory - получение детальной информации о слотах памяти
  · lshw -class memory - альтернативный метод через lshw

2. Новые переменные в отчете

· safe_max_physical_memory - максимальная физически поддерживаемая память
· safe_memory_slots_details - детали о слотах памяти (размер, тип, скорость, производитель)

3. Обновленный CSV формат

Измененные и новые столбцы:

· Current Memory (GB) - текущая установленная оперативная память
· Max Physical Memory - максимальная физически поддерживаемая память
· Memory Details - детали о модулях памяти (размер, тип, скорость)

4. Обработка различных форматов данных

· Парсинг вывода dmidecode для извлечения максимальной емкости
· Обработка информации о слотах памяти
· Фильтрация пустых и неизвестных значений

Пример вывода в CSV:

```
Hostname,...,Current Memory (GB),Max Physical Memory,Memory Details,...
"server1",...,"16.0","64 GB","16 GB DDR4 2400 MHz; 16 GB DDR4 2400 MHz",...
```

Методы определения максимальной памяти:

1. dmidecode -t 16 - основной метод для получения максимальной поддерживаемой памяти
2. dmidecode -t memory - для получения информации об установленных модулях памяти
3. lshw - альтернативный метод для систем без dmidecode

Особенности:

· Если dmidecode недоступен, значения будут "N/A"
· Информация о модулях памяти объединяется в одну строку для удобства чтения
· Фильтруются пустые слоты и неизвестные модули

Теперь отчет будет содержать полную информацию не только о текущей установленной оперативной памяти, но и о максимальной физически поддерживаемой памяти системой, а также детали об установленных модулях памяти.




Отличная идея! Добавим сбор информации о физических дисках, их размере, типе и других характеристиках. Вот улучшенная версия роли:

## Обновленный defaults/main.yml

```yaml
---
# Настройки сбора информации о памяти
memory_info:
  collect_max_memory: true
  collect_memory_slots: true
  use_dmidecode: true
  use_lshw: true

# Настройки сбора информации о дисках
disk_info:
  collect_physical_disks: true
  collect_partition_info: true
  collect_disk_health: false  # Будет использовать smartctl если доступно
  exclude_loop_devices: true
  exclude_removable: true
  min_disk_size_gb: 1  # Минимальный размер диска для отображения

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
collect_detailed_service_info: true  # Сбор расширенной информации о службах

# Фильтрация дисков (точки монтирования)
exclude_mount_types:
  - "tmpfs"
  - "devtmpfs"
  - "squashfs"
  - "overlay"
  - "cgroup"
  - "mqueue"
  - "devpts"
  - "proc"
  - "sysfs"
  - "fusectl"
  - "debugfs"
  - "securityfs"
  - "configfs"
  - "pstore"
  - "autofs"
  - "hugetlbfs"
  - "fuse.gvfsd-fuse"

exclude_mount_points:
  - "/proc"
  - "/sys"
  - "/dev"
  - "/run"
  - "/snap"
  - "/var/lib/docker"
  - "/sys/fs/cgroup"
  - "/sys/fs/fuse/connections"
  - "/sys/kernel/security"
  - "/dev/shm"
  - "/run/lock"
  - "/run/user"

# Формат времени для отчета
timestamp_format: "%Y-%m-%d_%H-%M-%S"
```

## Новые задачи для сбора информации о физических дисках

Добавьте эти задачи после сбора информации о памяти и перед сбором информации о CPU:

```yaml
- name: Collect physical disk information
  block:
    - name: Install necessary disk utilities
      package:
        name: "{{ item }}"
        state: present
      loop:
        - util-linux
        - lshw
        - hdparm
      ignore_errors: yes
      when: disk_info.collect_physical_disks | default(true)

    - name: Get physical disks information using lsblk
      shell: |
        lsblk -b -o NAME,TYPE,SIZE,MODEL,VENDOR,ROTA,MOUNTPOINT,FSTYPE,TRAN,SERIAL --json 2>/dev/null || echo '{"blockdevices": []}'
      register: lsblk_output
      changed_when: false
      ignore_errors: yes
      when: disk_info.collect_physical_disks | default(true)

    - name: Parse physical disks information
      set_fact:
        physical_disks_parsed: |
          {%- set disks = [] -%}
          {%- if lsblk_output.stdout and lsblk_output.stdout != '{"blockdevices": []}' -%}
            {%- set lsblk_data = lsblk_output.stdout | from_json -%}
            {%- for device in lsblk_data.blockdevices -%}
              {%- if device.type == "disk" -%}
                {%- set skip_disk = false -%}
                
                {# Пропускаем loop устройства если включено #}
                {%- if disk_info.exclude_loop_devices | default(true) and device.name.startswith('loop') -%}
                  {%- set skip_disk = true -%}
                {%- endif -%}
                
                {# Пропускаем маленькие диски #}
                {%- set size_gb = (device.size | default(0) | int / (1024**3)) | round(2) -%}
                {%- if size_gb < (disk_info.min_disk_size_gb | default(1)) -%}
                  {%- set skip_disk = true -%}
                {%- endif -%}
                
                {%- if not skip_disk -%}
                  {%- set disk_info_dict = {
                    'name': device.name,
                    'type': device.type,
                    'size_gb': size_gb,
                    'size_bytes': device.size | default(0),
                    'model': device.model | default('Unknown'),
                    'vendor': device.vendor | default('Unknown'),
                    'rotational': (device.rota | default(1) == 1),
                    'transport': device.tran | default('Unknown'),
                    'serial': device.serial | default('Unknown')
                  } -%}
                  
                  {# Собираем информацию о разделах для этого диска #}
                  {%- set partitions = [] -%}
                  {%- if device.children is defined and disk_info.collect_partition_info | default(true) -%}
                    {%- for child in device.children -%}
                      {%- if child.type == "part" -%}
                        {%- set part_size_gb = (child.size | default(0) | int / (1024**3)) | round(2) -%}
                        {%- if partitions.append({
                          'name': child.name,
                          'size_gb': part_size_gb,
                          'fstype': child.fstype | default('Unknown'),
                          'mountpoint': child.mountpoint | default('Not mounted')
                        }) -%}{%- endif -%}
                      {%- endif -%}
                    {%- endfor -%}
                  {%- endif -%}
                  {%- if disk_info_dict.update({'partitions': partitions}) -%}{%- endif -%}
                  
                  {%- if disks.append(disk_info_dict) -%}{%- endif -%}
                {%- endif -%}
              {%- endif -%}
            {%- endfor -%}
          {%- endif -%}
          {{ disks }}

    - name: Get additional disk information using lshw
      shell: |
        lshw -class disk -json 2>/dev/null || echo '[]'
      register: lshw_disk_output
      changed_when: false
      ignore_errors: yes
      when: disk_info.collect_physical_disks | default(true)

    - name: Enhance disk information with lshw data
      set_fact:
        enhanced_physical_disks: |
          {%- set enhanced_disks = [] -%}
          {%- for disk in physical_disks_parsed -%}
            {%- set enhanced_disk = disk -%}
            {%- if lshw_disk_output.stdout and lshw_disk_output.stdout != '[]' -%}
              {%- set lshw_data = lshw_disk_output.stdout | from_json -%}
              {%- for lshw_disk in lshw_data -%}
                {%- if lshw_disk.logicalname is defined and disk.name in lshw_disk.logicalname -%}
                  {%- if enhanced_disk.update({
                    'description': lshw_disk.description | default('Unknown'),
                    'product': lshw_disk.product | default('Unknown'),
                    'vendor': lshw_disk.vendor | default(enhanced_disk.vendor),
                    'serial': lshw_disk.serial | default(enhanced_disk.serial),
                    'capacity_bytes': lshw_disk.size | default(enhanced_disk.size_bytes),
                    'configuration': lshw_disk.configuration | default({})
                  }) -%}{%- endif -%}
                {%- endif -%}
              {%- endfor -%}
            {%- endif -%}
            {%- if enhanced_disks.append(enhanced_disk) -%}{%- endif -%}
          {%- endfor -%}
          {{ enhanced_disks }}

    - name: Get disk usage and performance information
      block:
        - name: Get disk read-ahead and scheduler info
          shell: |
            for disk in /sys/block/sd* /sys/block/vd* /sys/block/nvme*; do
              if [ -b "/dev/$(basename $disk)" ]; then
                echo "Disk: $(basename $disk)"
                echo "  Read-ahead: $(cat $disk/queue/read_ahead_kb 2>/dev/null || echo 'N/A') KB"
                echo "  Scheduler: $(cat $disk/queue/scheduler 2>/dev/null || echo 'N/A')"
                echo "  Size: $(cat $disk/size 2>/dev/null | awk '{print $1*512/1024/1024/1024 " GB"}' || echo 'N/A')"
                echo "---"
              fi
            done
          register: disk_sys_info
          changed_when: false
          ignore_errors: yes

        - name: Parse disk system information
          set_fact:
            disk_sysinfo_parsed: |
              {%- set disk_info_list = [] -%}
              {%- if disk_sys_info.stdout -%}
                {%- set disk_blocks = disk_sys_info.stdout.split('---') -%}
                {%- for block in disk_blocks -%}
                  {%- if block.strip() -%}
                    {%- set disk_data = {} -%}
                    {%- for line in block.strip().split('\n') -%}
                      {%- if 'Disk:' in line -%}
                        {%- set disk_name = line.split(':')[1] | trim -%}
                        {%- if disk_data.update({'name': disk_name}) -%}{%- endif -%}
                      {%- elif 'Read-ahead:' in line -%}
                        {%- set read_ahead = line.split(':')[1] | trim -%}
                        {%- if disk_data.update({'read_ahead': read_ahead}) -%}{%- endif -%}
                      {%- elif 'Scheduler:' in line -%}
                        {%- set scheduler = line.split(':')[1] | trim -%}
                        {%- if disk_data.update({'scheduler': scheduler}) -%}{%- endif -%}
                      {%- elif 'Size:' in line -%}
                        {%- set size_str = line.split(':')[1] | trim -%}
                        {%- if disk_data.update({'sys_size': size_str}) -%}{%- endif -%}
                      {%- endif -%}
                    {%- endfor -%}
                    {%- if disk_info_list.append(disk_data) -%}{%- endif -%}
                  {%- endif -%}
                {%- endfor -%}
              {%- endif -%}
              {{ disk_info_list }}

      when: disk_info.collect_physical_disks | default(true)

  rescue:
    - name: Handle disk collection errors
      debug:
        msg: "Failed to collect detailed disk information"
      ignore_errors: yes

- name: Set safe disk facts
  set_fact:
    safe_physical_disks: "{{ enhanced_physical_disks | default(physical_disks_parsed) | default([]) }}"
    safe_disk_sysinfo: "{{ disk_sysinfo_parsed | default([]) }}"

- name: Format physical disks for report
  set_fact:
    physical_disks_formatted: |
      {%- if safe_physical_disks -%}
        {%- set disk_strings = [] -%}
        {%- for disk in safe_physical_disks -%}
          {%- set disk_type = 'HDD' if disk.rotational else 'SSD' -%}
          {%- set disk_str = disk.name ~ ':' ~ disk.size_gb ~ 'GB:' ~ disk_type ~ ':' ~ disk.model -%}
          {%- if disk_strings.append(disk_str) -%}{%- endif -%}
        {%- endfor -%}
        {{ disk_strings | join('; ') }}
      {%- else -%}
        "No physical disks detected"
      {%- endif -%}

- name: Format disk partitions for report
  set_fact:
    disk_partitions_formatted: |
      {%- if safe_physical_disks -%}
        {%- set partition_strings = [] -%}
        {%- for disk in safe_physical_disks -%}
          {%- for partition in disk.partitions -%}
            {%- set part_str = partition.name ~ ':' ~ partition.size_gb ~ 'GB:' ~ partition.fstype ~ ':' ~ partition.mountpoint -%}
            {%- if partition_strings.append(part_str) -%}{%- endif -%}
          {%- endfor -%}
        {%- endfor -%}
        {{ partition_strings | join('; ') }}
      {%- else -%}
        "No partitions detected"
      {%- endif -%}
```

## Обновленный CSV заголовок и данные

Обновите задачу добавления CSV заголовка:

```yaml
- name: Add CSV header (only once)
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: "Hostname,Primary IP,All IPs,Distribution,Version,Architecture,CPU Model,CPU Frequency (MHz),CPU Cores,CPU Sockets,CPU Threads,CPU Cache,CPU Topology,CPU Architecture,Current Memory (GB),Max Physical Memory,Memory Details,Swap (GB),Virtualization,Mounted Disks,Physical Disks,Disk Partitions,Running Services,Service Details"
    create: yes
    insertbefore: BOF
  delegate_to: localhost
  run_once: true
  when: inventory_hostname == play_hosts[0]
```

Обновите задачу добавления данных в отчет:

```yaml
- name: Add host data to consolidated report
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: |
      "{{ safe_hostname }}",
      "{{ primary_ip }}",
      "{{ ip_addresses | join('; ') }}",
      "{{ safe_distribution }}",
      "{{ safe_distribution_version }}",
      "{{ safe_architecture }}",
      "{{ safe_cpu_model | replace('"', '""') }}",
      "{{ safe_cpu_frequency_mhz }}",
      "{{ safe_processor_cores }}",
      "{{ safe_processor_count }}",
      "{{ safe_processor_vcpus }}",
      "{{ safe_cpu_cache }}",
      "{{ safe_cpu_topology }}",
      "{{ safe_cpu_arch_details }}",
      "{{ safe_memory }}",
      "{{ safe_max_physical_memory }}",
      "{{ safe_memory_slots_details | replace('"', '""') }}",
      "{{ safe_swap }}",
      "{{ safe_virtualization_type }}/{{ safe_virtualization_role }}",
      "{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB({{ disk.fstype }}){% if not loop.last %}; {% endif %}{% endfor %}",
      "{{ physical_disks_formatted }}",
      "{{ disk_partitions_formatted }}",
      "{% for service in services_with_details %}{{ service.name }}{% if not loop.last %}; {% endif %}{% endfor %}",
      "{% for service in services_with_details %}{% if service.description is defined %}{{ service.name }}:{{ service.description }}:{{ service.active_state }}{% if not loop.last %}|{% endif %}{% else %}{{ service.name }}{% if not loop.last %}|{% endif %}{% endif %}{% endfor %}"
    create: yes
  delegate_to: localhost
  run_once: false
```

## Новый шаблон для детального отчета о дисках

Добавьте этот шаблон как `templates/disk_report.md.j2`:

```jinja2
# Physical Disk Report - {{ safe_hostname }}

## Physical Disks Summary
{% for disk in safe_physical_disks %}
### Disk {{ disk.name }}
- **Model**: {{ disk.model }} ({{ disk.vendor }})
- **Size**: {{ disk.size_gb }} GB ({{ disk.size_bytes }} bytes)
- **Type**: {% if disk.rotational %}HDD (Rotational){% else %}SSD (Non-rotational){% endif %}
- **Transport**: {{ disk.transport }}
- **Serial**: {{ disk.serial }}
{% if disk.description and disk.description != 'Unknown' %}- **Description**: {{ disk.description }}{% endif %}
{% if disk.product and disk.product != 'Unknown' %}- **Product**: {{ disk.product }}{% endif %}

### Partitions on {{ disk.name }}
{% if disk.partitions %}
{% for part in disk.partitions %}
- **{{ part.name }}**: {{ part.size_gb }} GB, {{ part.fstype }}, Mount: {{ part.mountpoint }}
{% endfor %}
{% else %}
- No partitions found
{% endif %}
{% if not loop.last %}
---
{% endif %}
{% else %}
No physical disks detected
{% endfor %}

## Disk System Information
{% for disk_sys in safe_disk_sysinfo %}
- **{{ disk_sys.name }}**: Scheduler: {{ disk_sys.scheduler | default('N/A') }}, Read-ahead: {{ disk_sys.read_ahead | default('N/A') }}
{% else %}
- No system disk information available
{% endfor %}

## Mounted Filesystems
{% for disk in disk_info %}
- **{{ disk.mount }}**: {{ disk.size_gb }} GB ({{ disk.fstype }})
{% else %}
- No mounted filesystems information available
{% endfor %}
```

## Основные возможности сбора информации о дисках:

1. **Физические диски**:
   - Имя устройства (sda, sdb, nvme0n1 и т.д.)
   - Модель и производитель
   - Размер в GB и байтах
   - Тип (HDD/SSD) на основе rotational flag
   - Интерфейс подключения (SATA, SAS, NVMe и т.д.)
   - Серийный номер

2. **Разделы**:
   - Имена разделов
   - Размеры разделов
   - Типы файловых систем
   - Точки монтирования

3. **Системная информация**:
   - Планировщики ввода-вывода
   - Read-ahead настройки
   - Размеры через sysfs

4. **Гибкая конфигурация**:
   - Исключение loop-устройств
   - Минимальный размер диска для отображения
   - Возможность отключения сбора

## Пример вывода в CSV:

```
Physical Disks: "sda:476.94GB:SSD:Samsung SSD 860; sdb:931.51GB:HDD:WDC WD10EFRX"
Disk Partitions: "sda1:0.49GB:ext4:/boot; sda2:476.45GB:LVM2_member:Not mounted"
```

## Пример структурированных данных:

```json
{
  "name": "sda",
  "size_gb": 476.94,
  "type": "disk", 
  "model": "Samsung SSD 860",
  "vendor": "Samsung",
  "rotational": false,
  "transport": "sata",
  "serial": "S1234567890",
  "partitions": [
    {
      "name": "sda1",
      "size_gb": 0.49,
      "fstype": "ext4",
      "mountpoint": "/boot"
    }
  ]
}
```

Этот улучшенный сбор информации о дисках дает полную картину о физической структуре хранилища системы, что особенно полезно для планирования апгрейдов, миграций и оптимизации хранилища.





Проблема в том, что в текущей роли Ansible отсутствует вывод информации об установленном программном обеспечении в итоговый CSV отчет. Нужно добавить колонку для отображения отфильтрованного ПО.

Вот исправления, которые нужно внести в файл `main.yml`:

## 1. Добавьте колонку для ПО в заголовок CSV

Замените задачу `Add CSV header`:

```yaml
- name: Add CSV header (only once)
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: "Hostname,Primary IP,All IPs,Distribution,Version,Architecture,CPU Model,CPU Frequency (MHz),CPU Cores,CPU Sockets,CPU Threads,CPU Cache,CPU Topology,CPU Architecture,Current Memory (GB),Max Physical Memory,Memory Details,Swap (GB),Virtualization,Disks,Running Services,Service Details,Installed Software"
    create: yes
    insertbefore: BOF
  delegate_to: localhost
  run_once: true
  when: inventory_hostname == play_hosts[0]
```

## 2. Добавьте данные об установленном ПО в строку отчета

Замените задачу `Add host data to consolidated report`:

```yaml
- name: Add host data to consolidated report
  lineinfile:
    path: "{{ report_path }}/{{ output_filename }}.csv"
    line: |
      "{{ safe_hostname }}","{{ primary_ip }}","{{ ip_addresses | join('; ') }}","{{ safe_distribution }}","{{ safe_distribution_version }}","{{ safe_architecture }}","{{ safe_cpu_model }}","{{ safe_cpu_frequency_mhz }}","{{ safe_processor_cores }}","{{ safe_processor_count }}","{{ safe_processor_vcpus }}","{{ safe_cpu_cache }}","{{ safe_cpu_topology }}","{{ safe_cpu_arch_details }}","{{ safe_memory }}","{{ safe_max_physical_memory }}","{{ safe_memory_slots_details }}","{{ safe_swap }}","{{ safe_virtualization_type }}/{{ safe_virtualization_role }}","{% for disk in disk_info %}{{ disk.mount }}:{{ disk.size_gb }}GB{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_details %}{{ service.name }}{% if not loop.last %}; {% endif %}{% endfor %}","{% for service in services_with_details %}{% if service.description is defined %}Name: {{ service.name }}; Description: {{ service.description }}; Status: {{ service.active_state }}; PID: {{ service.main_pid }}; Memory: {{ service.memory_usage }}; Config: {{ service.config_path }}{% else %}{{ service.name }}{% endif %}{% if not loop.last %}| {% endif %}{% endfor %}","{% for software in filtered_software %}{{ software.name }} ({{ software.version }}){% if not loop.last %}; {% endif %}{% endfor %}"
    create: yes
  delegate_to: localhost
  run_once: false
```

## 3. Улучшите фильтрацию ПО (опционально)

Если хотите улучшить фильтрацию, можно модифицировать задачу `Filter software based on patterns`:

```yaml
- name: Filter software based on patterns
  set_fact:
    filtered_software: |
      {%- set software_list = [] -%}
      {%- if ansible_facts.packages is defined -%}
        {%- for pattern in software_filter -%}
          {%- for pkg_name, pkg_info in ansible_facts.packages.items() -%}
            {%- set pkg_lower = pkg_name | lower -%}
            {%- set pattern_lower = pattern | lower -%}
            {%- if pattern_lower in pkg_lower or pkg_lower in pattern_lower -%}
              {%- set software_item = {"name": pkg_name, "version": pkg_info[0].version} -%}
              {%- if software_item not in software_list -%}
                {%- if software_list.append(software_item) -%}{%- endif -%}
              {%- endif -%}
            {%- endif -%}
          {%- endfor -%}
        {%- endfor -%}
      {%- endif -%}
      {{ software_list | unique | sort(attribute='name') }}
```

## 4. Убедитесь, что переменные определены

Добавьте в файл `defaults/main.yml` или в переменные плейбука:

```yaml
# Фильтры для сбора информации
software_filter: []
service_filter: []
exclude_mount_types: []
exclude_mount_points: []

# Настройки отчета
report_path: "/tmp/ansible_reports"
output_filename: "system_inventory"
include_all_ips: false
collect_detailed_service_info: false
collect_service_descriptions: false
encrypt_report: false
remove_unencrypted: false
vault_password_file: ""
```

## 5. Пример использования

При запуске роли укажите фильтры для ПО:

```yaml
- hosts: all
  vars:
    software_filter:
      - "python"
      - "nginx"
      - "mysql"
      - "postgresql"
      - "docker"
    service_filter:
      - "nginx"
      - "mysql"
      - "docker"
  roles:
    - your-role-name
```

После этих изменений в CSV отчете появится новая колонка "Installed Software", в которой будет перечислено все найденное программное обеспечение, соответствующее указанным фильтрам, в формате: `package_name (version); package_name2 (version2)`








