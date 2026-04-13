Ниже представлен пример Ansible роли, которая копирует shell-скрипт на удалённый сервер, запускает его и выводит результат выполнения.

Структура роли

```
copy_and_run_script/
├── tasks/
│   └── main.yml
└── files/
    └── myscript.sh          # сам скрипт (может быть передан через переменные)
```

Содержимое tasks/main.yml

```yaml
---
- name: Копировать скрипт на удалённый сервер
  ansible.builtin.copy:
    src: myscript.sh          # файл из каталога files/ роли
    dest: /tmp/myscript.sh
    mode: '0755'              # сделать исполняемым
  register: copy_result

- name: Запустить скрипт на удалённом сервере
  ansible.builtin.command: /tmp/myscript.sh
  register: script_output
  changed_when: false

- name: Вывести результат выполнения скрипта
  ansible.builtin.debug:
    msg:
      - "stdout: {{ script_output.stdout }}"
      - "stderr: {{ script_output.stderr }}"
      - "rc: {{ script_output.rc }}"
```

Альтернатива – использование модуля script

Если не требуется сохранять копию скрипта на сервере, можно обойтись одной задачей с модулем script, который сам копирует и выполняет скрипт:

```yaml
- name: Запустить локальный скрипт на удалённом сервере и вывести результат
  ansible.builtin.script: myscript.sh
  register: script_output
  changed_when: false

- name: Вывод результата
  ansible.builtin.debug:
    var: script_output.stdout
```

Но по условию требуется явное копирование, поэтому первый вариант предпочтительнее.

Использование роли в плейбуке

```yaml
- hosts: target_servers
  roles:
    - copy_and_run_script
```

Замечания

· Убедитесь, что файл myscript.sh существует в files/ роли.
· Для вывода только успешного результата можно использовать ansible.builtin.debug с проверкой script_output.rc == 0.
· Если скрипт долгий, добавьте async и poll для асинхронного выполнения.