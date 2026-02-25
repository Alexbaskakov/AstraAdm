# Лабораторная работа №3
## Разработка и выполнение первых Ansible Playbook

## Цель лабораторной работы
Научиться описывать желаемое состояние устройств в виде кода (Infrastructure as Code, IaC) с помощью Ansible Playbook, освоить базовую структуру Playbook, принципы идемпотентности и безопасную проверку конфигураций перед применением.

## Задачи лабораторной работы
1. Создать простой Ansible Playbook для массовой настройки banner login.
2. Разработать Playbook для базовой конфигурации Linux-хостов.
3. Разработать Playbook для базовой конфигурации MikroTik RouterOS.
4. Использовать режим dry-run (`--check`) для проверки Playbook.
5. Зафиксировать созданные Playbook в Git-репозитории.

## Ожидаемый результат
После выполнения лабораторной работы студент:
- понимает структуру Ansible Playbook;
- умеет писать простые Playbook на YAML;
- осознаёт принцип идемпотентности;
- умеет безопасно проверять изменения перед применением;
- хранит Playbook в системе контроля версий.

---

# Теоретическая часть

**Ansible Playbook** — это YAML-файл, в котором описывается:
- на каких устройствах выполнять действия;
- какие действия нужно выполнить;
- в каком порядке.

Playbook отвечает на вопрос:  
**«Какое состояние должно быть у системы»**, а не **«какие команды выполнить»**.

### Почему Playbook лучше, чем команды вручную
- можно запускать сколько угодно раз;
- повторный запуск не ломает систему;
- конфигурация хранится в Git;
- одинаковая настройка на всех узлах.

### Идемпотентность
Идемпотентность означает:
> если система уже настроена правильно, Ansible ничего не меняет.

---

# Практическая часть

### Что должно быть готово перед началом

1. Сервер управления `Mgmt`.
2. Установленный Ansible.
3. Рабочий SSH-доступ к Linux-узлам.
4. Файл `hosts.ini` с группами `linux` и (опционально) `mikrotik`.

Минимальный пример `hosts.ini`:
```
[linux]
r1 ansible_host=10.0.12.1
r2 ansible_host=10.0.12.2
```
```
[mikrotik]
mt1 ansible_host=10.0.99.2
```
- Проверка связи:

```
ansible linux -i hosts.ini -m ping
```
- Если нет pong — дальше не идём.

# 1. Подготовка рабочей директории
```
cd ~/ansible
```
- Проверка:
```
ls
```

# 2. Playbook №1 — Массовая настройка banner login
### 2.1 Что такое banner login

- Banner login — это текст, который отображается перед входом в систему
(файл /etc/issue).

### 2.2 Создание Playbook
```
cat > banner.yml <<EOF
---
- name: Configure login banner
  hosts: linux
  become: true

  tasks:
    - name: Set banner in /etc/issue
      copy:
        dest: /etc/issue
        content: |
          ****************************************
          *  Authorized access only!            *
          *  All actions are monitored.         *
          ****************************************
EOF
```

### 2.3 Dry-run
```
ansible-playbook -i hosts.ini banner.yml --check
```
### 2.4 Реальный запуск
```
ansible-playbook -i hosts.ini banner.yml
```
### 2.5 Проверка результата
```
ansible linux -i hosts.ini -m command -a "cat /etc/issue" --become
```

# 3. Playbook №2 — Базовая конфигурация Linux-хостов
### 3.1 Что будем настраивать

- hostname
- пользователя admin
- SSH-ключи
- базовые пакеты

### 3.2 Создание Playbook

```
cat > linux_base.yml <<EOF
---
- name: Base Linux configuration
  hosts: linux
  become: true

  tasks:
    - name: Set hostname
      hostname:
        name: "{{ inventory_hostname }}"

    - name: Create admin user
      user:
        name: admin
        groups: sudo
        shell: /bin/bash
        state: present

    - name: Create .ssh directory
      file:
        path: /home/admin/.ssh
        state: directory
        owner: admin
        group: admin
        mode: 0700

    - name: Install base packages
      apt:
        name:
          - vim
          - curl
          - git
        state: present
        update_cache: true
EOF
```

### 3.3 Dry-run

```
ansible-playbook -i hosts.ini linux_base.yml --check
```
### 3.4 Применение
```
ansible-playbook -i hosts.ini linux_base.yml
```

### 3.5 Проверка
```
ansible linux -i hosts.ini -m command -a "hostname"
ansible linux -i hosts.ini -m command -a "id admin"
```

# 4. Playbook №3 — Базовая конфигурация MikroTik
⚠️ Этот Playbook не обязательно запускать, он нужен для понимания структуры.

### 4.1 Создание Playbook
```
cat > mikrotik_base.yml <<EOF
---
- name: Base MikroTik configuration
  hosts: mikrotik
  gather_facts: false

  tasks:
    - name: Set router identity
      community.routeros.command:
        commands:
          - /system identity set name=LAB-MT

    - name: Create admin user
      community.routeros.command:
        commands:
          - /user add name=admin group=full password=StrongPassword
EOF
```

### 4.2 Dry-run невозможен
- RouterOS не поддерживает полноценный dry-run.
- Поэтому сначала всегда проверяем шаблоны и команды вручную.

# 5. Принцип идемпотентности

Повторный запуск:

```
ansible-playbook -i hosts.ini linux_base.yml
```

Ожидаемо:

- **ok** вместо **changed**

- система не ломается

- ничего лишнего не происходит

# 6. Коммит Playbook в Git

```
git status
git add banner.yml linux_base.yml mikrotik_base.yml
git commit -m "Lab3: first Ansible playbooks (banner, linux base, mikrotik base)"
```

# Контрольные вопросы

- Что такое Ansible Playbook?

- В чём отличие Playbook от ad-hoc команд?

- Что означает идемпотентность?

- Зачем использовать --check?

- Почему Playbook удобно хранить в Git?

- Почему IaC снижает количество ошибок?

# Заключение
В ходе лабораторной работы были разработаны и выполнены первые Ansible Playbook. Освоены базовая структура Playbook, принципы идемпотентности и безопасной проверки конфигураций. Полученные навыки являются основой для дальнейшего изучения автоматизации и управления инфраструктурой.
