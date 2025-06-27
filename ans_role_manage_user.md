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