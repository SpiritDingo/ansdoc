Вот Ansible плейбук для сбора информации о каталоге /var/www/html и поиска содержимого web_sync:

```yaml
---
- name: Проверка каталога /var/www/html и поиск web_sync
  hosts: all
  gather_facts: false
  
  tasks:
    - name: Проверить существование каталога /var/www/html
      stat:
        path: /var/www/html
      register: www_dir_info

    - name: Вывести информацию о каталоге
      debug:
        msg: |
          Каталог /var/www/html:
          - Существует: {{ www_dir_info.stat.exists }}
          {% if www_dir_info.stat.exists %}
          - Тип: {{ www_dir_info.stat.type }}
          - Владелец: {{ www_dir_info.stat.pw_name }}
          - Группа: {{ www_dir_info.stat.gr_name }}
          - Права: {{ www_dir_info.stat.mode }}
          - Размер: {{ www_dir_info.stat.size }} байт
          - Дата изменения: {{ www_dir_info.stat.mtime }}
          {% endif %}

    - name: Поиск файлов/каталогов с именем web_sync в /var/www/html
      find:
        paths: /var/www/html
        name: "*web_sync*"
        recurse: yes
      register: web_sync_files
      when: www_dir_info.stat.exists

    - name: Вывести найденные файлы/каталоги web_sync
      debug:
        msg: |
          Найдено {{ web_sync_files.files | length }} объектов с именем web_sync:
          {% for file in web_sync_files.files %}
          - Путь: {{ file.path }}
            Тип: {{ file.file_type }}
            Размер: {{ file.size }} байт
          {% endfor %}
      when: www_dir_info.stat.exists and web_sync_files.files | length > 0

    - name: Сообщение если web_sync не найден
      debug:
        msg: "Файлы/каталоги с именем web_sync не найдены в /var/www/html"
      when: www_dir_info.stat.exists and web_sync_files.files | length == 0

    - name: Сообщение если каталог /var/www/html не существует
      debug:
        msg: "Каталог /var/www/html не существует"
      when: not www_dir_info.stat.exists
```

Также вот упрощенная версия плейбука:

```yaml
---
- name: Проверка /var/www/html и поиск web_sync
  hosts: all
  gather_facts: false
  
  tasks:
    - name: Проверить каталог /var/www/html
      stat:
        path: /var/www/html
      register: www_dir

    - name: Поиск web_sync содержимого
      find:
        paths: /var/www/html
        patterns: "*web_sync*"
        recurse: yes
      register: web_sync_results
      when: www_dir.stat.exists

    - name: Вывести результаты
      debug:
        var: web_sync_results.files
      when: www_dir.stat.exists and web_sync_results.files | length > 0

    - name: Сообщение если ничего не найдено
      debug:
        msg: "Каталог /var/www/html не существует или web_sync не найден"
      when: not www_dir.stat.exists or web_sync_results.files | length == 0
```

Для запуска плейбука сохраните его в файл (например, check_web_sync.yml) и выполните:

```bash
ansible-playbook -i inventory check_web_sync.yml
```

Где inventory - ваш файл с перечнем хостов.

Плейбук выполнит:

1. Проверку существования каталога /var/www/html
2. Выведет подробную информацию о каталоге (если существует)
3. Поиск всех файлов и каталогов, содержащих web_sync в имени
4. Выведет результаты поиска