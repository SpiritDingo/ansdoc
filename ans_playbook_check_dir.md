Вот Ansible плейбук для проверки каталога и поиска значения web_sync в файлах:

```yaml
---
- name: Проверка каталога /var/www/html и поиск значения web_sync в файлах
  hosts: all
  gather_facts: false
  
  tasks:
    - name: Проверка существования каталога /var/www/html
      stat:
        path: /var/www/html
      register: www_dir
      
    - name: Поиск всех файлов в каталоге (если он существует)
      find:
        paths: /var/www/html
        recurse: yes
        file_type: file
      register: all_files
      when: www_dir.stat.exists and www_dir.stat.isdir
      
    - name: Поиск значения web_sync во всех файлах
      shell: |
        grep -r -l "web_sync" /var/www/html 2>/dev/null || true
      register: files_with_web_sync
      changed_when: false
      when: www_dir.stat.exists and www_dir.stat.isdir
      
    - name: Получение детальной информации о файлах с web_sync
      stat:
        path: "{{ item }}"
      register: web_sync_files_info
      loop: "{{ files_with_web_sync.stdout_lines }}"
      when: 
        - www_dir.stat.exists 
        - www_dir.stat.isdir
        - files_with_web_sync.stdout_lines | length > 0
        
    - name: Поиск точного значения web_sync с контекстом
      shell: |
        grep -n -H "web_sync" "{{ item }}" 2>/dev/null || true
      register: web_sync_matches
      loop: "{{ files_with_web_sync.stdout_lines }}"
      when: 
        - www_dir.stat.exists 
        - www_dir.stat.isdir
        - files_with_web_sync.stdout_lines | length > 0
      changed_when: false
      
    - name: Вывод результатов поиска
      debug:
        msg: |
          Каталог /var/www/html существует: {{ www_dir.stat.exists }}
          {% if www_dir.stat.exists %}
          Найдено файлов с упоминанием 'web_sync': {{ files_with_web_sync.stdout_lines | length }}
          {% if files_with_web_sync.stdout_lines | length > 0 %}
          Файлы и совпадения:
          {% for file in files_with_web_sync.stdout_lines %}
          - Файл: {{ file }}
            Совпадения:
            {% for match in web_sync_matches.results %}
            {% if match.item == file and match.stdout %}
            {{ match.stdout }}
            {% endif %}
            {% endfor %}
          {% endfor %}
          {% else %}
          Значение 'web_sync' не найдено в файлах
          {% endif %}
          {% else %}
          Каталог /var/www/html не существует - поиск пропущен
          {% endif %}
      var: msg
      
    - name: Альтернативный вывод (более структурированный)
      debug:
        msg: "Файл: {{ item.item }}, Совпадения: {{ item.stdout_lines | join(', ') }}"
      loop: "{{ web_sync_matches.results }}"
      when: 
        - www_dir.stat.exists 
        - www_dir.stat.isdir
        - item.stdout != ''
        - files_with_web_sync.stdout_lines | length > 0
```

Более простая версия плейбука:

```yaml
---
- name: Поиск значения web_sync в каталоге /var/www/html
  hosts: all
  gather_facts: false
  
  tasks:
    - name: Проверка существования каталога /var/www/html
      stat:
        path: /var/www/html
      register: www_dir
      
    - name: Поиск значения web_sync в файлах (если каталог существует)
      shell: |
        if [ -d "/var/www/html" ]; then
          grep -r -n -H "web_sync" /var/www/html 2>/dev/null || echo "Значение 'web_sync' не найдено"
        else
          echo "Каталог /var/www/html не существует"
        fi
      register: web_sync_search
      changed_when: false
      
    - name: Вывод результатов поиска
      debug:
        msg: "{{ web_sync_search.stdout_lines }}"
```

Ключевые особенности плейбука:

1. Проверка существования каталога с помощью модуля stat
2. Поиск рекурсивно во всех файлах каталога
3. Поиск именно значения web_sync, а не файлов с таким именем
4. Вывод пути к файлу и строки с совпадением
5. Пропуск задачи если каталог не существует
6. Обработка ошибок - игнорирование ошибок доступа и т.д.

Плейбук покажет:

· Существует ли каталог /var/www/html
· В каких файлах найдено значение web_sync
· Полные пути к этим файлам
· Конкретные строки с совпадениями (с номерами строк)