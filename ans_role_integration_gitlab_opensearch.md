# Роль Ansible для интеграции GitLab с OpenSearch

Вот пример роли Ansible для настройки интеграции GitLab с OpenSearch (форк Elasticsearch):

## Структура роли

```
roles/gitlab_opensearch_integration/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── gitlab_opensearch.json.j2
└── README.md
```

## defaults/main.yml

```yaml
---
# Адрес сервера OpenSearch
opensearch_host: "https://opensearch.example.com:9200"

# Учетные данные для подключения к OpenSearch
opensearch_username: "admin"
opensearch_password: "securepassword"

# Настройки интеграции GitLab
gitlab_external_url: "https://gitlab.example.com"
gitlab_admin_token: "gitlab_admin_token_here"

# Настройки индексации
gitlab_opensearch_index_name: "gitlab"
gitlab_opensearch_index_schedule: "0 * * * *"  # каждый час
```

## tasks/main.yml

```yaml
---
- name: Установка зависимостей (curl, jq)
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - curl
    - jq
  when: ansible_os_family == 'Debian'

- name: Проверка подключения к OpenSearch
  uri:
    url: "{{ opensearch_host }}"
    method: GET
    user: "{{ opensearch_username }}"
    password: "{{ opensearch_password }}"
    validate_certs: no  # для тестов, в production используйте корректные сертификаты
    status_code: 200
  register: opensearch_check
  retries: 3
  delay: 10

- name: Создание шаблона индекса в OpenSearch
  uri:
    url: "{{ opensearch_host }}/_index_template/gitlab_template"
    method: PUT
    body: "{{ lookup('template', 'gitlab_opensearch.json.j2') }}"
    body_format: json
    user: "{{ opensearch_username }}"
    password: "{{ opensearch_password }}"
    validate_certs: no
    status_code: 200
  when: opensearch_check.status == 200

- name: Настройка интеграции в GitLab
  uri:
    url: "{{ gitlab_external_url }}/api/v4/application/settings"
    method: PUT
    headers:
      PRIVATE-TOKEN: "{{ gitlab_admin_token }}"
    body:
      elasticsearch_indexing: true
      elasticsearch_search: true
      elasticsearch_url: "{{ opensearch_host }}"
      elasticsearch_username: "{{ opensearch_username }}"
      elasticsearch_password: "{{ opensearch_password }}"
      elasticsearch_index_name: "{{ gitlab_opensearch_index_name }}"
      elasticsearch_aws: false
      elasticsearch_reindexing: false
    body_format: json
    validate_certs: "{{ gitlab_external_url.startswith('https') }}"
    status_code: 200
```

## templates/gitlab_opensearch.json.j2

```json
{
  "index_patterns": ["{{ gitlab_opensearch_index_name }}-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index": {
        "refresh_interval": "30s"
      }
    },
    "mappings": {
      "properties": {
        "type": {
          "type": "keyword"
        },
        "id": {
          "type": "integer"
        },
        "title": {
          "type": "text",
          "analyzer": "standard"
        },
        "description": {
          "type": "text",
          "analyzer": "standard"
        },
        "created_at": {
          "type": "date"
        },
        "updated_at": {
          "type": "date"
        },
        "project_id": {
          "type": "integer"
        },
        "author_id": {
          "type": "integer"
        },
        "visibility_level": {
          "type": "integer"
        }
      }
    }
  }
}
```

## Использование роли

1. Добавьте роль в ваш playbook:

```yaml
- hosts: gitlab_server
  roles:
    - gitlab_opensearch_integration
```

2. Переопределите переменные в вашем inventory или playbook:

```yaml
vars:
  opensearch_host: "https://opensearch.prod.example.com:9200"
  opensearch_username: "gitlab_user"
  opensearch_password: "{{ vault_opensearch_password }}"
  gitlab_external_url: "https://gitlab.prod.example.com"
  gitlab_admin_token: "{{ vault_gitlab_admin_token }}"
```

## Дополнительные шаги

После применения роли вам может потребоваться:

1. Запустить первоначальную индексацию данных через API GitLab:
```bash
curl --request POST --header "PRIVATE-TOKEN: <your_admin_token>" "https://gitlab.example.com/api/v4/elasticsearch/index"
```

2. Настроить расписание для периодической