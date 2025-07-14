# Универсальная Ansible роль для установки RPM и DEB пакетов

Эта роль позволяет устанавливать пакеты как на Oracle Linux 9 (RPM), так и на Ubuntu 22.04/24.04 (DEB) из файлов, хранящихся в самой роли.

## Структура роли

```
universal_package_installer/
├── defaults/
│   └── main.yml
├── files/
│   ├── ubuntu/
│   │   └── package.deb
│   └── oraclelinux/
│       └── package.rpm
├── tasks/
│   └── main.yml
└── vars/
    └── main.yml
```

## Код роли

### defaults/main.yml

```yaml
---
# Имена пакетов (без расширения)
package_name: "my_package"

# Путь для временного хранения пакетов на целевых хостах
temp_path: "/tmp"

# Удалять пакет после установки (true/false)
remove_package_after_install: true
```

### tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ item }}"
  with_first_found:
    - "{{ ansible_distribution|lower }}_{{ ansible_distribution_major_version }}.yml"
    - "{{ ansible_distribution|lower }}.yml"
    - "{{ ansible_os_family|lower }}.yml"
  tags: always

- name: Copy package to target host
  ansible.builtin.copy:
    src: "{{ package_source_path }}"
    dest: "{{ temp_path }}/{{ package_file_name }}"
    mode: '0644'

- name: Install package on RPM-based systems
  ansible.builtin.yum:
    name: "{{ temp_path }}/{{ package_file_name }}"
    state: present
  when: ansible_pkg_mgr == "yum"

- name: Install package on DEB-based systems
  ansible.builtin.apt:
    deb: "{{ temp_path }}/{{ package_file_name }}"
    state: present
  when: ansible_pkg_mgr == "apt"

- name: Remove package file if configured
  ansible.builtin.file:
    path: "{{ temp_path }}/{{ package_file_name }}"
    state: absent
  when: remove_package_after_install
```

### vars/oraclelinux_9.yml

```yaml
---
package_file_name: "{{ package_name }}.rpm"
package_source_path: "oraclelinux/{{ package_file_name }}"
```

### vars/ubuntu.yml

```yaml
---
package_file_name: "{{ package_name }}.deb"
package_source_path: "ubuntu/{{ package_file_name }}"
```

## Использование роли

1. Поместите ваши пакеты в соответствующие подкаталоги:
   - RPM пакеты в `files/oraclelinux/`
   - DEB пакеты в `files/ubuntu/`

2. Добавьте роль в ваш playbook:

```yaml
- hosts: all
  roles:
    - universal_package_installer
```

3. Вы можете переопределить параметры по умолчанию:

```yaml
- hosts: all
  roles:
    - role: universal_package_installer
      vars:
        package_name: "custom_package"
        remove_package_after_install: false
```

## Особенности работы

1. Роль автоматически определяет тип ОС и использует соответствующий менеджер пакетов (yum для Oracle Linux, apt для Ubuntu).
2. Пакеты копируются во временную директорию перед установкой.
3. После установки пакет может быть автоматически удален (настраивается параметром `remove_package_after_install`).
4. Роль поддерживает как Ubuntu 22.04, так и 24.04 через общий файл переменных для Ubuntu.

## Дополнительные улучшения

Для более сложных сценариев вы можете расширить роль, добавив:

1. Проверку контрольных сумм пакетов
2. Установку зависимостей
3. Обработку нескольких пакетов одновременно
4. Кастомные скрипты до/после установки

Эта роль предоставляет базовый функционал для кроссплатформенной установки пакетов из файлов в Ansible.