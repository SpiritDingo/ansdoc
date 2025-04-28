# Ansible скрипт для проверки доступности серверов с помощью ping

Вот несколько вариантов Ansible playbook для проверки доступности серверов:

## 1. Базовый скрипт для проверки ping (ping_check.yml)

```yaml
---
- name: Проверка доступности серверов через ping
  hosts: all
  gather_facts: no

  tasks:
    - name: Проверка соединения с серверами
      ansible.builtin.ping:
```

### Как запустить:
```bash
ansible-playbook -i inventory.ini ping_check.yml
```

## 2. Расширенная версия с выводом результатов (advanced_ping_check.yml)

```yaml
---
- name: Расширенная проверка доступности серверов
  hosts: all
  gather_facts: no
  any_errors_fatal: false

  tasks:
    - name: Проверка соединения ping
      ansible.builtin.ping:
      register: ping_result
      ignore_errors: yes

    - name: Вывод результатов проверки
      debug:
        msg: "Сервер {{ inventory_hostname }} ({{ ansible_host }}) {% if ping_result is success %}ДОСТУПЕН{% else %}НЕДОСТУПЕН{% endif %}"
```

## 3. Скрипт с записью результатов в файл (ping_check_with_output.yml)

```yaml
---
- name: Проверка доступности с сохранением результатов
  hosts: all
  gather_facts: no
  vars:
    output_file: "/tmp/ping_check_results.txt"

  tasks:
    - name: Проверка соединения
      ansible.builtin.ping:
      register: ping_result
      ignore_errors: yes

    - name: Запись результатов в файл
      block:
        - name: Создание директории для результатов
          file:
            path: "{{ output_file | dirname }}"
            state: directory
            mode: '0755'
          when: output_file | dirname != ''

        - name: Запись статуса доступности
          lineinfile:
            path: "{{ output_file }}"
            line: "{{ inventory_hostname }} - {% if ping_result is success %}OK{% else %}FAIL{% endif %}"
            create: yes
      delegate_to: localhost
      run_once: yes
```

## 4. Проверка доступности без playbook (прямая команда)

Вы можете использовать модуль ping напрямую без создания playbook:

```bash
ansible all -i inventory.ini -m ping
```

## 5. Пример inventory файла (inventory.ini)

```ini
[web_servers]
web1 ansible_host=192.168.1.10
web2 ansible_host=192.168.1.11

[db_servers]
db1 ansible_host=192.168.1.20
db2 ansible_host=192.168.1.21

[all:vars]
ansible_user=admin
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

## 6. Дополнительные опции проверки

Если вам нужно проверить доступность по определенному порту (не только ICMP ping):

```yaml
---
- name: Проверка доступности порта
  hosts: all
  gather_facts: no

  tasks:
    - name: Проверка TCP соединения на порт 22
      wait_for:
        host: "{{ ansible_host }}"
        port: 22
        timeout: 5
      delegate_to: localhost
```

## 7. Параллельная проверка

Для ускорения проверки множества серверов можно увеличить количество параллельных соединений:

```bash
ansible all -i inventory.ini -m ping -f 10  # 10 параллельных процессов
```

или для playbook:

```bash
ansible-playbook -i inventory.ini ping_check.yml -f 10
```

Эти скрипты помогут вам быстро проверить доступность ваших серверов и идентифицировать проблемные узлы.