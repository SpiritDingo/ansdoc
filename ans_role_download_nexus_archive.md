Вот пример Ansible роли для скачивания ZIP-архива из Nexus с авторизацией, распаковки и удаления архива:

Структура роли:

```
roles/download_nexus_archive/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
└── vars/
    └── main.yml
```

defaults/main.yml (переменные по умолчанию):

```yaml
---
# URL Nexus репозитория
nexus_url: "https://nexus.example.com"
# Путь к артефакту в Nexus
artifact_path: "repository/path/to/artifact.zip"
# Имя файла для сохранения
archive_name: "artifact.zip"
# Кредиты Nexus
nexus_username: "username"
nexus_password: "password"
# Целевой каталог на сервере
target_dir: "/opt/myapp"
# Временный каталог для загрузки
temp_dir: "/tmp"
```

vars/main.yml (основные переменные):

```yaml
---
# Полный URL для загрузки
download_url: "{{ nexus_url }}/{{ artifact_path }}"
# Полный путь к архиву
archive_path: "{{ temp_dir }}/{{ archive_name }}"
```

tasks/main.yml (основные задачи):

```yaml
---
- name: Create target directory if not exists
  ansible.builtin.file:
    path: "{{ target_dir }}"
    state: directory
    mode: '0755'

- name: Download archive from Nexus with authentication
  ansible.builtin.get_url:
    url: "{{ download_url }}"
    dest: "{{ archive_path }}"
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    validate_certs: no  # установите yes если есть валидный SSL сертификат
    force: yes
    mode: '0644'
  register: download_result

- name: Extract archive to target directory
  ansible.builtin.unarchive:
    src: "{{ archive_path }}"
    dest: "{{ target_dir }}"
    remote_src: yes
    creates: "{{ target_dir }}/extracted_marker"  # опционально, чтобы не распаковывать каждый раз
  when: download_result.changed

- name: Remove downloaded archive
  ansible.builtin.file:
    path: "{{ archive_path }}"
    state: absent
  when: download_result.changed
```

Пример использования в playbook:

playbook.yml:

```yaml
---
- name: Download and extract archive from Nexus
  hosts: all
  roles:
    - role: download_nexus_archive
      vars:
        nexus_url: "https://nexus.company.com"
        artifact_path: "repository/myapp/releases/v1.0/app.zip"
        nexus_username: "deploy_user"
        nexus_password: "{{ nexus_deploy_password }}"  # лучше использовать vault
        target_dir: "/opt/myapp"
        temp_dir: "/tmp"
```

Безопасная версия с использованием Ansible Vault:

1. Создайте зашифрованный файл с паролями:

```bash
ansible-vault create secrets.yml
```

1. Добавьте в secrets.yml:

```yaml
---
nexus_deploy_password: "your_secure_password"
```

1. Обновите playbook:

```yaml
---
- name: Download and extract archive from Nexus
  hosts: all
  vars_files:
    - secrets.yml
  roles:
    - role: download_nexus_archive
      vars:
        nexus_password: "{{ nexus_deploy_password }}"
        # остальные переменные...
```

Улучшенная версия с проверками:

```yaml
---
- name: Check if target directory exists
  ansible.builtin.stat:
    path: "{{ target_dir }}"
  register: target_dir_stat

- name: Create target directory
  ansible.builtin.file:
    path: "{{ target_dir }}"
    state: directory
    mode: '0755'
  when: not target_dir_stat.stat.exists

- name: Download archive from Nexus
  ansible.builtin.get_url:
    url: "{{ download_url }}"
    dest: "{{ archive_path }}"
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    validate_certs: no
    timeout: 30
    headers:
      User-Agent: "ansible-{{ ansible_version }}"
    checksum: "sha256:{{ expected_checksum }}"  # опционально для проверки целостности
  register: download_result
  until: download_result is success
  retries: 3
  delay: 5

- name: Extract archive
  ansible.builtin.unarchive:
    src: "{{ archive_path }}"
    dest: "{{ target_dir }}"
    remote_src: yes
    owner: "{{ target_owner | default(omit) }}"
    group: "{{ target_group | default(omit) }}"
    keep_newer: yes
  when: download_result.changed

- name: Cleanup temporary archive
  ansible.builtin.file:
    path: "{{ archive_path }}"
    state: absent
  when: download_result.changed

- name: Verify extraction
  ansible.builtin.find:
    paths: "{{ target_dir }}"
    file_type: file
  register: extracted_files
  when: download_result.changed
```

Запуск playbook:

```bash
# Обычный запуск
ansible-playbook -i inventory playbook.yml

# С vault паролем
ansible-playbook -i inventory playbook.yml --ask-vault-pass

# Или если пароль в файле
ansible-playbook -i inventory playbook.yml --vault-password-file ~/.vault_pass.txt
```

Эта роль обеспечивает:

1. Загрузку архива из Nexus с аутентификацией
2. Создание целевого каталога при необходимости
3. Распаковку архива в указанный каталог
4. Удаление временного архива после распаковки
5. Идемпотентность (повторный запуск не будет повторять действия без изменений)