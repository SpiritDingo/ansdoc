Универсальная Ansible роль для развертывания артефактов из Nexus

Роль предназначена для автоматизации процесса доставки и развертывания архивированных артефактов из локального репозитория Nexus на целевые Linux-серверы. Роль скачивает архив, распаковывает его во временную директорию, а затем копирует содержимое в указанную целевую папку с заменой файлов.

Возможности

· Скачивание артефакта с Nexus по HTTP/HTTPS (поддержка Basic Auth).
· Проверка контрольной суммы (SHA256 и др.).
· Распаковка архивов различных форматов (tar, tar.gz, zip, tar.bz2, tar.xz и т.д.).
· Копирование содержимого в целевую директорию с заменой (опционально – полная очистка целевой директории перед копированием).
· Очистка временных файлов после выполнения.
· Идемпотентность: повторный запуск с неизменным архивом не приводит к повторному копированию (если используется проверка изменений).

Структура роли

```
deploy_artifact/
├── defaults/
│   └── main.yml          # Значения переменных по умолчанию
├── tasks/
│   └── main.yml          # Основные задачи
└── meta/
    └── main.yml          # Метаданные роли (опционально)
```

Переменные роли

Все параметры настраиваются через переменные. В defaults/main.yml заданы значения по умолчанию.

Переменная Описание Значение по умолчанию
nexus_url Базовый URL Nexus-сервера (без /repository/...) http://nexus.local
nexus_repository Имя репозитория в Nexus releases
artifact_path Путь к артефакту внутри репозитория (например, com/example/app/1.0/app-1.0.tar.gz) обязательный
archive_name Имя файла архива на целевом сервере (если отличается от имени в URL) artifact.tar.gz
temp_dir Временная директория для скачивания и распаковки /tmp/ansible_artifact_deploy
extracted_subdir Поддиректория внутри temp_dir для распакованных файлов extracted
target_dir Конечная директория, куда копируется содержимое архива /opt/application
target_clean Удалять ли целевую директорию перед копированием (true – удалить и создать заново) false
cleanup_temp Удалять временную директорию после завершения true
nexus_user Имя пользователя для Basic Auth (если требуется) ""
nexus_password Пароль для Basic Auth ""
force_basic_auth Принудительно использовать Basic Auth false
checksum Контрольная сумма для проверки скачанного файла (формат sha256:...) ""
copy_method Способ копирования: shell (cp -rf) или synchronize (rsync) shell

Пример defaults/main.yml

```yaml
---
nexus_url: "http://nexus.local"
nexus_repository: "releases"
artifact_path: ""               # обязательно переопределить
archive_name: "artifact.tar.gz"
temp_dir: "/tmp/ansible_artifact_deploy"
extracted_subdir: "extracted"
target_dir: "/opt/application"
target_clean: false
cleanup_temp: true
nexus_user: ""
nexus_password: ""
force_basic_auth: false
checksum: ""
copy_method: "shell"            # или "synchronize"
```

Основные задачи (tasks/main.yml)

```yaml
---
- name: Ensure temporary directory exists
  file:
    path: "{{ temp_dir }}"
    state: directory

- name: Download artifact from Nexus
  get_url:
    url: "{{ nexus_url }}/repository/{{ nexus_repository }}/{{ artifact_path }}"
    dest: "{{ temp_dir }}/{{ archive_name }}"
    url_username: "{{ nexus_user }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: "{{ force_basic_auth }}"
    checksum: "{{ checksum }}"
    mode: '0644'
  register: download_result

- name: Remove old extraction directory if target_clean is true
  file:
    path: "{{ temp_dir }}/{{ extracted_subdir }}"
    state: absent
  when: target_clean | bool

- name: Create extraction directory
  file:
    path: "{{ temp_dir }}/{{ extracted_subdir }}"
    state: directory

- name: Extract archive into temporary directory
  unarchive:
    src: "{{ temp_dir }}/{{ archive_name }}"
    dest: "{{ temp_dir }}/{{ extracted_subdir }}"
    remote_src: yes
  when: download_result.changed or (target_clean | bool)

- name: Check if target directory exists
  stat:
    path: "{{ target_dir }}"
  register: target_stat

- name: Create target directory if it does not exist
  file:
    path: "{{ target_dir }}"
    state: directory
  when: not target_stat.stat.exists

- name: Clean target directory (if target_clean is true)
  file:
    path: "{{ target_dir }}"
    state: absent
  when: target_clean | bool

- name: Recreate target directory after cleaning
  file:
    path: "{{ target_dir }}"
    state: directory
  when: target_clean | bool

- name: Copy extracted content to target directory (using shell)
  shell: "cp -rf {{ temp_dir }}/{{ extracted_subdir }}/. {{ target_dir }}/"
  when: copy_method == "shell"

- name: Copy extracted content to target directory (using synchronize)
  synchronize:
    src: "{{ temp_dir }}/{{ extracted_subdir }}/"
    dest: "{{ target_dir }}/"
    recursive: yes
    delete: "{{ target_clean | bool }}"
    rsync_opts:
      - "--chmod=Du=rwx,Dg=rx,Do=rx,Fu=rw,Fg=r,Fo=r"
  when: copy_method == "synchronize"

- name: Cleanup temporary directory
  file:
    path: "{{ temp_dir }}"
    state: absent
  when: cleanup_temp | bool
```

Примечания по реализации

1. Скачивание – используется модуль get_url с возможностью указания учётных данных и контрольной суммы. Если checksum не задан, проверка не выполняется.
2. Распаковка – модуль unarchive с remote_src: yes распаковывает локальный архив на управляемом хосте. Формат архива определяется автоматически по расширению файла.
3. Идемпотентность – если архив не изменился (по changed от get_url), и target_clean не включён, то задачи распаковки и копирования пропускаются. Это позволяет избежать лишних операций при повторных запусках. Если требуется безусловное обновление, установите target_clean: true или всегда принудительно запускайте роль с --force-handlers (не рекомендуется).
4. Копирование – по умолчанию используется cp -rf через shell, что гарантирует замену файлов и сохранение структуры. Альтернативно можно выбрать synchronize (требует установленного rsync на обеих сторонах) с опцией delete для удаления лишних файлов в целевой директории, если target_clean: true.
5. Очистка целевой директории – если target_clean: true, целевая директория полностью удаляется и создаётся заново перед копированием. Это гарантирует, что после развертывания в ней будут только файлы из архива.
6. Обработка ошибок – при сбое задачи unarchive или copy временные файлы можно оставить для диагностики, отключив cleanup_temp.

Пример использования в playbook

```yaml
---
- name: Deploy application from Nexus
  hosts: app_servers
  roles:
    - role: deploy_artifact
      vars:
        nexus_url: "http://nexus.corp.local"
        nexus_repository: "releases"
        artifact_path: "com/mycompany/myapp/1.0.0/myapp-1.0.0.tar.gz"
        target_dir: "/opt/myapp"
        target_clean: true
        cleanup_temp: true
        nexus_user: "deployer"
        nexus_password: "secret"
        checksum: "sha256:abc123..."
```

Требования

· Ansible 2.9 или выше.
· На управляемом хосте должны быть доступны стандартные утилиты (tar, unzip, cp).
· Для метода synchronize требуется установленный rsync на управляемом хосте и на control-ноде (или доступ через SSH).
· Если Nexus требует аутентификации, укажите nexus_user и nexus_password.

Роль может быть легко расширена (например, добавление уведомлений, перезапуск сервисов после деплоя через handlers).

_____________
_____________


Роль полностью поддерживает архивы формата zip. Модуль unarchive в Ansible автоматически определяет тип архива по расширению файла и использует соответствующий инструмент для распаковки (для zip — unzip). Убедитесь, что на целевой машине установлен пакет unzip (обычно входит в стандартные репозитории).

Пример настройки для zip-архива

```yaml
- name: Deploy zip artifact from Nexus
  hosts: app_servers
  roles:
    - role: deploy_artifact
      vars:
        nexus_url: "http://nexus.local"
        nexus_repository: "releases"
        artifact_path: "com/example/app/1.0/app-1.0.zip"
        archive_name: "app-1.0.zip"          # имя локального файла
        target_dir: "/opt/application"
        target_clean: true                   # очистить целевую директорию перед копированием
        cleanup_temp: true
        # Опционально: контрольная сумма и аутентификация
        checksum: "sha256:abc123..."
        nexus_user: "deployer"
        nexus_password: "secret"
```

Если unzip не установлен

Добавьте в роль отдельную задачу (например, в tasks/main.yml перед распаковкой), чтобы гарантировать наличие unzip на управляемом хосте:

```yaml
- name: Ensure unzip is installed (for zip archives)
  package:
    name: unzip
    state: present
  when: archive_name.endswith('.zip')
```

Это делает роль более самодостаточной для zip-артефактов, не требуя ручной установки на каждом сервере.