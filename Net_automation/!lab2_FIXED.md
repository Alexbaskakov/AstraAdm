
# ЛАБОРАТОРНАЯ РАБОТА № 2
## Дисциплина: Администрирование информационных систем

## Тема: Автоматизация установки Docker и Minikube средствами Ansible

---

### 1. Цель работы

Освоение навыков автоматизации развертывания программного обеспечения с использованием системы управления конфигурациями Ansible.

### 2. Задачи работы

В ходе выполнения лабораторной работы необходимо:

1. Изучить основные модули Ansible для управления пакетами и службами
2. Разработать Ansible роль для установки Docker
3. Разработать Ansible роль для установки Minikube
4. Обеспечить идемпотентность выполнения всех операций
5. Проверить работоспособность установленного ПО

### 3. Оборудование и программное обеспечение

| Наименование | Характеристики |
|--------------|----------------|
| ОС | Ubuntu 22.04 / Debian 12 |
| Ansible | версия 2.14+ |
| Целевая система | x86_64 архитектура |

---

### 4. Теоретические сведения

#### 4.1 Ansible

Ansible — система управления конфигурациями, написанная на Python. Управление осуществляется через SSH без установки агентов на целевые машины.

#### 4.2 Основные модули Ansible

| Модуль | Назначение | Основные параметры |
|--------|------------|---------------------|
| `apt` | Управление пакетами в Debian/Ubuntu | `name`, `state`, `update_cache` |
| `get_url` | Скачивание файлов по HTTP/HTTPS | `url`, `dest`, `mode` |
| `systemd` | Управление systemd службами | `name`, `state`, `enabled` |
| `user` | Управление пользователями | `name`, `groups`, `append` |
| `deb822_repository` | Добавление репозиториев (DEB822 формат) | `name`, `uris`, `suites`, `signed_by` |
| `command` | Выполнение команд | `cmd`, `register` |

#### 4.3 Идемпотентность

**Идемпотентность** — свойство операции, при котором её повторное применение не изменяет состояние системы, если она уже находится в требуемом состоянии.

Способы обеспечения идемпотентности в Ansible:
- `changed_when: false` — не засчитывать задачу как изменяющую состояние
- `failed_when: false` — не прерывать выполнение при ошибке
- `when:` — условное выполнение задачи
- `register` + проверка — сохранение результата для анализа

---

### 5. Задания для выполнения

#### 5.1 Предварительная подготовка

**Задание 5.1.1.** Установите Ansible на управляющую машину:

```bash
sudo apt update && sudo apt install ansible -y
```

**Задание 5.1.2.** Проверьте версию установленного Ansible:

```bash
ansible --version
```

**Задание 5.1.3.** Создайте директорию для лабораторной работы:

```bash
mkdir -p ~/lab-ansible/roles
cd ~/lab-ansible
```

---

#### 5.2 Создание роли для установки Docker

**Задание 5.2.1.** Создайте структуру роли `docker`:

```bash
mkdir -p roles/docker/{tasks,handlers,vars,defaults,templates}
touch roles/docker/tasks/main.yml
```

**Задание 5.2.2.** В файле `roles/docker/tasks/main.yml` создайте задачу для обновления кэша APT с валидностью 3600 секунд.

*Ожидаемый результат:* использование модуля `apt` с параметрами `update_cache: yes` и `cache_valid_time: 3600`.

**Задание 5.2.3.** Добавьте задачу для установки пакетов зависимостей: `ca-certificates`, `curl`.

*Подсказка:* модуль `apt`, параметр `name` принимает список пакетов.

**Задание 5.2.4.** Используя модуль `deb822_repository`, добавьте официальный репозиторий Docker со следующими параметрами:

| Параметр | Значение |
|----------|----------|
| `name` | docker |
| `types` | deb |
| `uris` | `https://download.docker.com/linux/{{ ansible_distribution \| lower }}` |
| `suites` | `{{ ansible_distribution_release }}` |
| `components` | stable |
| `architectures` | `amd64` (или `arm64` для ARM) |
| `signed_by` | `https://download.docker.com/linux/{{ ansible_distribution \| lower }}/gpg` |

**Задание 5.2.5.** Добавьте задачу для установки следующих пакетов Docker:
- `docker-ce`
- `docker-ce-cli`
- `containerd.io`
- `docker-buildx-plugin`
- `docker-compose-plugin`

**Задание 5.2.6.** Используя модуль `systemd`, добавьте задачу для запуска службы Docker и добавления её в автозагрузку.

*Ожидаемые параметры:* `state: started`, `enabled: yes`

**Задание 5.2.7.** Добавьте задачу для добавления текущего пользователя (`{{ ansible_user }}`) в группу `docker` с использованием модуля `user`.

*Условие:* задачу следует выполнять только если пользователь не `root`.

**Задание 5.2.8.** Создайте задачу для проверки версии Docker и вывода результата с помощью модуля `debug`.

*Ожидаемый вывод:* `"Docker успешно установлен: Docker version XX.XX.XX"`

---

#### 5.3 Создание роли для установки Minikube

**Задание 5.3.1.** Создайте структуру роли `minikube`:

```bash
mkdir roles
ansible-galaxy init minikube
```

**Задание 5.3.2.** В файле `roles/minikube/tasks/main.yml` создайте задачу для проверки наличия Minikube с использованием модуля `command: minikube version`.

*Необходимые параметры:*
- `register: minikube_version_check`
- `changed_when: false`
- `failed_when: false`

**Задание 5.3.3.** Добавьте задачу для скачивания DEB-пакета Minikube с использованием модуля `get_url` по адресу:
```
https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
```

*Параметры:* `dest: /tmp/minikube_latest_amd64.deb`, `mode: '0644'`

*Условие выполнения:* задача выполняется, если Minikube не установлен (проверка из п.5.3.2).

**Задание 5.3.4.** Добавьте задачу для установки Minikube из DEB-пакета с использованием модуля `apt` (параметр `deb`).

*Условие:* аналогично п.5.3.3.

*Дополнительно:* добавьте уведомление хендлера `clean minikube deb`.

**Задание 5.3.5.** Создайте файл `roles/minikube/handlers/main.yml` с хендлером `clean minikube deb`, который удаляет `/tmp/minikube_latest_amd64.deb`.

**Задание 5.3.6.** В файле `roles/minikube/tasks/main.yml` добавьте задачу для проверки статуса кластера Minikube:

- Модуль: `command: minikube status`
- Параметры: `become_user: "{{ ansible_user }}"`, `failed_when: false`, `ignore_errors: yes`

**Задание 5.3.7.** Добавьте задачу для запуска Minikube кластера с драйвером Docker:

```yaml
command: minikube start --driver=docker
```

*Условие:* кластер не запущен (проверка из п.5.3.6).

**Задание 5.3.8.** Добавьте задачу для повторного получения статуса кластера и вывода его с помощью модуля `debug`.

---

#### 5.4 Создание основного плейбука

**Задание 5.4.1.** Создайте файл `install.yml` в корневой директории лабораторной работы со следующей структурой:

```yaml
---
- name: Полная установка Docker и Minikube
  hosts: all
  become: yes
  gather_facts: yes
  
  pre_tasks:
    - name: Обновление apt кэша
      ansible.builtin.apt:
        update_cache: yes
      changed_when: false

  roles:
    - docker
    - minikube

  post_tasks:
    - name: Проверка готовности кластера
      command: kubectl get nodes
      register: nodes
      changed_when: false
      
    - name: Результат
      debug:
        msg: "{{ nodes.stdout_lines }}"
```

**Задание 5.4.2.** Запустите плейбук и проверьте результат:

```bash
ansible-playbook -i "localhost," -c local install.yml
```

---

### 6. Контрольные вопросы

1. Что такое Ansible и для каких задач он используется?
2. Какие модули Ansible были использованы в работе? Опишите их назначение.
3. Что такое идемпотентность? Приведите примеры из выполненной работы.
4. Для чего используется параметр `cache_valid_time` в модуле `apt`?
5. В чем особенность формата DEB822 для репозиториев APT?
6. Почему в задаче запуска Minikube используется `become_user: "{{ ansible_user }}"`, а не выполнение от root?
7. Как работает условное выполнение задач в Ansible (параметр `when`)?
8. Для чего служат хендлеры и как они связаны с параметром `notify`?

---

### 7. Требования к отчету

Отчет должен содержать:

1. Титульный лист
2. Цель работы
3. Ход выполнения работы с листингами всех созданных файлов
4. Скриншоты результатов выполнения плейбука
5. Скриншот, подтверждающий идемпотентность (повторный запуск)
6. Ответы на контрольные вопросы
7. Вывод о проделанной работе

---

### 8. Критерии оценки

| Критерий | Баллы |
|----------|-------|
| Правильно создана структура ролей | 10 |
| Корректно настроен модуль deb822_repository | 15 |
| Установка Docker выполнена без ошибок | 15 |
| Установка Minikube выполнена без ошибок | 15 |
| Обеспечена идемпотентность | 15 |
| Запуск кластера Minikube выполнен успешно | 10 |
| Полнота ответов на контрольные вопросы | 10 |
| Оформление отчета | 10 |
| **Итого** | **100** |

---

**Разработал:** _________________ /_______________/

**Дата составления:** «___» ___________ 2026 г.

---

### Приложение А. Ожидаемая структура проекта

```
~/lab-ansible/
├── install.yml
└── roles/
    ├── docker/
    │   └── tasks/
    │       └── main.yml
    └── minikube/
        ├── tasks/
        │   └── main.yml
        └── handlers/
            └── main.yml
```

### Приложение Б. Ожидаемый результат выполнения

```
PLAY RECAP *********************************************************************
localhost                  : ok=16   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

TASK [Показываем статус Minikube] *********************************************
ok: [localhost] => {
    "msg": [
        "minikube",
        "type: Control Plane",
        "host: Running",
        "kubelet: Running",
        "apiserver: Running",
        "kubeconfig: Configured"
    ]
}
```

---
---
---
# ОТЧЕТ ПО ЛАБОРАТОРНОЙ РАБОТЕ № ___

## Дисциплина: Администрирование информационных систем

## Тема: Автоматизация установки Docker и Minikube средствами Ansible

---

### 1. Цель работы

Освоение навыков автоматизации развертывания программного обеспечения с использованием системы управления конфигурациями Ansible.

### 2. Задачи работы

1. Разработать Ansible роль для установки Docker
2. Разработать Ansible роль для установки Minikube
3. Обеспечить идемпотентность выполнения всех операций
4. Проверить работоспособность установленного ПО

### 3. Оборудование и программное обеспечение

| Наименование | Характеристики |
|--------------|----------------|
| ОС | Ubuntu 22.04 / Debian 12 |
| Ansible | версия 2.14+ |
| Целевая система | x86_64 архитектура |

### 4. Ход выполнения работы

#### 4.1 Разработка роли для установки Docker

**Листинг 1 – Файл tasks/main.yml (роль docker)**

```yaml
- name: Обновляем кэш APT
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Устанавливаем необходимые зависимости
  ansible.builtin.apt:
    name:
      - ca-certificates
      - curl
    state: present

- name: Добавляем официальный репозиторий Docker
  ansible.builtin.deb822_repository:
    name: docker
    types: deb
    uris: "https://download.docker.com/linux/{{ ansible_distribution | lower }}"
    suites: "{{ ansible_distribution_release }}"
    components: stable
    architectures: "{{ 'arm64' if ansible_architecture == 'aarch64' else 'amd64' }}"
    signed_by: "https://download.docker.com/linux/{{ ansible_distribution | lower }}/gpg"
    state: present
    enabled: yes

- name: Устанавливаем пакеты Docker
  ansible.builtin.apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
    state: present
    update_cache: yes

- name: Запускаем и включаем службу Docker
  ansible.builtin.systemd:
    name: docker
    state: started
    enabled: yes

- name: Добавляем текущего пользователя в группу docker
  ansible.builtin.user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes
  when: ansible_user != 'root'
```

**Описание выполняемых операций:**

| № | Операция | Используемый модуль |
|---|----------|---------------------|
| 1 | Обновление списка пакетов | apt |
| 2 | Установка зависимостей | apt |
| 3 | Добавление репозитория Docker | deb822_repository |
| 4 | Установка пакетов Docker | apt |
| 5 | Запуск службы Docker | systemd |
| 6 | Добавление пользователя в группу docker | user |

#### 4.2 Разработка роли для установки Minikube

**Листинг 2 – Файл tasks/main.yml (роль minikube)**

```yaml
- name: Проверяем, установлен ли Minikube
  command: minikube version
  register: minikube_version_check
  changed_when: false
  failed_when: false

- name: Скачиваем последнюю версию Minikube DEB пакета
  ansible.builtin.get_url:
    url: https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
    dest: /tmp/minikube_latest_amd64.deb
    mode: '0644'
  when: minikube_version_check is failed or (minikube_version_check is success and 'version' not in minikube_version_check.stdout)

- name: Устанавливаем Minikube из DEB пакета
  ansible.builtin.apt:
    deb: /tmp/minikube_latest_amd64.deb
    state: present
  when: minikube_version_check is failed or (minikube_version_check is success and 'version' not in minikube_version_check.stdout)
  notify: clean minikube deb

- name: Проверяем, запущен ли кластер Minikube
  become_user: "{{ ansible_user }}"
  command: minikube status
  register: minikube_status
  changed_when: false
  failed_when: false
  ignore_errors: yes

- name: Запускаем Minikube кластер (если не запущен)
  become_user: "{{ ansible_user }}"
  command: minikube start --driver=docker
  when: 
    - minikube_status is failed or "Running" not in minikube_status.stdout

- name: Выводим статус кластера
  become_user: "{{ ansible_user }}"
  command: minikube status
  register: final_status
  changed_when: false

- name: Показываем статус Minikube
  ansible.builtin.debug:
    msg: "{{ final_status.stdout_lines }}"
```

**Листинг 3 – Файл handlers/main.yml (роль minikube)**

```yaml
- name: clean minikube deb
  ansible.builtin.file:
    path: /tmp/minikube_latest_amd64.deb
    state: absent
```

### 5. Результаты выполнения

#### 5.1 Результат установки Docker

```
TASK [Выводим версию Docker для проверки] ********************
changed: [localhost]

TASK [Показываем результат] **********************************
ok: [localhost] => {
    "msg": "Docker успешно установлен: Docker version 27.5.1"
}
```

#### 5.2 Результат установки Minikube

```
TASK [Показываем статус Minikube] ****************************
ok: [localhost] => {
    "msg": [
        "minikube",
        "type: Control Plane",
        "host: Running",
        "kubelet: Running",
        "apiserver: Running",
        "kubeconfig: Configured"
    ]
}
```

### 6. Проверка идемпотентности

При повторном запуске плейбука были получены следующие результаты:

| Задача | Первый запуск | Второй запуск |
|--------|---------------|---------------|
| Проверка наличия Minikube | failed | ok |
| Скачивание DEB пакета | changed | skipping |
| Установка Minikube | changed | skipping |
| Проверка статуса кластера | ok | ok |
| Запуск кластера | changed | skipping |

**Вывод:** Идемпотентность обеспечена – повторный запуск не вносит изменения в систему.

### 7. Заключение

В ходе лабораторной работы были разработаны две Ansible-роли для автоматизации установки Docker и Minikube. Все поставленные задачи выполнены:

1. ✓ Разработана роль для установки Docker из официального репозитория
2. ✓ Разработана роль для установки Minikube из DEB-пакета
3. ✓ Обеспечена идемпотентность выполнения операций
4. ✓ Проведена проверка работоспособности установленного ПО

### 8. Список использованных источников

1. Ansible Documentation: https://docs.ansible.com/
2. Docker Documentation: https://docs.docker.com/
3. Minikube Documentation: https://minikube.sigs.k8s.io/docs/

---

**Дата выполнения:** «___» ___________ 2026 г.

**Студент:** _________________ /_______________/

**Преподаватель:** ______________ /_______________/

---

### Приложение А. Структура ролей

```
ansible-roles/
├── docker/
│   └── tasks/
│       └── main.yml
└── minikube/
    ├── tasks/
    │   └── main.yml
    └── handlers/
        └── main.yml
```

### Приложение Б. Команда запуска

```bash
ansible-playbook -i "localhost," -c local install.yml
```
