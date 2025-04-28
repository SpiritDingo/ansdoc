# Ansible роль для сбора информации о пользователях на Linux сервере

Эта роль собирает различную информацию о пользователях на Linux сервере, включая системных и обычных пользователей, их привилегии и активность.

## Структура роли

```
roles/user_info/
├── tasks/
│   └── main.yml
├── templates/
│   └── user_report.j2
├── vars/
│   └── main.yml
└── defaults/
    └── main.yml
```

## Файлы роли

### tasks/main.yml

```yaml
---
- name: Gather system information
  setup:
    gather_subset:
      - '!all'
      - '!min'
    filter: ansible_distribution*

- name: Create output directory
  ansible.builtin.file:
    path: "{{ output_dir }}"
    state: directory
    mode: '0755'

- name: Get all users
  ansible.builtin.command: getent passwd
  register: all_users
  changed_when: false

- name: Get sudoers
  ansible.builtin.command: grep -Po '^sudo.+:\K.*$' /etc/group
  register: sudo_users
  changed_when: false

- name: Get currently logged in users
  ansible.builtin.command: who
  register: logged_users
  changed_when: false

- name: Get last logged in users
  ansible.builtin.command: last -n 10
  register: last_logged_users
  changed_when: false

- name: Get users with login shells
  ansible.builtin.command: grep -v '/nologin\|/false' /etc/passwd
  register: shell_users
  changed_when: false

- name: Get locked users
  ansible.builtin.command: "getent shadow | awk -F: '$2 !~ /^[!*]/ {print $1}'"
  register: locked_users
  changed_when: false

- name: Generate user report
  ansible.builtin.template:
    src: user_report.j2
    dest: "{{ output_dir }}/user_report_{{ ansible_hostname }}.txt"
    mode: '0644'

- name: Show report path
  ansible.builtin.debug:
    msg: "User report generated at {{ output_dir }}/user_report_{{ ansible_hostname }}.txt"
```

### templates/user_report.j2

```jinja2
User Information Report for {{ ansible_hostname }}
Generated on {{ ansible_date_time.iso8601 }}
OS: {{ ansible_distribution }} {{ ansible_distribution_version }}

=== All Users ===
{{ all_users.stdout }}

=== Sudo Users ===
{{ sudo_users.stdout }}

=== Currently Logged In Users ===
{{ logged_users.stdout }}

=== Last Logged In Users (last 10) ===
{{ last_logged_users.stdout }}

=== Users with Login Shells ===
{{ shell_users.stdout }}

=== Locked Users ===
{{ locked_users.stdout_lines | join('\n') }}
```

### defaults/main.yml

```yaml
---
# Default output directory for reports
output_dir: "/tmp/user_reports"
```

## Использование роли

1. Создайте playbook (например, `gather_users.yml`):

```yaml
---
- hosts: all
  become: yes
  roles:
    - user_info
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini gather_users.yml
```

## Дополнительные возможности

Вы можете расширить эту роль, добавив:

1. Сбор информации о домашних директориях пользователей
2. Проверку паролей на истечение срока действия
3. Анализ активности пользователей (когда последний раз логинились)
4. Проверку SSH ключей пользователей
5. Формирование отчета в HTML или JSON формате

Для этого можно добавить дополнительные задачи в `tasks/main.yml` и соответствующие шаблоны.