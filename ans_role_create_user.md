# Ansible роль для создания пользователя с правами sudo на Linux

Создадим Ansible роль, которая будет добавлять нового пользователя, настраивать для него SSH-доступ с авторизацией по ключу и предоставлять права sudo.

## Структура роли

Создадим следующую структуру файлов:

```
roles/sudo_user/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── sudoers.j2
└── README.md
```

## 1. defaults/main.yml

```yaml
---
# Имя пользователя по умолчанию
sudo_user_name: "deploy"
# Группы, в которые будет добавлен пользователь (через запятую)
sudo_user_groups: "sudo"
# Дополнительные группы (опционально)
sudo_user_extra_groups: ""
# Путь к SSH-ключу пользователя (публичный ключ)
sudo_user_ssh_key: ""
# Закомментируйте следующую строку, если не хотите отключать парольную аутентификацию
sudo_user_disable_password_auth: true
# Shell для пользователя (по умолчанию /bin/bash)
sudo_user_shell: "/bin/bash"
# Срок действия пароля (в днях, -1 для отключения)
sudo_user_password_max_days: -1
```

## 2. tasks/main.yml

```yaml
---
- name: Ensure sudo package is installed
  apt:
    name: sudo
    state: present
  when: ansible_os_family == 'Debian'

- name: Create sudo user group if it doesn't exist
  group:
    name: "{{ sudo_user_groups }}"
    state: present

- name: Create user with specified groups
  user:
    name: "{{ sudo_user_name }}"
    groups: "{{ sudo_user_groups }}{% if sudo_user_extra_groups %},{{ sudo_user_extra_groups }}{% endif %}"
    append: yes
    shell: "{{ sudo_user_shell }}"
    password_lock: "{{ sudo_user_disable_password_auth }}"
    update_password: on_create

- name: Set password expiration
  command: "chage -M {{ sudo_user_password_max_days }} {{ sudo_user_name }}"
  when: sudo_user_password_max_days != -1

- name: Set up authorized key for the user
  authorized_key:
    user: "{{ sudo_user_name }}"
    key: "{{ sudo_user_ssh_key }}"
    state: present
  when: sudo_user_ssh_key != ""

- name: Ensure sudoers.d directory exists
  file:
    path: /etc/sudoers.d
    state: directory
    mode: 0750

- name: Configure sudo privileges
  template:
    src: sudoers.j2
    dest: /etc/sudoers.d/{{ sudo_user_name }}
    mode: 0440
    validate: 'visudo -cf %s'
```

## 3. templates/sudoers.j2

```
# Sudo configuration for {{ sudo_user_name }}
{{ sudo_user_name }} ALL=(ALL:ALL) NOPASSWD: ALL
```

## 4. README.md

```markdown
# Sudo User Role

This role creates a user with sudo privileges on Linux systems.

## Variables

- `sudo_user_name`: Username to create (default: "deploy")
- `sudo_user_groups`: Primary group (default: "sudo")
- `sudo_user_extra_groups`: Additional groups (optional)
- `sudo_user_ssh_key`: SSH public key for the user
- `sudo_user_disable_password_auth`: Disable password authentication (default: true)
- `sudo_user_shell`: User shell (default: "/bin/bash")
- `sudo_user_password_max_days`: Password expiration in days (-1 to disable)

## Example Playbook

```yaml
- hosts: servers
  roles:
    - role: sudo_user
      vars:
        sudo_user_name: "admin"
        sudo_user_ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC..."
        sudo_user_extra_groups: "docker,www-data"
```

## Использование роли

1. Создайте playbook (например, `create_user.yml`):

```yaml
---
- hosts: all
  become: yes
  roles:
    - role: sudo_user
      vars:
        sudo_user_name: "admin"
        sudo_user_ssh_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
        sudo_user_extra_groups: "docker"
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini create_user.yml -K
```

## Примечания

1. Роль поддерживает как Debian-based, так и RHEL-based системы.
2. Для безопасности рекомендуется:
   - Всегда использовать SSH-ключи вместо паролей
   - Ограничивать sudo-права при необходимости (изменив шаблон sudoers.j2)
   - Регулярно обновлять SSH-ключи
3. Файл sudoers проверяется на валидность перед применением через `visudo -cf`.