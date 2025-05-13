# Ansible Role for MS Defender Information Collection on Linux Servers

Below is an Ansible role that collects information about Microsoft Defender status on Linux servers, including hostname, Defender status, and Organization ID.

## Role Structure

```
msdefender_info_collector/
├── tasks/
│   └── main.yml
├── templates/
│   └── report.j2
└── defaults/
    └── main.yml
```

## Role Implementation

### tasks/main.yml

```yaml
---
- name: Check if Microsoft Defender is installed
  stat:
    path: /opt/microsoft/mdatp/bin/mdatp
  register: mdatp_installed

- name: Get Defender status information
  command: /opt/microsoft/mdatp/bin/mdatp health
  register: defender_status
  when: mdatp_installed.stat.exists
  ignore_errors: yes

- name: Get organization ID
  command: /opt/microsoft/mdatp/bin/mdatp health --field org_id
  register: org_id
  when: mdatp_installed.stat.exists
  ignore_errors: yes

- name: Set default values when Defender not installed
  set_fact:
    defender_status: "Microsoft Defender not installed"
    org_id: "N/A"
  when: not mdatp_installed.stat.exists

- name: Generate report
  template:
    src: report.j2
    dest: /tmp/msdefender_report.csv
  delegate_to: localhost
  run_once: true
```

### templates/report.j2

```jinja2
Host,Defender Status,Org ID
{% for host in ansible_play_hosts %}
{{ hostvars[host].inventory_hostname }},{{ hostvars[host].defender_status.stdout | default('N/A') | replace('\n', ' ') | replace(',', ';') }},{{ hostvars[host].org_id.stdout | default('N/A') }}
{% endfor %}
```

### defaults/main.yml

```yaml
---
# Default output file location
report_path: /tmp/msdefender_report.csv
```

## Usage

1. Create the role directory structure as shown above
2. Place the files in their respective directories
3. Create a playbook to use the role:

```yaml
---
- name: Collect MS Defender information from Linux servers
  hosts: linux_servers
  become: yes
  roles:
    - msdefender_info_collector
```

4. Run the playbook:

```bash
ansible-playbook defender_info_collection.yml
```

## Output

The role will generate a CSV report at `/tmp/msdefender_report.csv` with the following format:

```
Host,Defender Status,Org ID
server1.example.com,"{""healthy"":true,""healthy_without_exclusions"":true,...}",org12345
server2.example.com,Microsoft Defender not installed,N/A
```

## Notes

1. The role requires root privileges to check Defender status
2. The report will be generated on the Ansible control machine
3. For servers without Defender installed, it will mark them appropriately
4. The health output is sanitized to be CSV-friendly by replacing commas with semicolons

You can customize the report format by modifying the `report.j2` template as needed.