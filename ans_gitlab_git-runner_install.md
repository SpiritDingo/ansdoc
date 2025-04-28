# Ansible Playbook для установки и настройки GitLab Runner

Вот пример Ansible playbook для установки и настройки GitLab Runner на Linux-системах (поддерживает Ubuntu/Debian и CentOS/RHEL):

```yaml
---
- name: Install and configure GitLab Runner
  hosts: all
  become: yes
  vars:
    gitlab_runner_version: "16.11.0"
    gitlab_runner_registration_token: "your-registration-token"
    gitlab_runner_coordinator_url: "https://gitlab.com"
    gitlab_runner_executor: "shell"
    gitlab_runner_description: "Ansible deployed runner"
    gitlab_runner_tags: ["ansible", "shell"]
    gitlab_runner_concurrent: 4
    gitlab_runner_user: "gitlab-runner"

  tasks:
    # Установка зависимостей
    - name: Install required packages
      package:
        name:
          - curl
          - ca-certificates
          - gnupg
        state: present

    # Добавление репозитория GitLab Runner
    - name: Add GitLab Runner repository key
      apt_key:
        url: "https://packages.gitlab.com/gitlab/gitlab-runner/gpgkey"
        state: present
      when: ansible_facts['os_family'] == 'Debian'

    - name: Add GitLab Runner repository (Debian/Ubuntu)
      apt_repository:
        repo: "deb https://packages.gitlab.com/gitlab/gitlab-runner/{{ ansible_facts['distribution']|lower }} {{ ansible_facts['distribution_release'] }} main"
        state: present
        filename: "gitlab-runner"
      when: ansible_facts['os_family'] == 'Debian'

    - name: Add GitLab Runner repository (RHEL/CentOS)
      yum_repository:
        name: gitlab-runner
        description: GitLab Runner Repository
        baseurl: "https://packages.gitlab.com/gitlab/gitlab-runner/el/{{ ansible_facts['distribution_major_version'] }}/$basearch"
        gpgkey: "https://packages.gitlab.com/gitlab/gitlab-runner/gpgkey"
        gpgcheck: yes
        enabled: yes
      when: ansible_facts['os_family'] == 'RedHat'

    # Установка GitLab Runner
    - name: Install GitLab Runner
      package:
        name: gitlab-runner
        state: present
        version: "{{ gitlab_runner_version }}"

    # Регистрация раннера
    - name: Register GitLab Runner
      command: >
        gitlab-runner register
        --non-interactive
        --url "{{ gitlab_runner_coordinator_url }}"
        --registration-token "{{ gitlab_runner_registration_token }}"
        --executor "{{ gitlab_runner_executor }}"
        --description "{{ gitlab_runner_description }}"
        --tag-list "{{ gitlab_runner_tags | join(',') }}"
        --run-untagged="false"
        --locked="false"
      args:
        creates: /etc/gitlab-runner/config.toml
      register: runner_register
      changed_when: "'Runner registered successfully' in runner_register.stdout"

    # Настройка конфигурации
    - name: Configure concurrent runners
      lineinfile:
        path: /etc/gitlab-runner/config.toml
        regexp: '^concurrent = '
        line: 'concurrent = {{ gitlab_runner_concurrent }}'
        state: present
      notify: restart gitlab-runner

    # Запуск и включение сервиса
    - name: Ensure GitLab Runner is running and enabled
      service:
        name: gitlab-runner
        state: started
        enabled: yes

  handlers:
    - name: restart gitlab-runner
      service:
        name: gitlab-runner
        state: restarted
```

## Использование

1. Сохраните этот код в файл `install-gitlab-runner.yml`
2. Подготовьте инвентарный файл с хостами, где нужно установить runner
3. Замените `your-registration-token` на ваш реальный токен из GitLab
4. Запустите playbook:

```bash
ansible-playbook -i inventory.ini install-gitlab-runner.yml
```

## Настройки

Вы можете изменить следующие параметры в секции `vars`:

- `gitlab_runner_version` - версия GitLab Runner
- `gitlab_runner_registration_token` - токен регистрации из GitLab
- `gitlab_runner_coordinator_url` - URL вашего GitLab сервера
- `gitlab_runner_executor` - тип исполнителя (shell, docker, kubernetes и т.д.)
- `gitlab_runner_description` - описание раннера
- `gitlab_runner_tags` - теги для раннера
- `gitlab_runner_concurrent` - количество одновременных задач
- `gitlab_runner_user` - пользователь под которым будет работать runner

## Дополнительные настройки

Для более сложных конфигураций (например, Docker executor) вам нужно добавить дополнительные задачи в playbook.