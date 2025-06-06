# Ansible роль для установки агента BI.ZONE EDR в Linux

Вот пример Ansible роли для установки агента BI.ZONE EDR в Linux системах (RPM и DEB):

## Структура роли

```
roles/install_bz_agent/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── env_vars.j2
└── vars
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки по умолчанию
bz_agent_package_path: ""
bz_agent_type: ""  # rpm или deb

# Параметры агента
bz_authority_service: ""
bz_sensors_service: ""
bz_polling_period: "300s"
bz_dial_timeout: "15s"
bz_agent_groups: ""
```

### tasks/main.yml

```yaml
---
- name: Проверка обязательных параметров
  assert:
    that:
      - bz_agent_package_path != ""
      - bz_agent_type in ['rpm', 'deb']
      - bz_authority_service != ""
      - bz_sensors_service != ""
    fail_msg: "Не указаны обязательные параметры для установки агента"

- name: Создание временного файла с переменными окружения
  template:
    src: env_vars.j2
    dest: /tmp/bz_agent_env_vars
    mode: 0644

- name: Установка RPM пакета
  when: bz_agent_type == 'rpm'
  ansible.builtin.command: |
    set -a && . /tmp/bz_agent_env_vars && sudo -E yum install -y {{ bz_agent_package_path }}
  args:
    executable: /bin/bash

- name: Установка DEB пакета
  when: bz_agent_type == 'deb'
  ansible.builtin.command: |
    set -a && . /tmp/bz_agent_env_vars && sudo -E apt-get install -y {{ bz_agent_package_path }}
  args:
    executable: /bin/bash

- name: Проверка состояния службы bzsenagent
  ansible.builtin.command: systemctl status bzsenagent.service
  register: service_status
  ignore_errors: yes

- name: Вывод статуса службы
  ansible.builtin.debug:
    var: service_status.stdout_lines

- name: Удаление временного файла с переменными
  ansible.builtin.file:
    path: /tmp/bz_agent_env_vars
    state: absent
```

### templates/env_vars.j2

```bash
# Environment variables for BI.ZONE EDR agent installation
export BZ_AUTHORITY_SERVICE={{ bz_authority_service }}
export BZ_SENSORS_SERVICE={{ bz_sensors_service }}
export BZ_POLLING_PERIOD={{ bz_polling_period }}
export BZ_DIAL_TIMEOUT={{ bz_dial_timeout }}
{% if bz_agent_groups != "" %}
export BZ_AGENT_GROUPS='{{ bz_agent_groups }}'
{% endif %}
```

## Пример использования роли

```yaml
- hosts: linux_servers
  become: yes
  roles:
    - role: install_bz_agent
      vars:
        bz_agent_package_path: "/path/to/bzagent.rpm"
        bz_agent_type: "rpm"
        bz_authority_service: "management.example.com:80"
        bz_sensors_service: "management.example.com:443"
        bz_polling_period: "300s"
        bz_dial_timeout: "15s"
        bz_agent_groups: "nixprod"
```

## Примечания

1. Роль поддерживает установку как RPM, так и DEB пакетов
2. Все переменные окружения передаются с префиксом BZ_ как требуется
3. Используется sudo -E для сохранения переменных окружения
4. После установки проверяется статус службы bzsenagent.service
5. Для работы роли необходимо указать обязательные параметры:
   - Путь к пакету
   - Тип пакета (rpm/deb)
   - Адреса серверов управления (authority и sensors)

Роль можно расширить для поддержки дополнительных параметров агента или добавить обработку ошибок.