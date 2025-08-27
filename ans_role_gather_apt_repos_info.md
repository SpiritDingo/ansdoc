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