Для автоматической установки агента администрирования Kaspersky (klnagent) с помощью Ansible можно использовать готовые роли, доступные на GitHub и других платформах. Вот как это можно сделать.

🚀 Готовые роли на GitHub

1. ansible-kaspersky-role от AlexKirmasov

Это готовая роль для CentOS 7, которая включает в себя установку как агента администрирования (klnagent), так и Kaspersky Endpoint Security (KESL).

Ключевые особенности:

· Структура: Роль имеет типовую структуру, что упрощает её интеграцию в ваш проект.
· Методы установки: Предлагает два варианта:
  · autonomous_package: Установка из автономного пакета (например, klnagent64-11.0.0-38.x86_64.sh).
  · custom_install: Кастомная установка, при которой пакет загружается по указанному URL.
· Переменные: Для настройки используются следующие переменные:
  · metod_klnagent_param: Выбор метода установки (custom_install или autonomous_package).
  · klnagent_url: URL-адрес для скачивания пакета агента.
  · klnagent_autonomous_package_name: Имя файла автономного пакета.
  · KLNAGENT_SERVER: IP-адрес или имя сервера администрирования Kaspersky Security Center.
  · KLNAGENT_PORT: Порт для подключения к серверу администрирования (по умолчанию 14000).
  · KLNAGENT_SSLPORT: SSL-порт (по умолчанию 13000).
  · KLNAGENT_USESSL: Использовать ли SSL (1 — да, 0 — нет).
· Запуск: Плейбуки можно запускать как для всех хостов, так и для выборочных.
  ```bash
  # Для всех хостов
  ansible-playbook service_install_klnagent_all_servers.yml --vault-password-file=/path/to/secret --diff -vv
  
  # Для выбранных хостов
  ansible-playbook service_install_klnagent_select_servers.yml -e "host=test01" --vault-password-file=/path/to/secret --diff -vv
  ```
  Для хранения конфиденциальных данных (логинов и паролей от репозитория) используется ansible-vault.

2. Репозиторий ittxx/kaspersky

Этот репозиторий также содержит материалы по установке KESL и может быть полезен в дополнение к основной роли.

📝 Пример плейбука для установки агента

Ниже представлен упрощенный пример, который вы можете адаптировать под свои нужды.

```yaml
---
- name: Установка агента администрирования Kaspersky
  hosts: all
  become: yes
  vars:
    # Метод установки: autonomous_package или custom_install
    metod_klnagent_param: "custom_install"
    # URL для скачивания дистрибутива
    klnagent_url: "http://your-repo-server/klnagent64-11.0.0-38.x86_64.sh"
    # Настройки подключения к KSC
    KLNAGENT_SERVER: "192.168.1.100"
    KLNAGENT_PORT: "14000"
    KLNAGENT_SSLPORT: "13000"
    KLNAGENT_USESSL: "1"
    # Имя файла дистрибутива
    klnagent_file_name: "klnagent64-11.0.0-38.x86_64.sh"

  tasks:
    - name: Создать временную директорию
      file:
        path: /tmp/kaspersky_install
        state: directory
        mode: '0755'

    - name: Скачать дистрибутив агента
      get_url:
        url: "{{ klnagent_url }}"
        dest: "/tmp/kaspersky_install/{{ klnagent_file_name }}"
        mode: '0755'
      when: metod_klnagent_param == "custom_install"

    - name: Запустить установку агента
      shell: "/tmp/kaspersky_install/{{ klnagent_file_name }} --autoinstall"
      args:
        creates: /opt/kaspersky/klnagent64/sbin/klnagent
      when: metod_klnagent_param == "custom_install"
```

💡 Важные моменты

· Система-хранилище (репозиторий): Для успешной работы через custom_install необходимо развернуть HTTP-сервер (например, Nginx), который будет раздавать дистрибутивы. В официальной документации Kaspersky есть инструкции по настройке такого репозитория.
· Автоответы (autoanswers): При использовании метода autonomous_package или кастомной установки для автоматизации ответов на запросы установщика можно подготовить файл с настройками, например, autoanswers.conf, где указываются параметры подключения к серверу администрирования.
· Безопасность: Всегда следуйте принципу наименьших привилегий и храните учетные данные (логины, пароли) в зашифрованном виде с помощью ansible-vault.

Надеюсь, эта информация поможет вам в автоматизации. Если у вас появятся дополнительные вопросы, обращайтесь.


Конечно, добавлю важный блок предварительной настройки, который обычно опускают в базовых примерах. Ниже — расширенная роль, включающая подготовку перед установкой агента Kaspersky.

🛠 Подготовка перед установкой (pre-tasks)

Перед запуском установщика необходимо:

1. Проверить поддерживаемую ОС (например, RHEL/CentOS 7/8/9, Ubuntu 20.04/22.04).
2. Удалить старые/конфликтующие версии агента или KES (если есть).
3. Установить обязательные зависимости (glibc, libstdc++, wget, perl).
4. Настроить SELinux (для CentOS/RHEL — перевести в Permissive или добавить политики).
5. Открыть порты (firewalld/iptables) для связи с KSC: 13000 (SSL), 14000 (обычный), 15000 (активация).
6. Создать временную директорию и загрузить дистрибутив (если нет локального репозитория).
7. Проверить наличие установщика и корректность прав.

📄 Пример блока подготовки в роли kaspersky_agent

Структура роли:

```
roles/kaspersky_agent/
├── tasks/
│   ├── main.yml
│   ├── prepare.yml
│   └── install.yml
├── vars/
│   └── main.yml
└── files/
    └── klnagent64-11.0.0-38.x86_64.sh
```

vars/main.yml (настройки)

```yaml
kaspersky_agent_version: "11.0.0-38"
kaspersky_agent_pkg: "klnagent64-{{ kaspersky_agent_version }}.x86_64.sh"
kaspersky_agent_download_url: "http://repo.example.com/kaspersky/{{ kaspersky_agent_pkg }}"
ksc_server: "192.168.1.100"
ksc_port: 14000
ksc_ssl_port: 13000
ksc_use_ssl: 1
temp_dir: "/tmp/kaspersky_install"
```

tasks/prepare.yml (все шаги подготовки)

```yaml
---
- name: Проверка ОС (только RedHat/CentOS/Ubuntu)
  fail:
    msg: "Неподдерживаемая ОС. Поддерживаются: RedHat, CentOS, Ubuntu"
  when:
    - ansible_os_family != "RedHat"
    - ansible_distribution != "Ubuntu"

- name: Установка зависимостей (RedHat)
  package:
    name:
      - glibc
      - libstdc++
      - wget
      - perl
      - policycoreutils-python-utils  # для управления SELinux
    state: present
  when: ansible_os_family == "RedHat"

- name: Установка зависимостей (Ubuntu)
  package:
    name:
      - libc6
      - libstdc++6
      - wget
      - perl
    state: present
  when: ansible_distribution == "Ubuntu"

- name: Удаление предыдущих версий агента Kaspersky
  package:
    name: "{{ item }}"
    state: absent
  loop:
    - klnagent64
    - kaspersky-endpoint-security
  ignore_errors: yes

- name: Остановка служб Kaspersky (если запущены)
  systemd:
    name: "{{ item }}"
    state: stopped
  loop:
    - klnagent
    - kesl
  ignore_errors: yes

- name: Временное отключение SELinux (если включён)
  selinux:
    state: permissive
    policy: targeted
  when:
    - ansible_os_family == "RedHat"
    - ansible_selinux.status == "enabled"

- name: Открыть порты в firewalld (RedHat)
  firewalld:
    port: "{{ item }}/tcp"
    permanent: yes
    immediate: yes
    state: enabled
  loop:
    - "{{ ksc_port }}"
    - "{{ ksc_ssl_port }}"
    - 15000
  when: ansible_os_family == "RedHat"

- name: Создать временную директорию
  file:
    path: "{{ temp_dir }}"
    state: directory
    mode: '0755'

- name: Загрузить дистрибутив агента (если нет локального)
  get_url:
    url: "{{ kaspersky_agent_download_url }}"
    dest: "{{ temp_dir }}/{{ kaspersky_agent_pkg }}"
    mode: '0755'
    timeout: 60
  register: download_result
  until: download_result is success
  retries: 3
  delay: 5
```

tasks/install.yml (установка после подготовки)

```yaml
---
- name: Запуск установки агента с автоответами
  shell: |
    cd {{ temp_dir }}
    ./{{ kaspersky_agent_pkg }} \
      --autoinstall \
      --server {{ ksc_server }} \
      --port {{ ksc_port }} \
      --ssl-port {{ ksc_ssl_port }} \
      --use-ssl {{ ksc_use_ssl }}
  args:
    creates: /opt/kaspersky/klnagent64/sbin/klnagent
  register: install_result

- name: Проверка успешности установки
  fail:
    msg: "Ошибка установки: {{ install_result.stderr }}"
  when: install_result.rc != 0

- name: Запуск и включение службы klnagent
  systemd:
    name: klnagent
    state: started
    enabled: yes
```

tasks/main.yml (точка входа)

```yaml
---
- name: Подготовка системы к установке Kaspersky Agent
  import_tasks: prepare.yml

- name: Установка агента Kaspersky
  import_tasks: install.yml
```

🧪 Пример плейбука для вызова роли

```yaml
- hosts: servers
  become: yes
  roles:
    - kaspersky_agent
  vars:
    ksc_server: "10.10.10.5"
    ksc_port: 14000
    kaspersky_agent_download_url: "https://internal-repo/klnagent64-11.0.0-38.x86_64.sh"
```

⚠️ Важные замечания по подготовке

Действие Почему важно
Удаление старых версий Конфликт версий агента вызывает ошибки соединения с KSC
Отключение SELinux (или политики) Установщик Kaspersky не всегда корректно работает с Enforcing
Открытие портов Агент должен инициировать соединение к KSC на указанные порты
Зависимости Без perl и libstdc++ установщик выдаст ошибку cannot execute binary file
Создание временной папки Некоторые установщики требуют исполняемый файл в локальной FS, не в смонтированной сети

После выполнения роли агент сам зарегистрируется в KSC (если указаны корректные параметры сервера). Если нужен отложенный старт (например, после перезагрузки), добавьте handler для перезапуска klnagent.

Такой подход покрывает 90% проблем, возникающих при ручной установке.


-----------------------------

Чтобы добавить в вашу Ansible роль установку KESL (Kaspersky Endpoint Security for Linux), нужно учесть, что скрипт kesl-12.4.0-1342.sh — это универсальный установщик, который сам определит тип вашей ОС и установит подходящий пакет (RPM или DEB). Ниже я опишу шаги, которые необходимо добавить в роль.

⚙️ Основные шаги для установки KESL

Процесс добавления KESL в вашу Ansible роль будет включать четыре основных этапа.

1. Подготовка системы: Установка необходимых зависимостей и удаление старых версий.
2. Загрузка дистрибутива: Скачивание универсального скрипта kesl-12.4.0-1342.sh из вашего Nexus-репозитория.
3. Запуск установки: Выполнение скрипта для автоматической установки пакета.
4. Пост-настройка: Запуск скрипта инициализации с файлом ответов autoinstall.ini для настройки продукта.

---

📝 Добавление задач в Ansible роль

Вот примерный код задач, которые нужно добавить в ваш плейбук.

```yaml
# tasks/install_kesl.yml
---
- name: 1. Установка зависимостей для KESL (RedHat/CentOS)
  ansible.builtin.package:
    name:
      - glibc
      - libstdc++
      - wget
      - perl
      - kernel-devel
      - make
      - gcc
    state: present
  when: ansible_os_family == "RedHat"

- name: 1. Установка зависимостей для KESL (Debian/Ubuntu)
  ansible.builtin.package:
    name:
      - libc6
      - libstdc++6
      - wget
      - perl
      - linux-headers-{{ ansible_kernel }}
      - make
      - gcc
    state: present
  when: ansible_distribution in ["Debian", "Ubuntu"]

- name: 2. Скачать дистрибутив KESL из Nexus
  ansible.builtin.get_url:
    url: "{{ kesl_download_url }}"
    dest: "{{ temp_dir }}/kesl-{{ kesl_version }}.sh"
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: yes
    mode: '0755'

- name: 3. Запустить установку KESL
  ansible.builtin.shell: |
    cd {{ temp_dir }}
    ./kesl-{{ kesl_version }}.sh --autoinstall
  args:
    creates: /opt/kaspersky/kesl/bin/kesl-setup.pl
  register: kesl_install
  failed_when: kesl_install.rc != 0

- name: 4. Создать файл autoinstall.ini для пост-настройки
  ansible.builtin.copy:
    dest: "{{ temp_dir }}/autoinstall.ini"
    content: |
      # Согласие с лицензией и политикой конфиденциальности
      EULA_AGREED=yes
      PRIVACY_POLICY_AGREED=yes
      
      # Настройка использования KSN (Kaspersky Security Network)
      USE_KSN=no
      
      # Настройка источника обновлений (SCServer = Сервер администрирования KSC)
      UPDATER_SOURCE=SCServer
      
      # Запустить обновление антивирусных баз после настройки
      UPDATE_EXECUTE=yes
      
      # Автоматически скачать и скомпилировать модуль ядра
      KERNEL_SRCS_INSTALL=yes
      
      # Локаль сервиса (по желанию)
      SERVICE_LOCALE=en_US.utf8
    mode: '0644'

- name: 5. Запустить автоматическую пост-настройку KESL
  ansible.builtin.shell: |
    /opt/kaspersky/kesl/bin/kesl-setup.pl --autoinstall {{ temp_dir }}/autoinstall.ini
  register: kesl_setup
  failed_when: kesl_setup.rc != 0

- name: 6. Запустить и включить службу KESL
  ansible.builtin.systemd:
    name: kesl
    state: started
    enabled: yes
```

🔧 Необходимые переменные

Не забудьте определить переменные, которые используются в задачах.

```yaml
# vars/main.yml
kesl_version: "12.4.0-1342"
kesl_download_url: "https://nexus.company.com/repository/kaspersky/kesl-{{ kesl_version }}.sh"
temp_dir: "/tmp/kaspersky_install"
```

⚠️ Важные замечания

· Параметр --autoinstall: Он позволяет скрипту kesl-12.4.0-1342.sh работать в тихом режиме, не задавая вопросов.
· Файл autoinstall.ini: Это ключевой элемент для настройки. В примере выше приведен минимальный набор параметров, но вы можете добавлять другие, например INSTALL_LICENSE для активации ключом или PROXY_SERVER.
· Компиляция модуля ядра: Параметр KERNEL_SRCS_INSTALL=yes может потребовать установки пакетов kernel-devel и gcc, которые уже добавлены в шаг с зависимостями.
· KSN: Параметр USE_KSN=no отключает участие в Kaspersky Security Network. Включите его (yes), если это разрешено вашей политикой безопасности.
· Источник обновлений: Если у вас есть локальный сервер администрирования KSC, укажите UPDATER_SOURCE=SCServer.

---

🧪 Запуск плейбука

Запустите плейбук, включающий обновленную роль. Убедитесь, что переменные с паролями для Nexus зашифрованы с помощью ansible-vault.

```bash
ansible-playbook -i inventory/production install_kaspersky.yml --ask-vault-pass
```

Если возникнут вопросы по адаптации параметров autoinstall.ini под вашу инфраструктуру, спрашивайте — я помогу разобраться.