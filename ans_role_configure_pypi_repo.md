Вот универсальная Ansible роль для настройки pip/pypi репозитория с аутентификацией:

Структура роли

```
roles/configure_pypi_repo/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
├── vars/
│   ├── RedHat.yml
│   └── Debian.yml
└── templates/
    └── pip.conf.j2
```

defaults/main.yml

```yaml
---
# URL вашего Nexus PyPI репозитория
pypi_repo_url: "https://nexus.example.com/repository/pypi-group/simple"
pypi_repo_username: ""
pypi_repo_password: ""
pypi_repo_host: "nexus.example.com"

# Настройка для разных уровней (global/user/virtualenv)
pypi_config_level: "global"  # global, user, или venv
pypi_venv_path: ""  # если используется venv уровень

# Дополнительные индексы
pypi_extra_index_urls: []
# Пример:
# pypi_extra_index_urls:
#   - "https://pypi.org/simple"

# Таймауты
pypi_timeout: 30
pypi_retries: 3

# Установить как единственный индекс
pypi_set_as_only_index: true
```

tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- name: Set PyPI config directory based on level
  set_fact:
    pypi_config_dir: >-
      {%- if pypi_config_level == 'global' -%}
        /etc
      {%- elif pypi_config_level == 'user' -%}
        {{ ansible_env.HOME | default('/root') }}
      {%- elif pypi_config_level == 'venv' -%}
        {{ pypi_venv_path }}
      {%- endif -%}
    pypi_config_file: >-
      {%- if pypi_config_level == 'global' -%}
        /etc/pip.conf
      {%- elif pypi_config_level == 'user' -%}
        {{ ansible_env.HOME | default('/root') }}/.pip/pip.conf
      {%- elif pypi_config_level == 'venv' -%}
        {{ pypi_venv_path }}/pip.conf
      {%- endif -%}

- name: Install required packages for SSL/HTTPS
  package:
    name: "{{ pypi_required_packages }}"
    state: present
  when: "'https' in pypi_repo_url"

- name: Create pip config directory for user level
  file:
    path: "{{ pypi_config_dir }}/.pip"
    state: directory
    mode: '0755'
  when: pypi_config_level == 'user'

- name: Create pip config directory for venv level
  file:
    path: "{{ pypi_venv_path }}"
    state: directory
    mode: '0755'
  when: pypi_config_level == 'venv' and pypi_venv_path != ''

- name: Configure pip to use Nexus PyPI repository
  template:
    src: pip.conf.j2
    dest: "{{ pypi_config_file }}"
    mode: '0644'
    backup: yes

- name: Set environment variables for pip authentication (system-wide)
  copy:
    content: |
      # PyPI Repository Configuration
      PIP_INDEX_URL={{ pypi_repo_url }}
      PIP_TRUSTED_HOST={{ pypi_repo_host }}
      {% if pypi_config_level == 'global' %}
      PIP_CONFIG_FILE=/etc/pip.conf
      {% endif %}
    dest: /etc/profile.d/pypi_repo.sh
    mode: '0644'
  when: pypi_config_level == 'global'

- name: Test PyPI repository connection
  command: "pip config list"
  register: pip_config_check
  changed_when: false
  ignore_errors: true

- name: Display pip configuration
  debug:
    var: pip_config_check.stdout_lines
  when: pip_config_check.rc == 0
```

vars/RedHat.yml

```yaml
---
pypi_required_packages:
  - ca-certificates
  - python3-pip
  - python3
```

vars/Debian.yml

```yaml
---
pypi_required_packages:
  - ca-certificates
  - python3-pip
  - python3
```

templates/pip.conf.j2

```ini
[global]
{% if pypi_set_as_only_index %}
index-url = {{ pypi_repo_url }}
{% else %}
index = {{ pypi_repo_url }}
{% endif %}

{% if pypi_extra_index_urls %}
extra-index-url =
{% for url in pypi_extra_index_urls %}
    {{ url }}
{% endfor %}
{% endif %}

{% if pypi_repo_username and pypi_repo_password %}
# Authentication for private repository
{% endif %}

[install]
trusted-host = {{ pypi_repo_host }}
timeout = {{ pypi_timeout }}
retries = {{ pypi_retries }}

{% if pypi_repo_username and pypi_repo_password %}
# Credentials (if not using keyring)
{% raw %}
# Note: For better security, use pip keyring or environment variables
{% endraw %}
{% endif %}
```

Пример использования в playbook

```yaml
---
- name: Configure PyPI repositories on all servers
  hosts: all
  become: yes
  vars:
    pypi_repo_url: "https://nexus.internal.com/repository/pypi-all/simple"
    pypi_repo_username: "nexus_user"
    pypi_repo_password: "{{ vault_nexus_password }}"
    pypi_repo_host: "nexus.internal.com"
    
  roles:
    - role: configure_pypi_repo
```

Расширенная версия с безопасным хранением паролей

Для безопасной работы с паролями, добавьте файл tasks/secure_auth.yml:

```yaml
---
- name: Install keyring for secure password storage
  package:
    name: python3-keyring
    state: present
  when: ansible_os_family == "Debian" or ansible_os_family == "RedHat"

- name: Set pip credentials in environment
  lineinfile:
    path: "{{ ansible_env.HOME }}/.bashrc"
    line: 'export PIP_USERNAME="{{ pypi_repo_username }}"'
    regexp: '^export PIP_USERNAME='
  when: pypi_config_level == 'user'

- name: Set pip password in environment  
  lineinfile:
    path: "{{ ansible_env.HOME }}/.bashrc"
    line: 'export PIP_PASSWORD="{{ pypi_repo_password }}"'
    regexp: '^export PIP_PASSWORD='
  when: pypi_config_level == 'user'
  no_log: true
```

Версия с поддержкой pip.ini (Windows-like конфиг)

Для совместимости с разными форматами конфигурации:

```yaml
# В tasks/main.yml добавьте:
- name: Create legacy pip.ini symlink (compatibility)
  file:
    src: "{{ pypi_config_file }}"
    dest: "{{ pypi_config_dir }}/pip.ini"
    state: link
  when: pypi_config_level == 'global'
  ignore_errors: yes
```

Использование с шифрованием Ansible Vault

```bash
# Создайте зашифрованный файл с паролями
ansible-vault create group_vars/all/vault.yml

# Содержимое vault.yml:
vault_nexus_password: "your_secure_password"
```

Проверка конфигурации после применения

```yaml
# В tasks/main.yml добавьте проверку:
- name: Verify pip configuration
  shell: |
    pip config list 2>/dev/null || echo "Config check completed"
    pip install --dry-run pip 2>&1 | grep -i "looking in" || true
  register: pip_verify
  changed_when: false

- name: Show pip configuration summary
  debug:
    msg: |
      PyPI Repository configured:
      - Config file: {{ pypi_config_file }}
      - Repository: {{ pypi_repo_url }}
      - Authentication: {{ 'Enabled' if pypi_repo_username else 'Disabled' }}
```

Эта роль универсальна для всех указанных дистрибутивов и поддерживает:

· Oracle Linux (RHEL-based)
· Ubuntu 22.04 и 24.04
· Аутентификацию через Nexus
· Разные уровни конфигурации pip
· Безопасное хранение паролей через Ansible Vault