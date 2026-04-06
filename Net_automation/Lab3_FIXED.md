# Лабораторная работа №3
## Разработка и выполнение первых Ansible Playbook

## Цель лабораторной работы

Научиться описывать желаемое состояние устройств в виде кода (Infrastructure as Code, IaC) с помощью Ansible Playbook, освоить базовую структуру Playbook, принципы идемпотентности и безопасную проверку конфигураций перед применением.

## Задачи лабораторной работы
1. Создать простой Ansible Playbook для массовой настройки banner login.
2. Разработать Playbook для базовой конфигурации MikroTik RouterOS.
3. Разработать Playbook для самостоятельной конфигурации MikroTik RouterOS.
4. Зафиксировать созданные Playbook в Git-репозитории.

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

Возможно два варианта выполнения лабораторной:

Первый:
 Зайти в GNS3, создать там проект и работать с образами Mikrotik, UbuntuDockerGuest и Cloud.
 Однако в данном варианте есть подвох UbuntuDockerGuest переодически забывает установленные пакеты. 
 UbuntuDockerGuest нужно соединить через dhcp с Cloud, а с микротиком соединить по статическим IP.

Второй:
 На собственном компьютере создать виртуальные машины с образами Микротика и Unix-системой.
 Их нужно соединить между собой по статическим IP.


<img width="833" height="442" alt="Схема для лабы3_ADM" src="https://github.com/user-attachments/assets/c2d21d53-e9be-4784-b26e-9eada6ae73f5" />


На Unix-систему нужно установить
```
apt update && apt install ansible sshpass
```
А на микротике нужно прописать команду
```
/ip ssh set always-allow-password-login=yes # Говорим, что можно подключиться по ssh без ключа
```

Перед началом работы, не забудьте проверить связность между Unix-системой и Микротиком. 
И Что Unix-система имеет доступ к интернету.
 
# 1. Создание файла `hosts.ini`:
 Пример 
```
[mikrotik]
mt1 ansible_host=Здесь ваш IP-mikrotik
[mikrotik:vars]
ansible_user=admin
ansible_password=Здесь ваш пароль от микротика

```

# 2. Playbook — Базовая конфигурация MikroTik

### 2.1 Создание Playbook
```
cat > mikrotik_ssh.yml <<EOF
---
- name: Configure MikroTik via direct SSH
  hosts: mikrotik
  gather_facts: no
  connection: local  

  tasks:
    - name: Ensure IP address 192.168.2.1/24 exists on ether2
      command: >
        sshpass -p '{{ ansible_password }}'
        ssh -o StrictHostKeyChecking=no
        {{ ansible_user }}@{{ ansible_host }}
        "/ip address add address=192.168.2.1/24 interface=ether2"
      register: result
      failed_when: result.rc != 0 and 'already have such address' not in result.stdout
      changed_when: result.rc == 0 and 'already have such address' not in result.stdout

    - name: Show all IP addresses
      command: >
        sshpass -p '{{ ansible_password }}'
        ssh -o StrictHostKeyChecking=no
        {{ ansible_user }}@{{ ansible_host }}
        "/ip address print"
      register: ip_list

    - debug:
        var: ip_list.stdout_lines
EOF
```

Объяснение playbook:
```
- name: Configure MikroTik via direct SSH
  # Хост (или группа хостов) из инвентарного файла
  hosts: mikrotik
  # Не собираем факты об устройстве (MikroTik не предоставляет стандартные факты Ansible)
  gather_facts: no
  # Запускаем задачи с управляющего узла (не пытаемся подключаться к цели через SSH от Ansible)
  connection: local

  tasks:
    # Задача 1: Добавление IP-адреса 192.168.2.1/24 на интерфейс ether2
    - name: Ensure IP address 192.168.2.1/24 exists on ether2
      # Используем модуль command для выполнения произвольной команды на управляющем узле
      command: >
        # sshpass - позволяет передать пароль через аргумент (небезопасно, но удобно для учёбы)
        sshpass -p '{{ ansible_password }}'
        # Отключаем проверку ключа хоста (для упрощения в лабораторной среде)
        ssh -o StrictHostKeyChecking=no
        # Пользователь и хост берутся из инвентаря
        {{ ansible_user }}@{{ ansible_host }}
        # Сама команда на MikroTik в кавычках
        "/ip address add address=192.168.2.1/24 interface=ether2"
      # Регистрируем вывод команды в переменную result
      register: result
      # Условие, при котором задача считается провалившейся:
      # код возврата не 0 И при этом в выводе нет фразы "already have such address"
      failed_when: 
	- result.rc != 0 
        - ('already have such address' not in (result.stdout | default('') + result.stderr | default('')))
      # Условие, при котором задача считается изменившей состояние:
      # код возврата 0 И фраза "already have such address" отсутствует
      changed_when: result.rc == 0 and ('already have such address' not in (result.stdout | default('') + result.stderr | default('')))

    # Задача 2: Показать все IP-адреса (для проверки)
    - name: Show all IP addresses
      command: >
        sshpass -p '{{ ansible_password }}'
        ssh -o StrictHostKeyChecking=no
        {{ ansible_user }}@{{ ansible_host }}
        "/ip address print"
      register: ip_list

    # Задача 3: Вывести полученный список адресов на экран
    - debug:
        var: ip_list.stdout_lines
```

### 2.2 Проверка playbook
```
ansible-playbook -i hosts.ini mikrotik_ssh.yml --check
```
 Вот какой вывод ожидается:
```
PLAY [Configure MikroTik via direct SSH] ***************************************

TASK [Ensure IP address 192.168.2.1/24 exists on ether2] ***********************
skipping: [mt1]

TASK [Show all IP addresses] ***************************************************
skipping: [mt1]

TASK [debug] *******************************************************************
ok: [mt1] => {
    "ip_list.stdout_lines": "VARIABLE IS NOT DEFINED!"
}

PLAY RECAP *********************************************************************
mt1 : ok=1    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
```

### 2.3 Запуск playbook

ansible-playbook -i hosts.ini mikrotik_ssh.yml 

В выводе мы не должны увидеть ошибок, а на микротике должен появиться ip address на ether2.

# 3. Написать и выполнить собственный playbook

В данном разделе нужно через playbook превратить mikrotik в dhcp-server с hostname=dhcp-server и сохранить настройки.


# 4. Коммит Playbook в Git

```
git status
git add hosts.ini mikrotik_ssh.yml 
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
В ходе лабораторной работы были разработаны и выполнены первые Ansible Playbook. Освоены базовая структура Playbook и безопасной проверки конфигураций. Полученные навыки являются основой для дальнейшего изучения автоматизации и управления инфраструктурой.

by: nik7zol and Svetlana Kosareva
