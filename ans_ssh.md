
Ansible-роль для управления SSH-ключами на нескольких хостах, включая проверку, добавление и удаление ключей.
Вот полноценная Ansible-роль для управления SSH-ключами на нескольких хостах, включая проверку, добавление и удаление ключей.

Структура роли
```
roles/ssh_key_management/
├── defaults
│   └── main.yml       # переменные по умолчанию
├── tasks
│   └── main.yml       # основные задачи
├── templates
│   └── authorized_keys.j2  # шаблон для authorized_keys (опционально)
└── README.md
```

1. defaults/main.yml
```YAML
---
# Пользователь, для которого управляем ключами
ssh_user: "{{ ansible_user }}"

# SSH-ключи для добавления (список)
ssh_public_keys_to_add: []

# SSH-ключи для удаления (список)
ssh_public_keys_to_remove: []

# Строгое соответствие (удаляет все ключи, кроме указанных в ssh_public_keys_to_add)
strict_mode: false

# Права для .ssh директории и authorized_keys
ssh_dir_mode: "0700"
authorized_keys_mode: "0600"
```
2. tasks/main.yml
```YAML
---
- name: Ensure .ssh directory exists
  ansible.builtin.file:
    path: "/home/{{ ssh_user }}/.ssh"
    state: directory
    mode: "{{ ssh_dir_mode }}"
    owner: "{{ ssh_user }}"
    group: "{{ ssh_user }}"
  when: ssh_user != 'root'

- name: Ensure .ssh directory exists for root
  ansible.builtin.file:
    path: "/root/.ssh"
    state: directory
    mode: "{{ ssh_dir_mode }}"
  when: ssh_user == 'root'

- name: Get current authorized_keys content
  ansible.builtin.shell:
    cmd: "cat /home/{{ ssh_user }}/.ssh/authorized_keys 2>/dev/null || true"
  when: ssh_user != 'root'
  register: current_keys
  changed_when: false

- name: Get current authorized_keys content for root
  ansible.builtin.shell:
    cmd: "cat /root/.ssh/authorized_keys 2>/dev/null || true"
  when: ssh_user == 'root'
  register: current_keys
  changed_when: false

- name: Initialize keys list
  ansible.builtin.set_fact:
    final_keys: "{{ current_keys.stdout_lines | default([]) }}"

# Удаление указанных ключей
- name: Remove specified keys
  ansible.builtin.set_fact:
    final_keys: "{{ final_keys | reject('match', item) | list }}"
  loop: "{{ ssh_public_keys_to_remove }}"
  when: ssh_public_keys_to_remove | length > 0

# Режим strict: оставить только ключи из ssh_public_keys_to_add
- name: Enforce strict mode (remove all keys not in ssh_public_keys_to_add)
  ansible.builtin.set_fact:
    final_keys: "{{ ssh_public_keys_to_add }}"
  when: strict_mode

# Добавление новых ключей (если их еще нет)
- name: Add new keys
  ansible.builtin.set_fact:
    final_keys: "{{ (final_keys + ssh_public_keys_to_add) | unique }}"
  when: ssh_public_keys_to_add | length > 0 and not strict_mode

- name: Create authorized_keys file
  ansible.builtin.copy:
    dest: "/home/{{ ssh_user }}/.ssh/authorized_keys"
    content: "{{ final_keys | join('\n') }}"
    owner: "{{ ssh_user }}"
    group: "{{ ssh_user }}"
    mode: "{{ authorized_keys_mode }}"
  when: ssh_user != 'root'

- name: Create authorized_keys file for root
  ansible.builtin.copy:
    dest: "/root/.ssh/authorized_keys"
    content: "{{ final_keys | join('\n') }}"
    mode: "{{ authorized_keys_mode }}"
  when: ssh_user == 'root'

- name: Verify keys were applied
  ansible.builtin.debug:
    msg: |
      SSH keys for {{ ssh_user }}:
      {{ final_keys | join('\n      ') }}
```
3. Пример использования роли
playbook.yml
---
- hosts: all
  become: yes
  roles:
    - role: ssh_key_management
      vars:
        ssh_user: "deploy"
        ssh_public_keys_to_add:
          - "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC1... user1@example.com"
          - "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC2... user2@example.com"
        ssh_public_keys_to_remove:
          - "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC0... olduser@example.com"
        strict_mode: false
4. Дополнительные возможности
Проверка ключей без изменений
Для проверки текущих ключей без внесения изменений:

- hosts: all
  tasks:
    - name: Check current SSH keys
      ansible.builtin.shell: "cat /home/{{ ansible_user }}/.ssh/authorized_keys 2>/dev/null || echo 'No keys found'"
      register: ssh_keys
      changed_when: false
    
    - name: Display current keys
      ansible.builtin.debug:
        var: ssh_keys.stdout_lines
Примеры запуска
Добавить ключи:

ansible-playbook -i inventory playbook.yml
Проверить ключи (dry-run):

ansible-playbook -i inventory playbook.yml --tags debug --check
Строгий режим (оставить только указанные ключи):

strict_mode: true
5. Безопасность
Всегда проверяйте добавляемые SSH-ключи
Используйте ansible-vault для защиты sensitive data
Рекомендуется использовать strict_mode: true для production-серверов
Эта роль обеспечивает полный цикл управления SSH-ключами и может быть легко интегрирована в ваш Ansible workflow.