# Ansible роль для интеграции NetBox с Ansible Semaphore

Эта роль настраивает интеграцию между NetBox (IPAM/DCIM) и Ansible Semaphore (веб-интерфейс для Ansible), позволяя использовать данные из NetBox как динамический инвентарь в Semaphore.

## Структура роли

```
netbox-semaphore-integration/
├── defaults/
│   └── main.yml          # Значения переменных по умолчанию
├── files/
│   ├── netbox-inventory.py  # Скрипт динамического инвентаря
│   └── netbox.env        # Файл с переменными окружения
├── templates/
│   └── netbox-credentials.j2  # Шаблон для хранения учетных данных
├── tasks/
│   └── main.yml          # Основные задачи установки
└── vars/
    └── main.yml          # Основные переменные роли
```

## Основные файлы роли

### defaults/main.yml

```yaml
---
# Настройки NetBox
netbox_url: "https://netbox.example.com"
netbox_api_token: "your_netbox_api_token"

# Настройки Semaphore
semaphore_install_dir: "/opt/semaphore"
semaphore_user: "semaphore"
semaphore_group: "semaphore"

# Настройки динамического инвентаря
inventory_script_path: "{{ semaphore_install_dir }}/inventory/netbox.py"
inventory_config_path: "{{ semaphore_install_dir }}/inventory/netbox.env"

# Параметры запросов к NetBox
netbox_query_timeout: 30
netbox_verify_ssl: true
```

### tasks/main.yml

```yaml
---
- name: Установка зависимостей
  apt:
    name:
      - python3
      - python3-pip
      - python3-requests
    state: present
    update_cache: yes

- name: Создание директории для скриптов инвентаря
  file:
    path: "{{ semaphore_install_dir }}/inventory"
    state: directory
    owner: "{{ semaphore_user }}"
    group: "{{ semaphore_group }}"
    mode: '0755'

- name: Копирование скрипта динамического инвентаря
  copy:
    src: "netbox-inventory.py"
    dest: "{{ inventory_script_path }}"
    owner: "{{ semaphore_user }}"
    group: "{{ semaphore_group }}"
    mode: '0755'

- name: Создание конфигурационного файла для NetBox
  template:
    src: "netbox-credentials.j2"
    dest: "{{ inventory_config_path }}"
    owner: "{{ semaphore_user }}"
    group: "{{ semaphore_group }}"
    mode: '0600'

- name: Установка необходимых Python библиотек
  pip:
    name:
      - pynetbox
      - requests
    executable: pip3

- name: Проверка работы скрипта инвентаря
  command: "python3 {{ inventory_script_path }} --list"
  register: inventory_test
  changed_when: false
  ignore_errors: yes

- name: Настройка задачи в Semaphore для синхронизации инвентаря
  community.general.ini_file:
    path: "{{ semaphore_install_dir }}/config.json"
    section: inventory
    option: "NetBox Dynamic Inventory"
    value: "{{ inventory_script_path }}"
    backup: yes
  notify: restart semaphore
```

### files/netbox-inventory.py

```python
#!/usr/bin/env python3

import os
import json
import pynetbox
from argparse import ArgumentParser

def load_config():
    config = {
        'NETBOX_URL': os.getenv('NETBOX_URL'),
        'NETBOX_TOKEN': os.getenv('NETBOX_TOKEN'),
        'NETBOX_SSL_VERIFY': os.getenv('NETBOX_SSL_VERIFY', 'true').lower() == 'true'
    }
    return config

def get_netbox_inventory():
    config = load_config()
    
    nb = pynetbox.api(
        config['NETBOX_URL'],
        token=config['NETBOX_TOKEN']
    )
    nb.http_session.verify = config['NETBOX_SSL_VERIFY']
    
    inventory = {
        '_meta': {
            'hostvars': {}
        },
        'all': {
            'hosts': [],
            'children': []
        }
    }
    
    # Получаем устройства из NetBox
    devices = nb.dcim.devices.filter(status='active')
    
    for device in devices:
        hostname = device.name
        inventory['all']['hosts'].append(hostname)
        inventory['_meta']['hostvars'][hostname] = {
            'ansible_host': device.primary_ip.address.split('/')[0] if device.primary_ip else '',
            'device_role': device.device_role.slug,
            'device_type': device.device_type.model,
            'site': device.site.slug,
            'tenant': device.tenant.name if device.tenant else '',
            'platform': device.platform.slug if device.platform else '',
            'tags': [tag.slug for tag in device.tags]
        }
        
        # Группы по ролям устройств
        role_group = f"role_{device.device_role.slug}"
        if role_group not in inventory:
            inventory[role_group] = {'hosts': []}
        inventory[role_group]['hosts'].append(hostname)
        
        # Группы по сайтам
        site_group = f"site_{device.site.slug}"
        if site_group not in inventory:
            inventory[site_group] = {'hosts': []}
        inventory[site_group]['hosts'].append(hostname)
        
        # Группы по платформам
        if device.platform:
            platform_group = f"platform_{device.platform.slug}"
            if platform_group not in inventory:
                inventory[platform_group] = {'hosts': []}
            inventory[platform_group]['hosts'].append(hostname)
    
    return inventory

def main():
    parser = ArgumentParser()
    parser.add_argument('--list', action='store_true')
    parser.add_argument('--host')
    args = parser.parse_args()
    
    if args.list:
        print(json.dumps(get_netbox_inventory(), indent=2))
    elif args.host:
        inventory = get_netbox_inventory()
        hostvars = inventory['_meta']['hostvars'].get(args.host, {})
        print(json.dumps(hostvars, indent=2))
    else:
        parser.print_help()

if __name__ == '__main__':
    main()
```

### templates/netbox-credentials.j2

```ini
# NetBox API credentials
NETBOX_URL={{ netbox_url }}
NETBOX_TOKEN={{ netbox_api_token }}
NETBOX_SSL_VERIFY={{ 'true' if netbox_verify_ssl else 'false' }}
```

## Использование роли

1. Создайте playbook для использования роли:

```yaml
---
- hosts: semaphore_server
  become: yes
  roles:
    - netbox-semaphore-integration
  vars:
    netbox_url: "https://your-netbox-instance.com"
    netbox_api_token: "your_api_token_here"
```

2. Примените playbook:

```bash
ansible-playbook -i inventory.ini semaphore_netbox_integration.yml
```

## Настройка в Semaphore

После применения роли необходимо выполнить следующие шаги в веб-интерфейсе Semaphore:

1. Перейдите в раздел "Inventory"
2. Нажмите "Add Inventory"
3. Введите имя (например, "NetBox Dynamic Inventory")
4. Выберите тип "Static / File"
5. Укажите путь к скрипту: `/opt/semaphore/inventory/netbox.py`
6. Сохраните изменения

## Дополнительные возможности

1. **Фильтрация устройств**: Вы можете модифицировать скрипт `netbox-inventory.py` для фильтрации устройств по определенным критериям (например, только активные устройства с определенными тегами).

2. **Дополнительные переменные**: Расширьте скрипт для включения дополнительных переменных из NetBox (например, custom fields).

3. **Периодическая синхронизация**: Настройте задачу в Semaphore для регулярного обновления инвентаря из NetBox.

4. **Безопасность**: Для защиты API токена NetBox используйте Ansible Vault:

```bash
ansible-vault encrypt_string 'your_api_token_here' --name 'netbox_api_token'
```

Вставьте полученное значение в переменные вместо открытого токена.

Эта интеграция позволит вам использовать актуальные данные из NetBox (устройства, IP-адреса, роли, сайты) непосредственно в Ansible Semaphore для выполнения playbook.