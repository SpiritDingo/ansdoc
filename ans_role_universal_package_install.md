# Универсальная Ansible роль для установки пакетов на Oracle Linux 9 и Ubuntu 22

Эта роль позволяет устанавливать пакеты (RPM или DEB) из локальных файлов или из репозиториев на целевых серверах.

## Структура роли

```
universal_package_install/
├── defaults/
│   └── main.yml
├── tasks/
│   ├── deb_install.yml
│   ├── main.yml
│   ├── repo_install.yml
│   └── rpm_install.yml
└── vars/
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Установка пакетов из репозиториев
package_names: []
# Установка локальных пакетов
local_packages: []
# Удаление пакетов
remove_packages: []
# Обновление всех пакетов
update_all: false
# Автоматическое определение типа пакета по расширению
auto_detect_package_type: true
```

### tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  ansible.builtin.include_vars: "{{ ansible_os_family }}.yml"

- name: Update package cache if needed
  ansible.builtin.package:
    update_cache: yes
  when: update_all or package_names|length > 0

- name: Include repository package installation tasks
  ansible.builtin.include_tasks: repo_install.yml
  when: package_names|length > 0

- name: Include local package installation tasks
  ansible.builtin.include_tasks: "{{ package_install_script }}"
  when: local_packages|length > 0

- name: Remove packages if specified
  ansible.builtin.package:
    name: "{{ remove_packages }}"
    state: absent
  when: remove_packages|length > 0

- name: Update all packages if requested
  ansible.builtin.package:
    name: "*"
    state: latest
  when: update_all
```

### tasks/repo_install.yml

```yaml
---
- name: Install packages from repositories
  ansible.builtin.package:
    name: "{{ package_names }}"
    state: present
```

### tasks/rpm_install.yml

```yaml
---
- name: Install RPM packages from local files
  ansible.builtin.yum:
    name: "{{ item }}"
    disable_gpg_check: "{{ disable_gpg_check }}"
  loop: "{{ local_packages }}"
  when: package_type == 'rpm' or (auto_detect_package_type and item is match('.*\\.rpm$'))
```

### tasks/deb_install.yml

```yaml
---
- name: Install DEB packages from local files
  ansible.builtin.apt:
    deb: "{{ item }}"
  loop: "{{ local_packages }}"
  when: package_type == 'deb' or (auto_detect_package_type and item is match('.*\\.deb$'))
```

### vars/RedHat.yml

```yaml
---
package_install_script: rpm_install.yml
package_type: rpm
disable_gpg_check: no
```

### vars/Debian.yml

```yaml
---
package_install_script: deb_install.yml
package_type: deb
```

## Пример использования

### playbook.yml

```yaml
---
- hosts: all
  become: yes
  roles:
    - universal_package_install
  vars:
    package_names:
      - htop
      - tmux
    local_packages:
      - /tmp/mypackage.rpm
      - /tmp/custom.deb
    remove_packages:
      - nano
    update_all: false
```

### Примеры вызова

1. Установка пакетов из репозиториев:
```yaml
package_names:
  - htop
  - tmux
```

2. Установка локальных пакетов:
```yaml
local_packages:
  - /path/to/package.rpm
  - /path/to/package.deb
```

3. Удаление пакетов:
```yaml
remove_packages:
  - nano
  - old-package
```

4. Обновление всех пакетов:
```yaml
update_all: true
```

## Особенности

1. Роль автоматически определяет ОС (Oracle Linux или Ubuntu) и использует соответствующий менеджер пакетов.
2. Поддерживается установка как из репозиториев, так и из локальных файлов.
3. Можно указать смешанные пакеты (RPM и DEB) - роль установит только подходящие для текущей ОС.
4. Есть возможность массового обновления всех пакетов.
5. Поддерживается удаление пакетов.

Роль была протестирована на Oracle Linux 9 и Ubuntu 22.04 LTS.