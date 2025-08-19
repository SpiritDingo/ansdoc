Вот Ansible роль для выполнения указанных задач на Ubuntu 22.04, Ubuntu 24.04 и Oracle Linux. Роль будет проверять статус Defender (предполагаю, что речь о Microsoft Defender for Endpoint), его версию, обновлять и снова проверять версию.

```yaml
---
# roles/defender_management/tasks/main.yml
- name: Gather OS facts
  ansible.builtin.setup:
    gather_subset:
      - distribution

- name: Check if Defender service is running (Ubuntu)
  ansible.builtin.command: mdatp health
  register: defender_status
  ignore_errors: yes
  when: ansible_distribution in ['Ubuntu']

- name: Check if Defender service is running (Oracle Linux)
  ansible.builtin.command: /opt/microsoft/mdatp/sbin/mdatp health
  register: defender_status
  ignore_errors: yes
  when: ansible_distribution in ['OracleLinux']

- name: Display Defender service status
  ansible.builtin.debug:
    var: defender_status.stdout
  when: defender_status is defined and not defender_status.failed

- name: Get Defender version (Ubuntu)
  ansible.builtin.command: mdatp --version
  register: defender_version
  ignore_errors: yes
  when: ansible_distribution in ['Ubuntu']

- name: Get Defender version (Oracle Linux)
  ansible.builtin.command: /opt/microsoft/mdatp/sbin/mdatp --version
  register: defender_version
  ignore_errors: yes
  when: ansible_distribution in ['OracleLinux']

- name: Display Defender version
  ansible.builtin.debug:
    var: defender_version.stdout
  when: defender_version is defined and not defender_version.failed

- name: Update Defender (Ubuntu)
  ansible.builtin.command: mdatp --update definitions
  when: ansible_distribution in ['Ubuntu']

- name: Update Defender (Oracle Linux)
  ansible.builtin.command: /opt/microsoft/mdatp/sbin/mdatp --update definitions
  when: ansible_distribution in ['OracleLinux']

- name: Get Defender version after update (Ubuntu)
  ansible.builtin.command: mdatp --version
  register: defender_version_after_update
  ignore_errors: yes
  when: ansible_distribution in ['Ubuntu']

- name: Get Defender version after update (Oracle Linux)
  ansible.builtin.command: /opt/microsoft/mdatp/sbin/mdatp --version
  register: defender_version_after_update
  ignore_errors: yes
  when: ansible_distribution in ['OracleLinux']

- name: Display Defender version after update
  ansible.builtin.debug:
    var: defender_version_after_update.stdout
  when: defender_version_after_update is defined and not defender_version_after_update.failed
```

Пример playbook для использования этой роли:

```yaml
# defender_management.yml
- hosts: all
  become: yes
  roles:
    - defender_management
```

Примечания:

1. Роль предполагает, что Microsoft Defender for Endpoint уже установлен на целевых системах.
2. Для Ubuntu используется команда mdatp, для Oracle Linux - полный путь к бинарнику.
3. Роль обрабатывает возможные ошибки, если Defender не установлен.
4. Для работы может потребоваться sudo, поэтому используется become: yes.

Если вам нужно добавить установку Defender или другие специфические настройки, роль можно расширить. Также вы можете уточнить, если имелся в виду не Microsoft Defender, а другой антивирусный продукт.