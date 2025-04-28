# Ansible роль для проверки смонтированных сетевых папок

Роль проверяет состояние смонтированных сетевых ресурсов (NFS, CIFS) и их доступность.

## Структура роли

```
roles/check_mounts/
├── defaults
│   └── main.yml
├── tasks
│   ├── main.yml
│   ├── check_nfs.yml
│   ├── check_cifs.yml
│   └── report.yml
├── templates
│   └── report.j2
└── handlers
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки проверки
check_mounts:
  - mount_point: "/mnt/nfs_share"   # Точка монтирования для проверки
    expected_type: "nfs"            # Ожидаемый тип (nfs, cifs, smbfs)
    test_file: ".mount_test"        # Имя тестового файла для проверки записи
    required: true                  # Обязательно ли должно быть смонтировано

# Параметры проверки
check_timeout: 10                   # Таймаут проверки в секундах
check_retries: 2                    # Количество попыток проверки
```

### tasks/main.yml

```yaml
---
- name: Gather mounted filesystems facts
  setup:
    gather_subset:
      - '!all'
      - 'mounts'

- name: Include NFS checks
  include_tasks: check_nfs.yml
  when: item.expected_type == 'nfs'
  loop: "{{ check_mounts }}"
  loop_control:
    loop_var: mount_item

- name: Include CIFS checks
  include_tasks: check_cifs.yml
  when: item.expected_type in ['cifs', 'smbfs']
  loop: "{{ check_mounts }}"
  loop_control:
    loop_var: mount_item

- name: Generate check report
  include_tasks: report.yml
```

### tasks/check_nfs.yml

```yaml
---
- name: Check if NFS mount is present
  fail:
    msg: "NFS mount {{ mount_item.mount_point }} not found in mounted filesystems"
  when: >
    mount_item.mount_point not in ansible_mounts|map(attribute='mount')|list
    and mount_item.required

- name: Verify NFS mount type
  fail:
    msg: "Mount {{ mount_item.mount_point }} has wrong type {{ ansible_mounts[mount_item.mount_point]['fstype'] }}, expected {{ mount_item.expected_type }}"
  when: >
    mount_item.mount_point in ansible_mounts
    and ansible_mounts[mount_item.mount_point]['fstype'] != mount_item.expected_type

- name: Test NFS mount accessibility (read)
  stat:
    path: "{{ mount_item.mount_point }}"
  register: mount_stat
  ignore_errors: yes
  retries: "{{ check_retries }}"
  delay: "{{ check_timeout }}"

- name: Verify NFS mount accessibility
  fail:
    msg: "Cannot access NFS mount point {{ mount_item.mount_point }}"
  when: mount_stat is failed and mount_item.required

- name: Test NFS mount write operation
  block:
    - name: Create test file
      file:
        path: "{{ mount_item.mount_point }}/{{ mount_item.test_file }}"
        state: touch
      register: test_file_create

    - name: Verify test file creation
      fail:
        msg: "Cannot create test file in {{ mount_item.mount_point }}"
      when: test_file_create is failed and mount_item.required

    - name: Remove test file
      file:
        path: "{{ mount_item.mount_point }}/{{ mount_item.test_file }}"
        state: absent
      when: test_file_create is success
  when: mount_stat is success
```

### tasks/check_cifs.yml

```yaml
---
- name: Check if CIFS mount is present
  fail:
    msg: "CIFS mount {{ mount_item.mount_point }} not found in mounted filesystems"
  when: >
    mount_item.mount_point not in ansible_mounts|map(attribute='mount')|list
    and mount_item.required

- name: Verify CIFS mount type
  fail:
    msg: "Mount {{ mount_item.mount_point }} has wrong type {{ ansible_mounts[mount_item.mount_point]['fstype'] }}, expected {{ mount_item.expected_type }}"
  when: >
    mount_item.mount_point in ansible_mounts
    and ansible_mounts[mount_item.mount_point]['fstype'] != mount_item.expected_type|replace('smbfs','cifs')

- name: Test CIFS mount accessibility (read)
  stat:
    path: "{{ mount_item.mount_point }}"
  register: mount_stat
  ignore_errors: yes
  retries: "{{ check_retries }}"
  delay: "{{ check_timeout }}"

- name: Verify CIFS mount accessibility
  fail:
    msg: "Cannot access CIFS mount point {{ mount_item.mount_point }}"
  when: mount_stat is failed and mount_item.required

- name: Test CIFS mount write operation
  block:
    - name: Create test file
      file:
        path: "{{ mount_item.mount_point }}/{{ mount_item.test_file }}"
        state: touch
      register: test_file_create

    - name: Verify test file creation
      fail:
        msg: "Cannot create test file in {{ mount_item.mount_point }}"
      when: test_file_create is failed and mount_item.required

    - name: Remove test file
      file:
        path: "{{ mount_item.mount_point }}/{{ mount_item.test_file }}"
        state: absent
      when: test_file_create is success
  when: mount_stat is success
```

### tasks/report.yml

```yaml
---
- name: Collect mount check results
  set_fact:
    mount_check_results: >-
      {{
        mount_check_results | default([]) + [{
          'mount_point': item.mount_point,
          'mounted': item.mount_point in ansible_mounts,
          'type': ansible_mounts[item.mount_point]['fstype'] if item.mount_point in ansible_mounts else 'none',
          'accessible': mount_stat is defined and mount_stat is success,
          'writable': test_file_create is defined and test_file_create is success,
          'required': item.required
        }]
      }}
  loop: "{{ check_mounts }}"
  loop_control:
    loop_var: item
  when: mount_stat is defined

- name: Generate HTML report
  template:
    src: report.j2
    dest: "/var/log/mount_checks_report.html"
  when: mount_check_results is defined
```

### templates/report.j2

```html
<!DOCTYPE html>
<html>
<head>
    <title>Network Mounts Check Report</title>
    <style>
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
        .ok { background-color: #dff0d8; }
        .warning { background-color: #fcf8e3; }
        .error { background-color: #f2dede; }
    </style>
</head>
<body>
    <h1>Network Mounts Check Report</h1>
    <p>Generated on {{ ansible_date_time.iso8601 }}</p>
    
    <table>
        <tr>
            <th>Mount Point</th>
            <th>Mounted</th>
            <th>Type</th>
            <th>Accessible</th>
            <th>Writable</th>
            <th>Status</th>
        </tr>
        {% for item in mount_check_results %}
        <tr class="{% if item.mounted and item.accessible and (not item.required or item.writable) %}ok{% elif not item.required %}warning{% else %}error{% endif %}">
            <td>{{ item.mount_point }}</td>
            <td>{{ item.mounted | ternary('YES', 'NO') }}</td>
            <td>{{ item.type }}</td>
            <td>{{ item.accessible | ternary('YES', 'NO') }}</td>
            <td>{{ item.writable | ternary('YES', 'NO') if item.accessible else 'N/A' }}</td>
            <td>
                {% if item.mounted and item.accessible and (not item.required or item.writable) %}
                    OK
                {% elif not item.required %}
                    WARNING
                {% else %}
                    ERROR
                {% endif %}
            </td>
        </tr>
        {% endfor %}
    </table>
</body>
</html>
```

## Пример использования

1. Создайте playbook `check_network_mounts.yml`:

```yaml
---
- hosts: all
  become: yes
  roles:
    - check_mounts
  vars:
    check_mounts:
      - mount_point: "/mnt/nfs_share"
        expected_type: "nfs"
        test_file: ".mount_test"
        required: true

      - mount_point: "/mnt/samba"
        expected_type: "cifs"
        test_file: ".mount_test"
        required: false
```

2. Запустите playbook:
```bash
ansible-playbook -i inventory check_network_mounts.yml
```

3. Отчет будет сохранен в `/var/log/mount_checks_report.html`

## Особенности

- Проверка наличия монтирования в системе
- Проверка правильного типа файловой системы
- Тестирование доступности на чтение
- Тестирование возможности записи
- Генерация HTML-отчета с цветовой индикацией
- Поддержка обязательных и необязательных монтирований
- Возможность настройки количества попыток и таймаутов

Роль может быть расширена для проверки дополнительных параметров:
- Проверка свободного места
- Проверка прав доступа
- Проверка скорости доступа
- Интеграция с системами мониторинга (Zabbix, Prometheus)