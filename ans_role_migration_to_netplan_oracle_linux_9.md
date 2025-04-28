# Роль Ansible для миграции на Netplan в Oracle Linux 9

Эта роль обеспечивает полную миграцию с традиционных сетевых инструментов RHEL (NetworkManager/network-scripts) на Netplan в Oracle Linux 9.

## Структура роли

```
migrate_to_netplan/
├── defaults/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── install.yml
│   ├── migrate.yml
│   └── cleanup.yml
├── templates/
│   └── 99-migrated.yaml.j2
└── handlers/
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Основные параметры
netplan_backup_dir: "/var/backups/network_config"
netplan_config_file: "/etc/netplan/99-migrated.yaml"

# Сохранять ли старые конфиги
keep_old_configs: true

# Отключать ли NetworkManager полностью
disable_network_manager: true

# Автоматически применять конфигурацию
auto_apply: true
```

### tasks/main.yml

```yaml
---
- name: Включение EPEL репозитория
  include_tasks: install.yml
  tags: install

- name: Миграция конфигурации сети
  include_tasks: migrate.yml
  tags: migrate

- name: Очистка старых конфигов
  include_tasks: cleanup.yml
  tags: cleanup
```

### tasks/install.yml

```yaml
---
- name: Установка EPEL репозитория
  dnf:
    name: epel-release
    state: present

- name: Установка Netplan и зависимостей
  dnf:
    name:
      - netplan
      - python3-netplan
      - network-scripts  # Для обратной совместимости
    state: present
```

### tasks/migrate.yml

```yaml
---
- name: Создание директории для резервных копий
  file:
    path: "{{ netplan_backup_dir }}"
    state: directory
    mode: 0755

- name: Резервное копирование текущей конфигурации сети
  block:
    - name: Копирование ifcfg файлов
      copy:
        remote_src: yes
        src: "/etc/sysconfig/network-scripts/"
        dest: "{{ netplan_backup_dir }}/network-scripts"
    - name: Копирование NetworkManager конфигов
      copy:
        remote_src: yes
        src: "/etc/NetworkManager/"
        dest: "{{ netplan_backup_dir }}/NetworkManager"
  when: keep_old_configs

- name: Получение текущей сетевой конфигурации
  command: nmcli -t -f DEVICE,TYPE,STATE,CONNECTION dev status
  register: nmcli_devices
  changed_when: false

- name: Создание конфигурации Netplan на основе текущих настроек
  template:
    src: 99-migrated.yaml.j2
    dest: "{{ netplan_config_file }}"
    mode: 0644
  notify: apply netplan config

- name: Остановка и отключение NetworkManager
  service:
    name: NetworkManager
    state: stopped
    enabled: false
  when: disable_network_manager

- name: Остановка и отключение network-scripts
  service:
    name: network
    state: stopped
    enabled: false
```

### tasks/cleanup.yml

```yaml
---
- name: Удаление старых конфигурационных файлов (опционально)
  block:
    - name: Удаление ifcfg файлов
      file:
        path: "/etc/sysconfig/network-scripts/ifcfg-{{ item }}"
        state: absent
      loop: "{{ (nmcli_devices.stdout_lines | select('match', 'ethernet') | map('split', ':') | map('first') | list) }}"
    
    - name: Удаление NetworkManager конфигов
      file:
        path: "/etc/NetworkManager/system-connections/"
        state: absent
  when: not keep_old_configs
```

### handlers/main.yml

```yaml
---
- name: apply netplan config
  command: netplan apply
  when: auto_apply
```

### templates/99-migrated.yaml.j2

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    {% for device in nmcli_devices.stdout_lines %}
    {% if ':ethernet:' in device %}
    {% set dev_name = device.split(':')[0] %}
    {{ dev_name }}:
      dhcp4: {{ 'yes' if 'dhcp' in (lookup('ini', '/etc/sysconfig/network-scripts/ifcfg-' + dev_name + ' section=^BOOTPROXY$ file=/etc/sysconfig/network-scripts/ifcfg-' + dev_name + ' re=True') | default('dhcp', true)) else 'no' }}
      {% if 'dhcp' not in (lookup('ini', '/etc/sysconfig/network-scripts/ifcfg-' + dev_name + ' section=^BOOTPROXY$ file=/etc/sysconfig/network-scripts/ifcfg-' + dev_name + ' re=True') | default('dhcp', true)) %}
      addresses: [{{ lookup('ini', '/etc/sysconfig/network-scripts/ifcfg-' + dev_name + ' section=^IPADDR$ file=/etc/sysconfig/network-scripts/ifcfg-' + dev_name) }}/{{ lookup('ini', '/etc/sysconfig/network-scripts/ifcfg-' + dev_name + ' section=^PREFIX$ file=/etc/sysconfig/network-scripts/ifcfg-' + dev_name) | default('24') }}]
      gateway4: {{ lookup('ini', '/etc/sysconfig/network-scripts/ifcfg-' + dev_name + ' section=^GATEWAY$ file=/etc/sysconfig/network-scripts/ifcfg-' + dev_name) | default('') }}
      nameservers:
        addresses: [{{ lookup('ini', '/etc/sysconfig/network-scripts/ifcfg-' + dev_name + ' section=^DNS1$ file=/etc/sysconfig/network-scripts/ifcfg-' + dev_name) | default('') }}, {{ lookup('ini', '/etc/sysconfig/network-scripts/ifcfg-' + dev_name + ' section=^DNS2$ file=/etc/sysconfig/network-scripts/ifcfg-' + dev_name) | default('') }}]
      {% endif %}
    {% endif %}
    {% endfor %}
```

## Использование роли

1. Добавьте роль в ваш playbook:

```yaml
- hosts: oracle_servers
  become: yes
  roles:
    - migrate_to_netplan
```

2. Пример использования с параметрами:

```yaml
- hosts: oracle_servers
  become: yes
  roles:
    - role: migrate_to_netplan
      vars:
        keep_old_configs: false
        disable_network_manager: true
        auto_apply: true
```

## Особенности реализации

1. **Автоматическое преобразование конфигов**: Роль автоматически конвертирует существующие ifcfg-файлы в формат Netplan.

2. **Безопасность**: По умолчанию сохраняются резервные копии всех сетевых конфигураций.

3. **Гибкость**: Можно выбрать, отключать ли полностью NetworkManager или оставить его для управления другими типами подключений.

4. **Обработка ошибок**: Если автоматическое преобразование не работает, всегда можно восстановить исходную конфигурацию из бэкапа.

## Рекомендации

1. Перед применением в production протестируйте роль в тестовой среде.
2. Убедитесь, что у вас есть физический доступ к серверу или KVM-консоль на случай проблем с сетью.
3. Для сложных сетевых конфигураций может потребоваться ручная донастройка после миграции.
4. Учтите, что Netplan в Oracle Linux - это нестандартное решение и может потребовать дополнительной поддержки.