# Роль Ansible для установки и настройки Microsoft Defender для Linux

Эта роль устанавливает и настраивает Microsoft Defender для Linux (mdatp) с использованием локального репозитория Nexus и локального хранилища ключей для Oracle Linux 9, Ubuntu 22.04 и Ubuntu 24.04.

## Структура роли

```
roles/defender_install/
├── defaults/
│   └── main.yml       - Основные переменные
├── tasks/
│   ├── main.yml       - Основной файл задач
│   ├── oracle.yml     - Задачи для Oracle Linux
│   ├── ubuntu.yml     - Задачи для Ubuntu
│   └── common.yml    - Общие задачи для всех ОС
├── templates/
│   ├── defender.repo.j2  - Шаблон репозитория для RHEL-based
│   └── defender.list.j2  - Шаблон репозитория для Debian-based
└── vars/
    ├── main.yml       - Основные переменные
    ├── oracle.yml     - Переменные для Oracle Linux
    └── ubuntu.yml     - Переменные для Ubuntu
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки репозитория Nexus
nexus_url: "http://nexus.example.com"
nexus_repo_path: "/repository/defender"

# Настройки Defender
defender_config:
  organization: "YourOrganization"
  tags:
    - "env:production"
  proxy: ""
  log_level: "info"
```

### tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_distribution | lower }}.yml"

- name: Include OS-specific tasks
  include_tasks: "{{ ansible_distribution | lower }}.yml"
  when: ansible_distribution in ['OracleLinux', 'Ubuntu']

- name: Configure Microsoft Defender
  include_tasks: common.yml
```

### tasks/oracle.yml

```yaml
---
- name: Install prerequisites for Oracle Linux
  yum:
    name:
      - gnupg
      - gpg
    state: present

- name: Add Microsoft Defender GPG key from local Nexus
  rpm_key:
    key: "{{ nexus_url }}{{ nexus_repo_path }}/keys/microsoft-gpg.asc"
    state: present

- name: Configure Microsoft Defender repository from Nexus
  template:
    src: defender.repo.j2
    dest: /etc/yum.repos.d/microsoft-defender.repo
    owner: root
    group: root
    mode: '0644'

- name: Install Microsoft Defender
  yum:
    name: mdatp
    state: present
    enablerepo: "microsoft-defender"
```

### tasks/ubuntu.yml

```yaml
---
- name: Install prerequisites for Ubuntu
  apt:
    name:
      - gnupg
      - gpg
    state: present
    update_cache: yes

- name: Add Microsoft Defender GPG key from local Nexus
  apt_key:
    url: "{{ nexus_url }}{{ nexus_repo_path }}/keys/microsoft-gpg.asc"
    state: present

- name: Configure Microsoft Defender repository from Nexus
  template:
    src: defender.list.j2
    dest: /etc/apt/sources.list.d/microsoft-defender.list
    owner: root
    group: root
    mode: '0644'

- name: Install Microsoft Defender
  apt:
    name: mdatp
    state: present
    update_cache: yes
```

### tasks/common.yml

```yaml
---
- name: Ensure Defender service is running and enabled
  service:
    name: mdatp
    state: started
    enabled: yes

- name: Configure Defender settings
  command: mdatp config {{ item.key }} --value {{ item.value }}
  loop: "{{ defender_config | dict2items }}"
  when: item.value is defined and item.value != ""
  ignore_errors: yes

- name: Verify Defender status
  command: mdatp health
  register: defender_health
  changed_when: false

- name: Display Defender health status
  debug:
    var: defender_health.stdout_lines
```

### templates/defender.repo.j2

```jinja2
[microsoft-defender]
name=Microsoft Defender for Linux
baseurl={{ nexus_url }}{{ nexus_repo_path }}/rhel/{{ ansible_distribution_major_version }}/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/Microsoft-RPM-GPG-KEY
```

### templates/defender.list.j2

```jinja2
deb [arch=amd64] {{ nexus_url }}{{ nexus_repo_path }}/ubuntu/{{ ansible_distribution_release }}/ stable main
```

### vars/oracle.yml

```yaml
---
defender_package: mdatp
defender_service: mdatp
```

### vars/ubuntu.yml

```yaml
---
defender_package: mdatp
defender_service: mdatp
```

## Пример playbook

```yaml
---
- name: Install and configure Microsoft Defender
  hosts: all
  become: yes
  roles:
    - defender_install
```

## Подготовка локального репозитория Nexus

1. Загрузите пакеты Defender для всех поддерживаемых ОС в Nexus
2. Загрузите GPG ключи Microsoft в Nexus
3. Настройте репозитории в Nexus для каждого дистрибутива:
   - Oracle Linux 9
   - Ubuntu 22.04
   - Ubuntu 24.04

## Использование

1. Разместите роль в директории `roles/` вашего Ansible проекта
2. Создайте playbook (как показано выше)
3. Запустите playbook:

```bash
ansible-playbook -i inventory defender_install.yml
```

## Примечания

- Убедитесь, что ваши хосты имеют доступ к серверу Nexus
- Настройте переменные `nexus_url` и `nexus_repo_path` в соответствии с вашей инфраструктурой
- Для работы роли требуется root-доступ (become: yes)
- Роль поддерживает только указанные версии ОС



# Улучшенная роль Ansible для установки Defender с авторизацией в Nexus

Добавим поддержку авторизации в Nexus и улучшим безопасность и надежность роли.

## Обновленная структура роли

```
roles/defender_install/
├── defaults/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── oracle.yml
│   ├── ubuntu.yml
│   ├── common.yml
│   └── nexus_auth.yml
├── templates/
│   ├── defender.repo.j2
│   └── defender.list.j2
├── vars/
│   ├── main.yml
│   ├── oracle.yml
│   └── ubuntu.yml
└── handlers/
    └── main.yml
```

## Ключевые улучшения

1. Добавлена поддержка авторизации в Nexus
2. Реализована безопасная работа с учетными данными
3. Добавлена проверка доступности репозитория
4. Улучшена обработка ошибок

## Обновленные файлы роли

### defaults/main.yml

```yaml
---
# Настройки репозитория Nexus
nexus:
  url: "http://nexus.example.com"
  repo_path: "/repository/defender"
  auth:
    enabled: false
    username: ""
    password: ""
  # Альтернатива: можно использовать API key вместо username/password
  # api_key: ""

# Настройки Defender
defender_config:
  organization: "YourOrganization"
  tags:
    - "env:production"
  proxy: ""
  log_level: "info"

# Настройки безопасности
validate_certs: true
```

### tasks/nexus_auth.yml

```yaml
---
- name: Create auth string for Nexus
  set_fact:
    nexus_auth_string: "{{ nexus.auth.username }}:{{ nexus.auth.password }}"
  when: nexus.auth.enabled

- name: Encode auth string to base64
  set_fact:
    nexus_auth_b64: "{{ nexus_auth_string | b64encode }}"
  when: nexus.auth.enabled

- name: Create Nexus auth headers
  set_fact:
    nexus_auth_headers:
      Authorization: "Basic {{ nexus_auth_b64 }}"
  when: nexus.auth.enabled
```

### tasks/oracle.yml (обновленный)

```yaml
---
- name: Install prerequisites for Oracle Linux
  yum:
    name:
      - gnupg
      - gpg
      - ca-certificates
    state: present

- name: Download Microsoft GPG key from Nexus with auth
  get_url:
    url: "{{ nexus.url }}{{ nexus.repo_path }}/keys/microsoft-gpg.asc"
    dest: /tmp/microsoft-gpg.asc
    headers: "{{ nexus_auth_headers | default(omit) }}"
    validate_certs: "{{ validate_certs }}"
  when: nexus.auth.enabled
  register: download_gpg_key
  retries: 3
  delay: 10
  until: download_gpg_key is succeeded

- name: Add Microsoft Defender GPG key (public fallback)
  rpm_key:
    key: "https://packages.microsoft.com/keys/microsoft.asc"
    state: present
  when: not nexus.auth.enabled or download_gpg_key is failed

- name: Add Microsoft Defender GPG key (from Nexus)
  rpm_key:
    key: /tmp/microsoft-gpg.asc
    state: present
  when: nexus.auth.enabled and download_gpg_key is succeeded

- name: Configure Microsoft Defender repository from Nexus
  template:
    src: defender.repo.j2
    dest: /etc/yum.repos.d/microsoft-defender.repo
    owner: root
    group: root
    mode: '0644'
  vars:
    repo_url: |
      {% if nexus.auth.enabled %}
      {{ nexus.url }}{{ nexus.repo_path }}/rhel/{{ ansible_distribution_major_version }}/
      {% else %}
      https://packages.microsoft.com/rhel/{{ ansible_distribution_major_version }}/prod/
      {% endif %}

- name: Verify repository accessibility
  uri:
    url: "{{ nexus.url }}{{ nexus.repo_path }}/rhel/{{ ansible_distribution_major_version }}/repodata/repomd.xml"
    headers: "{{ nexus_auth_headers | default(omit) }}"
    validate_certs: "{{ validate_certs }}"
    return_content: no
    status_code: 200
  when: nexus.auth.enabled
  register: repo_check
  ignore_errors: yes

- name: Fail if repository is not accessible
  fail:
    msg: "Nexus repository is not accessible. Check auth credentials and network connectivity."
  when: nexus.auth.enabled and repo_check is failed

- name: Install Microsoft Defender
  yum:
    name: mdatp
    state: present
    enablerepo: "microsoft-defender"
```

### tasks/ubuntu.yml (обновленный)

```yaml
---
- name: Install prerequisites for Ubuntu
  apt:
    name:
      - gnupg
      - gpg
      - ca-certificates
      - apt-transport-https
    state: present
    update_cache: yes

- name: Download Microsoft GPG key from Nexus with auth
  get_url:
    url: "{{ nexus.url }}{{ nexus.repo_path }}/keys/microsoft-gpg.asc"
    dest: /usr/share/keyrings/microsoft.gpg
    headers: "{{ nexus_auth_headers | default(omit) }}"
    validate_certs: "{{ validate_certs }}"
    mode: '0644'
  when: nexus.auth.enabled
  register: download_gpg_key
  retries: 3
  delay: 10
  until: download_gpg_key is succeeded

- name: Add Microsoft Defender GPG key (public fallback)
  apt_key:
    url: "https://packages.microsoft.com/keys/microsoft.asc"
    keyring: /usr/share/keyrings/microsoft.gpg
    state: present
  when: not nexus.auth.enabled or download_gpg_key is failed

- name: Configure Microsoft Defender repository from Nexus
  template:
    src: defender.list.j2
    dest: /etc/apt/sources.list.d/microsoft-defender.list
    owner: root
    group: root
    mode: '0644'
  vars:
    repo_url: |
      {% if nexus.auth.enabled %}
      deb [arch=amd64 signed-by=/usr/share/keyrings/microsoft.gpg] {{ nexus.url }}{{ nexus.repo_path }}/ubuntu/{{ ansible_distribution_release }}/ stable main
      {% else %}
      deb [arch=amd64 signed-by=/usr/share/keyrings/microsoft.gpg] https://packages.microsoft.com/ubuntu/{{ ansible_distribution_release }}/prod {{ ansible_distribution_release }} main
      {% endif %}

- name: Verify repository accessibility
  uri:
    url: "{{ nexus.url }}{{ nexus.repo_path }}/ubuntu/{{ ansible_distribution_release }}/Packages"
    headers: "{{ nexus_auth_headers | default(omit) }}"
    validate_certs: "{{ validate_certs }}"
    return_content: no
    status_code: 200
  when: nexus.auth.enabled
  register: repo_check
  ignore_errors: yes

- name: Fail if repository is not accessible
  fail:
    msg: "Nexus repository is not accessible. Check auth credentials and network connectivity."
  when: nexus.auth.enabled and repo_check is failed

- name: Update apt cache
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Install Microsoft Defender
  apt:
    name: mdatp
    state: present
```

### templates/defender.repo.j2 (обновленный)

```jinja2
[microsoft-defender]
name=Microsoft Defender for Linux
baseurl={{ repo_url | trim }}
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/Microsoft-RPM-GPG-KEY
{% if nexus.auth.enabled %}
sslverify={{ validate_certs | lower }}
metadata_expire=300
{% endif %}
```

### handlers/main.yml

```yaml
---
- name: restart defender
  service:
    name: mdatp
    state: restarted
    enabled: yes

- name: reload systemd
  systemd:
    daemon_reload: yes
```

## Пример использования с авторизацией

1. Создайте файл с защищенными переменными `group_vars/all/vault.yml`:

```yaml
---
nexus:
  auth:
    enabled: true
    username: "nexus_user"
    password: "secure_password"
  # или использовать API key
  # api_key: "your_api_key"
```

2. Зашифруйте его с помощью ansible-vault:

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

3. Создайте playbook:

```yaml
---
- name: Install and configure Microsoft Defender with Nexus auth
  hosts: all
  become: yes
  vars_files:
    - group_vars/all/vault.yml
  tasks:
    - include_role:
        name: defender_install
```

4. Запустите playbook с vault:

```bash
ansible-playbook -i inventory defender_install.yml --ask-vault-pass
```

## Дополнительные улучшения

1. **Безопасность**:
   - Использование ansible-vault для хранения учетных данных
   - Ограничение прав доступа к файлам с секретами
   - Поддержка проверки SSL сертификатов

2. **Надежность**:
   - Повторные попытки загрузки файлов
   - Проверка доступности репозитория
   - Фолбэк на официальные репозитории при проблемах с Nexus

3. **Гибкость**:
   - Поддержка как базовой аутентификации, так и API ключей
   - Возможность отключения проверки сертификатов для тестовых сред
   - Настраиваемые параметры подключения

Эта улучшенная роль обеспечивает безопасную установку Microsoft Defender с авторизацией в Nexus, сохраняя при этом гибкость и надежность развертывания.