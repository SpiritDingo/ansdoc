# Роль Ansible для обновления серверов Linux с уведомлением о недоступных серверах

Создадим роль, которая будет обновлять серверы Linux (apt/yum) и отправлять email с списком недоступных серверов.

## Структура роли

```
roles/update_servers/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── email_template.j2
└── README.md
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки почты для уведомлений
mail_to: "admin@example.com"
mail_from: "ansible@example.com"
mail_subject: "Недоступные серверы при обновлении"
smtp_server: "smtp.example.com"
smtp_port: 587
smtp_user: "user@example.com"
smtp_pass: "password"

# Настройки обновления
update_packages: yes
upgrade_packages: yes
autoremove: yes
```

### tasks/main.yml

```yaml
---
- name: Проверить доступность серверов
  ping:
  register: ping_results
  ignore_errors: yes
  tags: always

- name: Собрать список недоступных серверов
  set_fact:
    unreachable_hosts: "{{ unreachable_hosts|default([]) + [item] }}"
  loop: "{{ ansible_play_hosts }}"
  when: hostvars[item].ping_results is undefined or hostvars[item].ping_results.failed
  tags: always

- name: Обновить индексы пакетов (APT)
  apt:
    update_cache: yes
  when: 
    - ansible_pkg_mgr == 'apt'
    - update_packages
    - ping_results is not failed

- name: Обновить индексы пакетов (YUM)
  yum:
    update_cache: yes
  when: 
    - ansible_pkg_mgr == 'yum'
    - update_packages
    - ping_results is not failed

- name: Обновить все пакеты (APT)
  apt:
    upgrade: dist
    autoremove: "{{ autoremove }}"
  when: 
    - ansible_pkg_mgr == 'apt'
    - upgrade_packages
    - ping_results is not failed

- name: Обновить все пакеты (YUM)
  yum:
    name: '*'
    state: latest
  when: 
    - ansible_pkg_mgr == 'yum'
    - upgrade_packages
    - ping_results is not failed

- name: Отправить email с недоступными серверами
  community.general.mail:
    host: "{{ smtp_server }}"
    port: "{{ smtp_port }}"
    username: "{{ smtp_user }}"
    password: "{{ smtp_pass }}"
    to: "{{ mail_to }}"
    subject: "{{ mail_subject }}"
    body: "{{ lookup('template', 'email_template.j2') }}"
    from: "{{ mail_from }}"
  when: unreachable_hosts is defined and unreachable_hosts|length > 0
  delegate_to: localhost
  run_once: yes
```

### templates/email_template.j2

```jinja2
Следующие серверы были недоступны во время выполнения обновления:

{% for host in unreachable_hosts %}
- {{ host }}
{% endfor %}

Дата выполнения: {{ ansible_date_time.iso8601 }}

Это автоматическое сообщение, пожалуйста не отвечайте на него.
```

## Использование роли

1. Создайте playbook (например, `update_servers.yml`):

```yaml
---
- name: Обновление серверов Linux
  hosts: all
  become: yes
  gather_facts: yes

  roles:
    - update_servers
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini update_servers.yml
```

## Дополнительные настройки

1. Для работы отправки email необходимо установить коллекцию community.general:
```bash
ansible-galaxy collection install community.general
```

2. Можно переопределить параметры по умолчанию при вызове роли:

```yaml
- name: Обновление серверов Linux
  hosts: all
  become: yes
  vars:
    mail_to: "your-email@example.com"
    upgrade_packages: no  # только обновить индексы, без обновления пакетов
  roles:
    - update_servers
```

3. Для тестирования доступности серверов без выполнения обновления:
```bash
ansible-playbook -i inventory.ini update_servers.yml --tags always
```

Эта роль обеспечит безопасное обновление серверов и уведомление администратора о проблемах с доступностью.