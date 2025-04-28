# Ansible роль для установки и настройки standalone Nexus Repository Manager

Роль для развертывания автономного экземпляра Sonatype Nexus Repository Manager 3.

## Структура роли

```
roles/nexus_standalone/
├── defaults
│   └── main.yml
├── files
│   └── nexus.service
├── handlers
│   └── main.yml
├── tasks
│   ├── install.yml
│   ├── configure.yml
│   └── main.yml
├── templates
│   ├── nexus.properties.j2
│   └── nexus.vmoptions.j2
└── vars
    └── main.yml
```

## Основные файлы роли

### defaults/main.yml

```yaml
---
# Версия Nexus
nexus_version: "3.58.1-02"
nexus_download_url: "https://download.sonatype.com/nexus/3/nexus-{{ nexus_version }}-unix.tar.gz"

# Параметры установки
nexus_install_dir: "/opt/nexus"
nexus_data_dir: "/nexus-data"
nexus_user: "nexus"
nexus_group: "nexus"
nexus_port: 8081

# Параметры JVM
nexus_jvm_min_heap: "2G"
nexus_jvm_max_heap: "2G"
nexus_jvm_extra_opts: "-XX:MaxDirectMemorySize=2G -Djava.util.prefs.userRoot={{ nexus_data_dir }}/javaprefs"

# Настройки администратора
nexus_admin_password: "admin123"  # Рекомендуется изменить через vault
nexus_anonymous_access: false
```

### tasks/main.yml

```yaml
---
- name: Install Nexus Repository Manager
  include_tasks: install.yml

- name: Configure Nexus Repository Manager
  include_tasks: configure.yml
```

### tasks/install.yml

```yaml
---
- name: Ensure Java is installed
  package:
    name: openjdk-11-jdk
    state: present

- name: Create nexus user and group
  user:
    name: "{{ nexus_user }}"
    group: "{{ nexus_group }}"
    system: yes
    create_home: no
    shell: /bin/bash

- name: Create directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ nexus_user }}"
    group: "{{ nexus_group }}"
    mode: '0755'
  loop:
    - "{{ nexus_install_dir }}"
    - "{{ nexus_data_dir }}"

- name: Download Nexus
  get_url:
    url: "{{ nexus_download_url }}"
    dest: "/tmp/nexus-{{ nexus_version }}-unix.tar.gz"
    checksum: "sha256:<актуальная_контрольная_сумма>"

- name: Extract Nexus
  unarchive:
    src: "/tmp/nexus-{{ nexus_version }}-unix.tar.gz"
    dest: "{{ nexus_install_dir }}"
    remote_src: yes
    extra_opts: "--strip-components=1"
    owner: "{{ nexus_user }}"
    group: "{{ nexus_group }}"

- name: Configure Nexus as systemd service
  template:
    src: "nexus.service.j2"
    dest: "/etc/systemd/system/nexus.service"
    owner: root
    group: root
    mode: '0644'

- name: Reload systemd and enable service
  systemd:
    daemon_reload: yes
    enabled: yes
    name: nexus
    state: started
```

### tasks/configure.yml

```yaml
---
- name: Configure JVM options
  template:
    src: "nexus.vmoptions.j2"
    dest: "{{ nexus_install_dir }}/bin/nexus.vmoptions"
    owner: "{{ nexus_user }}"
    group: "{{ nexus_group }}"
    mode: '0644'

- name: Configure Nexus properties
  template:
    src: "nexus.properties.j2"
    dest: "{{ nexus_data_dir }}/etc/nexus.properties"
    owner: "{{ nexus_user }}"
    group: "{{ nexus_group }}"
    mode: '0644'

- name: Wait for Nexus to start
  uri:
    url: "http://localhost:{{ nexus_port }}/service/rest/v1/status"
    method: GET
    status_code: 200
    timeout: 30
  register: result
  until: result.status == 200
  retries: 10
  delay: 30

- name: Change default admin password
  uri:
    url: "http://localhost:{{ nexus_port }}/service/rest/v1/security/users/admin/change-password"
    method: PUT
    user: "admin"
    password: "admin"
    body: "{{ nexus_admin_password }}"
    body_format: text
    force_basic_auth: yes
    status_code: 204

- name: Configure anonymous access
  uri:
    url: "http://localhost:{{ nexus_port }}/service/rest/v1/security/anonymous"
    method: PUT
    user: "admin"
    password: "{{ nexus_admin_password }}"
    body_format: json
    body:
      enabled: "{{ nexus_anonymous_access }}"
      userId: "anonymous"
      realmName: "NexusAuthorizingRealm"
    status_code: 204
```

### templates/nexus.vmoptions.j2

```
-Xms{{ nexus_jvm_min_heap }}
-Xmx{{ nexus_jvm_max_heap }}
{{ nexus_jvm_extra_opts }}
```

### templates/nexus.properties.j2

```
# Jetty configuration
application-port={{ nexus_port }}
application-host=0.0.0.0
nexus-args=${jetty.etc}/jetty.xml,${jetty.etc}/jetty-http.xml,${jetty.etc}/jetty-requestlog.xml
nexus-context-path=/

# Nexus configuration
nexus.scripts.allowCreation=true
```

### files/nexus.service.j2

```
[Unit]
Description=Nexus Service
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart={{ nexus_install_dir }}/bin/nexus start
ExecStop={{ nexus_install_dir }}/bin/nexus stop
User={{ nexus_user }}
Group={{ nexus_group }}
Restart=on-abort

[Install]
WantedBy=multi-user.target
```

## Использование роли

Пример playbook для использования этой роли:

```yaml
---
- name: Install and configure Nexus Repository Manager
  hosts: nexus_server
  become: yes
  vars:
    nexus_admin_password: "{{ vault_nexus_admin_password }}"  # Рекомендуется использовать Ansible Vault
    nexus_jvm_min_heap: "4G"
    nexus_jvm_max_heap: "4G"
  
  roles:
    - nexus_standalone
```

## Дополнительные рекомендации

1. **Безопасность**:
   - Всегда меняйте пароль администратора по умолчанию
   - Используйте HTTPS вместо HTTP
   - Ограничьте доступ к порту Nexus с помощью брандмауэра

2. **Резервное копирование**:
   - Регулярно создавайте резервные копии директории данных (`/nexus-data`)
   - Экспортируйте конфигурацию через REST API

3. **Мониторинг**:
   - Настройте мониторинг дискового пространства
   - Контролируйте использование памяти JVM

4. **Обновление**:
   - Регулярно проверяйте наличие новых версий Nexus
   - Тестируйте обновления в тестовой среде перед применением в production

5. **Оптимизация**:
   - Настройте задачи обслуживания (очистка кэша, compact blob store)
   - Оптимизируйте параметры JVM в зависимости от нагрузки

Эта роль предоставляет базовую установку standalone Nexus. Для production-среды рекомендуется дополнительно настроить:
- SSL/TLS
- Аутентификацию LDAP/AD
- Репликацию репозиториев (если требуется)
- Интеграцию с системами мониторинга