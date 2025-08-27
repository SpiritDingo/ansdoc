На основе предоставленных результатов поиска, я создам роль Ansible для сбора информации о настройках репозиториев Ubuntu в файлах /etc/apt/sources.list и /etc/apt/sources.list.d/. Роль будет включать в себя задачи для сбора фактов, извлечения данных о репозиториях и структурирования вывода.

Роль Ansible: gather_apt_repos_info

Структура роли

```
roles/
    gather_apt_repos_info/
        tasks/
            main.yml
        templates/
            repos_info.j2
```

Содержание файла tasks/main.yml

```yaml
---
- name: Gather Ubuntu distribution facts
  ansible.builtin.setup:
    gather_subset:
      - distribution
  when: ansible_facts['os_family'] == "Debian"

- name: Check existence of /etc/apt/sources.list
  ansible.builtin.stat:
    path: /etc/apt/sources.list
  register: sources_list_stat

- name: Read content of /etc/apt/sources.list
  ansible.builtin.slurp:
    src: /etc/apt/sources.list
  register: sources_list_content
  when: sources_list_stat.stat.exists

- name: Check existence of /etc/apt/sources.list.d/ directory
  ansible.builtin.stat:
    path: /etc/apt/sources.list.d
  register: sources_list_d_stat

- name: Find all .list and .sources files in /etc/apt/sources.list.d/
  ansible.builtin.find:
    paths: /etc/apt/sources.list.d
    patterns: "*.list,*.sources"
    recurse: no
  register: sources_list_d_files
  when: sources_list_d_stat.stat.isdir

- name: Read content of all .list files in /etc/apt/sources.list.d/
  ansible.builtin.slurp:
    src: "{{ item.path }}"
  register: sources_list_d_content
  loop: "{{ sources_list_d_files.files | selectattr('path', 'match', '.*\\.list$') | list }}"
  loop_control:
    label: "{{ item.path }}"

- name: Read content of all .sources files in /etc/apt/sources.list.d/
  ansible.builtin.slurp:
    src: "{{ item.path }}"
  register: sources_sources_d_content
  loop: "{{ sources_list_d_files.files | selectattr('path', 'match', '.*\\.sources$') | list }}"
  loop_control:
    label: "{{ item.path }}"

- name: Decode and process sources.list content
  ansible.builtin.set_fact:
    sources_list_decoded: "{{ sources_list_content.content | b64decode | split('\n') | select('match', '^\\s*deb') | list }}"
  when: sources_list_stat.stat.exists and sources_list_content.content | length > 0

- name: Decode and process .list files content
  ansible.builtin.set_fact:
    sources_list_d_decoded: |
      {%- set output = [] -%}
      {%- for file in sources_list_d_content.results -%}
        {%- if file.content | length > 0 -%}
          {%- set content = file.content | b64decode | split('\n') | select('match', '^\\s*deb') | list -%}
          {%- if content | length > 0 -%}
            {%- set _ = output.append({'file': file.item, 'content': content}) -%}
          {%- endif -%}
        {%- endif -%}
      {%- endfor -%}
      {{ output }}

- name: Decode and process .sources files content
  ansible.builtin.set_fact:
    sources_sources_d_decoded: |
      {%- set output = [] -%}
      {%- for file in sources_sources_d_content.results -%}
        {%- if file.content | length > 0 -%}
          {%- set content = file.content | b64decode -%}
          {%- if content | length > 0 -%}
            {%- set _ = output.append({'file': file.item, 'content': content}) -%}
          {%- endif -%}
        {%- endif -%}
      {%- endfor -%}
      {{ output }}

- name: Generate repositories report
  ansible.builtin.template:
    src: repos_info.j2
    dest: "/tmp/apt_repositories_report_{{ ansible_facts['hostname'] }}.txt"
  vars:
    sources_list_exists: "{{ sources_list_stat.stat.exists }}"
    sources_list_content: "{{ sources_list_decoded | default([]) }}"
    sources_list_d_files: "{{ sources_list_d_files.files | map(attribute='path') | list }}"
    sources_list_d_content: "{{ sources_list_d_decoded | default([]) }}"
    sources_sources_d_content: "{{ sources_sources_d_decoded | default([]) }}"
```

Содержание файла templates/repos_info.j2

```jinja2
APT Repositories Report for {{ ansible_facts['hostname'] }}
Generated on {{ ansible_date_time.iso8601 }}

Distribution: {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}

=== /etc/apt/sources.list ===
{% if sources_list_exists %}
{% if sources_list_content | length > 0 %}
{% for line in sources_list_content %}
{{ line }}
{% endfor %}
{% else %}
No active repository entries found.
{% endif %}
{% else %}
File does not exist.
{% endif %}

=== /etc/apt/sources.list.d/ ===
{% if sources_list_d_files | length > 0 %}
Found {{ sources_list_d_files | length }} files in sources.list.d directory:
{% for file in sources_list_d_files %}
- {{ file }}
{% endfor %}

{% for item in sources_list_d_content %}
--- Content of {{ item.file }} ---
{% for line in item.content %}
{{ line }}
{% endfor %}
{% endfor %}

{% for item in sources_sources_d_content %}
--- Content of {{ item.file }} (deb822 format) ---
{{ item.content }}
{% endfor %}
{% else %}
No files found in /etc/apt/sources.list.d/
{% endif %}
```

Использование роли в playbook

Создайте playbook gather_repos.yml:

```yaml
- hosts: all
  roles:
    - gather_apt_repos_info
```

Запустите playbook:

```bash
ansible-playbook -i inventory gather_repos.yml
```

Особенности роли

1. Сбор фактов: Использует модуль setup для определения дистрибутива и его версии .
2. Проверка файлов: Проверяет существование /etc/apt/sources.list и директории /etc/apt/sources.list.d/.
3. Чтение файлов: Читает содержимое файлов с помощью модуля slurp.
4. Обработка содержимого:
   · Для .list файлов: извлекает строки, начинающиеся с deb (активные репозитории).
   · Для .sources файлов: выводит всё содержимое, так как они используют структурированный формат deb822 .
5. Формирование отчета: Генерирует отчет в текстовом файле с использованием шаблона Jinja2.

Примечания

· Роль предназначена для систем на базе Debian/Ubuntu.
· Для обработки файлов в формате deb822 (как в Ubuntu 24.04) может потребоваться дополнительная парсинг-логика, которую можно добавить в шаблон или в задачи.
· Роль не изменяет конфигурацию репозиториев, а только собирает информацию.

Этот подход позволяет эффективно собирать информацию о репозиториях across multiple Ubuntu systems и может быть расширен для анализа или валидации конфигураций.


Чтобы создать Ansible роль для сбора информации о настройках репозиториев в Ubuntu 22.04 и 24.04 (включая файлы /etc/apt/sources.list и /etc/apt/sources.list.d/), вы можете использовать следующие шаги, основываясь на результатах поиска.

📁 Структура роли

Создайте структуру каталогов для роли следующим образом:

```bash
roles/
    gather_apt_repos/
        tasks/
            main.yml
        defaults/
            main.yml
        meta/
            main.yml
```

📝 Содержание файлов роли

1. tasks/main.yml

Этот файл содержит задачи для сбора информации о репозиториях.

```yaml
---
- name: Gather information about APT repositories
  block:
    - name: Check existence of /etc/apt/sources.list
      ansible.builtin.stat:
        path: /etc/apt/sources.list
      register: sources_list_stat

    - name: Read contents of /etc/apt/sources.list if it exists
      ansible.builtin.slurp:
        src: /etc/apt/sources.list
      register: sources_list_content
      when: sources_list_stat.stat.exists

    - name: Find all .list files in /etc/apt/sources.list.d/
      ansible.builtin.find:
        paths: /etc/apt/sources.list.d
        patterns: '*.list'
      register: sources_list_d_files

    - name: Read contents of all .list files in /etc/apt/sources.list.d/
      ansible.builtin.slurp:
        src: "{{ item.path }}"
      register: sources_list_d_content
      loop: "{{ sources_list_d_files.files }}"

    - name: Find all .sources files in /etc/apt/sources.list.d/
      ansible.builtin.find:
        paths: /etc/apt/sources.list.d
        patterns: '*.sources'
      register: sources_list_d_sources_files

    - name: Read contents of all .sources files in /etc/apt/sources.list.d/
      ansible.builtin.slurp:
        src: "{{ item.path }}"
      register: sources_list_d_sources_content
      loop: "{{ sources_list_d_sources_files.files }}"

  rescue:
    - name: Handle errors during repository information gathering
      ansible.builtin.debug:
        msg: "Error gathering repository information: {{ ansible_failed_result.msg }}"

- name: Set facts for gathered repository information
  ansible.builtin.set_fact:
    apt_repositories_info: {
      "sources_list": "{{ sources_list_content.content | b64decode if sources_list_stat.stat.exists else '' }}",
      "sources_list_d_list_files": [
        {% for item in sources_list_d_content.results %}
        {
          "path": "{{ item.item }}",
          "content": "{{ item.content | b64decode }}"
        }{% if not loop.last %},{% endif %}
        {% endfor %}
      ],
      "sources_list_d_sources_files": [
        {% for item in sources_list_d_sources_content.results %}
        {
          "path": "{{ item.item }}",
          "content": "{{ item.content | b64decode }}"
        }{% if not loop.last %},{% endif %}
        {% endfor %}
      ]
    }

- name: Display gathered repository information
  ansible.builtin.debug:
    var: apt_repositories_info
```

2. defaults/main.yml

Здесь вы можете определить переменные по умолчанию для роли, хотя в данном случае они могут не понадобиться. Файл можно оставить пустым или определить:

```yaml
---
# defaults file for gather_apt_repos
```

3. meta/main.yml

Этот файл содержит метаданные роли:

```yaml
---
galaxy_info:
  author: Your Name
  description: Role to gather APT repository information from Ubuntu systems
  company: Your Company
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions:
        - jammy
        - noble
  galaxy_tags:
    - system
    - repositories
    - apt
```

🛠 Использование роли

1. Создайте плейбук для использования роли, например gather_repos.yml:

```yaml
---
- hosts: all
  roles:
    - gather_apt_repos
```

1. Запустите плейбук:

```bash
ansible-playbook -i inventory gather_repos.yml
```

💡 Примечания

· Чтение файлов: Для чтения файлов используется модуль slurp, который возвращает содержимое в кодировке base64. Поэтому используется фильтр b64decode для декодирования .
· Обработка ошибок: Блок rescue обрабатывает возможные ошибки при сборе информации.
· Сбор информации: Роля собирает информацию из:
  · /etc/apt/sources.list
  · Всех файлов с расширением .list в /etc/apt/sources.list.d/
  · Всех файлов с расширением .sources в /etc/apt/sources.list.d/ (актуально для Ubuntu 24.04 Noble) .
· Вывод информации: Собранная информация выводится с помощью debug и сохраняется в факте apt_repositories_info для дальнейшего использования.

Эта роль обеспечивает сбор информации о репозиториях в формате, пригодном для последующего анализа или документирования конфигурации системы.


Дополню роль для сохранения собранной информации в отчеты различных форматов.

📁 Обновленная структура роли

```
roles/
    gather_apt_repos/
        tasks/
            main.yml
        defaults/
            main.yml
        meta/
            main.yml
        templates/
            report.j2
```

📝 Обновленные файлы роли

1. defaults/main.yml - добавление переменных для отчетов

```yaml
---
# defaults file for gather_apt_repos
gather_apt_repos_report_dir: "/tmp/apt_repos_reports"
gather_apt_repos_report_format: "txt"  # txt, json, yaml, html
gather_apt_repos_create_timestamp: true
gather_apt_repos_include_hostname: true
```

2. tasks/main.yml - добавление задач для создания отчетов

```yaml
---
- name: Create report directory
  ansible.builtin.file:
    path: "{{ gather_apt_repos_report_dir }}"
    state: directory
    mode: '0755'
  when: gather_apt_repos_report_dir != ""

- name: Gather information about APT repositories
  block:
    - name: Check existence of /etc/apt/sources.list
      ansible.builtin.stat:
        path: /etc/apt/sources.list
      register: sources_list_stat

    - name: Read contents of /etc/apt/sources.list if it exists
      ansible.builtin.slurp:
        src: /etc/apt/sources.list
      register: sources_list_content
      when: sources_list_stat.stat.exists

    - name: Find all .list files in /etc/apt/sources.list.d/
      ansible.builtin.find:
        paths: /etc/apt/sources.list.d
        patterns: '*.list'
      register: sources_list_d_files

    - name: Read contents of all .list files in /etc/apt/sources.list.d/
      ansible.builtin.slurp:
        src: "{{ item.path }}"
      register: sources_list_d_content
      loop: "{{ sources_list_d_files.files }}"

    - name: Find all .sources files in /etc/apt/sources.list.d/
      ansible.builtin.find:
        paths: /etc/apt/sources.list.d
        patterns: '*.sources'
      register: sources_list_d_sources_files

    - name: Read contents of all .sources files in /etc/apt/sources.list.d/
      ansible.builtin.slurp:
        src: "{{ item.path }}"
      register: sources_list_d_sources_content
      loop: "{{ sources_list_d_sources_files.files }}"

  rescue:
    - name: Handle errors during repository information gathering
      ansible.builtin.debug:
        msg: "Error gathering repository information: {{ ansible_failed_result.msg }}"

- name: Set facts for gathered repository information
  ansible.builtin.set_fact:
    apt_repositories_info: {
      "hostname": "{{ ansible_hostname }}",
      "timestamp": "{{ ansible_date_time.iso8601 }}",
      "sources_list_exists": "{{ sources_list_stat.stat.exists }}",
      "sources_list_content": "{{ sources_list_content.content | b64decode if sources_list_stat.stat.exists else 'FILE_NOT_EXISTS' }}",
      "sources_list_d_list_files": [
        {% for item in sources_list_d_content.results %}
        {
          "path": "{{ item.item.path }}",
          "content": "{{ item.content | b64decode }}"
        }{% if not loop.last %},{% endif %}
        {% endfor %}
      ],
      "sources_list_d_sources_files": [
        {% for item in sources_list_d_sources_content.results %}
        {
          "path": "{{ item.item.path }}",
          "content": "{{ item.content | b64decode }}"
        }{% if not loop.last %},{% endif %}
        {% endfor %}
      ]
    }

- name: Create timestamp for filename
  ansible.builtin.set_fact:
    report_timestamp: "{{ ansible_date_time.iso8601 | regex_replace('[-:]', '') | regex_replace('\\.\\d+', '') }}"
  when: gather_apt_repos_create_timestamp

- name: Set report filename
  ansible.builtin.set_fact:
    report_filename: "apt_repos_report{% if gather_apt_repos_include_hostname %}_{{ ansible_hostname }}{% endif %}{% if gather_apt_repos_create_timestamp %}_{{ report_timestamp }}{% endif %}.{{ gather_apt_repos_report_format }}"
  when: gather_apt_repos_report_dir != ""

- name: Generate TXT report
  ansible.builtin.template:
    src: report.j2
    dest: "{{ gather_apt_repos_report_dir }}/{{ report_filename }}"
    mode: '0644'
  when: gather_apt_repos_report_format == "txt" and gather_apt_repos_report_dir != ""

- name: Generate JSON report
  ansible.builtin.copy:
    content: "{{ apt_repositories_info | to_nice_json }}"
    dest: "{{ gather_apt_repos_report_dir }}/{{ report_filename }}"
    mode: '0644'
  when: gather_apt_repos_report_format == "json" and gather_apt_repos_report_dir != ""

- name: Generate YAML report
  ansible.builtin.copy:
    content: "{{ apt_repositories_info | to_nice_yaml }}"
    dest: "{{ gather_apt_repos_report_dir }}/{{ report_filename }}"
    mode: '0644'
  when: gather_apt_repos_report_format == "yaml" and gather_apt_repos_report_dir != ""

- name: Display gathered repository information
  ansible.builtin.debug:
    var: apt_repositories_info

- name: Show report location
  ansible.builtin.debug:
    msg: "Report saved to {{ gather_apt_repos_report_dir }}/{{ report_filename }}"
  when: gather_apt_repos_report_dir != ""
```

3. templates/report.j2 - шаблон для текстового отчета

```jinja2
# APT Repository Report
# Generated: {{ apt_repositories_info.timestamp }}
# Hostname: {{ apt_repositories_info.hostname }}

{% if apt_repositories_info.sources_list_exists %}
=== /etc/apt/sources.list ===
{{ apt_repositories_info.sources_list_content }}

{% else %}
=== /etc/apt/sources.list ===
File does not exist

{% endif %}
=== Files in /etc/apt/sources.list.d/ ===

{% if apt_repositories_info.sources_list_d_list_files %}
*.list files:
{% for file in apt_repositories_info.sources_list_d_list_files %}
--- {{ file.path }} ---
{{ file.content }}

{% endfor %}
{% else %}
No .list files found in /etc/apt/sources.list.d/
{% endif %}

{% if apt_repositories_info.sources_list_d_sources_files %}
*.sources files:
{% for file in apt_repositories_info.sources_list_d_sources_files %}
--- {{ file.path }} ---
{{ file.content }}

{% endfor %}
{% else %}
No .sources files found in /etc/apt/sources.list.d/
{% endif %}

# Summary:
- /etc/apt/sources.list: {{ "EXISTS" if apt_repositories_info.sources_list_exists else "MISSING" }}
- .list files: {{ apt_repositories_info.sources_list_d_list_files | length }}
- .sources files: {{ apt_repositories_info.sources_list_d_sources_files | length }}
```

🎯 Пример плейбука с настройками отчетов

```yaml
---
- hosts: all
  vars:
    gather_apt_repos_report_dir: "/tmp/apt_reports"
    gather_apt_repos_report_format: "txt"  # или "json", "yaml"
    gather_apt_repos_create_timestamp: true
    gather_apt_repos_include_hostname: true
  
  roles:
    - gather_apt_repos
  
  tasks:
    - name: Display report contents
      ansible.builtin.shell: |
        ls -la "{{ gather_apt_repos_report_dir }}" && \
        echo "=== Last report ===" && \
        ls -t "{{ gather_apt_repos_report_dir }}" | head -1 | xargs -I {} cat "{{ gather_apt_repos_report_dir }}/{}"
      register: report_contents
      changed_when: false
    
    - name: Show report summary
      ansible.builtin.debug:
        msg: "{{ report_contents.stdout }}"
```

📊 Дополнительные возможности

Для HTML отчета добавьте шаблон templates/report_html.j2:

```jinja2
<!DOCTYPE html>
<html>
<head>
    <title>APT Repository Report - {{ apt_repositories_info.hostname }}</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .header { background: #f0f0f0; padding: 10px; border-radius: 5px; }
        .file-section { margin: 15px 0; }
        .file-content { background: #f8f8f8; padding: 10px; border: 1px solid #ddd; }
        .summary { background: #e8f4f8; padding: 10px; border-radius: 5px; }
    </style>
</head>
<body>
    <div class="header">
        <h1>APT Repository Report</h1>
        <p><strong>Generated:</strong> {{ apt_repositories_info.timestamp }}</p>
        <p><strong>Hostname:</strong> {{ apt_repositories_info.hostname }}</p>
    </div>

    <div class="file-section">
        <h2>/etc/apt/sources.list</h2>
        {% if apt_repositories_info.sources_list_exists %}
        <pre class="file-content">{{ apt_repositories_info.sources_list_content }}</pre>
        {% else %}
        <p>File does not exist</p>
        {% endif %}
    </div>

    <div class="file-section">
        <h2>Files in /etc/apt/sources.list.d/</h2>
        
        <h3>*.list files</h3>
        {% if apt_repositories_info.sources_list_d_list_files %}
        {% for file in apt_repositories_info.sources_list_d_list_files %}
        <h4>{{ file.path }}</h4>
        <pre class="file-content">{{ file.content }}</pre>
        {% endfor %}
        {% else %}
        <p>No .list files found</p>
        {% endif %}

        <h3>*.sources files</h3>
        {% if apt_repositories_info.sources_list_d_sources_files %}
        {% for file in apt_repositories_info.sources_list_d_sources_files %}
        <h4>{{ file.path }}</h4>
        <pre class="file-content">{{ file.content }}</pre>
        {% endfor %}
        {% else %}
        <p>No .sources files found</p>
        {% endif %}
    </div>

    <div class="summary">
        <h2>Summary</h2>
        <ul>
            <li>/etc/apt/sources.list: {{ "EXISTS" if apt_repositories_info.sources_list_exists else "MISSING" }}</li>
            <li>.list files: {{ apt_repositories_info.sources_list_d_list_files | length }}</li>
            <li>.sources files: {{ apt_repositories_info.sources_list_d_sources_files | length }}</li>
        </ul>
    </div>
</body>
</html>
```

И добавьте задачу для HTML формата в tasks/main.yml:

```yaml
- name: Generate HTML report
  ansible.builtin.template:
    src: report_html.j2
    dest: "{{ gather_apt_repos_report_dir }}/{{ report_filename }}"
    mode: '0644'
  when: gather_apt_repos_report_format == "html" and gather_apt_repos_report_dir != ""
```

🚀 Использование

```bash
# Запуск с текстовым отчетом
ansible-playbook -i inventory gather_repos.yml

# Запуск с JSON отчетом
ansible-playbook -i inventory gather_repos.yml -e "gather_apt_repos_report_format=json"

# Запуск с HTML отчетом
ansible-playbook -i inventory gather_repos.yml -e "gather_apt_repos_report_format=html"
```

Теперь роль будет сохранять подробные отчеты в указанной директории с возможностью выбора формата (TXT, JSON, YAML, HTML) и дополнительными метаданными.