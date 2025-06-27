# Универсальная Ansible роль для создания пользователя (с SSH-ключом и/или паролем)

Вот объединенная роль, которая позволяет создавать пользователей с:
- SSH-ключом
- Паролем
- Или обоими методами аутентификации

## Структура роли

```
roles/manage_user/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
├── templates/
│   └── authorized_keys.j2
└── README.md
```

## Содержание файлов

### defaults/main.yml
```yaml
---
# Основные параметры пользователя
user_name: ansible
user_groups: "sudo"
user_home: "/home/{{ user_name }}"
user_shell: "/bin/bash"
user_state: present  # или 'absent' для удаления пользователя

# Настройки SSH-ключа
user_ssh_key: ""  # публичный SSH-ключ (если пустой - ключ не добавляется)
user_ssh_key_manage: true  # управлять SSH-ключами пользователя

# Настройки пароля
user_password: ""  # хэшированный пароль (если пустой - пароль не меняется)
user_password_manage: false  # управлять паролем пользователя

# Настройки sudo
user_sudo_access: true  # давать ли права sudo
user_sudo_nopasswd: true  # разрешить sudo без пароля
```

### tasks/main.yml
```yaml
---
- name: Ensure group exists
  group:
    name: "{{ user_name }}"
    state: "{{ user_state }}"
  when: user_state == 'present'

- name: Manage user account
  user:
    name: "{{ user_name }}"
    group: "{{ user_name }}"
    groups: "{{ user_groups }}"
    append: yes
    shell: "{{ user_shell }}"
    home: "{{ user_home }}"
    password: "{{ user_password if user_password_manage else omit }}"
    state: "{{ user_state }}"
    system: no
    create_home: yes
    generate_ssh_key: no

- name: Setup SSH authorized key
  block:
    - name: Ensure .ssh directory exists
      file:
        path: "{{ user_home }}/.ssh"
        state: directory
        owner: "{{ user_name }}"
        group: "{{ user_name }}"
        mode: 0700

    - name: Add authorized key
      template:
        src: authorized_keys.j2
        dest: "{{ user_home }}/.ssh/authorized_keys"
        owner: "{{ user_name }}"
        group: "{{ user_name }}"
        mode: 0600
  when: 
    - user_state == 'present'
    - user_ssh_key_manage
    - user_ssh_key != ""

- name: Configure sudo access
  copy:
    dest: "/etc/sudoers.d/{{ user_name }}"
    content: |
      {{ user_name }} ALL=(ALL) {% if user_sudo_nopasswd %}NOPASSWD:{% endif %}ALL
    validate: "visudo -cf %s"
  when: 
    - user_state == 'present'
    - user_sudo_access

- name: Clean up sudo config when user is removed
  file:
    path: "/etc/sudoers.d/{{ user_name }}"
    state: absent
  when: user_state == 'absent'
```

### templates/authorized_keys.j2
```
{{ user_ssh_key }}
```

## Примеры использования

### 1. Создать пользователя с SSH-ключом
```yaml
- hosts: all
  become: yes
  roles:
    - role: manage_user
      vars:
        user_name: "deployer"
        user_ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ..."
```

### 2. Создать пользователя с паролем
```yaml
- hosts: all
  become: yes
  roles:
    - role: manage_user
      vars:
        user_name: "admin"
        user_password_manage: true
        user_password: "$6$rounds=656000$WX6bUJ5dtzQx9Ni/$6XewumE7W2nGZ1HcR9ZrO2YQYMS0vL7sFZQ7yY2X..."
```

### 3. Создать пользователя с обоими методами аутентификации
```yaml
- hosts: all
  become: yes
  roles:
    - role: manage_user
      vars:
        user_name: "superuser"
        user_ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ..."
        user_password_manage: true
        user_password: "$6$rounds=656000$WX6bUJ5dtzQx9Ni/$6XewumE7W2nGZ1HcR9ZrO2YQYMS0vL7sFZQ7yY2X..."
        user_sudo_nopasswd: false  # требовать пароль для sudo
```

### 4. Удалить пользователя
```yaml
- hosts: all
  become: yes
  roles:
    - role: manage_user
      vars:
        user_name: "olduser"
        user_state: absent
```

## Генерация хэша пароля

Для генерации хэша пароля выполните:
```bash
python3 -c 'import crypt; print(crypt.crypt("ваш_пароль", crypt.mksalt(crypt.METHOD_SHA512)))'
```

## Особенности роли

1. **Гибкость**: Можно управлять всеми аспектами пользователя независимо
2. **Безопасность**: Пароль и SSH-ключ обрабатываются раздельно
3. **Идемпотентность**: Роль можно запускать многократно без побочных эффектов
4. **Удаление**: Поддержка полного удаления пользователя
5. **Совместимость**: Работает на всех современных Linux-дистрибутивах

Для максимальной безопасности рекомендуется:
- Использовать Ansible Vault для хранения паролей
- По возможности использовать только SSH-ключи
- Ограничивать sudo-права только необходимыми командами




# Ошибка "Group sudo does not exist" в Ansible роли

Эта ошибка возникает, когда на целевом хосте отсутствует группа `sudo`, которую вы пытаетесь назначить пользователю. Давайте модифицируем универсальную роль, чтобы сделать её более устойчивой к таким ситуациям.

## Исправленная роль с обработкой отсутствующих групп

### Изменения в `tasks/main.yml`:

```yaml
---
- name: Check if sudo group exists
  stat:
    path: "/etc/group"
  register: group_info
  changed_when: false

- name: Ensure sudo group exists (for Debian/Ubuntu systems)
  group:
    name: sudo
    state: present
  when: 
    - "'sudo' in user_groups"
    - not group_info.stat.exists or "'sudo:' not in lookup('file', '/etc/group')"

- name: Ensure wheel group exists (for RHEL/CentOS systems)
  group:
    name: wheel
    state: present
  when: 
    - "'wheel' in user_groups"
    - not group_info.stat.exists or "'wheel:' not in lookup('file', '/etc/group')"

- name: Ensure primary group exists
  group:
    name: "{{ user_name }}"
    state: "{{ user_state }}"
  when: user_state == 'present'

- name: Manage user account
  user:
    name: "{{ user_name }}"
    group: "{{ user_name }}"
    groups: "{{ user_groups }}"
    append: yes
    shell: "{{ user_shell }}"
    home: "{{ user_home }}"
    password: "{{ user_password if user_password_manage else omit }}"
    state: "{{ user_state }}"
    system: no
    create_home: yes
    generate_ssh_key: no

# Остальные задачи остаются без изменений...
```

### Изменения в `defaults/main.yml`:

```yaml
---
# Основные параметры пользователя
user_name: ansible
user_groups: "{{ 'sudo' if ansible_facts['os_family'] == 'Debian' else 'wheel' }}"
user_home: "/home/{{ user_name }}"
user_shell: "/bin/bash"
user_state: present

# Остальные параметры без изменений...
```

## Решения проблемы

### Вариант 1: Автоматическое определение правильной группы (рекомендуется)

Роль теперь автоматически определяет правильную группу администраторов:
- Для Debian/Ubuntu использует `sudo`
- Для RHEL/CentOS использует `wheel`

### Вариант 2: Явное указание группы в playbook

```yaml
- hosts: all
  become: yes
  roles:
    - role: manage_user
      vars:
        user_name: "deployer"
        user_groups: "wheel"  # Явно указываем группу для RHEL/CentOS
        user_ssh_key: "ssh-rsa AAAAB3..."
```

### Вариант 3: Проверка и создание группы перед выполнением роли

Добавьте эту задачу в ваш playbook перед вызовом роли:

```yaml
- hosts: all
  become: yes
  tasks:
    - name: Detect admin group
      set_fact:
        admin_group: "{{ 'sudo' if ansible_facts['os_family'] == 'Debian' else 'wheel' }}"
    
    - name: Ensure admin group exists
      group:
        name: "{{ admin_group }}"
        state: present
    
    - name: Include user management role
      include_role:
        name: manage_user
      vars:
        user_groups: "{{ admin_group }}"
        user_name: "deployer"
        user_ssh_key: "ssh-rsa AAAAB3..."
```

## Дополнительные улучшения

1. **Проверка существования групп**:
```yaml
- name: Verify requested groups exist
  assert:
    that:
      - "item in ansible_facts['groups']"
    fail_msg: "Group {{ item }} does not exist on the target system"
    success_msg: "Group {{ item }} exists"
  loop: "{{ user_groups.split(',') if user_groups is string else user_groups }}"
  when: user_state == 'present'
```

2. **Логирование**:
```yaml
- name: Display group information
  debug:
    msg: "Available groups: {{ ansible_facts['groups'].keys() | list }}"
  verbosity: 1
```

Эти изменения сделают вашу роль более надежной при работе с разными дистрибутивами Linux.


# Добавление смены пароля в универсальную Ansible роль

Доработаем роль для поддержки смены пароля существующего пользователя. Вот полная реализация:

## Обновленная структура роли

```
roles/manage_user/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
├── templates/
│   └── authorized_keys.j2
└── README.md
```

## Обновленные файлы роли

### defaults/main.yml
```yaml
---
# Основные параметры пользователя
user_name: ansible
user_groups: "{{ 'sudo' if ansible_facts['os_family'] == 'Debian' else 'wheel' }}"
user_home: "/home/{{ user_name }}"
user_shell: "/bin/bash"
user_state: present  # или 'absent' для удаления пользователя

# Настройки SSH-ключа
user_ssh_key: ""  # публичный SSH-ключ (если пустой - ключ не добавляется)
user_ssh_key_manage: true  # управлять SSH-ключами пользователя

# Настройки пароля
user_password: ""  # хэшированный пароль (если пустой - пароль не меняется)
user_password_manage: false  # управлять паролем пользователя
user_password_update: false  # принудительно обновить пароль, даже если пользователь существует

# Настройки sudo
user_sudo_access: true  # давать ли права sudo
user_sudo_nopasswd: true  # разрешить sudo без пароля
```

### tasks/main.yml
```yaml
---
- name: Check if sudo group exists
  stat:
    path: "/etc/group"
  register: group_info
  changed_when: false

- name: Ensure sudo group exists (for Debian/Ubuntu systems)
  group:
    name: sudo
    state: present
  when: 
    - "'sudo' in user_groups"
    - not group_info.stat.exists or "'sudo:' not in lookup('file', '/etc/group')"

- name: Ensure wheel group exists (for RHEL/CentOS systems)
  group:
    name: wheel
    state: present
  when: 
    - "'wheel' in user_groups"
    - not group_info.stat.exists or "'wheel:' not in lookup('file', '/etc/group')"

- name: Check if user exists
  stat:
    path: "{{ user_home }}"
  register: user_info
  changed_when: false

- name: Ensure primary group exists
  group:
    name: "{{ user_name }}"
    state: "{{ user_state }}"
  when: user_state == 'present'

- name: Manage user account
  user:
    name: "{{ user_name }}"
    group: "{{ user_name }}"
    groups: "{{ user_groups }}"
    append: yes
    shell: "{{ user_shell }}"
    home: "{{ user_home }}"
    password: "{{ user_password if (user_password_manage and (user_password_update or not user_info.stat.exists)) else omit }}"
    state: "{{ user_state }}"
    system: no
    create_home: yes
    generate_ssh_key: no
    update_password: "{{ 'always' if (user_password_manage and user_password_update) else 'on_create' }}"

- name: Setup SSH authorized key
  block:
    - name: Ensure .ssh directory exists
      file:
        path: "{{ user_home }}/.ssh"
        state: directory
        owner: "{{ user_name }}"
        group: "{{ user_name }}"
        mode: 0700

    - name: Add authorized key
      template:
        src: authorized_keys.j2
        dest: "{{ user_home }}/.ssh/authorized_keys"
        owner: "{{ user_name }}"
        group: "{{ user_name }}"
        mode: 0600
  when: 
    - user_state == 'present'
    - user_ssh_key_manage
    - user_ssh_key != ""

- name: Configure sudo access
  copy:
    dest: "/etc/sudoers.d/{{ user_name }}"
    content: |
      {{ user_name }} ALL=(ALL) {% if user_sudo_nopasswd %}NOPASSWD:{% endif %}ALL
    validate: "visudo -cf %s"
  when: 
    - user_state == 'present'
    - user_sudo_access

- name: Clean up sudo config when user is removed
  file:
    path: "/etc/sudoers.d/{{ user_name }}"
    state: absent
  when: user_state == 'absent'
```

## Примеры использования

### 1. Создать пользователя с паролем
```yaml
- hosts: all
  become: yes
  roles:
    - role: manage_user
      vars:
        user_name: "admin"
        user_password_manage: true
        user_password: "$6$rounds=656000$WX6bUJ5dtzQx9Ni/$6XewumE7W2nGZ1HcR9ZrO2YQYMS0vL7sFZQ7yY2X..."
```

### 2. Изменить пароль существующего пользователя
```yaml
- hosts: all
  become: yes
  roles:
    - role: manage_user
      vars:
        user_name: "existing_user"
        user_password_manage: true
        user_password_update: true  # Принудительно обновить пароль
        user_password: "$6$newhash$newhashvalue..."
```

### 3. Создать пользователя с SSH-ключом и паролем
```yaml
- hosts: all
  become: yes
  roles:
    - role: manage_user
      vars:
        user_name: "superuser"
        user_ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ..."
        user_password_manage: true
        user_password: "$6$rounds=656000$WX6bUJ5dtzQx9Ni/$6XewumE7W2nGZ1HcR9ZrO2YQYMS0vL7sFZQ7yY2X..."
```

## Генерация хэша пароля

Для генерации хэша пароля выполните:
```bash
python3 -c 'import crypt; print(crypt.crypt("ваш_пароль", crypt.mksalt(crypt.METHOD_SHA512)))'
```

## Особенности реализации

1. **Гибкое управление паролями**:
   - `user_password_manage: true` - включить управление паролями
   - `user_password_update: true` - принудительно обновить пароль (даже для существующих пользователей)

2. **Безопасность**:
   - Пароль всегда передается в хэшированном виде
   - По умолчанию пароль устанавливается только при создании пользователя

3. **Идемпотентность**:
   - Без `user_password_update: true` пароль не будет меняться у существующих пользователей
   - При `user_password_update: true` пароль будет обновляться при каждом запуске

Для максимальной безопасности рекомендуется хранить хэши паролей в Ansible Vault.