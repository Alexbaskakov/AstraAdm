Лабораторная работа №4
Использование Jinja2-шаблонов для управления конфигурацией сетевых устройств
---

<img width="797" height="763" alt="image" src="https://github.com/user-attachments/assets/5382597b-c525-474b-89bf-dfe49081fae4" />

1. Цель лабораторной работы
Научиться генерировать конфигурационные файлы автоматически с помощью Jinja2-шаблонов и переменных Ansible, а затем применять сгенерированные конфиги на Linux-хостах и маршрутизаторах MikroTik RouterOS в среде GNS3.
2. Задачи лабораторной работы
Создать и настроить топологию сети в GNS3 (MikroTik + Linux + Cloud).
Выполнить начальную (базовую) конфигурацию всех устройств топологии.
Подготовить структуру проекта Ansible с inventory, переменными и шаблонами.
Создать Jinja2-шаблоны для конфигурационных файлов:
`/etc/chrony/chrony.conf` — синхронизация времени (NTP) на Linux-хосте;
`/etc/ssh/sshd_config` — усиление безопасности SSH на Linux-хосте;
конфигурация MikroTik RouterOS в формате `.rsc` — сетевые настройки маршрутизатора.
Написать Ansible Playbook, который разворачивает шаблоны на устройства.
Запустить Playbook в режиме `--check` (dry-run) для безопасной проверки.
Применить Playbook и верифицировать результат на каждом устройстве.
3. Ожидаемый результат
После выполнения лабораторной работы студент:
понимает принцип работы Jinja2-шаблонизатора в контексте Ansible;
умеет хранить переменные отдельно от кода в `group_vars/` и `host_vars/`;
умеет писать Jinja2-шаблоны с циклами, условиями и подстановкой переменных;
умеет применять сгенерированные конфиги через модуль `template` на Linux и MikroTik;
понимает, что изменится при запуске в режиме `--check`;
умеет проверять, что конфигурация действительно обновилась на целевых устройствах.
---
4. Теоретическая часть
4.1 Jinja2 — шаблонизатор для конфигураций
Jinja2 — это шаблонизатор, встроенный в Ansible по умолчанию. В шаблоне можно писать обычный текст конфигурационного файла, а переменные подставлять через синтаксис `{{ ... }}`. Ansible берёт файл с расширением `.j2`, подставляет значения переменных из `group_vars/`, `host_vars/` или командной строки, и записывает итоговый файл на целевое устройство через модуль `template`.
Основные конструкции Jinja2:
Конструкция	Синтаксис	Назначение
Вывод переменной	`{{ variable }}`	Подставляет значение переменной в текст
Цикл	`{% for item in list %}...{% endfor %}`	Итерация по списку элементов
Условие	`{% if condition %}...{% endif %}`	Вывод блока при выполнении условия
Комментарий шаблона	`{# comment #}`	Комментарий, не попадает в итоговый файл
Почему шаблоны выгодны по сравнению с ручной редакцией:
один шаблон обслуживает множество серверов — достаточно изменить переменную, чтобы конфиг обновился везде;
изменения версонируются в Git — всегда можно откатиться;
исключается человеческий фактор (опечатки, забытые строки);
конфигурация становится декларативной — описывается желаемый результат, а не последовательность команд.
4.2 Модуль `template` в Ansible
Модуль `template` отличается от `copy` тем, что перед отправкой файла на целевой хост он обрабатывает файл через Jinja2-движок. Если итоговый файл отличается от того, что уже лежит на хосте, задача помечается как `changed`. Это позволяет использовать handlers — обработчики, которые запускаются только при фактическом изменении конфигурации (например, перезапуск сервиса SSH только если `sshd_config` действительно изменился).
4.3 Управление MikroTik RouterOS через Ansible
MikroTik RouterOS не поддерживает классический SSH-доступ в формате, ожидаемом Ansible по умолчанию. Для управления MikroTik в данной лабораторной работе применяется подход `connection: local` с использованием утилиты `sshpass` для передачи команд через API-консоль RouterOS. Конфигурация MikroTik генерируется из Jinja2-шаблона в формате `.rsc` (RouterOS Script), а затем импортируется на маршрутизатор через модуль `community.routeros.command`.
---
5. Топология сети в GNS3
5.1 Описание топологии
Для выполнения лабораторной работы используется следующая топология сети, развёрнутая в эмуляторе GNS3:
```
┌─────────────┐         192.168.1.0/24          ┌─────────────┐         203.0.113.0/24          ┌─────────────┐
│             │    ether1 ──────── eth0          │             │    eth1 ──────── eth0           │             │
│  Mikrotik-1 │◄────────────────────────────────►│  LinuxCLI-1 │◄──────────────────────────────►│   Cloud1    │
│  (RouterOS) │                                  │   (Debian)  │                                │  (NAT/ WAN) │
└─────────────┘                                  └─────────────┘                                └─────────────┘
  192.168.1.1/24                                192.168.1.2/24                                203.0.113.1/24
                                                 203.0.113.2/24
```

Назначение устройств:
Устройство	Роль	Описание
Mikrotik-1	Маршрутизатор (LAN-шлюз)	Маршрутизирует трафик между LAN-сегментом и LinuxCLI-1.
LinuxCLI-1	Управляемый Linux-хост	Выступает как целевой хост для Ansible, а также как шлюз к Cloud1.
Cloud1	Внешняя сеть (WAN)	Эмулирует подключение к Интернет через NAT.
5.2 План адресации
Устройство	Интерфейс	IP-адрес	Маска подсети	Шлюз по умолчанию
Mikrotik-1	ether1	192.168.1.1	255.255.255.0	192.168.1.2
LinuxCLI-1	eth0	192.168.1.2	255.255.255.0	—
LinuxCLI-1	eth1	203.0.113.2	255.255.255.0	203.0.113.1
Cloud1	eth0	203.0.113.1	255.255.255.0	—
Обоснование выбора адресации: Подсеть `192.168.1.0/24` выбрана как стандартная частная LAN-сеть для связи между MikroTik и Linux-хостом. Подсеть `203.0.113.0/24` относится к диапазону TEST-NET-3 (RFC 5737), зарезервированному для документации и тестирования — она идеально подходит для эмуляции WAN-подключения в учебной среде, так как не конфликтует с реальными адресами.

---
6. Практическая часть
6.1 Создание проекта в GNS3
6.1.1 Запуск GNS3 и создание нового проекта
Запустите GNS3 и создайте новый проект с именем `Lab4_Jinja2_Templates`.
Тип проекта: Blank project.
Сохраните проект в удобную директорию.
6.1.2 Добавление узлов
Из панели узлов перетащите на рабочее поле следующие устройства:
Узел	Тип в GNS3	Количество
Mikrotik-1	MikroTik RouterOS (v6/v7)	1
LinuxCLI-1	Linux CLI (Debian/Ubuntu)	1
Cloud1	Cloud (NAT)	1
Обоснование выбора образов: MikroTik RouterOS выбран как популярное и доступное сетевое оборудование с богатым функционалом маршрутизации. Debian/Ubuntu Linux CLI используется как стандартный целевой хост для Ansible-автоматизации, полностью поддерживающий SSH, systemd и пакетный менеджер `apt`. Cloud (NAT) обеспечивает выход в Интернет для установки пакетов на Linux-хост.
6.1.3 Соединение узлов
Соедините устройства кабелями (drag-and-drop между интерфейсами):
Соединение	Тип линка
Mikrotik-1 (ether1) ↔ LinuxCLI-1 (eth0)	Ethernet
LinuxCLI-1 (eth1) ↔ Cloud1 (eth0)	Ethernet
6.1.4 Запуск всех устройств
Нажмите кнопку Start/Resume all nodes (зелёная стрелка ▶) на панели инструментов GNS3. Дождитесь полной загрузки всех устройств (индикаторы станут зелёными). Для LinuxCLI-1 это может занять 1–2 минуты.
---
6.2 Начальная конфигурация LinuxCLI-1
Подключитесь к консоли LinuxCLI-1 в GNS3 (двойной клик на узле).
6.2.1 Настройка сетевых интерфейсов
Назначьте IP-адреса на оба интерфейса LinuxCLI-1. В Debian/Ubuntu для этого используется `ip` из пакета `iproute2`.
```bash
# Настройка eth0 — связь с MikroTik (LAN)
ip addr add 192.168.1.2/24 dev eth0
ip link set eth0 up

# Настройка eth1 — связь с Cloud1 (WAN)
ip addr add 203.0.113.2/24 dev eth1
ip link set eth1 up

# Шлюз по умолчанию через Cloud1 (для выхода в Интернет)
ip route add default via 203.0.113.1
```
Обоснование: LinuxCLI-1 работает как dual-homed host — он одновременно подключён к локальной сети (MikroTik) и к внешней сети (Cloud1). Шлюз по умолчанию указывает на Cloud1, чтобы обеспечить доступ к репозиториям пакетов для установки Ansible, chrony и других компонентов.
6.2.2 Проверка связности
```bash
# Проверка связи с MikroTik
ping -c 3 192.168.1.1

# Проверка связи с Cloud1
ping -c 3 203.0.113.1

# Проверка выхода в Интернет (через NAT Cloud1)
ping -c 3 8.8.8.8
```
Если все три команды получают ответы — сетевая связность настроена корректно.
6.2.3 Настройка SSH-сервера на LinuxCLI-1
Ansible управляет устройствами через SSH, поэтому необходимо убедиться, что SSH-сервер запущен и принимает подключения.
```bash
# Обновление списка пакетов и установка SSH-сервера
apt update
apt install -y openssh-server

# Запуск SSH-сервера
systemctl enable --now ssh

# Проверка статуса
systemctl status ssh --no-pager
```
Для подключения Ansible к LinuxCLI-1 по SSH необходимо создать пользователя и установить пароль (в учебной среде):
```bash
# Создание пользователя для Ansible
useradd -m -s /bin/bash -G sudo ansible
echo "ansible:P@ssw0rd" | chpasswd
```
> **Важно:** В рабочей среде использование паролей не рекомендуется. Для учебных целей допустимо, но в production необходимо применять SSH-ключи.
6.2.4 Установка Ansible на LinuxCLI-1
В данной лабораторной работе LinuxCLI-1 выступает одновременно как управляющий узел (control node) и как целевой хост (managed node). Это допустимо для учебной среды.
```bash
# Установка Ansible
apt update
apt install -y ansible sshpass

# Проверка установки
ansible --version
```
`sshpass` необходим для подключения к MikroTik RouterOS через Ansible, так как RouterOS не поддерживает стандартную аутентификацию по ключам в контексте Ansible `connection: local`.
---
6.3 Начальная конфигурация Mikrotik-1
Подключитесь к консоли Mikrotik-1 в GNS3 (двойной клик на узле).
6.3.1 Сброс к заводским настройкам и базовая конфигурация
```routeros
# Сброс конфигурации (нажимайте 'y' на все вопросы)
/system reset-configuration no-defaults=yes

# После перезагрузки:
# Настройка IP-адреса на ether1
/ip address add address=192.168.1.1/24 interface=ether1

# Настройка шлюза по умолчанию (через LinuxCLI-1)
/ip route add gateway=192.168.1.2

# Установка имени маршрутизатора
/system identity set name=LAB-MT1

# Включение SSH-доступа для Ansible
/ip service enable ssh

# Установка пароля администратора
/user set 0 password=MikroTikPass

# Проверка
/ip address print
/ip route print
/system identity print
```
Обоснование: MikroTik-1 настроен с минимальным набором параметров, необходимым для взаимодействия с LinuxCLI-1 и Ansible. Шлюз по умолчанию указывает на LinuxCLI-1, чтобы MikroTik мог получать пакеты из WAN-сети при необходимости (например, обновление системного времени через NTP). SSH-сервис включён, так как Ansible управляет MikroTik через SSH-соединение.
6.3.2 Проверка связности с MikroTik
Вернитесь в консоль LinuxCLI-1 и проверьте связь:
```bash
# Проверка связи с MikroTik
ping -c 3 192.168.1.1

# Проверка SSH-подключения к MikroTik (sshpass используется для пароля)
sshpass -p 'MikroTikPass' ssh -o StrictHostKeyChecking=no admin@192.168.1.1 "/system identity print"
```
Если команда возвращает имя `LAB-MT1` — подключение к MikroTik работает корректно.
---
6.4 Подготовка проекта Ansible
Все действия в этом разделе выполняются в консоли LinuxCLI-1.
6.4.1 Создание структуры каталогов
```bash
cd ~
mkdir -p ansible/{templates,group_vars,host_vars}
cd ansible
```
Обоснование структуры:
Каталог	Назначение
`templates/`	Jinja2-шаблоны конфигурационных файлов (`.j2`)
`group_vars/`	Переменные, общие для группы устройств (например, для `[all]`)
`host_vars/`	Переменные, специфичные для конкретного хоста
6.4.2 Создание файла инвентаризации `hosts.ini`
Создайте файл `hosts.ini`, который описывает все устройства топологии:
```bash
cat > hosts.ini << 'EOF'
[linux]
linuxcli1 ansible_host=192.168.1.2 ansible_user=ansible ansible_password=P@ssw0rd ansible_python_interpreter=/usr/bin/python3

[mikrotik]
mikrotik1 ansible_host=192.168.1.1 ansible_user=admin ansible_password=MikroTikPass ansible_connection=local
EOF
```
Обоснование параметров инвентаризации:
Параметр	Значение	Описание
`ansible_host`	192.168.1.x	IP-адрес целевого устройства в топологии GNS3
`ansible_user`	ansible / admin	Имя пользователя для SSH-подключения
`ansible_password`	P@ssw0rd / MikroTikPass	Пароль пользователя (для `sshpass` при `connection: local` у MikroTik)
`ansible_connection=local`	(только MikroTik)	Ansible выполняет команды локально, подключаясь к MikroTik через `sshpass` + `community.routeros`
`ansible_python_interpreter`	/usr/bin/python3	Указание пути к Python для корректной работы модулей на Linux-хосте
6.4.3 Проверка связности Ansible
```bash
# Проверка связи с Linux-группой
ansible linux -i hosts.ini -m ping

# Ожидаемый результат:
# linuxcli1 | SUCCESS => {
#     "ping": "pong"
# }
```
Если `ping` возвращает `pong` — Ansible может управлять LinuxCLI-1. Если возникает ошибка — проверьте SSH-доступ (шаг 6.2.3).
---
6.5 Создание переменных в `group_vars/`
6.5.1 Переменные для Linux-группы
Создайте файл с переменными, которые будут использоваться в Jinja2-шаблонах для LinuxCLI-1:
```bash
cat > group_vars/linux.yml << 'EOF'
---
# Параметры NTP-синхронизации
ntp_servers:
  - 0.pool.ntp.org
  - 1.pool.ntp.org
  - 2.pool.ntp.org
  - 3.pool.ntp.org

ntp_allow_networks:
  - 192.168.1.0/24

# Параметры SSH-сервера
ssh_port: 22
ssh_permit_root_login: "no"
ssh_password_auth: "yes"
ssh_max_auth_tries: 3
ssh_login_grace_time: 60
ssh_allow_users: "ansible"

# Имя хоста
hostname: linuxcli1
EOF
```
Обоснование выбора переменных:
ntp_servers — список публичных NTP-серверов пула `pool.ntp.org`. Использование четырёх серверов обеспечивает отказоустойчивость синхронизации времени. Параметр `iburst` в шаблоне ускоряет начальную синхронизацию.
ntp_allow_networks — подсети, которым разрешено запрашивать время у chrony. В учебной среде достаточно одной подсети `192.168.1.0/24`.
ssh_port — стандартный порт SSH. В рабочей среде рекомендуется изменить на нестандартный, но для учебных целей оставляем `22`.
ssh_permit_root_login: "no" — прямой вход root по SSH отключён из соображений безопасности. Управление осуществляется через пользователя `ansible` с `sudo`.
ssh_password_auth: "yes" — аутентификация по паролю включена для учебной среды. В production рекомендуется `"no"` с использованием SSH-ключей.
ssh_max_auth_tries: 3 — ограничение количества попыток аутентификации для защиты от брутфорса.
ssh_allow_users — явное указание пользователей, которым разрешён SSH-доступ.
6.5.2 Переменные для MikroTik-группы
```bash
cat > group_vars/mikrotik.yml << 'EOF'
---
# Параметры MikroTik RouterOS
router_identity: LAB-MT1

# Сетевые интерфейсы
interfaces:
  - name: ether1
    address: 192.168.1.1/24
    comment: "LAN - connection to LinuxCLI-1"

# Маршрутизация
default_gateway: 192.168.1.2

# DNS
dns_servers:
  - 8.8.8.8
  - 8.8.4.4

# NTP для RouterOS
ntp_servers:
  - 0.pool.ntp.org
  - 1.pool.ntp.org
EOF
```
Обоснование: Данные переменные полностью описывают сетевую конфигурацию MikroTik в декларативном виде. Изменение IP-адреса или шлюза требует правки одного файла (переменных), а не переписывания шаблона. Это демонстрирует ключевой принцип Infrastructure as Code — разделение данных и логики.
---
6.6 Создание Jinja2-шаблонов
6.6.1 Шаблон `/etc/chrony/chrony.conf`
Шаблон `chrony.conf.j2` генерирует конфигурацию службы синхронизации времени chrony для LinuxCLI-1:
```bash
cat > templates/chrony.conf.j2 << 'EOF'
# chrony.conf - Generated by Ansible Jinja2 template
# Do not edit manually. Changes will be overwritten on next playbook run.

# NTP-серверы для синхронизации времени
{% for server in ntp_servers %}
server {{ server }} iburst
{% endfor %}

# Параметры хранения drift-файла
driftfile /var/lib/chrony/drift
rtcsync

# Шаг коррекции времени (секунды, количество шагов)
makestep 1.0 3

# Логирование
logdir /var/log/chrony

# Разрешение запросов из локальной сети
{% for net in ntp_allow_networks %}
allow {{ net }}
{% endfor %}
EOF
```
Обоснование структуры шаблона:
Директива `{% for server in ntp_servers %}` автоматически разворачивает список NTP-серверов из переменных — при добавлении нового сервера достаточно добавить его в `group_vars/linux.yml`, шаблон сам подставит строку `server ... iburst`.
Параметр `iburst` отправляет четыре запроса с интервалом 2 секунды при старте, что ускоряет первичную синхронизацию (RFC 5905).
`makestep 1.0 3` — разрешает корректировку времени не более чем на 1 секунду за шаг, максимум 3 шага. Это предотвращает резкий скачок времени при большой рассинхронизации.
Блок `{% for net in ntp_allow_networks %}` управляет ACL для chrony — только указанные подсети могут запрашивать время.
Проверьте содержимое шаблона:
```bash
cat templates/chrony.conf.j2
```
6.6.2 Шаблон `/etc/ssh/sshd_config`
Шаблон `sshd_config.j2` генерирует усиленный конфиг SSH-сервера:
```bash
cat > templates/sshd_config.j2 << 'EOF'
# sshd_config - Generated by Ansible Jinja2 template
# Do not edit manually. Changes will be overwritten on next playbook run.

# Сетевые параметры
Port {{ ssh_port }}
AddressFamily any
ListenAddress 0.0.0.0

# Аутентификация
PermitRootLogin {{ ssh_permit_root_login }}
MaxAuthTries {{ ssh_max_auth_tries }}
LoginGraceTime {{ ssh_login_grace_time }}
PasswordAuthentication {{ ssh_password_auth }}
PermitEmptyPasswords no
ChallengeResponseAuthentication no
UsePAM yes

# Ограничение доступа по пользователям
AllowUsers {{ ssh_allow_users }}

# Безопасность сессий
X11Forwarding no
PrintMotd no
AcceptEnv LANG LC_*

# Логирование
Subsystem sftp /usr/lib/openssh/sftp-server
EOF
```
Обоснование выбора директив:
Директива	Значение	Почему
`Port {{ ssh_port }}`	22	Порт SSH, параметризован для возможности быстрого изменения без редактирования шаблона.
`PermitRootLogin no`	no	Прямой вход root запрещён. Все действия выполняются через `sudo` от имени пользователя `ansible`.
`MaxAuthTries 3`	3	Ограничение попыток аутентификации — защита от брутфорс-атак.
`LoginGraceTime 60`	60	Таймаут ввода пароля 60 секунд — после этого подключение разрывается.
`PasswordAuthentication yes`	yes	Включена для учебной среды (Ansible использует `sshpass`). В production — `no`.
`PermitEmptyPasswords no`	no	Запрещены пустые пароли — базовое требование безопасности.
`ChallengeResponseAuthentication no`	no	Отключён устаревший метод challenge-response аутентификации.
`AllowUsers ansible`	ansible	Только пользователь `ansible` может подключаться по SSH. Остальные — нет.
`X11Forwarding no`	no	Проброс X11-графики отключён — не нужен на сетевом устройстве и снижает атаковую поверхность.
> **Примечание:** Директива `Protocol 2` из старых версий OpenSSH здесь **не используется**, так как в современных версиях OpenSSH (7.0+) поддерживается только протокол версии 2, и указание `Protocol 2` вызывает предупреждение об устаревании.
Проверьте содержимое шаблона:
```bash
cat templates/sshd_config.j2
```
6.6.3 Шаблон конфигурации MikroTik RouterOS (`.rsc`)
Шаблон `mikrotik_config.rsc.j2` генерирует скрипт конфигурации MikroTik:
```bash
cat > templates/mikrotik_config.rsc.j2 << 'EOF'
# MikroTik RouterOS Configuration Script
# Generated by Ansible Jinja2 template
# Router: {{ router_identity }}

# Установка имени маршрутизатора
/system identity set name={{ router_identity }}

# Настройка интерфейсов
{% for iface in interfaces %}
/ip address add address={{ iface.address }} interface={{ iface.name }} comment="{{ iface.comment }}"
{% endfor %}

# Настройка маршрутизации
/ip route add gateway={{ default_gateway }}

# Настройка DNS
/ip dns set servers="{{ dns_servers | join(',') }}"

# Настройка NTP-серверов
/system ntp client set primary-ntp={{ ntp_servers[0] }} secondary-ntp={{ ntp_servers[1] }} enabled=yes
EOF
```
Обоснование структуры шаблона:
Цикл `{% for iface in interfaces %}` позволяет задавать произвольное количество интерфейсов через переменные — при добавлении нового интерфейса в топологию достаточно добавить запись в `group_vars/mikrotik.yml`.
Фильтр `| join(',')` объединяет список DNS-серверов в строку с разделителями-запятыми, как того требует синтаксис RouterOS.
Использование `{{ ntp_servers[0] }}` и `{{ ntp_servers[1] }}` демонстрирует обращение к элементам списка по индексу, что необходимо для директив RouterOS, принимающих отдельные значения, а не список.
Проверьте содержимое шаблона:
```bash
cat templates/mikrotik_config.rsc.j2
```
---
6.7 Создание Ansible Playbook
6.7.1 Playbook для Linux-хостов
Создайте Playbook `linux_templates.yml`, который развёртывает Jinja2-шаблоны на LinuxCLI-1:
```bash
cat > linux_templates.yml << 'EOF'
---
- name: Deploy configuration templates to Linux hosts
  hosts: linux
  become: true
  gather_facts: true

  tasks:
    - name: Set hostname
      hostname:
        name: "{{ hostname }}"

    - name: Ensure chrony package is installed
      apt:
        name: chrony
        state: present
        update_cache: true

    - name: Deploy chrony.conf from Jinja2 template
      template:
        src: templates/chrony.conf.j2
        dest: /etc/chrony/chrony.conf
        owner: root
        group: root
        mode: '0644'
      notify: restart chrony

    - name: Deploy sshd_config from Jinja2 template
      template:
        src: templates/sshd_config.j2
        dest: /etc/ssh/sshd_config
        owner: root
        group: root
        mode: '0644'
      notify: restart ssh

    - name: Ensure chrony service is enabled and started
      service:
        name: chrony
        state: started
        enabled: true

  handlers:
    - name: restart chrony
      service:
        name: chrony
        state: restarted

    - name: restart ssh
      service:
        name: ssh
        state: restarted
EOF
```
Обоснование порядка задач:
hostname — устанавливается первым, чтобы в логах сервисов корректно отображалось имя хоста.
apt: chrony — пакет устанавливается до развертывания шаблона `chrony.conf`, чтобы директория `/etc/chrony/` существовала. Если изменить порядок, задача `template` завершится с ошибкой отсутствующей директории.
template: chrony.conf — развёртывает конфигурацию chrony. Если файл изменился, запускается handler `restart chrony`.
template: sshd_config — развёртывает конфигурацию SSH. Если файл изменился, запускается handler `restart ssh`. Поскольку `ssh_password_auth: "yes"`, текущее SSH-подключение не будет разорвано.
service: chrony — гарантирует, что chrony запущен и добавлен в автозагрузку.
Проверьте содержимое Playbook:
```bash
cat linux_templates.yml
```
6.7.2 Playbook для MikroTik
Создайте Playbook `mikrotik_templates.yml`, который генерирует и применяет конфигурацию на MikroTik:
```bash
cat > mikrotik_templates.yml << 'EOF'
---
- name: Generate MikroTik configuration from template
  hosts: mikrotik
  gather_facts: false
  connection: local

  tasks:
    - name: Generate MikroTik config file locally from Jinja2 template
      template:
        src: templates/mikrotik_config.rsc.j2
        dest: "/tmp/mikrotik_{{ inventory_hostname }}.rsc"
        mode: '0644'

    - name: Push configuration to MikroTik via SSH
      community.routeros.command:
        commands:
          - "/import file-name=mikrotik_{{ inventory_hostname }}.rsc"
      register: mikrotik_result
      changed_when: "'already' not in mikrotik_result.stdout[0]"

    - name: Display MikroTik identity
      community.routeros.command:
        commands:
          - /system identity print
      register: identity_output
      changed_when: false

    - name: Show current identity
      debug:
        var: identity_output.stdout_lines
EOF
```
Обоснование подхода к MikroTik:
`connection: local` — Ansible не подключается к MikroTik напрямую через SSH-сессию. Вместо этого он выполняет команды `sshpass` + SSH локально, перенаправляя команды на MikroTik. Это необходимо, так как RouterOS не поддерживает стандартный механизм `ansible_connection=ssh`.
Сначала шаблон генерирует `.rsc`-файл локально, затем этот файл импортируется на MikroTik через команду `/import`.
`changed_when: "'already' not in mikrotik_result.stdout[0]"` — Ansible помечает задачу как `changed` только если конфигурация действительно изменилась. Если MikroTik отвечает, что настройки уже применены, задача покажет `ok`.
---
6.8 Dry-run — безопасная проверка (Linux)
Перед реальным применением выполните проверку в режиме dry-run. Этот режим показывает, какие изменения Ansible планирует выполнить, но не применяет их фактически.
```bash
ansible-playbook -i hosts.ini linux_templates.yml --check -v
```
Ожидаемый вывод:
```
PLAY [Deploy configuration templates to Linux hosts] *************************

TASK [Gathering Facts] *******************************************************
ok: [linuxcli1]

TASK [Set hostname] **********************************************************
ok: [linuxcli1]

TASK [Ensure chrony package is installed] ************************************
ok: [linuxcli1]

TASK [Deploy chrony.conf from Jinja2 template] *******************************
changed: [linuxcli1]

TASK [Deploy sshd_config from Jinja2 template] ********************************
changed: [linuxcli1]

TASK [Ensure chrony service is enabled and started] **************************
ok: [linuxcli1]

PLAY RECAP *******************************************************************
linuxcli1   : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
Анализ вывода:
`changed` на задачах `template` означает, что сгенерированный из шаблона файл отличается от текущего на хосте — это нормально для первого запуска.
`ok` на задаче `hostname` означает, что имя хоста уже установлено корректно (идемпотентность).
Если видим ошибки вида `template not found` — проверьте путь к файлу шаблона.
Если видим ошибки `permission denied` — проверьте, что пользователь `ansible` входит в группу `sudo` и имеет права без пароля (или используйте `--become --ask-become-pass`).
> **Примечание:** Для MikroTik dry-run в полном объёме недоступен, так как RouterOS не поддерживает режим проверки. Перед применением MikroTik-Playbook выполните локальную генерацию шаблона и визуально проверьте содержимое `.rsc`-файла.
Локальная генерация шаблона для визуальной проверки:
```bash
ansible localhost -m template \
  -a "src=templates/mikrotik_config.rsc.j2 dest=/tmp/mikrotik_check.rsc" \
  -e "@group_vars/mikrotik.yml" \
  -e "router_identity=LAB-MT1"

cat /tmp/mikrotik_check.rsc
```
---
6.9 Реальное применение Playbook
Если dry-run прошёл без ошибок — выполняем реальное применение конфигурации.
6.9.1 Применение на LinuxCLI-1
```bash
ansible-playbook -i hosts.ini linux_templates.yml -v
```
Ожидаемый вывод: Аналогичен dry-run, но с фактическим применением изменений и запуском handlers.
6.9.2 Применение на Mikrotik-1
```bash
ansible-playbook -i hosts.ini mikrotik_templates.yml -v
```
Ожидаемый вывод:
```
TASK [Display MikroTik identity] *********************************************
ok: [mikrotik1] => {
    "identity_output.stdout_lines": [
        [
            "name: LAB-MT1"
        ]
    ]
}
```
> **Примечание:** При первом запуске MikroTik Playbook может потребоваться предварительное копирование `.rsc`-файла на маршрутизатор через SCP. В учебной среде допускается ручной импорт: подключитесь к консоли MikroTik в GNS3 и выполните `/import file-name=mikrotik_mikrotik1.rsc` (если автоматический импорт не сработал).
---
6.10 Верификация результатов
6.10.1 Проверка chrony на LinuxCLI-1
```bash
# Просмотр сгенерированного chrony.conf
ansible linux -i hosts.ini -m command -a "cat /etc/chrony/chrony.conf" --become

# Проверка статуса службы chrony
ansible linux -i hosts.ini -m command -a "systemctl status chrony --no-pager" --become

# Проверка текущих источников времени (chronyc sources)
ansible linux -i hosts.ini -m command -a "chronyc sources -v" --become
```
Ожидаемый результат:
В `chrony.conf` видны строки `server 0.pool.ntp.org iburst` и т.д.
`systemctl status chrony` показывает `active (running)`.
`chronyc sources` показывает NTP-серверы с состоянием `*` (текущий источник) или `+` (кандидат).
6.10.2 Проверка sshd_config на LinuxCLI-1
```bash
# Просмотр сгенерированного sshd_config
ansible linux -i hosts.ini -m command -a "cat /etc/ssh/sshd_config" --become

# Проверка, что SSH-служба перезапустилась
ansible linux -i hosts.ini -m command -a "systemctl status ssh --no-pager" --become

# Проверка, что SSH слушает на порту 22
ansible linux -i hosts.ini -m command -a "ss -tlnp | grep :22" --become
```
Ожидаемый результат:
В `sshd_config` видны строки `Port 22`, `PermitRootLogin no`, `AllowUsers ansible`.
SSH-служба `active (running)`.
Порт 22 открыт и прослушивается.
6.10.3 Проверка конфигурации MikroTik
Подключитесь к консоли MikroTik в GNS3:
```routeros
/system identity print
/ip address print
/ip route print
/ip dns print
/system ntp client print
```
Ожидаемый результат:
`name: LAB-MT1` — имя маршрутизатора установлено.
`192.168.1.1/24` на интерфейсе `ether1`.
Шлюз по умолчанию `192.168.1.2`.
DNS-серверы `8.8.8.8` и `8.8.4.4` настроены.
NTP-клиент включён с указанными серверами.
---
6.11 Индивидуализация конфигурации через `host_vars/`
Демонстрация возможности индивидуализации конфигурации для конкретных хостов. Например, если в топологии появляется второй Linux-хост с нестандартным SSH-портом.
6.11.1 Создание host_vars для конкретного хоста
```bash
# Пример: для хоста linuxcli1 изменить порт SSH на 2222
cat > host_vars/linuxcli1.yml << 'EOF'
---
ssh_port: 2222
EOF
```
6.11.2 Проверка через dry-run
```bash
ansible-playbook -i hosts.ini linux_templates.yml --check -v
```
В выводе задачи `Deploy sshd_config` Ansible покажет `changed` — шаблон сгенерирует конфиг с `Port 2222`.
6.11.3 Возврат к стандартному порту
> **Внимание:** Если вы реально примените Playbook с портом `2222`, последующие SSH-подключения Ansible к этому хосту перестанут работать (Ansible подключается по порту 22 по умолчанию). Для возврата удалите `host_vars/linuxcli1.yml` или верните `ssh_port: 22`:
```bash
cat > host_vars/linuxcli1.yml << 'EOF'
---
ssh_port: 22
EOF
```
---
6.12 Фиксация результатов в Git
Зафиксируйте все созданные файлы в Git-репозитории:
```bash
cd ~/ansible

# Инициализация Git-репозитория (если ещё не создан)
git init
git add -A

# Проверка статуса
git status

# Коммит
git commit -m "Lab4: Jinja2 templates for chrony, sshd_config and MikroTik config"
```
Обоснование: Хранение всех конфигураций и шаблонов в Git обеспечивает версионность — можно отследить, когда и кем была изменена конфигурация, а также откатиться к предыдущей версии при необходимости. Это один из ключевых принципов Infrastructure as Code.
---
7. Итоговая структура проекта
После выполнения всех шагов структура проекта Ansible должна выглядеть следующим образом:
```
~/ansible/
├── hosts.ini                          # Файл инвентаризации (Linux + MikroTik)
├── linux_templates.yml                # Playbook для Linux-хостов
├── mikrotik_templates.yml             # Playbook для MikroTik
├── templates/
│   ├── chrony.conf.j2                 # Jinja2-шаблон chrony.conf
│   ├── sshd_config.j2                 # Jinja2-шаблон sshd_config
│   └── mikrotik_config.rsc.j2         # Jinja2-шаблон конфигурации MikroTik
├── group_vars/
│   ├── linux.yml                      # Переменные Linux-группы
│   └── mikrotik.yml                   # Переменные MikroTik-группы
└── host_vars/
    └── linuxcli1.yml                  # Индивидуальные переменные хоста (по желанию)
```
---
8. Контрольные вопросы
Что делает модуль `template` в Ansible и чем он отличается от модуля `copy`?
Чем Jinja2-шаблон (`.j2`) отличается от обычного конфигурационного файла?
Зачем нужны директории `group_vars/` и `host_vars/`? В каком случае лучше использовать `host_vars/`?
Что показывает режим `--check` (dry-run)? Какие ограничения имеет dry-run для MikroTik?
Что такое handler в Ansible и при каких условиях он выполняется?
Почему выгодно разделять данные (переменные) и логику (шаблоны)? Приведите практический пример.
Почему директива `Protocol 2` не используется в современном `sshd_config`?
Объясните назначение параметра `ansible_connection=local` для MikroTik в инвентаризации.
Что произойдёт, если в Jinja2-шаблоне `chrony.conf.j2` изменить переменную `ntp_servers`, добавив ещё один сервер? Нужно ли менять шаблон?
В чём принцип идемпотентности применительно к модулю `template`?
