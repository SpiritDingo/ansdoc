Ниже представлена готовая Ansible-роль kernel_report, которая собирает информацию о текущей версии ядра и формирует единый CSV-отчёт по всем целевым хостам (Ubuntu 22.04/24.04, Oracle Linux 8/9).

Структура роли

```
roles/
└── kernel_report/
    ├── defaults/
    │   └── main.yml
    ├── tasks/
    │   └── main.yml
    └── templates/
        └── report.csv.j2
```

1. defaults/main.yml — путь к выходному файлу

```yaml
---
# Путь к итоговому CSV-файлу на управляющем узле
report_dest: "/tmp/kernel_report.csv"
```

2. tasks/main.yml — основные задачи

```yaml
---
- name: Убедиться, что факты собраны
  ansible.builtin.setup:
  when: ansible_facts is not defined or ansible_facts == {}

- name: Сформировать единый CSV-отчёт
  ansible.builtin.template:
    src: report.csv.j2
    dest: "{{ report_dest }}"
    mode: '0644'
  delegate_to: localhost
  run_once: true
```

3. templates/report.csv.j2 — шаблон CSV

```jinja2
Server Name,IP Address,OS Type,Kernel Version
{% for host in ansible_play_hosts_all %}
{{ hostvars[host].ansible_facts['hostname'] }},{{ hostvars[host].ansible_default_ipv4.address | default('N/A') }},{{ hostvars[host].ansible_facts['distribution'] }} {{ hostvars[host].ansible_facts['distribution_version'] }},{{ hostvars[host].ansible_facts['kernel'] }}
{% endfor %}
```

Пример использования (playbook)

```yaml
---
- name: Сбор информации о ядре
  hosts: all
  gather_facts: yes
  roles:
    - kernel_report
```

Запуск:

```bash
ansible-playbook -i inventory playbook.yml
```

Пояснения

· Роль собирает факты (если ещё не собраны) и затем один раз (run_once) с делегированием на localhost генерирует CSV на основе переменных всех хостов текущего плейбука (ansible_play_hosts_all).
· В отчёте фигурируют:
  · Server Name – ansible_facts.hostname
  · IP Address – IPv4-адрес по умолчанию (ansible_default_ipv4.address)
  · OS Type – дистрибутив + версия, например Ubuntu 22.04 или OracleLinux 9.2
  · Kernel Version – полная строка ядра (uname -r)
· Результат будет записан в файл, указанный в report_dest (по умолчанию /tmp/kernel_report.csv). При повторном запуске файл перезаписывается.

При необходимости путь к выходному файлу можно переопределить через переменную:

```bash
ansible-playbook ... -e report_dest=/home/user/report.csv
```


Ошибка возникает, когда Ansible пытается обратиться к hostvars[host].ansible_facts['hostname'], но для одного или нескольких хостов переменная ansible_facts не определена или не содержит ключа hostname.
Это типично для хостов, которые были недоступны во время сбора фактов, или для которых сбор фактов отключён.

Решение

Нужно обезопасить шаблон проверками и значениями по умолчанию.

Исправленный templates/report.csv.j2

```jinja2
Server Name,IP Address,OS Type,Kernel Version
{% for host in ansible_play_hosts_all %}
{% if hostvars[host].ansible_facts is defined %}
{{ hostvars[host].ansible_facts.hostname | default(host) }},{{ hostvars[host].ansible_default_ipv4.address | default('N/A') }},{{ hostvars[host].ansible_facts.distribution | default('Unknown') }} {{ hostvars[host].ansible_facts.distribution_version | default('') }},{{ hostvars[host].ansible_facts.kernel | default('N/A') }}
{% else %}
{{ host }},N/A,No facts collected,N/A
{% endif %}
{% endfor %}
```

Что изменилось

· Добавлена проверка {% if hostvars[host].ansible_facts is defined %} – если факты не собраны, выводится строка с пометкой «No facts collected».
· Для всех полей используются фильтры | default(...), чтобы избежать ошибки при отсутствии конкретных ключей.
· Вместо ansible_play_hosts_all можно использовать ansible_play_hosts (только хосты, для которых выполнялась текущая задача). Но в случае run_once на localhost это не всегда помогает, поэтому лучше оставить ansible_play_hosts_all с проверками.

Дополнительная рекомендация

Убедитесь, что в плейбуке включён gather_facts: yes (по умолчанию он включен).
Если какие-то хосты недоступны, выполните предварительный ping или используйте стратегию ignore_unreachable: yes на уровне плейбука, чтобы избежать остановки выполнения:

```yaml
- name: Сбор информации о ядре
  hosts: all
  gather_facts: yes
  ignore_unreachable: yes
  roles:
    - kernel_report
```

После исправления шаблона отчёт будет успешно сформирован даже при отсутствии данных по некоторым хостам.