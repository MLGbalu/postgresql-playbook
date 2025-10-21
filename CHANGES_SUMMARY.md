# Сводка изменений: Рефакторинг PostgreSQL Playbook

## 📋 Обзор

Проект был полностью рефакторен согласно **Red Hat Ansible Best Practices**. Все изменения задокументированы с подробными комментариями в коде.

---

## ✅ Выполненные задачи (12/12)

### 1. ✅ Установка python3-psycopg2
**Файл:** `postgres-playbook.yml` (строки 13-17) и `site.yml` (строки 20-24)

**Что сделано:**
```yaml
- name: "Install psycopg2 for PostgreSQL modules"
  ansible.builtin.apt:
    name: python3-psycopg2
    state: present
    update_cache: yes
```

**Зачем:** 
- `psycopg2` - обязательная зависимость для всех PostgreSQL модулей Ansible
- Без него `postgresql_user`, `postgresql_query` и др. не работают
- Устанавливается перед любыми операциями с PostgreSQL

---

### 2. ✅ Структура директорий
**Создано:**
```
inventory/production/        # Окружение production
├── hosts.yml               # YAML inventory вместо INI
└── group_vars/             # Переменные по группам
    ├── all.yml
    ├── postgres_master.yml
    ├── postgres_slaves.yml
    └── all_vault.yml.example
```

**Зачем:**
- Разделение по окружениям (production/staging/dev)
- YAML более читаем чем INI
- group_vars автоматически загружаются для групп

---

### 3. ✅ Вынос переменных в group_vars

#### `inventory/production/group_vars/all.yml`
**Что хранит:**
- Общие переменные для всех хостов
- Версия PostgreSQL
- Пути к конфигам
- Базовые пользователи

**Комментарии:**
```yaml
# === PostgreSQL общие настройки ===
# Версия PostgreSQL (для Ubuntu 20.04 доступны 12, 13, 14)
postgresql_version: "12"

# BEST PRACTICE: Отключаем логирование паролей в production
postgres_users_no_log: true
```

#### `inventory/production/group_vars/postgres_master.yml`
**Что хранит:**
- Конфигурация мастера
- Настройки WAL
- pg_hba.conf entries
- Пользователь репликации

**Комментарии:**
```yaml
# === PostgreSQL конфигурация для Master ===
# BEST PRACTICE: Централизованное управление конфигурацией через переменные
postgresql_master_config:
  # Слушать на IP мастера, а не localhost
  - regexp: "#listen_addresses = 'localhost'"
    line: "listen_addresses = '{{ postgres_master_ip }}'"
```

#### `inventory/production/group_vars/postgres_slaves.yml`
**Что хранит:**
- Конфигурация слейвов
- Hot standby настройки
- Recovery конфигурация

**Зачем разделение:**
- Модульность: легко понять что относится к какой роли
- Переиспользование: можно использовать в других проектах
- Безопасность: sensitive данные в отдельном vault файле

---

### 4-6. ✅ Роль postgresql_replication

#### Структура роли
```
roles/postgresql_replication/
├── defaults/main.yml        # Переменные по умолчанию (низкий приоритет)
├── handlers/main.yml        # Handlers для перезапуска сервисов
├── tasks/
│   ├── main.yml            # Orchestration - когда что запускать
│   ├── master.yml          # Настройка master сервера
│   └── slave.yml           # Настройка slave сервера
└── templates/
    ├── pg_hba_replication.conf.j2  # Шаблон pg_hba
    └── recovery.conf.j2             # Шаблон recovery.conf
```

#### `roles/postgresql_replication/defaults/main.yml`
**Комментарии:**
```yaml
# BEST PRACTICE: Defaults - значения по умолчанию с низким приоритетом
# Могут быть переопределены в group_vars или host_vars

# Версия PostgreSQL по умолчанию
postgresql_version: "12"
```

**Зачем:** Defaults имеют самый низкий приоритет, их легко переопределить

#### `roles/postgresql_replication/handlers/main.yml`
**Комментарии:**
```yaml
# BEST PRACTICE: Handlers - запускаются только при изменениях и только один раз
# Это предотвращает лишние перезапуски сервиса

- name: restart postgresql
  ansible.builtin.service:
    name: "{{ postgresql_daemon }}"
    state: restarted
  # BEST PRACTICE: Таймаут для безопасного перезапуска
  async: 60
  poll: 5
```

**Зачем:**
- Handler запускается только если таск сделал изменения (`changed=true`)
- Если 5 тасков изменили конфиг, handler запустится только 1 раз в конце
- Экономит время и предотвращает множественные рестарты

#### `roles/postgresql_replication/tasks/main.yml`
**Комментарии:**
```yaml
# BEST PRACTICE: main.yml - точка входа в роль
# Использует include_tasks для модульности и читаемости

- name: Include master configuration tasks
  ansible.builtin.include_tasks: master.yml
  # BEST PRACTICE: when условие определяет когда запускать таски
  when: inventory_hostname in groups['postgres_master']
  # BEST PRACTICE: tags для выборочного запуска
  tags:
    - postgres
    - postgres_master
    - replication
```

**Зачем:**
- Логика разделена: main.yml только решает что запускать
- when условие: master таски только на master серверах
- tags: можно запустить `ansible-playbook site.yml --tags postgres_master`

#### `roles/postgresql_replication/tasks/master.yml`
**Ключевые комментарии:**
```yaml
- name: Configure postgresql.conf for master replication
  # BEST PRACTICE: FQCN (Fully Qualified Collection Name) для всех модулей
  ansible.builtin.lineinfile:
    path: "{{ postgresql_config_path }}/postgresql.conf"
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
    backup: true  # BEST PRACTICE: создаем backup перед изменением
  # BEST PRACTICE: loop вместо устаревшего with_items
  loop: "{{ postgresql_master_config }}"
  # BEST PRACTICE: notify вместо прямого вызова restart
  notify: restart postgresql
```

**Объяснение изменений:**
- `ansible.builtin.lineinfile` вместо `lineinfile` - FQCN предотвращает конфликты имен модулей
- `loop:` вместо `with_items:` - современный унифицированный синтаксис
- `backup: true` вместо `yes` - правильный boolean тип
- `notify:` вместо прямого рестарта - handler запустится один раз после всех изменений

```yaml
- name: Create replication user
  community.postgresql.postgresql_user:
    name: "{{ replication_user.name }}"
    password: "{{ replication_user.password }}"
    role_attr_flags: "{{ replication_user.role_attr_flags }}"
    encrypted: true
  become: true
  become_user: postgres
  # BEST PRACTICE: no_log скрывает пароли из логов
  no_log: "{{ postgres_users_no_log }}"
```

**Зачем no_log:**
- Предотвращает показ паролей в Ansible output
- `no_log: "{{ postgres_users_no_log }}"` - управляется переменной
- В dev можно выключить для отладки, в production всегда true

#### `roles/postgresql_replication/tasks/slave.yml`
**Ключевой блок - block/rescue/always:**
```yaml
- name: Configure slave with rollback capability
  # BEST PRACTICE: block/rescue/always для error handling
  block:
    - name: Create fresh data directory
      # ... создание директории
    
    - name: Replicate data from master using pg_basebackup
      ansible.builtin.shell: |
        pg_basebackup -w -h {{ postgres_master_ip }} -U repluser ...
      environment:
        PGPASSWORD: "{{ replica_password }}"
      no_log: "{{ postgres_users_no_log }}"

  # BEST PRACTICE: rescue блок для rollback при ошибках
  rescue:
    - name: Remove corrupted data directory
    - name: Restore backup
    - name: Fail with clear message

  # BEST PRACTICE: always блок выполняется всегда
  always:
    - name: Remove backup directory if exists
    - name: Start PostgreSQL on slave
```

**Что делает:**
1. **block:** Пытается настроить репликацию
2. **rescue:** Если ошибка - откатывает изменения (удаляет новые данные, восстанавливает backup)
3. **always:** Выполняется всегда (удаляет backup, запускает PostgreSQL)

**Зачем:** Безопасность! Если что-то пошло не так, сервер вернется в исходное состояние

#### Templates
**`templates/pg_hba_replication.conf.j2`:**
```jinja
# BEST PRACTICE: Jinja2 template для pg_hba.conf
# Шаблонизация позволяет генерировать конфигурацию динамически

# Локальная репликация
host replication repluser 127.0.0.1/32 md5

# Репликация с мастера
host replication repluser {{ postgres_master_ip }}/32 md5
```

**Зачем шаблоны:**
- Динамическая генерация конфигов
- Легко поддерживать
- Переменные подставляются автоматически

---

### 7. ✅ Обновление синтаксиса

**Было:**
```yaml
with_items:
  - item1
  - item2

systemd:
  name: postgresql

encrypted: yes
```

**Стало:**
```yaml
loop:
  - item1
  - item2

ansible.builtin.systemd:
  name: postgresql

encrypted: true
```

**Таблица изменений:**

| Устаревшее | Современное | Зачем |
|------------|-------------|-------|
| `with_items` | `loop` | Унифицированный синтаксис циклов |
| `systemd` | `ansible.builtin.systemd` | FQCN - предотвращение конфликтов |
| `yes/no` | `true/false` | Правильные boolean типы |
| `shell: mv` | `command: mv` | Безопаснее если shell не нужен |

---

### 8. ✅ Tags для управления

**Добавлены tags во всех тасках:**
```yaml
tags:
  - install          # Только установка
  - configure        # Только конфигурация
  - replication      # Только репликация
  - postgres_master  # Только master сервер
  - postgres_slave   # Только slave серверы
  - healthcheck      # Только проверки
```

**Примеры использования:**
```bash
# Только установка PostgreSQL
ansible-playbook site.yml --tags install

# Только настройка репликации
ansible-playbook site.yml --tags replication

# Только master
ansible-playbook site.yml --tags postgres_master

# Только healthcheck
ansible-playbook site.yml --tags healthcheck

# Несколько tags
ansible-playbook site.yml --tags "install,configure"

# Пропустить healthcheck
ansible-playbook site.yml --skip-tags healthcheck
```

**Зачем:**
- Экономия времени: не нужно запускать весь playbook
- Отладка: можно протестировать отдельные части
- Maintenance: обновить только конфиги без переустановки

---

### 9. ✅ Ansible Vault для паролей

**Создан example файл:**
`inventory/production/group_vars/all_vault.yml.example`

```yaml
# SECURITY: Эти пароли будут зашифрованы через ansible-vault
vault_postgres_password: "changeme_strong_password_123"
vault_replica_password: "changeme_replica_password_456"
```

**Использование в переменных:**
```yaml
# В all.yml
postgresql_users:
  - name: postgres
    password: "{{ vault_postgres_password }}"

# В postgres_master.yml
replica_password: "{{ vault_replica_password }}"
```

**Workflow:**
```bash
# 1. Создать файл
cp all_vault.yml.example all_vault.yml

# 2. Отредактировать пароли
nano all_vault.yml

# 3. Зашифровать
ansible-vault encrypt all_vault.yml

# 4. Создать password file
echo "my_vault_password" > .vault_pass
chmod 600 .vault_pass

# 5. Использовать
ansible-playbook site.yml --vault-password-file .vault_pass
```

**Безопасность:**
- Пароли зашифрованы AES256
- Можно безопасно коммитить в Git
- `.vault_pass` в `.gitignore`

---

### 10. ✅ Главный playbook site.yml

**Структура:**
```yaml
---
# Play 1: Установка PostgreSQL на все серверы
- name: Install and configure PostgreSQL on all servers
  hosts: all
  tasks:
    - Install psycopg2
    - Install PostgreSQL (geerlingguy role)
    - Start PostgreSQL

# Play 2: Настройка репликации
- name: Configure PostgreSQL replication
  hosts: postgres
  roles:
    - postgresql_replication

# Play 3: Health checks
- name: Run health checks
  hosts: postgres
  tasks:
    - Check version
    - Check replication status
```

**Комментарии:**
```yaml
- name: Install psycopg2 for PostgreSQL modules
  # BEST PRACTICE: FQCN для всех модулей
  ansible.builtin.apt:
    name: python3-psycopg2
    state: present
    update_cache: yes
  tags:
    - dependencies

# BEST PRACTICE: Используем внешнюю роль для базовой установки
- name: Setup PostgreSQL on Debian/Ubuntu
  ansible.builtin.include_role:
    name: geerlingguy.postgresql
    tasks_from: setup-Debian
```

**Зачем разделение на plays:**
- Логическая группировка задач
- Разные hosts для разных plays
- Легко понять что происходит

---

### 11. ✅ ansible.cfg с best practices

**Основные секции:**

```ini
[defaults]
# Новая структура inventory
inventory = ./inventory/production

# BEST PRACTICE: Факты кешируются для скорости
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600

# BEST PRACTICE: Более информативный вывод
stdout_callback = yaml

# BEST PRACTICE: Логирование
log_path = ./ansible.log

[ssh_connection]
# BEST PRACTICE: SSH performance tuning
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

**Что улучшено:**

| Настройка | Что делает |
|-----------|------------|
| `gathering = smart` | Собирает факты только когда нужно |
| `fact_caching` | Кеширует факты в JSON файлы |
| `stdout_callback = yaml` | Красивый YAML вывод вместо JSON |
| `log_path` | Все логи в файл |
| `pipelining = True` | Ускоряет SSH в 2-3 раза |
| `ControlMaster` | Переиспользует SSH соединения |

---

### 12. ✅ Полноценный README.md

**Секции:**
1. 📋 Описание проекта
2. 🏗️ Архитектура (дерево файлов)
3. 🚀 Быстрый старт (4 шага)
4. 🏷️ Таблица tags
5. 🔧 Таблица переменных
6. 📊 Проверка репликации
7. 🔒 Безопасность
8. 🐛 Troubleshooting
9. 📚 Ссылки на документацию
10. 📝 TODO

**Дополнительно создано:**
- `MIGRATION_GUIDE.md` - подробное руководство по миграции
- `CHANGES_SUMMARY.md` - этот файл с описанием всех изменений

---

## 📊 Статистика изменений

### Файлы

| Категория | Количество | Описание |
|-----------|------------|----------|
| Созданные файлы | 17 | Новая структура |
| Обновленные файлы | 3 | postgres-playbook.yml, ansible.cfg, .gitignore |
| Документация | 3 | README.md, MIGRATION_GUIDE.md, CHANGES_SUMMARY.md |
| Всего файлов | 20+ | |

### Строки кода

| Файл | Строки | Комментариев | % комментариев |
|------|--------|--------------|----------------|
| site.yml | 143 | ~40 | ~28% |
| master.yml | 71 | ~25 | ~35% |
| slave.yml | 100 | ~30 | ~30% |
| group_vars/ | ~200 | ~60 | ~30% |

### Улучшения

| Метрика | До | После | Улучшение |
|---------|------|-------|-----------|
| Модульность | Низкая | Высокая | ✅ Роль можно переиспользовать |
| Безопасность | Пароли открыто | Vault | ✅ AES256 encryption |
| Читаемость | Средняя | Высокая | ✅ 30% комментариев |
| Тестируемость | Сложно | Легко | ✅ Можно тестировать по модулям |
| Документация | Нет | 3 файла | ✅ >500 строк документации |

---

## 🎯 Что теперь можно делать

### Выборочный запуск
```bash
# Только установка
ansible-playbook site.yml --tags install

# Только репликация
ansible-playbook site.yml --tags replication

# Только master
ansible-playbook site.yml --tags postgres_master
```

### Разные окружения
```bash
# Production
ansible-playbook -i inventory/production site.yml

# Staging (создайте inventory/staging)
ansible-playbook -i inventory/staging site.yml
```

### Безопасность
```bash
# С vault
ansible-playbook site.yml --ask-vault-pass

# Dry-run
ansible-playbook site.yml --check --diff
```

### Отладка
```bash
# Verbose mode
ansible-playbook site.yml -vvv

# Только синтаксис
ansible-playbook site.yml --syntax-check

# Список тасков
ansible-playbook site.yml --list-tasks
```

---

## 🔍 Ключевые Best Practices примененные

1. ✅ **FQCN** - Fully Qualified Collection Names для всех модулей
2. ✅ **Идемпотентность** - можно запускать многократно безопасно
3. ✅ **Модульность** - роли переиспользуемы
4. ✅ **DRY** - Don't Repeat Yourself, переменные в одном месте
5. ✅ **Handlers** - вместо прямых рестартов
6. ✅ **Templates** - для динамических конфигов
7. ✅ **Tags** - для гранулярного контроля
8. ✅ **Vault** - для sensitive данных
9. ✅ **block/rescue/always** - для error handling
10. ✅ **Комментарии** - объяснение каждого блока
11. ✅ **Документация** - подробная инструкция
12. ✅ **Backup** - перед каждым изменением конфигов

---

## 📝 Следующие шаги

1. Протестировать на dev окружении
2. Создать vault файл с реальными паролями
3. Запустить с --check для проверки
4. Применить на production
5. Настроить мониторинг
6. Добавить backup скрипты

---

## 👤 Кто создал

Рефакторинг выполнен согласно:
- [Ansible Best Practices (Red Hat)](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [Ansible Roles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html)
- [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)

Все файлы содержат подробные комментарии объясняющие **что** сделано и **зачем**.
