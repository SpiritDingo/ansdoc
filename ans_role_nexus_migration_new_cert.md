Добавлю задачи для подключения к Sonatype Nexus репозиторию. Вот обновленная структура роли с новыми задачами:

1. Сначала добавим новые переменные в `defaults/main.yml`:
```yaml
# Nexus repository settings
nexus_url: "https://nexus.example.com"
nexus_repo_name: "my-nexus-repo"
nexus_username: "deployment_user"
nexus_password: "password123"  # В реальном использовании следует использовать ansible-vault

# Для Debian/Ubuntu
nexus_deb_repo_distribution: "focal"  # Пример для Ubuntu 20.04
nexus_deb_repo_components: "main"
nexus_deb_repo_architectures: "amd64"

# Для RHEL/CentOS/OracleLinux
nexus_yum_gpgcheck: 1
nexus_yum_repo_gpgkey: "{{ nexus_url }}/repository/{{ nexus_repo_name }}/RPM-GPG-KEY-example"
```

2. Создадим новый файл задач `tasks/nexus_connection.yml`:
```yaml
---
- name: Configure Nexus repository connection for Debian/Ubuntu
  block:
    - name: Add Nexus APT repository (Ubuntu/Debian)
      apt_repository:
        repo: "deb {{ nexus_url }}/repository/{{ nexus_repo_name }} {{ nexus_deb_repo_distribution }} {{ nexus_deb_repo_components }}"
        filename: "nexus-{{ nexus_repo_name }}"
        state: present
        update_cache: no  # Кэш обновим позже
      when: ansible_distribution in ['Ubuntu', 'Debian']

    - name: Configure Nexus authentication for APT
      copy:
        dest: "/etc/apt/auth.conf.d/nexus.conf"
        content: |
          machine {{ nexus_url | regex_replace('^https?://', '') }}
          login {{ nexus_username }}
          password {{ nexus_password }}
        owner: root
        group: root
        mode: '0600'
      when: ansible_distribution in ['Ubuntu', 'Debian']
  
  when: ansible_distribution in ['Ubuntu', 'Debian']

- name: Configure Nexus repository connection for RHEL/CentOS/OracleLinux
  block:
    - name: Add Nexus YUM repository
      yum_repository:
        name: "nexus-{{ nexus_repo_name }}"
        description: "Nexus Repository {{ nexus_repo_name }}"
        baseurl: "{{ nexus_url }}/repository/{{ nexus_repo_name }}"
        gpgcheck: "{{ nexus_yum_gpgcheck }}"
        gpgkey: "{{ nexus_yum_repo_gpgkey }}"
        enabled: 1
        sslverify: 1
        state: present
      when: ansible_os_family == 'RedHat'

    - name: Configure Nexus credentials for YUM
      ini_file:
        path: "/etc/yum.repos.d/nexus-{{ nexus_repo_name }}.repo"
        section: "nexus-{{ nexus_repo_name }}"
        option: "username"
        value: "{{ nexus_username }}"
        mode: '0644'
      when: ansible_os_family == 'RedHat'

    - name: Configure Nexus password for YUM
      ini_file:
        path: "/etc/yum.repos.d/nexus-{{ nexus_repo_name }}.repo"
        section: "nexus-{{ nexus_repo_name }}"
        option: "password"
        value: "{{ nexus_password }}"
        mode: '0644'
      when: ansible_os_family == 'RedHat'
  
  when: ansible_os_family == 'RedHat'
```

3. Обновим `tasks/main.yml` чтобы включить новую задачу:
```yaml
---
- name: Include certificate tasks
  include_tasks: certificates.yml

- name: Include GPG keys tasks
  include_tasks: gpg_keys.yml

- name: Include repository files tasks
  include_tasks: repo_files.yml

- name: Configure Nexus repository connection
  include_tasks: nexus_connection.yml

- name: Include Ubuntu fix tasks
  include_tasks: ubuntu_fix.yml
  when: ansible_distribution == "Ubuntu"

- name: Update package manager cache
  include_tasks: update_cache.yml
```

4. Обновим `tasks/update_cache.yml` для обработки Nexus репозиториев:
```yaml
---
- name: Update APT cache on Ubuntu after Nexus config
  apt:
    update_cache: yes
  when: ansible_distribution == "Ubuntu"

- name: Clean YUM cache after Nexus config
  command: yum clean all
  when: ansible_os_family == 'RedHat'

- name: Rebuild YUM cache after Nexus config
  command: yum makecache
  when: ansible_os_family == 'RedHat'

- name: Verify Nexus repository accessibility (Debian/Ubuntu)
  command: apt-cache policy
  register: apt_cache_policy
  changed_when: false
  when: ansible_distribution in ['Ubuntu', 'Debian']

- name: Verify Nexus repository appears in policy (Debian/Ubuntu)
  debug:
    msg: "Nexus repository successfully configured"
  when: >
    ansible_distribution in ['Ubuntu', 'Debian'] and
    nexus_url in apt_cache_policy.stdout

- name: Verify Nexus repository accessibility (RHEL)
  command: yum repolist
  register: yum_repolist
  changed_when: false
  when: ansible_os_family == 'RedHat'

- name: Verify Nexus repository appears in repolist (RHEL)
  debug:
    msg: "Nexus repository successfully configured"
  when: >
    ansible_os_family == 'RedHat' and
    "nexus-{{ nexus_repo_name }}" in yum_repolist.stdout
```

Рекомендации по использованию:

1. Для безопасности рекомендуется хранить пароль в ansible-vault:
```bash
ansible-vault encrypt_string 'password123' --name 'nexus_password'
```

2. Затем в переменных (лучше в групповых или host_vars) использовать:
```yaml
nexus_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66386439653236336462653235653064336161623865633861366565643763313334333361623831
          6431623164396136323535313661346161346165666134330a663166353835313763626533373865
          32636530633662643861396366363263333963636561636337303938623937306635316462643962
          6531376663646366630a383963643736663936353763653662393366626235306665613162323439
          6562
```

3. Для разных окружений можно переопределить переменные:
```yaml
# group_vars/prod.yml
nexus_url: "https://nexus-prod.example.com"
nexus_repo_name: "prod-repo"

# group_vars/dev.yml
nexus_url: "https://nexus-dev.example.com"
nexus_repo_name: "dev-repo"
```

4. Для разных дистрибутивов можно настроить:
```yaml
# group_vars/debian.yml
nexus_deb_repo_distribution: "buster"  # Для Debian 10

# group_vars/ubuntu.yml
nexus_deb_repo_distribution: "focal"   # Для Ubuntu 20.04
```

Эта реализация предоставляет:
1. Поддержку как Debian/Ubuntu, так и RHEL/CentOS/OracleLinux
2. Настройку аутентификации для репозиториев
3. Проверку доступности репозиториев после настройки
4. Гибкость конфигурации через переменные
5. Интеграцию с существующей ролью по настройке сертификатов



Вот полная модифицированная Ansible роль для подключения хостов к Nexus репозиторию с учетом всех ваших требований:

```
roles/nexus_repo_connection/
├── defaults
│   └── main.yml
├── files
│   ├── certificates
│   │   ├── nexus.crt
│   │   └── CisEntCA.cer
│   ├── gpg_keys
│   │   ├── RPM-GPG-KEY-CentOS-5
│   │   ├── RPM-GPG-KEY-CentOS-6
│   │   ├── RPM-GPG-KEY-CentOS-7
│   │   ├── RPM-GPG-KEY-oracle
│   │   └── ubuntu-keyring-2018-archive.gpg
│   └── repo_files
│       ├── centos5
│       │   └── CentOS-Base.repo
│       ├── centos6
│       │   └── CentOS-Base.repo
│       ├── centos7
│       │   └── CentOS-Base.repo
│       ├── oracle_linux_9
│       │   └── oracle-linux-ol9.repo
│       ├── ubuntu22.04
│       │   └── sources.list
│       ├── ubuntu24.04
│       │   └── sources.list
│       └── nexus.conf
├── handlers
│   └── main.yml
├── tasks
│   ├── certificates.yml
│   ├── gpg_keys.yml
│   ├── nexus_connection.yml
│   ├── repo_files.yml
│   ├── ubuntu_fix.yml
│   ├── update_cache.yml
│   └── main.yml
└── templates
    ├── nexus_apt_repo.j2
    └── nexus_yum_repo.j2
```

### Содержимое файлов:

1. `defaults/main.yml`:
```yaml
---
# Default variables for nexus_repo_connection role

# Certificate paths
cert_src_path: "{{ role_path }}/files/certificates/nexus.crt"
cisentca_src_path: "{{ role_path }}/files/certificates/CisEntCA.cer"

cert_dest_ubuntu: /usr/local/share/ca-certificates/nexus.crt
cert_dest_rhel: /etc/pki/ca-trust/source/anchors/nexus.crt
cisentca_dest_ubuntu: /usr/local/share/ca-certificates/CisEntCA.cer
cisentca_dest_rhel: /etc/pki/ca-trust/source/anchors/CisEntCA.cer

# Nexus repository settings
nexus_url: "https://nexus.example.com"
nexus_repo_name: "my-nexus-repo"
nexus_username: "deployment_user"
nexus_password: "password123"  # Should be overridden with vault

# Debian/Ubuntu settings
nexus_deb_repo_distribution: "{{ ansible_distribution_release }}"
nexus_deb_repo_components: "main"
nexus_deb_repo_architectures: "amd64"

# RHEL/CentOS/OracleLinux settings
nexus_yum_gpgcheck: 1
nexus_yum_repo_gpgkey: "{{ nexus_url }}/repository/{{ nexus_repo_name }}/RPM-GPG-KEY-example"
nexus_yum_metadata_expire: "1440"  # 24 hours in minutes
```

2. `tasks/main.yml`:
```yaml
---
- name: Include certificate tasks
  include_tasks: certificates.yml

- name: Include GPG keys tasks
  include_tasks: gpg_keys.yml

- name: Include repository files tasks
  include_tasks: repo_files.yml

- name: Configure Nexus repository connection
  include_tasks: nexus_connection.yml

- name: Include Ubuntu fix tasks
  include_tasks: ubuntu_fix.yml
  when: ansible_distribution == "Ubuntu"

- name: Update package manager cache
  include_tasks: update_cache.yml
```

3. `tasks/certificates.yml` (без изменений из предыдущего ответа)

4. `tasks/gpg_keys.yml` (без изменений из предыдущего ответа)

5. `tasks/repo_files.yml` (без изменений из предыдущего ответа)

6. `tasks/ubuntu_fix.yml` (без изменений из предыдущего ответа)

7. `tasks/nexus_connection.yml`:
```yaml
---
- name: Ensure required directories exist (Debian/Ubuntu)
  file:
    path: "/etc/apt/auth.conf.d"
    state: directory
    mode: '0755'
  when: ansible_distribution in ['Ubuntu', 'Debian']

- name: Configure Nexus APT repository (Debian/Ubuntu)
  template:
    src: "nexus_apt_repo.j2"
    dest: "/etc/apt/sources.list.d/nexus-{{ nexus_repo_name }}.list"
    mode: '0644'
  when: ansible_distribution in ['Ubuntu', 'Debian']

- name: Configure Nexus authentication for APT
  copy:
    dest: "/etc/apt/auth.conf.d/nexus.conf"
    content: |
      machine {{ nexus_url | regex_replace('^https?://', '') }}
      login {{ nexus_username }}
      password {{ nexus_password }}
    owner: root
    group: root
    mode: '0600'
  when: ansible_distribution in ['Ubuntu', 'Debian']
  notify: Update APT cache

- name: Configure Nexus YUM repository (RHEL/CentOS/OracleLinux)
  template:
    src: "nexus_yum_repo.j2"
    dest: "/etc/yum.repos.d/nexus-{{ nexus_repo_name }}.repo"
    mode: '0644'
  when: ansible_os_family == 'RedHat'
  notify: Update YUM cache
```

8. `tasks/update_cache.yml`:
```yaml
---
- name: Update APT cache
  apt:
    update_cache: yes
  when: ansible_distribution in ['Ubuntu', 'Debian']

- name: Clean YUM cache
  command: yum clean all
  when: ansible_os_family == 'RedHat'

- name: Rebuild YUM cache
  command: yum makecache
  when: ansible_os_family == 'RedHat'

- name: Verify Nexus repository accessibility (Debian/Ubuntu)
  command: apt-cache policy
  register: apt_cache_policy
  changed_when: false
  when: ansible_distribution in ['Ubuntu', 'Debian']

- name: Verify Nexus repository appears in policy (Debian/Ubuntu)
  debug:
    msg: "Nexus repository successfully configured"
  when: >
    ansible_distribution in ['Ubuntu', 'Debian'] and
    nexus_url in apt_cache_policy.stdout

- name: Verify Nexus repository accessibility (RHEL)
  command: yum repolist
  register: yum_repolist
  changed_when: false
  when: ansible_os_family == 'RedHat'

- name: Verify Nexus repository appears in repolist (RHEL)
  debug:
    msg: "Nexus repository successfully configured"
  when: >
    ansible_os_family == 'RedHat' and
    "nexus-{{ nexus_repo_name }}" in yum_repolist.stdout
```

9. `handlers/main.yml`:
```yaml
---
- name: Update APT cache
  apt:
    update_cache: yes

- name: Update YUM cache
  block:
    - name: Clean YUM cache
      command: yum clean all
    
    - name: Rebuild YUM cache
      command: yum makecache
```

10. `templates/nexus_apt_repo.j2`:
```
# Managed by Ansible - DO NOT EDIT MANUALLY
deb [arch={{ nexus_deb_repo_architectures }}] {{ nexus_url }}/repository/{{ nexus_repo_name }} {{ nexus_deb_repo_distribution }} {{ nexus_deb_repo_components }}
```

11. `templates/nexus_yum_repo.j2`:
```
# Managed by Ansible - DO NOT EDIT MANUALLY
[nexus-{{ nexus_repo_name }}]
name=Nexus Repository - {{ nexus_repo_name }}
baseurl={{ nexus_url }}/repository/{{ nexus_repo_name }}
enabled=1
gpgcheck={{ nexus_yum_gpgcheck }}
gpgkey={{ nexus_yum_repo_gpgkey }}
metadata_expire={{ nexus_yum_metadata_expire }}
sslverify=1
username={{ nexus_username }}
password={{ nexus_password }}
```

### Пример использования роли:

```yaml
- name: Configure Nexus repository connection
  hosts: all
  become: yes
  gather_facts: yes

  vars:
    nexus_url: "https://nexus.internal.company.com"
    nexus_repo_name: "company-prod"
    nexus_username: "deployer"
    nexus_password: "{{ vault_nexus_password }}"  # From ansible-vault

  roles:
    - nexus_repo_connection
```

### Особенности реализации:

1. **Гибкая конфигурация** через переменные для разных дистрибутивов
2. **Безопасное хранение** учетных данных (рекомендуется использовать ansible-vault)
3. **Шаблонизация** конфигурационных файлов репозиториев
4. **Автоматическая проверка** доступности репозиториев после настройки
5. **Обработчики** для обновления кэша пакетных менеджеров
6. **Поддержка** Ubuntu/Debian и RHEL/CentOS/OracleLinux
7. **Интеграция** с существующей настройкой сертификатов и GPG-ключей

Для дополнительной безопасности рекомендуется:
1. Хранить пароли в ansible-vault
2. Использовать HTTPS для подключения к Nexus
3. Ограничивать доступ к репозиторию по IP/сети
4. Регулярно обновлять сертификаты и ключи