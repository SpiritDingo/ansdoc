Ниже представлена Ansible-роль ubuntu_repo_info, которая собирает информацию о настроенных APT-репозиториях в Ubuntu: имена файлов, строки с описанием репозиториев (deb/deb-src) и проверяет доступность URL каждого репозитория.

Структура роли

```
ubuntu_repo_info/
├── tasks/
│   └── main.yml
└── README.md (опционально)
```

tasks/main.yml

```yaml
---
- name: Найти все файлы .list в директориях apt
  ansible.builtin.find:
    paths:
      - /etc/apt/sources.list
      - /etc/apt/sources.list.d
    patterns: "*.list"
    file_type: file
  register: repo_files
  become: yes

- name: Прочитать содержимое найденных файлов
  ansible.builtin.slurp:
    src: "{{ item.path }}"
  loop: "{{ repo_files.files }}"
  register: repo_contents
  become: yes

- name: Извлечь строки репозиториев и подготовить список для проверки
  ansible.builtin.set_fact:
    repo_entries: >-
      {{
        repo_entries | default([]) +
        [
          {
            'file': item.item.path,
            'line': line,
            'url': (line | regex_replace('^\\s*deb(-src)?\\s+', '') | regex_replace('\\s+.*$', ''))
          }
        ]
      }}
  loop: "{{ repo_contents.results }}"
  loop_control:
    loop_var: item
  vars:
    lines: "{{ (item.content | b64decode).splitlines() | map('trim') | select('match', '^\\s*deb(-src)?\\s+.*$') | list }}"
  when: lines | length > 0

- name: Оставить только уникальные URL для проверки доступности
  ansible.builtin.set_fact:
    unique_urls: "{{ repo_entries | map(attribute='url') | unique | list }}"
  when: repo_entries is defined

- name: Проверить доступность каждого уникального URL репозитория
  ansible.builtin.uri:
    url: "{{ item }}"
    method: HEAD
    follow_redirects: safe
    timeout: 10
    validate_certs: no  # можно изменить при необходимости
  register: url_checks
  loop: "{{ unique_urls }}"
  ignore_errors: yes
  when: unique_urls is defined

- name: Сформировать итоговый список репозиториев с признаком доступности
  ansible.builtin.set_fact:
    repo_info: >-
      {{
        repo_entries | map('combine', {
          'accessible': (
            url_checks.results
            | selectattr('item', 'equalto', item.url)
            | map(attribute='status')
            | list
            | first
            | default(-1)
          ) in [200, 301, 302, 304, 401, 403]
        }) | list
      }}
  vars:
    url_checks_dict: "{{ url_checks.results | items2dict(key_name='item', value_name='status') }}"
  # Более простая версия с фильтром map('combine', ...) может не работать в старых версиях Jinja2.
  # Альтернативный подход через json_query или цикл:

- name: Альтернативное заполнение repo_info через цикл (если предыдущая задача не работает)
  ansible.builtin.set_fact:
    repo_info_final: "{{ repo_info_final | default([]) + [entry | combine({'accessible': is_accessible})] }}"
  loop: "{{ repo_entries }}"
  loop_control:
    loop_var: entry
  vars:
    url_status: "{{ url_checks.results | selectattr('item', 'equalto', entry.url) | map(attribute='status') | first | default(-1) }}"
    is_accessible: "{{ url_status in [200, 301, 302, 304, 401, 403] }}"
  when: repo_entries is defined

- name: Вывести результат в консоль
  ansible.builtin.debug:
    var: repo_info_final | default(repo_info)
```

Пояснение работы

1. Поиск файлов – модуль find ищет все *.list в /etc/apt/sources.list (как файл) и /etc/apt/sources.list.d (включая возможные подкаталоги).
2. Чтение содержимого – slurp читает файлы в бинарном виде, затем декодируется в текст.
3. Парсинг строк – из каждой строки, начинающейся с deb или deb-src, извлекается URL (всё, что после типа репозитория и до первого пробела после URL). Например, из deb http://archive.ubuntu.com/ubuntu/ jammy main будет http://archive.ubuntu.com/ubuntu/.
4. Уникальные URL – чтобы не проверять один и тот же URL многократно, выбираются уникальные значения.
5. Проверка доступности – для каждого уникального URL выполняется HEAD-запрос с таймаутом 10 секунд. Признак доступности: HTTP-статус 200, 301, 302, 304 (редиректы), 401 (требуется авторизация) или 403 (доступ запрещён, но сервер отвечает). Ошибки соединения или статусы 4xx/5xx считаются недоступностью.
6. Сборка финального списка – каждая запись о репозитории дополняется полем accessible (true/false).
7. Вывод – результат показывается через debug.

Пример вывода

```json
[
  {
    "file": "/etc/apt/sources.list.d/docker.list",
    "line": "deb [arch=amd64] https://download.docker.com/linux/ubuntu jammy stable",
    "url": "https://download.docker.com/linux/ubuntu",
    "accessible": true
  },
  {
    "file": "/etc/apt/sources.list",
    "line": "deb http://archive.ubuntu.com/ubuntu/ jammy main restricted",
    "url": "http://archive.ubuntu.com/ubuntu/",
    "accessible": true
  }
]
```

Использование

Добавьте роль в playbook:

```yaml
- hosts: ubuntu_servers
  roles:
    - ubuntu_repo_info
```

Если нужно сохранить результат в файл для последующего анализа, добавьте задачу:

```yaml
- name: Сохранить отчёт о репозиториях
  ansible.builtin.copy:
    content: "{{ repo_info_final | default(repo_info) | to_nice_json }}"
    dest: "/tmp/repo_accessibility_report.json"
  delegate_to: localhost
```

Примечания

· Роль требует become: yes для чтения файлов в /etc/apt/.
· Проверка доступности выполняется через HTTP/HTTPS. Если репозиторий использует протокол file:// или ftp://, проверка не сработает – в таких случаях поле accessible будет false. При необходимости можно расширить логику, добавив проверку через ansible.builtin.stat для локальных каталогов.
· Таймауты и игнорирование ошибок SSL можно настроить под свою среду.


_____


Ошибка возникла из‑за того, что в задаче Извлечь строки репозиториев... использовалась переменная line, которая не была определена. Цикл шёл по файлам, но внутри шаблона set_fact требовалась итерация по строкам каждого файла, а вложенного цикла предусмотрено не было.

Ниже приведён исправленный tasks/main.yml. Для каждого файла сначала формируется список словарей с полями file, line и url, а затем все списки объединяются в общий перечень. После этого проверяется доступность уникальных URL и собирается итоговый отчёт.

```yaml
---
- name: Найти все файлы .list в директориях apt
  ansible.builtin.find:
    paths:
      - /etc/apt/sources.list
      - /etc/apt/sources.list.d
    patterns: "*.list"
    file_type: file
  register: repo_files
  become: yes

- name: Прочитать содержимое найденных файлов
  ansible.builtin.slurp:
    src: "{{ item.path }}"
  loop: "{{ repo_files.files }}"
  register: repo_contents
  become: yes

- name: Извлечь строки репозиториев и URL для каждого файла
  ansible.builtin.set_fact:
    file_repos: >-
      {{
        file_repos | default({}) |
        combine({item.item.path: parsed_lines})
      }}
  loop: "{{ repo_contents.results }}"
  loop_control:
    loop_var: item
  vars:
    content_str: "{{ item.content | b64decode }}"
    raw_lines: "{{ content_str.splitlines() | map('trim') | select('match', '^\\s*deb(-src)?\\s+.*$') | list }}"
    parsed_lines: >-
      {%- set result = [] -%}
      {%- for line in raw_lines -%}
        {%- set url = line | regex_replace('^\\s*deb(-src)?\\s+', '') | regex_replace('\\s+.*$', '') -%}
        {%- set _ = result.append({'file': item.item.path, 'line': line, 'url': url}) -%}
      {%- endfor -%}
      {{ result }}
  when: content_str is defined

- name: Объединить все записи репозиториев в единый список
  ansible.builtin.set_fact:
    repo_entries: "{{ file_repos.values() | sum(start=[]) }}"
  when: file_repos is defined

- name: Оставить только уникальные URL для проверки доступности
  ansible.builtin.set_fact:
    unique_urls: "{{ repo_entries | map(attribute='url') | unique | list }}"
  when: repo_entries is defined

- name: Проверить доступность каждого уникального URL репозитория
  ansible.builtin.uri:
    url: "{{ item }}"
    method: HEAD
    follow_redirects: safe
    timeout: 10
    validate_certs: no  # при необходимости измените на yes
  register: url_checks
  loop: "{{ unique_urls }}"
  ignore_errors: yes
  when: unique_urls is defined

- name: Собрать итоговый список с признаком доступности
  ansible.builtin.set_fact:
    repo_info: "{{ repo_info | default([]) + [entry | combine({'accessible': is_accessible})] }}"
  loop: "{{ repo_entries }}"
  loop_control:
    loop_var: entry
  vars:
    url_status: "{{ url_checks.results | selectattr('item', 'equalto', entry.url) | map(attribute='status') | first | default(-1) }}"
    is_accessible: "{{ url_status in [200, 301, 302, 304, 401, 403] }}"
  when: repo_entries is defined

- name: Вывести результат в консоль
  ansible.builtin.debug:
    var: repo_info
```

Что было исправлено

· Устранена неопределённая переменная line – теперь для каждого файла отдельно перебираются строки, соответствующие шаблону deb/deb-src.
· Добавлена задача Объединить все записи репозиториев в единый список, чтобы получить плоский перечень всех найденных репозиториев.
· Итоговый список repo_info формируется через цикл по repo_entries с проверкой статуса из url_checks, что гарантирует корректную работу в любых версиях Jinja2.

Роль готова к использованию. При необходимости можно добавить задачу для сохранения отчёта в файл.

____


Чтобы получить единый JSON-отчёт со всех серверов, роль должна сначала собрать информацию на каждом хосте, а затем одной задачей (с run_once) объединить данные из hostvars и сохранить в файл на управляющей машине.

Ниже итоговый tasks/main.yml, который делает именно это. Результатом будет файл /tmp/all_hosts_repo_info.json на локальной машине (где запущен Ansible), содержащий словарь { "hostname": [ записи репозиториев ], ... }.

```yaml
---
- name: Найти все файлы .list в директориях apt
  ansible.builtin.find:
    paths:
      - /etc/apt/sources.list
      - /etc/apt/sources.list.d
    patterns: "*.list"
    file_type: file
  register: repo_files
  become: yes

- name: Прочитать содержимое найденных файлов
  ansible.builtin.slurp:
    src: "{{ item.path }}"
  loop: "{{ repo_files.files }}"
  register: repo_contents
  become: yes

- name: Извлечь строки репозиториев и URL для каждого файла
  ansible.builtin.set_fact:
    file_repos: >-
      {{
        file_repos | default({}) |
        combine({item.item.path: parsed_lines})
      }}
  loop: "{{ repo_contents.results }}"
  loop_control:
    loop_var: item
  vars:
    content_str: "{{ item.content | b64decode }}"
    raw_lines: "{{ content_str.splitlines() | map('trim') | select('match', '^\\s*deb(-src)?\\s+.*$') | list }}"
    parsed_lines: >-
      {%- set result = [] -%}
      {%- for line in raw_lines -%}
        {%- set url = line | regex_replace('^\\s*deb(-src)?\\s+', '') | regex_replace('\\s+.*$', '') -%}
        {%- set _ = result.append({'file': item.item.path, 'line': line, 'url': url}) -%}
      {%- endfor -%}
      {{ result }}
  when: content_str is defined

- name: Объединить все записи репозиториев в единый список
  ansible.builtin.set_fact:
    repo_entries: "{{ file_repos.values() | sum(start=[]) }}"
  when: file_repos is defined

- name: Оставить только уникальные URL для проверки доступности
  ansible.builtin.set_fact:
    unique_urls: "{{ repo_entries | map(attribute='url') | unique | list }}"
  when: repo_entries is defined

- name: Проверить доступность каждого уникального URL репозитория
  ansible.builtin.uri:
    url: "{{ item }}"
    method: HEAD
    follow_redirects: safe
    timeout: 10
    validate_certs: no
  register: url_checks
  loop: "{{ unique_urls }}"
  ignore_errors: yes
  when: unique_urls is defined

- name: Собрать итоговый список с признаком доступности
  ansible.builtin.set_fact:
    repo_info: "{{ repo_info | default([]) + [entry | combine({'accessible': is_accessible})] }}"
  loop: "{{ repo_entries }}"
  loop_control:
    loop_var: entry
  vars:
    url_status: "{{ url_checks.results | selectattr('item', 'equalto', entry.url) | map(attribute='status') | first | default(-1) }}"
    is_accessible: "{{ url_status in [200, 301, 302, 304, 401, 403] }}"
  when: repo_entries is defined

# --- Единый отчёт со всех хостов ---
- name: Агрегировать данные со всех серверов и сохранить единый JSON
  ansible.builtin.copy:
    content: "{{ all_hosts_data | to_nice_json }}"
    dest: "/tmp/all_hosts_repo_info.json"
  delegate_to: localhost
  run_once: true
  vars:
    all_hosts_data: >-
      {%- set result = {} -%}
      {%- for host in ansible_play_hosts_all -%}
        {%- if hostvars[host].repo_info is defined -%}
          {%- set _ = result.update({host: hostvars[host].repo_info}) -%}
        {%- endif -%}
      {%- endfor -%}
      {{ result }}
```

Как это работает

1. Каждый хост обрабатывает свои APT-файлы, извлекает строки репозиториев, проверяет доступность URL и сохраняет результат в факт repo_info.
2. Задача с run_once: true и delegate_to: localhost выполняется единожды на управляющей машине. Она перебирает весь список хостов из текущего плейбука (ansible_play_hosts_all), собирает их repo_info и формирует общий объект.
3. Этот объект записывается в /tmp/all_hosts_repo_info.json на локальной машине.

Формат итогового JSON

```json
{
  "server01": [
    {
      "file": "/etc/apt/sources.list",
      "line": "deb http://archive.ubuntu.com/ubuntu jammy main",
      "url": "http://archive.ubuntu.com/ubuntu/",
      "accessible": true
    }
  ],
  "server02": [
    ...
  ]
}
```

Использование в плейбуке

```yaml
- hosts: all_ubuntu
  become: yes
  roles:
    - ubuntu_repo_info
```

После выполнения в /tmp/all_hosts_repo_info.json будет готовый сводный отчёт.