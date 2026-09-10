Понял вас. Скорее всего, вы имели в виду официальный offboarding-скрипт от Microsoft, который корректно отключает устройство от портала Defender for Endpoint перед удалением агента.

Его выполнение можно добавить в роль как отдельную задачу, которая запускается перед удалением пакета.

Обновлённая структура роли

```text
roles/
└── mde_remove/
    ├── defaults/
    │   └── main.yml
    ├── files/
    │   └── MicrosoftDefenderATPOffboardingLinuxServer_valid_until_YYYY-MM-DD.py  # сюда положите скачанный скрипт
    ├── tasks/
    │   └── main.yml
    └── vars/
        └── main.yml
```

defaults/main.yml

Добавьте переменную для управления запуском offboarding-скрипта.

```yaml
# Удалять ли пакет mde-netfilter (сетевой фильтр, часто устанавливается вместе с MDE)
mde_remove_netfilter: true

# Удалять ли репозиторий Microsoft после удаления пакета
mde_remove_repo: true

# Запускать ли официальный offboarding-скрипт перед удалением
mde_run_offboard_script: true

# Имя файла скрипта (должен лежать в files/ роли)
mde_offboard_script_file: "MicrosoftDefenderATPOffboardingLinuxServer_valid_until_YYYY-MM-DD.py"

# Список пакетов для удаления (основной пакет и возможные зависимости)
mde_packages:
  - mdatp
```

tasks/main.yml

Основной файл задач. Теперь он сначала копирует и запускает offboarding-скрипт, а затем удаляет пакеты.

```yaml
---
- name: Проверка, что целевая ОС поддерживается
  ansible.builtin.assert:
    that:
      - ansible_facts['os_family'] in ['RedHat', 'Debian']
    fail_msg: "Роль поддерживает только RedHat (Oracle Linux 9) и Debian (Ubuntu)."

- name: Копирование offboarding-скрипта на целевой хост
  ansible.builtin.copy:
    src: "{{ mde_offboard_script_file }}"
    dest: "/tmp/{{ mde_offboard_script_file }}"
    mode: '0755'
  when: mde_run_offboard_script | bool

- name: Запуск offboarding-скрипта Microsoft Defender
  ansible.builtin.command:
    cmd: "python3 /tmp/{{ mde_offboard_script_file }}"
    chdir: /tmp
  register: mde_offboard_result
  failed_when:
    - mde_offboard_result.rc != 0
    - "'already offboarded' not in mde_offboard_result.stderr | lower"
    - "'not installed' not in mde_offboard_result.stderr | lower"
  changed_when: mde_offboard_result.rc == 0
  when: mde_run_offboard_script | bool

- name: Удаление временного offboarding-скрипта
  ansible.builtin.file:
    path: "/tmp/{{ mde_offboard_script_file }}"
    state: absent
  when: mde_run_offboard_script | bool

- name: Удаление пакета mdatp (Oracle Linux)
  ansible.builtin.yum:
    name: mdatp
    state: absent
  when: ansible_facts['os_family'] == 'RedHat'

- name: Удаление пакета mdatp (Ubuntu)
  ansible.builtin.apt:
    name: mdatp
    state: absent
    purge: yes
  when: ansible_facts['os_family'] == 'Debian'

- name: Удаление пакета mde-netfilter (опционально)
  ansible.builtin.package:
    name: mde-netfilter
    state: absent
  when: mde_remove_netfilter | bool

- name: Удаление репозитория Microsoft (опционально)
  ansible.builtin.yum_repository:
    name: microsoft-prod
    state: absent
  when:
    - mde_remove_repo | bool
    - ansible_facts['os_family'] == 'RedHat'

- name: Удаление репозитория Microsoft (Ubuntu)
  ansible.builtin.apt_repository:
    repo: "deb [arch=amd64,arm64] https://packages.microsoft.com/ubuntu/{{ ansible_distribution_version }}/prod {{ ansible_distribution_release }} main"
    state: absent
  when:
    - mde_remove_repo | bool
    - ansible_facts['os_family'] == 'Debian'

- name: Удаление остаточных файлов конфигурации
  ansible.builtin.file:
    path: "{{ item }}"
    state: absent
  loop:
    - /etc/opt/microsoft/mdatp
    - /var/opt/microsoft/mdatp
    - /var/log/microsoft/mdatp
  ignore_errors: yes
```

Как получить offboarding-скрипт

1. Войдите на портал Microsoft Defender (security.microsoft.com).
2. Перейдите в Settings → Endpoints → Device management → Offboarding.
3. Выберите Operating system: Linux Server, Deployment method: Local script.
4. Нажмите Download package и распакуйте архив.
5. Положите файл MicrosoftDefenderATPOffboardingLinuxServer_valid_until_YYYY-MM-DD.py в каталог roles/mde_remove/files/.
6. При необходимости поправьте mde_offboard_script_file в defaults/main.yml под точное имя скачанного файла.

Пример playbook

```yaml
---
- name: Удалить Microsoft Defender for Endpoint с offboarding
  hosts: all
  become: true
  roles:
    - mde_remove
```

Пояснения

· Порядок действий: сначала запускается offboarding-скрипт (он отключает устройство от портала), затем удаляется пакет. Это рекомендованный Microsoft порядок.
· Идемпотентность: если устройство уже отключено от портала, скрипт вернёт ненулевой код, но задача не упадёт — в failed_when учтены сообщения already offboarded и not installed.
· Отключение шага: если offboarding-скрипт не нужен (например, тенант уже недоступен), установите mde_run_offboard_script: false.
· Права: для запуска скрипта нужны права root — playbook уже использует become: true.

Если под «off border» вы имели в виду что-то другое (например, отключение tamper protection или сетевого фильтра), уточните — добавлю соответствующую задачу.