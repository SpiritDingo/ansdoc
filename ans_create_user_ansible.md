# Ansible скрипт для создания учётной записи на Linux сервере

Вот пример Ansible playbook для создания пользователя на Linux сервере:

## 1. Базовый playbook (create_user.yml)

```yaml
---
- name: Create user account on Linux servers
  hosts: all
  become: yes
  vars:
    username: "newuser"
    password: "securepassword123"
    groups: "sudo"
    ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ..."

  tasks:
    - name: Create user account
      user:
        name: "{{ username }}"
        password: "{{ password | password_hash('sha512') }}"
        groups: "{{ groups }}"
        append: yes
        shell: /bin/bash
        create_home: yes
        home: "/home/{{ username }}"

    - name: Set up authorized key for SSH access
      ansible.posix.authorized_key:
        user: "{{ username }}"
        state: present
        key: "{{ ssh_key }}"
```

## 2. Улучшенная версия с проверками (advanced_create_user.yml)

```yaml
---
- name: Advanced user creation playbook
  hosts: all
  become: yes
  vars:
    username: "newuser"
    password: "securepassword123"
    groups: "sudo"
    ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ..."
    user_comment: "Ansible created user"

  tasks:
    - name: Check if user already exists
      ansible.builtin.command: "id {{ username }}"
      register: user_exists
      ignore_errors: yes
      changed_when: false

    - name: Create user account if not exists
      user:
        name: "{{ username }}"
        password: "{{ password | password_hash('sha512') }}"
        groups: "{{ groups }}"
        append: yes
        shell: /bin/bash
        create_home: yes
        home: "/home/{{ username }}"
        comment: "{{ user_comment }}"
        system: no
      when: user_exists.rc != 0

    - name: Set up authorized key for SSH access
      ansible.posix.authorized_key:
        user: "{{ username }}"
        state: present
        key: "{{ ssh_key }}"
        exclusive: yes

    - name: Ensure sudo access for the user
      lineinfile:
        path: /etc/sudoers
        state: present
        line: "{{ username }} ALL=(ALL) NOPASSWD:ALL"
        validate: "visudo -cf %s"
```

## 3. Как использовать

1. Сохраните playbook в файл (например, `create_user.yml`)
2. Создайте inventory файл (например, `hosts.ini`):

```ini
[servers]
server1 ansible_host=192.168.1.100
server2 ansible_host=192.168.1.101

[servers:vars]
ansible_user=admin
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

3. Запустите playbook:

```bash
ansible-playbook -i hosts.ini create_user.yml
```

## 4. Безопасные рекомендации

1. **Пароли**: Лучше использовать Ansible Vault для хранения паролей
2. **SSH ключи**: Генерируйте уникальные ключи для каждого пользователя
3. **Привилегии**: Давайте минимально необходимые права (не всегда `NOPASSWD:ALL`)

## 5. Пример с Ansible Vault

1. Создайте зашифрованный файл с переменными (`vars.yml`):

```bash
ansible-vault create vars.yml
```

2. Добавьте в него:

```yaml
username: "secureuser"
password: "verysecretpassword"
ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ..."
```

3. Обновите playbook для использования vault:

```yaml
- name: Create user with vault
  hosts: all
  become: yes
  vars_files:
    - vars.yml
  tasks:
    # ... те же задачи, но теперь используют переменные из vault
```

4. Запустите с vault:

```bash
ansible-playbook -i hosts.ini --ask-vault-pass create_user.yml
```