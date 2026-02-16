Ниже представлена структура и содержимое Ansible роли info-collector, которая собирает информацию с целевых хостов и формирует JSON-отчёт на управляющей машине.

Структура роли

```
info-collector/
├── defaults/
│   └── main.yml
├── tasks/
│   └── main.yml
├── meta/
│   └── main.yml
└── README.md
```

Файлы роли

defaults/main.yml – переменные по умолчанию

```yaml
---
# Директория для сохранения отчётов на control node
report_dir: "./reports"

# Список пакетов для проверки (можно дополнять с учётом разных ОС)
package_names:
  - postgresql
  - nginx
  - apache2
  - httpd
```

tasks/main.yml – основные задачи

```yaml
---
- name: Сбор фактов о пакетах (package_facts)
  package_facts:
    manager: auto
  when: ansible_facts.packages is not defined

- name: Создание директории для отчётов на control node
  local_action:
    module: file
    path: "{{ report_dir }}"
    state: directory
  run_once: true
  delegate_to: localhost

- name: Сбор информации о физических дисках из фактов ansible_devices
  set_fact:
    physical_disks: >-
      {{ ansible_facts.devices | dict2items
         | selectattr('value.rotational', 'defined')
         | map(attribute='value') | list }}
  when: ansible_facts.devices is defined

- name: Формирование структуры отчёта
  set_fact:
    report_data:
      hostname: "{{ ansible_facts.hostname | default('') }}"
      ip_addresses:
        all_ipv4: "{{ ansible_facts.all_ipv4_addresses | default([]) }}"
        default_ipv4: "{{ ansible_facts.default_ipv4.address | default('') }}"
      cpu:
        model: "{{ ansible_facts.processor[0] | default('') }}"
        cores: "{{ ansible_facts.processor_cores | default(ansible_facts.processor_count) | default(0) }}"
        vcpus: "{{ ansible_facts.processor_vcpus | default(ansible_facts.processor_count) | default(0) }}"
      ram_mb: "{{ ansible_facts.memtotal_mb | default(0) }}"
      os:
        distribution: "{{ ansible_facts.distribution | default('') }}"
        version: "{{ ansible_facts.distribution_version | default('') }}"
        family: "{{ ansible_facts.os_family | default('') }}"
      disks: "{{ physical_disks | default([]) | map(attribute='value') | map(attribute='size') | list }}"
      physical_disks_detail: "{{ physical_disks | default([]) | map(attribute='value') | map(attribute='size') | list }}"
      installed_software: {}

- name: Детализация физических дисков (тип накопителя)
  set_fact:
    report_data: >-
      {{ report_data | combine({
        'physical_disks_detail': (
          ansible_facts.devices | dict2items
          | selectattr('value.rotational', 'defined')
          | map(attribute='value') | map(attribute='size') | list
        ) | default([])
      }) }}
  when: ansible_facts.devices is defined

- name: Сбор информации об установленных пакетах по фильтру
  set_fact:
    report_data: >-
      {{ report_data | combine({
        'installed_software': report_data.installed_software | combine({
          item: ansible_facts.packages[item] | default([])
        })
      }) }}
  loop: "{{ package_names }}"
  when: ansible_facts.packages is defined

- name: Запись отчёта в JSON файл на control node
  local_action:
    module: copy
    content: "{{ report_data | to_nice_json }}"
    dest: "{{ report_dir }}/{{ inventory_hostname }}_{{ ansible_date_time.date }}.json"
  delegate_to: localhost
  vars:
    ansible_date_time:
      date: "{{ lookup('pipe', 'date +%Y%m%d') }}"
```

meta/main.yml – метаданные роли (минимальные)

```yaml
---
galaxy_info:
  author: "Your Name"
  description: "Сбор информации о хосте и формирование JSON отчёта"
  license: "MIT"
  min_ansible_version: "2.9"
  platforms:
    - name: GenericLinux
      versions:
        - all
  galaxy_tags:
    - system
    - info
    - report
dependencies: []
```

README.md – описание роли

```markdown
# Info Collector Ansible Role

Собирает информацию с целевых хостов и формирует JSON-отчёт на управляющей машине.

## Собираемые данные
- имя хоста
- IP-адреса (все IPv4 и основной)
- CPU (модель, количество ядер, vCPU)
- RAM (в мегабайтах)
- версия ОС (дистрибутив, версия, семейство)
- список физических дисков (имя, размер, модель, тип: SSD/HDD)
- состояние установки пакетов PostgreSQL, nginx, Apache (версии, если установлены)

## Переменные

| Имя | По умолчанию | Описание |
|-----|--------------|----------|
| `report_dir` | `./reports` | Директория на control node для сохранения отчётов |
| `package_names` | `['postgresql', 'nginx', 'apache2', 'httpd']` | Список пакетов для проверки |

## Зависимости

Нет.

## Пример playbook

```yaml
- hosts: all
  gather_facts: yes
  roles:
    - info-collector
```

После выполнения в директории ./reports появятся файлы вида hostname_YYYYMMDD.json.

```

## Примечания

- Роль использует встроенные факты Ansible (собранные модулем `setup`) и модуль `package_facts`.
- Для определения типа диска используется атрибут `rotational` (0 – SSD, 1 – HDD).
- Отчёт сохраняется локально на управляющем хосте, имя файла содержит имя целевого хоста и дату.
- Список пакетов можно адаптировать под конкретные ОС через переменные или `vars/` в зависимости от окружения.