Вот пример Ansible playbook для создания пользователя `git-runner` на Linux-системе:

```yaml
---
- name: Create git-runner user
  hosts: all
  become: yes

  vars:
    git_runner_user: git-runner
    git_runner_group: git-runner
    git_runner_home: "/home/{{ git_runner_user }}"
    git_runner_shell: "/bin/bash"
    git_runner_ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD..."  # Замените на ваш публичный ключ

  tasks:
    - name: Ensure group exists
      group:
        name: "{{ git_runner_group }}"
        state: present

    - name: Ensure user exists
      user:
        name: "{{ git_runner_user }}"
        group: "{{ git_runner_group }}"
        home: "{{ git_runner_home }}"
        shell: "{{ git_runner_shell }}"
        create_home: yes
        state: present

    - name: Ensure .ssh directory exists
      file:
        path: "{{ git_runner_home }}/.ssh"
        state: directory
        owner: "{{ git_runner_user }}"
        group: "{{ git_runner_group }}"
        mode: '0700'

    - name: Add authorized key
      ansible.posix.authorized_key:
        user: "{{ git_runner_user }}"
        key: "{{ git_runner_ssh_key }}"
        state: present

    - name: Ensure user has sudo access (опционально)
      copy:
        dest: "/etc/sudoers.d/{{ git_runner_user }}"
        content: "{{ git_runner_user }} ALL=(ALL) NOPASSWD:ALL"
        mode: '0440'
      when: false  # По умолчанию отключено, включите если нужно
```

### Примечания по использованию:
1. Замените значение `git_runner_ssh_key` на реальный публичный SSH-ключ
2. Если не нужен доступ sudo для пользователя, оставьте `when: false` в последней задаче
3. Для изменения shell пользователя измените переменную `git_runner_shell` (например, на `/usr/sbin/nologin` если не нужен интерактивный вход)
4. Для добавления пользователя в дополнительные группы используйте параметр `groups` в задаче создания пользователя

### Использование:
Сохраните как `create_git_runner.yml` и выполните:
```bash
ansible-playbook -i your_inventory create_git_runner.yml
```

При необходимости вы можете добавить дополнительные задачи, например, для установки GitLab Runner или других зависимостей.