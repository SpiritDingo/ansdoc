# Ansible роль для создания пользователя и добавления в группу sudo в Oracle Linux 9

Для Oracle Linux 9 (основанного на RHEL) группа sudo называется `wheel`. Вот обновленная роль:

## Структура роли

```
roles/oracle-user/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
└── vars
    └── main.yml
```

## Содержание файлов

### `roles/oracle-user/defaults/main.yml`

```yaml
---
# Настройки по умолчанию
user_name: "oracle_user"
user_password: ""
user_shell: "/bin/bash"
user_groups: "wheel"  # В Oracle Linux используется группа wheel
user_ssh_key: ""
user_sudo_nopasswd: false
user_create_home: true
user_state: "present"
user_comment: "Oracle Linux User"
```

### `roles/oracle-user/tasks/main.yml`

```yaml
---
- name: Ensure wheel group exists
  group:
    name: wheel
    state: present

- name: Create user with password and add to wheel group
  user:
    name: "{{ user_name }}"
    password: "{{ user_password | password_hash('sha512') }}"
    shell: "{{ user_shell }}"
    groups: "{{ user_groups }}"
    append: yes
    create_home: "{{ user_create_home }}"
    state: "{{ user_state }}"
    comment: "{{ user_comment }}"
  when: user_password != ""

- name: Create user without password and add to wheel group
  user:
    name: "{{ user_name }}"
    shell: "{{ user_shell }}"
    groups: "{{ user_groups }}"
    append: yes
    create_home: "{{ user_create_home }}"
    state: "{{ user_state }}"
    comment: "{{ user_comment }}"
  when: user_password == ""

- name: Add SSH key for the user
  authorized_key:
    user: "{{ user_name }}"
    key: "{{ user_ssh_key }}"
    state: present
  when: user_ssh_key != ""

- name: Configure passwordless sudo if requested
  lineinfile:
    path: /etc/sudoers.d/{{ user_name }}
    state: present
    line: '{{ user_name }} ALL=(ALL) NOPASSWD: ALL'
    validate: 'visudo -cf %s'
    mode: '0440'
  when: user_sudo_nopasswd

- name: Ensure sudoers.d directory exists
  file:
    path: /etc/sudoers.d
    state: directory
    mode: '0750'
```

### `roles/oracle-user/vars/main.yml`

```yaml
---
# Специфичные для Oracle Linux 9 переменные
user_groups: "wheel,oinstall"  # Пример добавления в группы Oracle
```

## Пример playbook для Oracle Linux 9

Создайте файл `oracle-user.yml`:

```yaml
---
- hosts: oracle_servers
  become: yes
  vars:
    # Генерация хеша пароля (лучше использовать vault)
    user_password_hash: "{{ 'yourpassword' | password_hash('sha512') }}"
    
  roles:
    - role: oracle-user
      vars:
        user_name: "oracle_admin"
        user_password: "{{ user_password_hash }}"
        user_ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ..."
        user_sudo_nopasswd: true
        user_groups: "wheel,dba"
        user_comment: "Oracle DB Admin"
```

## Особенности для Oracle Linux 9

1. **Группа sudo**: В Oracle Linux используется группа `wheel` вместо `sudo`.

2. **Безопасность sudo**: Лучше создавать отдельные файлы в `/etc/sudoers.d/` вместо редактирования `/etc/sudoers`.

3. **Группы Oracle**: Если пользователь будет работать с Oracle DB, можно добавить его в группы `oinstall`, `dba`, `oper`.

4. **Сертификаты**: Oracle Linux 9 активно использует PKI, можно добавить задачи для настройки сертификатов.

5. **SELinux**: Убедитесь, что SELinux настроен правильно для нового пользователя.

## Запуск playbook

```bash
ansible-playbook -i inventory.ini oracle-user.yml
```

## Рекомендации по безопасности

1. Всегда используйте `ansible-vault` для хранения паролей:
   ```bash
   ansible-vault encrypt_string 'securepassword' --name 'user_password'
   ```

2. Для Oracle DB администраторов ограничьте sudo права:
   ```yaml
   - name: Configure limited sudo for Oracle admin
     lineinfile:
       path: /etc/sudoers.d/oracle_admin
       line: 'oracle_admin ALL=(oracle) NOPASSWD: /u01/app/oracle/product/*/bin/*'
       validate: 'visudo -cf %s'
   ```

3. Для аудита добавьте задачу:
   ```yaml
   - name: Enable user audit
     lineinfile:
       path: /etc/audit/audit.rules
       line: '-w /home/{{ user_name }} -p wa -k oracle_user'
   ```