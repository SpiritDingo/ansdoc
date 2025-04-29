# Ansible роль для создания пользователя с паролем и правами sudo на Linux

Вот полная Ansible роль, которая создает пользователя, устанавливает пароль и предоставляет права sudo.

## Структура роли

```
roles/sudo_user_with_password/
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
sudo_user_name: "admin"

# Пароль пользователя (в зашифрованном виде)
# Для генерации используйте: mkpasswd --method=SHA-512
# Или: python3 -c 'import crypt; print(crypt.crypt("пароль", crypt.mksalt(crypt.METHOD_SHA512)))'
sudo_user_password: ""

# Группы пользователя (через запятую)
sudo_user_groups: "sudo"

# Дополнительные группы (опционально)
sudo_user_extra_groups: ""

# Shell для пользователя
sudo_user_shell: "/bin/bash"

# Срок действия пароля (в днях, -1 для отключения)
sudo_user_password_max_days: -1

# Запретить вход по паролю (разрешить только SSH-ключ)
sudo_user_disable_password_auth: false
```

## 2. tasks/main.yml

```yaml
---
- name: Ensure sudo package is installed
  package:
    name: sudo
    state: present

- name: Create sudo group if it doesn't exist
  group:
    name: "{{ sudo_user_groups }}"
    state: present

- name: Create user with password
  user:
    name: "{{ sudo_user_name }}"
    groups: "{{ sudo_user_groups }}{% if sudo_user_extra_groups %},{{ sudo_user_extra_groups }}{% endif %}"
    append: yes
    shell: "{{ sudo_user_shell }}"
    password: "{{ sudo_user_password }}"
    update_password: on_create

- name: Lock password auth if required
  command: "passwd -l {{ sudo_user_name }}"
  when: sudo_user_disable_password_auth

- name: Set password expiration
  command: "chage -M {{ sudo_user_password_max_days }} {{ sudo_user_name }}"
  when: sudo_user_password_max_days != -1

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
{{ sudo_user_name }} ALL=(ALL:ALL) {% if sudo_user_disable_password_auth %}NOPASSWD:{% endif %} ALL
```

## 4. README.md

```markdown
# Sudo User with Password Role

Создает пользователя с паролем и правами sudo.

## Переменные

- `sudo_user_name`: Имя пользователя (по умолчанию: "admin")
- `sudo_user_password`: Зашифрованный пароль (обязательно)
- `sudo_user_groups`: Основные группы (по умолчанию: "sudo")
- `sudo_user_extra_groups`: Дополнительные группы
- `sudo_user_shell`: Shell пользователя (по умолчанию: "/bin/bash")
- `sudo_user_password_max_days`: Срок действия пароля (-1 - без срока)
- `sudo_user_disable_password_auth`: Отключить вход по паролю (false по умолчанию)

## Пример использования

1. Сгенерируйте зашифрованный пароль:

```bash
python3 -c 'import crypt; print(crypt.crypt("ваш_пароль", crypt.mksalt(crypt.METHOD_SHA512)))'
```

2. Создайте playbook `create_user.yml`:

```yaml
---
- hosts: all
  become: yes
  vars:
    encrypted_password: "$6$rounds=656000$W2X5.4UZJkDhgvB/$W8J..." # ваш зашифрованный пароль
  
  roles:
    - role: sudo_user_with_password
      vars:
        sudo_user_name: "superuser"
        sudo_user_password: "{{ encrypted_password }}"
        sudo_user_extra_groups: "docker,adm"
```

3. Запустите playbook:

```bash
ansible-playbook -i inventory.ini create_user.yml -K
```

## Безопасность

1. Для большей безопасности рекомендуется:
   - Использовать сложные пароли
   - Включить `sudo_user_disable_password_auth: true` и использовать SSH-ключи
   - Регулярно менять пароли
   - Ограничивать sudo-права при необходимости
```

## Альтернативный вариант с незашифрованным паролем

Если вы хотите передавать пароль в открытом виде (не рекомендуется для production):

1. Измените задачу создания пользователя в `tasks/main.yml`:

```yaml
- name: Create user with password
  user:
    name: "{{ sudo_user_name }}"
    groups: "{{ sudo_user_groups }}{% if sudo_user_extra_groups %},{{ sudo_user_extra_groups }}{% endif %}"
    append: yes
    shell: "{{ sudo_user_shell }}"
    password: "{{ sudo_user_password | password_hash('sha512') }}"
    update_password: on_create
```

2. Тогда можно передавать пароль в открытом виде:

```yaml
vars:
  sudo_user_password: "my_secure_password"
```

Однако этот подход менее безопасен, так как пароль будет виден в логах и истории.