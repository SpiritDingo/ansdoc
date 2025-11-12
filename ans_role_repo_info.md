Вот обновленная роль для создания отчета в формате CSV:

Обновленная структура роли

```
repo_info/
├── tasks
│   └── main.yml
└── templates
    └── repo_report.csv.j2
```

Содержание файлов

1. tasks/main.yml:

```yaml
---
- name: "Collect repository files information"
  block:
    - name: "Find APT repository files (Ubuntu)"
      ansible.builtin.find:
        paths:
          - /etc/apt/sources.list
          - /etc/apt/sources.list.d
        patterns: "*.list"
        file_type: file
      register: apt_repo_files
      when: ansible_distribution == 'Ubuntu'

    - name: "Read APT repository files content (Ubuntu)"
      ansible.builtin.slurp:
        src: "{{ item.path }}"
      register: apt_repo_contents
      loop: "{{ apt_repo_files.files }}"
      when: ansible_distribution == 'Ubuntu'

    - name: "Find YUM/DNF repository files (Oracle Linux 9)"
      ansible.builtin.find:
        paths: /etc/yum.repos.d
        patterns: "*.repo"
        file_type: file
      register: yum_repo_files
      when: ansible_distribution == 'OracleLinux' and ansible_distribution_major_version == '9'

    - name: "Read YUM/DNF repository files content (Oracle Linux 9)"
      ansible.builtin.slurp:
        src: "{{ item.path }}"
      register: yum_repo_contents
      loop: "{{ yum_repo_files.files }}"
      when: ansible_distribution == 'OracleLinux' and ansible_distribution_major_version == '9'

  rescue:
    - name: "Handle errors during collection"
      ansible.builtin.debug:
        msg: "Error collecting repo info for {{ ansible_distribution }}"

- name: "Create local reports directory"
  ansible.builtin.file:
    path: "{{ playbook_dir }}/reports"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true

- name: "Generate CSV report"
  ansible.builtin.template:
    src: repo_report.csv.j2
    dest: "{{ playbook_dir }}/repos_report.csv"
  delegate_to: localhost
  run_once: true
```

1. templates/repo_report.csv.j2:

```jinja2
Hostname,OS,Distribution,Version,Architecture,Repo File Path,Repo File Name,Repo File Content
{% for host in groups['all'] %}
{% set hostvars = hostvars[host] %}
{% if hostvars.ansible_distribution == 'Ubuntu' and hostvars.apt_repo_files is defined and hostvars.apt_repo_files.files %}
  {% for file in hostvars.apt_repo_files.files %}
    {% set content_index = hostvars.apt_repo_contents.results | selectattr('item.path', 'equalto', file.path) | first %}
"{{ hostvars.ansible_hostname }}","{{ hostvars.ansible_distribution }}","{{ hostvars.ansible_os_family }}","{{ hostvars.ansible_distribution_version }}","{{ hostvars.ansible_architecture }}","{{ file.path }}","{{ file.path | basename }}","{{ content_index.content | b64decode | replace('"', '""') | replace('\n', '\\n') | trim }}"
  {% endfor %}
{% elif hostvars.ansible_distribution == 'OracleLinux' and hostvars.ansible_distribution_major_version == '9' and hostvars.yum_repo_files is defined and hostvars.yum_repo_files.files %}
  {% for file in hostvars.yum_repo_files.files %}
    {% set content_index = hostvars.yum_repo_contents.results | selectattr('item.path', 'equalto', file.path) | first %}
"{{ hostvars.ansible_hostname }}","{{ hostvars.ansible_distribution }}","{{ hostvars.ansible_os_family }}","{{ hostvars.ansible_distribution_version }}","{{ hostvars.ansible_architecture }}","{{ file.path }}","{{ file.path | basename }}","{{ content_index.content | b64decode | replace('"', '""') | replace('\n', '\\n') | trim }}"
  {% endfor %}
{% else %}
"{{ hostvars.ansible_hostname }}","{{ hostvars.ansible_distribution }}","{{ hostvars.ansible_os_family }}","{{ hostvars.ansible_distribution_version }}","{{ hostvars.ansible_architecture }}","No repo files found","",""
{% endif %}
{% endfor %}
```

Альтернативный упрощенный шаблон (если сложный не работает)

templates/repo_report_simple.csv.j2:

```jinja2
Hostname,OS,Distribution,Version,Architecture,Repo File Path,Repo File Name,Repo File Content
{% for host in groups['all'] %}
{% set hostvars = hostvars[host] %}
{% if hostvars.ansible_distribution == 'Ubuntu' and hostvars.apt_repo_files is defined %}
  {% for file in hostvars.apt_repo_files.files %}
"{{ hostvars.ansible_hostname }}","{{ hostvars.ansible_distribution }}","{{ hostvars.ansible_os_family }}","{{ hostvars.ansible_distribution_version }}","{{ hostvars.ansible_architecture }}","{{ file.path }}","{{ file.path | basename }}","APT repository file"
  {% endfor %}
  {% if hostvars.apt_repo_files.files | length == 0 %}
"{{ hostvars.ansible_hostname }}","{{ hostvars.ansible_distribution }}","{{ hostvars.ansible_os_family }}","{{ hostvars.ansible_distribution_version }}","{{ hostvars.ansible_architecture }}","No APT repo files","",""
  {% endif %}
{% elif hostvars.ansible_distribution == 'OracleLinux' and hostvars.ansible_distribution_major_version == '9' and hostvars.yum_repo_files is defined %}
  {% for file in hostvars.yum_repo_files.files %}
"{{ hostvars.ansible_hostname }}","{{ hostvars.ansible_distribution }}","{{ hostvars.ansible_os_family }}","{{ hostvars.ansible_distribution_version }}","{{ hostvars.ansible_architecture }}","{{ file.path }}","{{ file.path | basename }}","YUM/DNF repository file"
  {% endfor %}
  {% if hostvars.yum_repo_files.files | length == 0 %}
"{{ hostvars.ansible_hostname }}","{{ hostvars.ansible_distribution }}","{{ hostvars.ansible_os_family }}","{{ hostvars.ansible_distribution_version }}","{{ hostvars.ansible_architecture }}","No YUM repo files","",""
  {% endif %}
{% else %}
"{{ hostvars.ansible_hostname }}","{{ hostvars.ansible_distribution }}","{{ hostvars.ansible_os_family }}","{{ hostvars.ansible_distribution_version }}","{{ hostvars.ansible_architecture }}","Unsupported OS","",""
{% endif %}
{% endfor %}
```

Playbook для использования

site.yml:

```yaml
---
- name: "Collect repository information from all hosts"
  hosts: all
  gather_facts: true
  roles:
    - repo_info
```

Запуск

```bash
ansible-playbook -i inventory.ini site.yml
```

Результат

Создается файл reports/repos_report.csv с содержимым:

```csv
Hostname,OS,Distribution,Version,Architecture,Repo File Path,Repo File Name,Repo File Content
"ubuntu-server","Ubuntu","Debian","22.04","x86_64","/etc/apt/sources.list","sources.list","deb http://archive.ubuntu.com/ubuntu jammy main restricted\ndeb http://archive.ubuntu.com/ubuntu jammy-updates main restricted"
"ubuntu-server","Ubuntu","Debian","22.04","x86_64","/etc/apt/sources.list.d/docker.list","docker.list","deb [arch=amd64] https://download.docker.com/linux/ubuntu jammy stable"
"oracle9-server","OracleLinux","RedHat","9.2","x86_64","/etc/yum.repos.d/oracle-linux-ol9.repo","oracle-linux-ol9.repo","[ol9_baseos_latest]\nname=Oracle Linux 9 BaseOS Latest\nbaseurl=https://yum.oracle.com/repo/OracleLinux/OL9/baseos/latest/\ngpgcheck=1\nenabled=1"
"oracle9-server","OracleLinux","RedHat","9.2","x86_64","/etc/yum.repos.d/epel.repo","epel.repo","[epel]\nname=Extra Packages for Enterprise Linux 9\nbaseurl=https://download.fedoraproject.org/pub/epel/9/Everything/x86_64/\nenabled=1\ngpgcheck=1"
```

Ключевые особенности:

· ✅ CSV формат с разделителями-запятыми
· ✅ Один файл для всех хостов
· ✅ Полные пути к файлам репозиториев
· ✅ Имена файлов в отдельной колонке
· ✅ Содержимое файлов с экранированием кавычек
· ✅ Поддержка Ubuntu и Oracle Linux 9
· ✅ Автоматическое создание директории reports
· ✅ Обработка специальных символов в CSV

Файл отчета будет создан в ./reports/repos_report.csv и может быть открыт в Excel, LibreOffice Calc или любом другом редакторе таблиц.


Ошибка возникает из-за проблем с доступностью переменных в hostvars. Вот исправленная версия роли:

Исправленная структура роли

```
repo_info/
├── tasks
│   └── main.yml
└── templates
    └── repo_report.csv.j2
```

Содержание файлов

1. tasks/main.yml:

```yaml
---
- name: "Collect repository files information"
  block:
    - name: "Find APT repository files (Ubuntu)"
      ansible.builtin.find:
        paths:
          - /etc/apt/sources.list
          - /etc/apt/sources.list.d
        patterns: "*.list"
        file_type: file
      register: apt_repo_files
      when: ansible_distribution == 'Ubuntu'

    - name: "Read APT repository files content (Ubuntu)"
      ansible.builtin.slurp:
        src: "{{ item.path }}"
      register: apt_repo_contents
      loop: "{{ apt_repo_files.files }}"
      when: ansible_distribution == 'Ubuntu'

    - name: "Find YUM/DNF repository files (Oracle Linux 9)"
      ansible.builtin.find:
        paths: /etc/yum.repos.d
        patterns: "*.repo"
        file_type: file
      register: yum_repo_files
      when: ansible_distribution == 'OracleLinux' and ansible_distribution_major_version == '9'

    - name: "Read YUM/DNF repository files content (Oracle Linux 9)"
      ansible.builtin.slurp:
        src: "{{ item.path }}"
      register: yum_repo_contents
      loop: "{{ yum_repo_files.files }}"
      when: ansible_distribution == 'OracleLinux' and ansible_distribution_major_version == '9'

  rescue:
    - name: "Handle errors during collection"
      ansible.builtin.debug:
        msg: "Error collecting repo info for {{ ansible_distribution }}"

- name: "Set facts for report generation"
  ansible.builtin.set_fact:
    host_repo_data: |
      {
        "hostname": "{{ ansible_hostname }}",
        "distribution": "{{ ansible_distribution }}",
        "os_family": "{{ ansible_os_family }}",
        "version": "{{ ansible_distribution_version }}",
        "architecture": "{{ ansible_architecture }}",
        "apt_repo_files": "{{ apt_repo_files.files | default([]) }}",
        "apt_repo_contents": "{{ apt_repo_contents.results | default([]) }}",
        "yum_repo_files": "{{ yum_repo_files.files | default([]) }}",
        "yum_repo_contents": "{{ yum_repo_contents.results | default([]) }}"
      }

- name: "Create local reports directory"
  ansible.builtin.file:
    path: "{{ playbook_dir }}/reports"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true

- name: "Generate CSV report"
  ansible.builtin.template:
    src: repo_report.csv.j2
    dest: "{{ playbook_dir }}/reports/repos_report.csv"
  delegate_to: localhost
  run_once: true
```

1. templates/repo_report.csv.j2:

```jinja2
Hostname,OS,OS Family,Version,Architecture,Repo File Path,Repo File Name,Repo File Content
{% for host in groups['all'] %}
{% if hostvars[host].host_repo_data is defined %}
{% set host_data = hostvars[host].host_repo_data %}
{% if host_data.distribution == 'Ubuntu' and host_data.apt_repo_files %}
  {% for file in host_data.apt_repo_files %}
    {% set file_content = '' %}
    {% for content_item in host_data.apt_repo_contents %}
      {% if content_item.item.path == file.path %}
        {% set file_content = content_item.content | b64decode | replace('"', '""') | replace('\n', '\\n') | trim %}
      {% endif %}
    {% endfor %}
"{{ host_data.hostname }}","{{ host_data.distribution }}","{{ host_data.os_family }}","{{ host_data.version }}","{{ host_data.architecture }}","{{ file.path }}","{{ file.path | basename }}","{{ file_content }}"
  {% endfor %}
{% elif host_data.distribution == 'OracleLinux' and host_data.yum_repo_files %}
  {% for file in host_data.yum_repo_files %}
    {% set file_content = '' %}
    {% for content_item in host_data.yum_repo_contents %}
      {% if content_item.item.path == file.path %}
        {% set file_content = content_item.content | b64decode | replace('"', '""') | replace('\n', '\\n') | trim %}
      {% endif %}
    {% endfor %}
"{{ host_data.hostname }}","{{ host_data.distribution }}","{{ host_data.os_family }}","{{ host_data.version }}","{{ host_data.architecture }}","{{ file.path }}","{{ file.path | basename }}","{{ file_content }}"
  {% endfor %}
{% else %}
"{{ host_data.hostname }}","{{ host_data.distribution }}","{{ host_data.os_family }}","{{ host_data.version }}","{{ host_data.architecture }}","No repo files found","",""
{% endif %}
{% endif %}
{% endfor %}
```

Альтернативная упрощенная версия (если сложная не работает)

templates/repo_report_simple.csv.j2:

```jinja2
Hostname,OS,OS Family,Version,Architecture,Repo File Path,Repo File Name,Repo File Content
{% for host in groups['all'] %}
{% if hostvars[host].host_repo_data is defined %}
{% set host_data = hostvars[host].host_repo_data %}
{% if host_data.distribution == 'Ubuntu' and host_data.apt_repo_files %}
  {% for file in host_data.apt_repo_files %}
"{{ host_data.hostname }}","{{ host_data.distribution }}","{{ host_data.os_family }}","{{ host_data.version }}","{{ host_data.architecture }}","{{ file.path }}","{{ file.path | basename }}","APT repository file"
  {% endfor %}
{% elif host_data.distribution == 'OracleLinux' and host_data.yum_repo_files %}
  {% for file in host_data.yum_repo_files %}
"{{ host_data.hostname }}","{{ host_data.distribution }}","{{ host_data.os_family }}","{{ host_data.version }}","{{ host_data.architecture }}","{{ file.path }}","{{ file.path | basename }}","YUM/DNF repository file"
  {% endfor %}
{% else %}
"{{ host_data.hostname }}","{{ host_data.distribution }}","{{ host_data.os_family }}","{{ host_data.version }}","{{ host_data.architecture }}","No repo files found","",""
{% endif %}
{% endif %}
{% endfor %}
```

Обновленный playbook

site.yml:

```yaml
---
- name: "Collect repository information from all hosts"
  hosts: all
  gather_facts: true
  tasks:
    - name: "Include repo_info role"
      ansible.builtin.include_role:
        name: repo_info
```

Запуск

```bash
ansible-playbook -i inventory.ini site.yml
```

Ключевые изменения:

1. Добавлен set_fact - собирает все данные в одну структуру
2. Исправлен доступ к переменным - через hostvars[host].host_repo_data
3. Упрощена логика шаблона - меньше сложных фильтров
4. Обработка ошибок - проверки на существование переменных

Если все равно возникают проблемы, попробуйте эту минимальную версию:

tasks/main.yml (минимальная версия):

```yaml
---
- name: "Collect APT repo files (Ubuntu)"
  ansible.builtin.find:
    paths:
      - /etc/apt/sources.list
      - /etc/apt/sources.list.d
    patterns: "*.list"
    file_type: file
  register: apt_repo_files
  when: ansible_distribution == 'Ubuntu'

- name: "Collect YUM repo files (Oracle Linux)"
  ansible.builtin.find:
    paths: /etc/yum.repos.d
    patterns: "*.repo"
    file_type: file
  register: yum_repo_files
  when: ansible_distribution == 'OracleLinux'

- name: "Create local reports directory"
  ansible.builtin.file:
    path: "{{ playbook_dir }}/reports"
    state: directory
    mode: '0755'
  delegate_to: localhost
  run_once: true

- name: "Generate simple CSV report"
  ansible.builtin.template:
    src: repo_report_simple.csv.j2
    dest: "{{ playbook_dir }}/reports/repos_report.csv"
  delegate_to: localhost
  run_once: true
```

Это должно решить проблему с неопределенными переменными в hostvars.