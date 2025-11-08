# Лабораторная работа №5: Сетевые настройки и базовые сетевые сервисы

## Цель лабораторной работы
Освоить практические навыки настройки сетевых интерфейсов, управления сетевой безопасностью и развертывания базовых сетевых сервисов (chrony, nginx, samba) в Astra Linux.

## Задачи лабораторной работы
1. Настроить статическую и динамическую (DHCP) адресацию через конфигурационные файлы `/etc/network/interfaces.d/`.  
2. Управлять межсетевым экраном **iptables** или **nftables**: просматривать, добавлять и удалять правила, открывать порты.  
3. Установить и настроить сервер точного времени **chrony**.  
4. Развернуть веб-сервер **nginx** для раздачи статического контента.  
5. Настроить общий сетевой доступ к файлам с помощью **Samba**, создав открытый и закрытый ресурсы.

## Ожидаемый результат
Студент получает практические навыки настройки сетевых интерфейсов, конфигурации файервола, работы с базовыми службами сети и развертывания простых серверов. Освоено управление сетевыми сервисами и безопасность локальных соединений.

---

## Теоретическая часть

### 1. Настройка сетевых интерфейсов

В Astra Linux используется классическая схема настройки сети через файлы `/etc/network/interfaces` и `/etc/network/interfaces.d/`.

#### Пример настройки статического IP:
Файл `/etc/network/interfaces.d/eth0`:
```bash
auto eth0
iface eth0 inet static
    address 192.168.1.10
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8
```

#### Пример настройки DHCP:
```bash
auto eth0
iface eth0 inet dhcp
```

После изменения конфигурации:
```bash
sudo systemctl restart networking
```

Проверка подключения:
```bash
ip a
ping -c 3 8.8.8.8
```

---

### 2. Межсетевой экран: iptables / nftables

#### Проверка текущих правил:
```bash
sudo iptables -L -n -v
```

#### Добавление правила для открытия порта (например, 80/tcp):
```bash
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

#### Сохранение правил:
```bash
sudo iptables-save > /etc/iptables/rules.v4
```

#### Очистка правил:
```bash
sudo iptables -F
```

**Современная альтернатива:** `nftables`  
Проверка таблиц:
```bash
sudo nft list ruleset
```

Добавление простого правила:
```bash
sudo nft add rule inet filter input tcp dport 22 accept
```

Сохранение конфигурации:
```bash
sudo nft list ruleset > /etc/nftables.conf
sudo systemctl enable nftables
```

---

### 3. Настройка сервера точного времени chrony

Chrony синхронизирует время с интернет-серверами NTP.

#### Установка:
```bash
sudo apt install chrony -y
```

#### Основной конфигурационный файл:
`/etc/chrony/chrony.conf`

#### Пример конфигурации:
```bash
pool ntp.ubuntu.com iburst
allow 192.168.1.0/24
local stratum 10
```

Перезапуск службы:
```bash
sudo systemctl restart chronyd
```

Проверка статуса:
```bash
chronyc tracking
chronyc sources
```

---

### 4. Установка и настройка nginx

#### Установка:
```bash
sudo apt install nginx -y
```

#### Проверка статуса службы:
```bash
sudo systemctl status nginx
```

#### Файлы конфигурации:
- Основной файл: `/etc/nginx/nginx.conf`
- Сайты: `/etc/nginx/sites-available/` и `/etc/nginx/sites-enabled/`

#### Пример минимального сайта:
Создать файл `/var/www/html/index.html`:
```html
<html>
  <body>
    <h1>Добро пожаловать в Astra Linux!</h1>
  </body>
</html>
```

Перезапустить nginx:
```bash
sudo systemctl restart nginx
```

Проверить работу:
```bash
curl http://localhost
```

---

### 5. Настройка Samba

Samba позволяет организовать сетевой доступ к файлам для клиентов Windows и Linux.

#### Установка:
```bash
sudo apt install samba -y
```

#### Файл конфигурации:
`/etc/samba/smb.conf`

#### Пример конфигурации:
```ini
[public]
   path = /srv/samba/public
   browsable = yes
   guest ok = yes
   read only = no
   create mask = 0777

[private]
   path = /srv/samba/private
   valid users = admin
   read only = no
   create mask = 0700
```

Создать каталоги:
```bash
sudo mkdir -p /srv/samba/public /srv/samba/private
sudo chown -R nobody:nogroup /srv/samba/public
sudo chown -R admin:admin /srv/samba/private
```

Добавить пользователя в Samba:
```bash
sudo smbpasswd -a admin
```

Перезапустить службу:
```bash
sudo systemctl restart smbd
```

Проверить доступ:
```bash
smbclient -L localhost -U admin
```

---

## Практическое задание

1. Настройте интерфейс `eth0` со статическим IP-адресом `192.168.1.10`.  
2. Проверьте подключение к интернету (`ping`, `ip a`).  
3. Настройте DHCP и убедитесь, что адрес выдается автоматически.  
4. Добавьте правило в iptables, разрешающее входящие подключения на порт 80.  
5. Установите и настройте chrony, убедитесь в синхронизации времени.  
6. Разверните nginx и проверьте доступность веб-страницы через браузер.  
7. Настройте общий доступ к папке через Samba:  
   - общий ресурс `/srv/samba/public` (без пароля);  
   - закрытый `/srv/samba/private` (для пользователя `admin`).  
8. Задокументируйте все шаги, команды и выведенные результаты.

---

## Контрольные вопросы
1. Чем отличается статическая настройка сети от DHCP?  
2. Как просмотреть текущие правила iptables?  
3. Для чего используется chrony и как проверить его работу?  
4. Где хранятся файлы конфигурации nginx?  
5. Как ограничить доступ к сетевому ресурсу Samba?  

---

## Дополнительная литература
- [Документация по iptables и nftables](https://wiki.archlinux.org/title/Iptables)  
- [Chrony — официальная документация](https://chrony.tuxfamily.org/documentation.html)  
- [Nginx Beginner’s Guide](https://nginx.org/en/docs/beginners_guide.html)  
- [Настройка Samba](https://wiki.debian.org/Samba/ServerSimple)  
