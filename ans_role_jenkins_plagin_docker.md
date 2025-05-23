# Роль Ansible для установки Jenkins с плагинами и отдельными volumes в Docker Compose

Вот усовершенствованная роль Ansible, которая развертывает Jenkins в Docker с:
- Отдельными volumes для разных типов данных
- Предустановкой плагинов
- Настройкой через JCasC (Jenkins Configuration as Code)
- Оптимизированной структурой для production

## Структура роли

```
roles/jenkins-docker/
├── defaults
│   └── main.yml
├── files
│   ├── jenkins-compose.yml
│   ├── plugins.txt
│   └── executors.groovy
├── tasks
│   ├── main.yml
│   ├── prerequisites.yml
│   ├── volumes.yml
│   ├── deploy.yml
│   ├── configure.yml
│   └── plugins.yml
├── templates
│   ├── casc.yaml.j2
│   └── jenkins.env.j2
└── vars
    └── main.yml
```

## Основные файлы роли

### defaults/main.yml

```yaml
---
# Docker settings
jenkins_service_name: jenkins
jenkins_image: jenkins/jenkins
jenkins_image_version: "lts-jdk17"
jenkins_network_name: jenkins-net
docker_compose_version: "3.8"

# Ports configuration
jenkins_http_port: 8080
jenkins_agent_port: 50000

# Volumes configuration
jenkins_home_volume: jenkins_home
jenkins_plugins_volume: jenkins_plugins
jenkins_config_volume: jenkins_config
jenkins_scripts_volume: jenkins_scripts

# Jenkins configuration
jenkins_admin_user: admin
jenkins_timezone: "Europe/Moscow"
jenkins_url: "http://{{ ansible_host }}:{{ jenkins_http_port }}"
jenkins_executors: 2

# Plugins configuration
jenkins_plugins:
  - workflow-aggregator
  - git
  - blueocean
  - docker-plugin
  - configuration-as-code
  - pipeline-utility-steps
  - ssh-agent
  - mailer
  - job-dsl
  - ansible
```

### vars/main.yml

```yaml
---
# Security sensitive variables (override in vault)
jenkins_admin_password: "changeme"
jenkins_java_opts: >-
  -Djenkins.install.runSetupWizard=false
  -Duser.timezone={{ jenkins_timezone }}
  -Dhudson.model.LoadStatistics.clock=2000
```

### files/jenkins-compose.yml

```yaml
version: '{{ docker_compose_version }}'

services:
  {{ jenkins_service_name }}:
    image: {{ jenkins_image }}:{{ jenkins_image_version }}
    container_name: {{ jenkins_service_name }}
    restart: unless-stopped
    user: "1000:1000"
    ports:
      - "{{ jenkins_http_port }}:8080"
      - "{{ jenkins_agent_port }}:50000"
    volumes:
      - {{ jenkins_home_volume }}:/var/jenkins_home
      - {{ jenkins_plugins_volume }}:/usr/share/jenkins/ref/plugins
      - {{ jenkins_config_volume }}:/var/jenkins_config
      - {{ jenkins_scripts_volume }}:/var/jenkins_scripts
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JAVA_OPTS={{ jenkins_java_opts }}
      - CASC_JENKINS_CONFIG=/var/jenkins_config/casc.yaml
      - JENKINS_ADMIN_ID={{ jenkins_admin_user }}
      - JENKINS_ADMIN_PASSWORD={{ jenkins_admin_password }}
    networks:
      - {{ jenkins_network_name }}

networks:
  {{ jenkins_network_name }}:
    driver: bridge

volumes:
  {{ jenkins_home_volume }}:
  {{ jenkins_plugins_volume }}:
  {{ jenkins_config_volume }}:
  {{ jenkins_scripts_volume }}:
```

### files/plugins.txt

```
{{ jenkins_plugins | join('\n') }}
```

### files/executors.groovy

```groovy
import hudson.model.*
import jenkins.model.*

Jenkins.instance.setNumExecutors({{ jenkins_executors }})
Jenkins.instance.save()
```

### templates/casc.yaml.j2

```yaml
jenkins:
  systemMessage: "Jenkins configured by Ansible with Docker"
  numExecutors: {{ jenkins_executors }}
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: {{ jenkins_admin_user }}
          password: {{ jenkins_admin_password }}
  authorizationStrategy:
    globalMatrix:
      permissions:
        - "Overall/Administer:{{ jenkins_admin_user }}"
        - "Overall/Read:anonymous"

unclassified:
  location:
    url: {{ jenkins_url }}
    adminAddress: admin@{{ ansible_host }}

tool:
  git:
    installations:
      - name: default
        home: /usr/bin/git
```

### tasks/main.yml

```yaml
---
- name: Include prerequisites tasks
  include_tasks: prerequisites.yml

- name: Create and configure volumes
  include_tasks: volumes.yml

- name: Deploy Jenkins container
  include_tasks: deploy.yml

- name: Configure Jenkins
  include_tasks: configure.yml

- name: Install plugins
  include_tasks: plugins.yml
```

### tasks/prerequisites.yml

```yaml
---
- name: Install Docker and dependencies
  block:
    - name: Install Docker (Debian)
      apt:
        name:
          - docker.io
          - docker-compose-plugin
          - git
        state: present
        update_cache: yes
      when: ansible_os_family == 'Debian'

    - name: Install Docker (RHEL)
      yum:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-compose-plugin
          - git
        state: present
      when: ansible_os_family == 'RedHat'

    - name: Start and enable Docker
      service:
        name: docker
        state: started
        enabled: yes

    - name: Add user to docker group
      user:
        name: "{{ ansible_user }}"
        groups: docker
        append: yes
```

### tasks/volumes.yml

```yaml
---
- name: Create Jenkins volumes
  community.docker.docker_volume:
    name: "{{ item }}"
    driver: local
  loop:
    - "{{ jenkins_home_volume }}"
    - "{{ jenkins_plugins_volume }}"
    - "{{ jenkins_config_volume }}"
    - "{{ jenkins_scripts_volume }}"

- name: Set correct permissions on volumes
  command: >
    docker run --rm -v {{ item }}:/mnt alpine
    chown -R 1000:1000 /mnt
  loop:
    - "{{ jenkins_home_volume }}"
    - "{{ jenkins_plugins_volume }}"
    - "{{ jenkins_config_volume }}"
    - "{{ jenkins_scripts_volume }}"
```

### tasks/deploy.yml

```yaml
---
- name: Copy Docker Compose file
  template:
    src: jenkins-compose.yml
    dest: "/opt/jenkins/docker-compose.yml"
    mode: '0644'

- name: Copy plugins list
  copy:
    src: plugins.txt
    dest: "/opt/jenkins/plugins.txt"
    mode: '0644'

- name: Copy initial configuration scripts
  copy:
    src: executors.groovy
    dest: "/opt/jenkins/executors.groovy"
    mode: '0644'

- name: Start Jenkins container
  community.docker.docker_compose:
    project_src: /opt/jenkins
    state: present
  register: compose_up_result

- name: Wait for Jenkins to start
  uri:
    url: "http://localhost:{{ jenkins_http_port }}"
    status_code: 200
    timeout: 30
  register: jenkins_status
  retries: 15
  delay: 10
  until: jenkins_status.status == 200
```

### tasks/configure.yml

```yaml
---
- name: Prepare configuration directory
  file:
    path: "/var/lib/docker/volumes/{{ jenkins_config_volume }}/_data"
    state: directory
    mode: '0755'

- name: Generate JCasC configuration
  template:
    src: casc.yaml.j2
    dest: "/var/lib/docker/volumes/{{ jenkins_config_volume }}/_data/casc.yaml"
    mode: '0644'

- name: Copy init scripts to volume
  copy:
    src: executors.groovy
    dest: "/var/lib/docker/volumes/{{ jenkins_scripts_volume }}/_data/executors.groovy"
    mode: '0644'

- name: Apply initial configuration
  command: >
    docker exec {{ jenkins_service_name }} 
    bash -c "java -jar /var/jenkins_home/war/WEB-INF/jenkins-cli.jar -s http://localhost:8080 -auth {{ jenkins_admin_user }}:{{ jenkins_admin_password }}
    groovy = < /var/jenkins_scripts/executors.groovy"
  when: "'Creating' in compose_up_result.stdout"
```

### tasks/plugins.yml

```yaml
---
- name: Install Jenkins plugins
  command: >
    docker exec {{ jenkins_service_name }}
    jenkins-plugin-cli --plugin-file /opt/jenkins/plugins.txt
  when: "'Creating' in compose_up_result.stdout"
  register: plugins_install
  changed_when: plugins_install.rc == 0

- name: Restart Jenkins after plugins installation
  command: docker restart {{ jenkins_service_name }}
  when: "'Creating' in compose_up_result.stdout"
```

## Использование роли

1. Создайте playbook `deploy-jenkins.yml`:

```yaml
---
- hosts: jenkins_servers
  become: yes
  vars_files:
    - secrets.yml  # Зашифрованный ansible-vault файл
  roles:
    - jenkins-docker
```

2. Файл `secrets.yml` (зашифруйте с помощью ansible-vault):

```yaml
---
jenkins_admin_password: "secure_password_here"
```

3. Запустите playbook:

```bash
ansible-playbook -i inventory deploy-jenkins.yml --ask-vault-pass
```

## Преимущества этой реализации

1. **Раздельные volumes**:
   - `jenkins_home` - основные данные Jenkins
   - `jenkins_plugins` - установленные плагины
   - `jenkins_config` - конфигурационные файлы
   - `jenkins_scripts` - скрипты инициализации

2. **Безопасность**:
   - Все volumes монтируются с правильными правами (UID 1000)
   - Пароли хранятся в зашифрованном виде
   - Отключен setup wizard

3. **Гибкость**:
   - Легко добавлять новые плагины через переменную `jenkins_plugins`
   - Конфигурация через JCasC позволяет легко масштабировать
   - Отдельные volumes упрощают backup и миграцию

4. **Производительность**:
   - Оптимизированные Java опции
   - Отдельный volume для плагинов ускоряет пересоздание контейнера

Для production использования рекомендуется добавить:
- Настройку SSL/TLS
- Регулярные backup важных volumes
- Мониторинг ресурсов
- Ограничение доступа к Jenkins API