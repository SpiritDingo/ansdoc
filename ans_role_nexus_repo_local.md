Вот модифицированная Ansible роль с поддержкой Ubuntu 22.04 и 24.04:

Структура роли:

```
roles/nexus_repo_setup/
├── tasks
│   └── main.yml
├── templates
│   └── nexus-repo.j2
├── vars
│   ├── main.yml
│   └── Ubuntu_22.04.yml
│   └── Ubuntu_24.04.yml
└── defaults
    └── main.yml
```

1. defaults/main.yml:

```yaml
nexus_repo_url: "http://your-nexus-repo/ubuntu"
gpg_key_url: "http://your-nexus-repo/keys/public.gpg"
repo_filename: "nexus-repo.list"
```

2. vars/Ubuntu_22.04.yml:

```yaml
codename: jammy
components: main
```

3. vars/Ubuntu_24.04.yml:

```yaml
codename: noble
components: main
```

4. templates/nexus-repo.j2:

```jinja2
deb {{ nexus_repo_url }} {{ codename }} {{ components }}
```

5. tasks/main.yml:

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_distribution }}_{{ ansible_distribution_version }}.yml"

- name: Download and install GPG key
  become: yes
  ansible.builtin.get_url:
    url: "{{ gpg_key_url }}"
    dest: /usr/share/keyrings/nexus-repo-keyring.gpg
    mode: '0644'

- name: Create repository configuration file
  become: yes
  ansible.builtin.template:
    src: nexus-repo.j2
    dest: "/etc/apt/sources.list.d/{{ repo_filename }}"
    mode: '0644'
  vars:
    codename: "{{ codename }}"
    components: "{{ components }}"

- name: Update package cache
  become: yes
  ansible.builtin.apt:
    update_cache: yes

- name: Upgrade mdatp package
  become: yes
  ansible.builtin.apt:
    name: mdatp
    only_upgrade: yes
    state: latest
```

Пример использования в playbook:

```yaml
- hosts: all
  gather_facts: yes
  roles:
    - nexus_repo_setup
  vars:
    nexus_repo_url: "http://your-actual-nexus-repo/ubuntu"
    gpg_key_url: "http://your-actual-nexus-repo/keys/public.gpg"
```

Особенности реализации:

1. Автоматическое определение версии Ubuntu через gather_facts
2. Использование отдельных файлов переменных для разных версий ОС
3. Современный способ добавления GPG ключа через /usr/share/keyrings/
4. Шаблон для создания файла репозитория в /etc/apt/sources.list.d/
5. Поддержка разных кодовых имен для разных версий Ubuntu

Примечания:

· Для работы роли необходим собранный facts (gather_facts: yes)
· Роль использует привилегированные операции (become: yes)
· Замените URL репозитория и GPG ключа на реальные значения
· При необходимости добавьте аутентификацию в URL репозитория



На основе ваших требований модифицирую Ansible роль для добавления аутентификации в локальном репозитории Nexus с использованием учетных данных, хранящихся в /etc/apt/auth.conf.d/.

Измененная структура роли:

```
roles/nexus_repo_auth/
├── tasks
│   └── main.yml
├── templates
│   ├── nexus-repo.j2
│   └── auth.conf.j2
├── vars
│   ├── main.yml
│   ├── Ubuntu_22.04.yml
│   └── Ubuntu_24.04.yml
└── defaults
    └── main.yml
```

Ключевые изменения:

1. Добавлена поддержка аутентификации через файл /etc/apt/auth.conf.d/nexus.conf
2. Использование переменных для учетных данных с возможностью шифрования через Ansible Vault
3. Проверка доступности репозитория перед настройкой

Содержание файлов:

1. defaults/main.yml:

```yaml
nexus_repo_url: "http://your-nexus-repo/ubuntu"
gpg_key_url: "http://your-nexus-repo/keys/public.gpg"
repo_filename: "nexus-repo.list"
auth_filename: "nexus.conf"
nexus_username: ""
nexus_password: ""
validate_repo: yes
```

2. tasks/main.yml:

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_distribution }}_{{ ansible_distribution_version }}.yml"

- name: Create authentication directory
  become: yes
  ansible.builtin.file:
    path: /etc/apt/auth.conf.d
    state: directory
    mode: '0755'

- name: Create authentication configuration
  become: yes
  ansible.builtin.template:
    src: auth.conf.j2
    dest: "/etc/apt/auth.conf.d/{{ auth_filename }}"
    mode: '0600'
  when: nexus_username | length > 0 and nexus_password | length > 0

- name: Download and install GPG key
  become: yes
  ansible.builtin.get_url:
    url: "{{ gpg_key_url }}"
    dest: /usr/share/keyrings/nexus-repo-keyring.gpg
    mode: '0644'
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: yes
  when: nexus_username | length > 0 and nexus_password | length > 0

- name: Create repository configuration file
  become: yes
  ansible.builtin.template:
    src: nexus-repo.j2
    dest: "/etc/apt/sources.list.d/{{ repo_filename }}"
    mode: '0644'

- name: Validate repository access
  become: yes
  ansible.builtin.apt:
    update_cache: yes
  ignore_errors: yes
  register: apt_update_result
  when: validate_repo | bool

- name: Fail if repository validation failed
  ansible.builtin.fail:
    msg: "Nexus repository validation failed. Please check credentials and network connectivity."
  when: apt_update_result is failed and validate_repo | bool

- name: Upgrade mdatp package
  become: yes
  ansible.builtin.apt:
    name: mdatp
    only_upgrade: yes
    state: latest
```

3. templates/auth.conf.j2:

```jinja2
machine {{ nexus_repo_url | replace('http://', '') | replace('https://', '') }}
login {{ nexus_username }}
password {{ nexus_password }}
```

4. templates/nexus-repo.j2:

```jinja2
deb [arch=amd64 signed-by=/usr/share/keyrings/nexus-repo-keyring.gpg] {{ nexus_repo_url }} {{ codename }} {{ components }}
```

Использование роли:

1. Создайте файл с учетными данными (например, group_vars/nexus.yml) с защитой через Ansible Vault:

```yaml
nexus_username: "your_username"
nexus_password: "your_password"
nexus_repo_url: "https://your-nexus-repo/ubuntu"
```

1. Запустите плейбук с использованием зашифрованных переменных:

```yaml
- hosts: all
  roles:
    - nexus_repo_auth
  vars_files:
    - group_vars/nexus.yml
```

Особенности реализации:

1. Безопасное хранение учетных данных:
   · Использование Ansible Vault для шифрования
   · Файл аутентификации создается с правами 0600
   · Учетные данные не сохраняются в истории apt
2. Проверка доступности репозитория:
   · Валидация репозитория после настройки
   · Возможность отключения проверки (validate_repo: no)
3. Совместимость с разными версиями Ubuntu:
   · Поддержка 22.04 (Jammy) и 24.04 (Noble)
   · Автоматическое определение версии ОС
4. Гибкая конфигурация:
   · Возможность использования без аутентификации
   · Настраиваемые имена файлов конфигурации

Для дополнительной безопасности рекомендуется:

1. Использовать HTTPS вместо HTTP
2. Регулярно обновлять GPG-ключи
3. Ограничить доступ к Nexus repository средствами сетевой безопасности

Данная реализация соответствует современным стандартам безопасности и обеспечивает надежную работу с аутентификацией в локальном репозитории Nexus.



Модифицирую роль для выбора соответствующего локального репозитория Nexus в зависимости от версии Ubuntu (22.04 или 24.04):

Обновленная структура роли:

```
roles/nexus_repo_auth/
├── tasks
│   └── main.yml
├── templates
│   ├── nexus-repo-22.04.j2
│   ├── nexus-repo-24.04.j2
│   └── auth.conf.j2
├── vars
│   ├── main.yml
│   ├── Ubuntu_22.04.yml
│   └── Ubuntu_24.04.yml
└── defaults
    └── main.yml
```

Ключевые изменения:

1. Раздельные шаблоны репозиториев для разных версий Ubuntu
2. Автоматический выбор шаблона на основе обнаруженной версии ОС
3. Поддержка разных путей репозиториев для разных версий

Содержание файлов:

1. defaults/main.yml:

```yaml
# Базовые URL репозиториев для разных версий Ubuntu
nexus_repo_base_url: "http://your-nexus-repo"
nexus_repo_path_22_04: "/repository/ubuntu-22.04"
nexus_repo_path_24_04: "/repository/ubuntu-24.04"

# Общие настройки
gpg_key_url: "{{ nexus_repo_base_url }}/repository/keys/public.gpg"
repo_filename: "nexus-repo.list"
auth_filename: "nexus.conf"
nexus_username: ""
nexus_password: ""
validate_repo: yes
```

2. vars/Ubuntu_22.04.yml:

```yaml
codename: jammy
components: main
nexus_repo_url: "{{ nexus_repo_base_url }}{{ nexus_repo_path_22_04 }}"
repo_template: "nexus-repo-22.04.j2"
```

3. vars/Ubuntu_24.04.yml:

```yaml
codename: noble
components: main
nexus_repo_url: "{{ nexus_repo_base_url }}{{ nexus_repo_path_24_04 }}"
repo_template: "nexus-repo-24.04.j2"
```

4. templates/nexus-repo-22.04.j2:

```jinja2
deb [arch=amd64 signed-by=/usr/share/keyrings/nexus-repo-keyring.gpg] {{ nexus_repo_url }} {{ codename }} {{ components }}
# Дополнительные настройки для Ubuntu 22.04
```

5. templates/nexus-repo-24.04.j2:

```jinja2
deb [arch=amd64 signed-by=/usr/share/keyrings/nexus-repo-keyring.gpg] {{ nexus_repo_url }} {{ codename }} {{ components }}
# Дополнительные настройки для Ubuntu 24.04
```

6. templates/auth.conf.j2:

```jinja2
machine {{ nexus_repo_url | replace('http://', '') | replace('https://', '') }}
login {{ nexus_username }}
password {{ nexus_password }}
```

7. tasks/main.yml:

```yaml
---
- name: Detect Ubuntu version and include appropriate variables
  block:
    - name: Include variables for Ubuntu 22.04
      include_vars: "Ubuntu_22.04.yml"
      when: ansible_distribution_version == "22.04"
      
    - name: Include variables for Ubuntu 24.04
      include_vars: "Ubuntu_24.04.yml"
      when: ansible_distribution_version == "24.04"
      
    - name: Fail for unsupported Ubuntu versions
      fail:
        msg: "Unsupported Ubuntu version {{ ansible_distribution_version }}. Only 22.04 and 24.04 are supported."
      when: ansible_distribution_version != "22.04" and ansible_distribution_version != "24.04"

- name: Create authentication directory
  become: yes
  file:
    path: /etc/apt/auth.conf.d
    state: directory
    mode: '0755'

- name: Create authentication configuration
  become: yes
  template:
    src: auth.conf.j2
    dest: "/etc/apt/auth.conf.d/{{ auth_filename }}"
    mode: '0600'
  when: nexus_username | length > 0 and nexus_password | length > 0

- name: Download and install GPG key
  become: yes
  get_url:
    url: "{{ gpg_key_url }}"
    dest: /usr/share/keyrings/nexus-repo-keyring.gpg
    mode: '0644'
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: yes
  when: nexus_username | length > 0 and nexus_password | length > 0

- name: Create repository configuration file
  become: yes
  template:
    src: "{{ repo_template }}"
    dest: "/etc/apt/sources.list.d/{{ repo_filename }}"
    mode: '0644'

- name: Validate repository access
  become: yes
  apt:
    update_cache: yes
  ignore_errors: yes
  register: apt_update_result
  when: validate_repo | bool

- name: Fail if repository validation failed
  fail:
    msg: "Nexus repository validation failed. Please check credentials and network connectivity."
  when: apt_update_result is failed and validate_repo | bool

- name: Upgrade mdatp package
  become: yes
  apt:
    name: mdatp
    only_upgrade: yes
    state: latest
```

Использование роли:

Playbook:

```yaml
- hosts: all
  gather_facts: yes
  roles:
    - nexus_repo_auth
  vars:
    nexus_repo_base_url: "http://your-actual-nexus-repo"
    nexus_repo_path_22_04: "/repository/ubuntu-22.04"
    nexus_repo_path_24_04: "/repository/ubuntu-24.04"
    nexus_username: "{{ vault_nexus_username }}"
    nexus_password: "{{ vault_nexus_password }}"
```

Файл с защищенными переменными (group_vars/nexus.yml):

```yaml
vault_nexus_username: "your_username"
vault_nexus_password: "your_password"
```

Особенности реализации:

1. Автоматическое определение версии Ubuntu через ansible_distribution_version
2. Раздельные пути репозиториев для разных версий ОС
3. Разные шаблоны конфигурации для максимальной гибкости
4. Единая точка конфигурации базового URL репозитория
5. Защита учетных данных через Ansible Vault

Дополнительные возможности:

Для большей гибкости можно добавить поддержку различных архитектур:

В templates/nexus-repo-*.j2:

```jinja2
deb [arch=amd64,i386 signed-by=/usr/share/keyrings/nexus-repo-keyring.gpg] {{ nexus_repo_url }} {{ codename }} {{ components }}
deb [arch=arm64,armhf signed-by=/usr/share/keyrings/nexus-repo-keyring.gpg] {{ nexus_repo_url }} {{ codename }} {{ components }}
```

Эта реализация обеспечивает гибкую настройку локальных репозиториев Nexus с учетом особенностей разных версий Ubuntu и обеспечивает безопасное хранение учетных данных.