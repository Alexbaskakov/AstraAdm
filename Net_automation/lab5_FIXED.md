# Лабораторная работа №5
## Ролевая модель (Roles) в Ansible и организация кода

## Цель лабораторной работы
Научиться структурировать проекты автоматизации с помощью Ansible Roles, разделять конфигурацию на логические части и переиспользовать код для управления различными типами устройств.

## Задачи лабораторной работы
1. Создать роль `common` для базовой настройки всех устройств.
2. Создать роль `linux` для настройки интерфейса на Linux-хосте.
3. Создать роль `mikrotik` для настройки DHCP-сервера на устройстве MikroTik.
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
- роль `linux` — всё, что связано с unix-системами;
- роль `mikrotik` — всё, что относится к RouterOS.

Роли позволяют:
- наводить порядок;
- переиспользовать код;
- применять одинаковые настройки на многих узлах;
- масштабировать проект.

---

# Практическая часть

## Что должно быть готово перед началом
Возможно два варианта выполнения лабораторной:

Первый: Зайти в GNS3, создать там проект и работать с образами Mikrotik, UbuntuDockerGuest и Cloud. Однако в данном варианте есть подвох UbuntuDockerGuest переодически забывает установленные пакеты. UbuntuDockerGuest нужно соединить через dhcp с Cloud, а с микротиком соединить по статическим IP.

Второй: На собственном компьютере создать виртуальные машины с образами Микротика и Unix-системой. Их нужно соединить между собой по статическим IP.

<img width="507" height="551" alt="Схема для лабы5_ADM" src="https://github.com/user-attachments/assets/677dac08-17c0-46ae-b54f-dc13abed9768" />

Конфиграция интерфесов на Ansible-server
```
#
# This is a sample network config, please uncomment lines to configure the network
#

# Uncomment this line to load custom interface files
# source /etc/network/interfaces.d/*

# Static config for eth0
auto eth1
iface eth1 inet static
	address 192.168.1.2
	netmask 255.255.255.0
# 	gateway 192.168.1.1
	up echo nameserver 192.168.1.1 > /etc/resolv.conf

auto eth2
iface eth2 inet static
	address 192.168.2.2
	netmask 255.255.255.0
# 	gateway 192.168.2.1
	up echo nameserver 192.168.2.1 > /etc/resolv.conf


# DHCP config for eth0
auto eth0
iface eth0 inet dhcp
	hostname UbuntuDockerGuest1-1

```

На Ansible-server нужно установить

```
apt update && apt install ansible sshpass tree
```
А на микротике нужно прописать команду
```
/ip address add address=192.168.1.1/24 interface=ether4
/ip ssh set always-allow-password-login=yes # Говорим, что можно подключиться по ssh без ключа
```
На LinuxCLI
Конфигурация интерфейсов

```
#
# This is a sample network config, please uncomment lines to configure the network
#

# Uncomment this line to load custom interface files
# source /etc/network/interfaces.d/*

# Static config for eth0
auto eth0
iface eth0 inet static
	address 192.168.2.1
	netmask 255.255.255.0
# 	gateway 192.168.2.2
	up echo nameserver 192.168.2.2 > /etc/resolv.conf

# DHCP config for eth2
auto eth2
iface eth2 inet dhcp
	hostname LinuxCLI-2

```

И команда которые нужно запустить на нём
```
apt update && apt install openssh-server sudo python3
service sshd start
useradd -m -s /bin/bash ansible
passwd ansible
usermod -aG sudo ansible

```

---

# 1. Подготовка структуры проекта

Перейти в каталог Ansible:

```
mkdir ansible
cd ansible/
```

В нём создадим hosts.ini

```
nano hosts.ini
```

```
[mikrotik]
mt1 ansible_host=192.168.1.1
[mikrotik:vars]
ansible_user=admin
ansible_password=Здесь ваш пароль от микротика

[linux]
192.168.1.2 

[linux:vars]
ansible_user=ansible
ansible_password=Здесь ваш пароль от пользователя ansible
ansible_become=yes
ansible_become_method=sudo
ansible_become_password=Здесь ваш пароль от пользователя ansible
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

tasks – главный файл main.yml содержит список действий, которые выполняет роль. Это обязательная папка – без неё роль не имеет логики.

handlers – здесь хранятся обработчики (handlers), которые запускаются только при вызове из задач. Файл main.yml определяет действия, которые будут выполнены по уведомлению.

templates – шаблоны Jinja2 (расширение .j2), которые позволяют динамически генерировать конфигурационные файлы на основе переменных. Например, nginx.conf.j2.

defaults – переменные по умолчанию. Они имеют самый низкий приоритет и легко переопределяются пользователем.

vars – переменные более высокого приоритета, чем в defaults. Их сложнее переопределить. Обычно кладут значения, которые пользователь вряд ли захочет менять, либо те, что жёстко задают логику роли.

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
ansible-galaxy init linux
```
### 3.2 Установка и настройка FRR (OSPF)
Открываем задачи:

```
nano linux/tasks/main.yml
```
Содержимое:

```
---
- name: Set static IP for eth1
  blockinfile:
    path: /etc/network/interfaces
    block: |
      auto eth1
      iface eth1 inet static
          address {{ eth1_ip }}
          netmask {{ eth1_netmask }}
          # gateway не указываем, если не нужен
    marker: "# {mark} ANSIBLE MANAGED BLOCK - eth1"
    insertafter: "^# Static config for eth0"  # или другое якорь
    backup: yes
  become: true
  notify: restart networking
  ```

Также добавим какой ip address и какой интерфейс используется на нашем линукс устройстве. Это можно уточнить в defaults/main.yml или в vars/main.yml

```
nano linux/defaults/main.yml
```

```
---
eth1_ip: 192.168.10.2
eth1_netmask: 255.255.255.0
```

### 3.3 Добавим обработчик (handler) перезапуска сети
```
nano linux/handlers/main.yml
```
```
---
- name: restart networking
  systemd:
    name: networking
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
- name: Ensure IP address 192.168.10.1/24 exists on ether1
  command: >
    sshpass -p '{{ ansible_password }}'
    ssh -o StrictHostKeyChecking=no
    {{ ansible_user }}@{{ ansible_host }}
    "/ip address add address=192.168.10.1/24 interface=ether1"
  register: result
  failed_when: result.rc != 0 and 'already have such address' not in result.stdout
  changed_when: result.rc == 0 and 'already have such address' not in result.stdout
```

# 5. Создание основного Playbook
Основной Playbook связывает роли и группы устройств.

```
cd ..
nano site.yml
```
Содержимое:

---
```
- name: Configure Linux routers
  hosts: linux
  roles:
    - linux

- name: Configure MikroTik devices
  hosts: mikrotik
  gather_facts: no
  connection: local  
  roles:
    - mikrotik
```
# 6. Dry-run
```
ansible-playbook -i hosts.ini site.yml --check
```

# 7. Применение ролей
```
ansible-playbook -i hosts.ini site.yml
```
# 8. Проверка результата

В связи с тем, что лабораторная делается на GNS3, handler скорее всего не запустится корректно. Так что перед проверкой перезапустите linuxCLI.
Заходим на Микротик и запускаем пинг на линукс.
```
ping 192.168.10.2
```

# 9. Самостоятельное задание

Самостоятельно превратите микротик в DHCP-server и сделайте так, чтобы он выдал ip address на linuxCLI.

# 10. Коммит в Git
```
git status
git add roles/ site.yml hosts.ini
git commit -m "Lab5: introduce Ansible roles (common, linux, mikrotik)"
```

# Контрольные вопросы

- Что такое Ansible Role?

- Зачем нужна ролевая модель?

- Чем роль отличается от Playbook?

- Какие каталоги входят в роль?

- Почему роли упрощают сопровождение кода?

- Что происходит при повторном запуске ролей?

# Заключение
В ходе лабораторной работы была реализована ролевая модель Ansible, выполнена структуризация кода автоматизации и внедрён подход переиспользования конфигураций. Полученные навыки позволяют создавать масштабируемые и поддерживаемые проекты автоматизации.![Uploading Схема для лабы5_ADM.PNG…]()
