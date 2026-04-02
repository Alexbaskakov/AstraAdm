# Лабораторная работа №5
## Ролевая модель (Roles) в Ansible и организация кода

## Цель лабораторной работы
Научиться структурировать проекты автоматизации с помощью Ansible Roles, разделять конфигурацию на логические части и переиспользовать код для управления различными типами устройств.

## Задачи лабораторной работы
1. Создать роль `common` для базовой настройки всех устройств.
2. Создать роль `router` для настройки IPv4-маршрутизации и OSPF на Linux-хостах с использованием FRR.
3. Создать роль `mikrotik` для настройки DHCP-сервера и firewall на устройствах MikroTik.
4. Создать основной Playbook, который применяет роли к нужным группам устройств.
5. Перенести (рефакторить) ранее написанные задачи в роли.
6. Закрепить понимание структуры ролей и переиспользования кода.

## Ожидаемый результат
После выполнения лабораторной работы студент:
- понимает, зачем нужны роли в Ansible;
- умеет создавать и использовать роли;
- понимает структуру каталогов роли;
- умеет применять разные роли к разным группам устройств;
- умеет поддерживать порядок и масштабируемость Ansible-проекта.

---

# Теоретическая часть

Когда задач становится много, один большой Playbook становится неудобным:
- трудно читать;
- трудно изменять;
- трудно переиспользовать.

**Ansible Role** — это логически завершённый набор:
- задач,
- шаблонов,
- переменных,
- обработчиков (handlers),

который отвечает **за одну функцию**.

Пример:
- роль `common` — базовые настройки для всех устройств;
- роль `router` — всё, что связано с маршрутизацией;
- роль `mikrotik` — всё, что относится к RouterOS.

Роли позволяют:
- наводить порядок;
- переиспользовать код;
- применять одинаковые настройки на многих узлах;
- масштабировать проект.

---

# Практическая часть

## Что должно быть готово перед началом
1. Сервер управления `Mgmt`.
2. Установленный Ansible.
3. Рабочий `hosts.ini` с группами `linux` и `mikrotik`.
4. Базовые Playbook из ЛР №3 и шаблоны из ЛР №4.

---

# 1. Подготовка структуры проекта

Перейти в каталог Ansible:

```
cd ~/ansible
```
Создать каталог для ролей:

```
mkdir roles
cd roles
```
# 2. Создание роли common
Роль common применяется ко всем устройствам
(пользователи, время, базовые пакеты).

### 2.1 Создание структуры роли
```
ansible-galaxy init common
```
Проверка структуры:

```
tree common
```
Должны быть каталоги:

- tasks

- handlers

- templates

- defaults

- vars

### 2.2 Настройка задач роли common
Открываем файл задач:

```
nano common/tasks/main.yml
```
Содержимое:

```
---
- name: Install base packages
  apt:
    name:
      - vim
      - curl
      - git
      - chrony
    state: present
    update_cache: true
  become: true

- name: Ensure chrony is running
  service:
    name: chrony
    state: started
    enabled: yes
  become: true

- name: Create admin user
  user:
    name: admin
    groups: sudo
    shell: /bin/bash
    state: present
  become: true
  ```
# 3. Создание роли router
Роль применяется только к Linux-маршрутизаторам.

### 3.1 Создание роли
```
ansible-galaxy init router
```
### 3.2 Установка и настройка FRR (OSPF)
Открываем задачи:

```
nano router/tasks/main.yml
```
Содержимое:

```
---
- name: Install FRR routing suite
  apt:
    name: frr
    state: present
    update_cache: true
  become: true

- name: Enable OSPF daemon
  lineinfile:
    path: /etc/frr/daemons
    regexp: '^ospfd='
    line: 'ospfd=yes'
  become: true
  notify: restart frr

- name: Deploy FRR configuration
  copy:
    dest: /etc/frr/frr.conf
    content: |
      frr version 8
      service integrated-vtysh-config
      router ospf
        network 10.0.0.0/8 area 0
      !
  become: true
  notify: restart frr
  ```
### 3.3 Handler для FRR
```
nano router/handlers/main.yml
```
```
---
- name: restart frr
  service:
    name: frr
    state: restarted
  become: true
  ```
# 4. Создание роли mikrotik


### 4.1 Создание роли
```
ansible-galaxy init mikrotik
```
### 4.2 Задачи роли mikrotik
```
nano mikrotik/tasks/main.yml
```

---

```
- name: Configure DHCP server
  community.routeros.command:
    commands:
      - /ip dhcp-server add name=dhcp1 interface=ether2 disabled=no

- name: Configure basic firewall rule
  community.routeros.command:
    commands:
      - /ip firewall filter add chain=input action=accept protocol=icmp
```

# 5. Создание основного Playbook
Основной Playbook связывает роли и группы устройств.

```
cd ~/ansible
nano site.yml
```
Содержимое:

---
```
- name: Apply common role to all Linux hosts
  hosts: linux
  roles:
    - common

- name: Configure Linux routers
  hosts: linux
  roles:
    - router

- name: Configure MikroTik devices
  hosts: mikrotik
  roles:
    - mikrotik
```
# 6. Dry-run
```
ansible-playbook -i hosts.ini site.yml --check
```
Если ошибок нет — можно применять.

# 7. Применение ролей
```
ansible-playbook -i hosts.ini site.yml
```
# 8. Проверка результата
```
ansible linux -i hosts.ini -m command -a "systemctl status chrony --no-pager" --become
ansible linux -i hosts.ini -m command -a "vtysh -c 'show ip ospf neighbor'" --become
```
# 9. Рефакторинг предыдущих работ
Ранее созданные задачи:

- установки пакетов,

- пользователей,

- сервисов,

- должны быть:

- удалены из старых Playbook;

- перенесены в роли common и router.

Старые Playbook можно оставить только как учебные примеры.

# 10. Коммит в Git
```
git status
git add roles/ site.yml
git commit -m "Lab5: introduce Ansible roles (common, router, mikrotik)"
```

# Контрольные вопросы

- Что такое Ansible Role?

- Зачем нужна ролевая модель?

- Чем роль отличается от Playbook?

- Какие каталоги входят в роль?

- Почему роли упрощают сопровождение кода?

- Что происходит при повторном запуске ролей?

# Заключение
В ходе лабораторной работы была реализована ролевая модель Ansible, выполнена структуризация кода автоматизации и внедрён подход переиспользования конфигураций. Полученные навыки позволяют создавать масштабируемые и поддерживаемые проекты автоматизации.
