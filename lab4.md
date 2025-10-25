# Лабораторная работа №4: Управление системой и службами с помощью systemd

## Цель лабораторной работы
Освоить основы работы с системой инициализации **systemd** в Astra Linux: управление службами, анализ системных журналов, диагностика проблем при загрузке и создание собственных сервисов.

## Задачи лабораторной работы
1. Изучить команды управления службами с помощью **systemctl** (start, stop, restart, enable, disable, status).  
2. Научиться анализировать журналы с помощью **journalctl**, выполнять фильтрацию по имени службы, времени и приоритету сообщений.  
3. Создать и протестировать простую собственную службу systemd для пользовательского скрипта.  
4. Ознакомиться с работой контейнеров на базе **systemd-nspawn**.  
5. Изучить систему целей (runlevels) в systemd: получение и изменение цели загрузки, диагностика проблем при старте системы.  

## Ожидаемый результат
Студент овладевает практическими навыками управления службами и процессами через systemd, анализом логов с journalctl и созданием собственных системных сервисов. Получено понимание механизмов загрузки системы, целей (targets) и диагностики сбоев.

---

## Теоретическая часть

### 1. Общие сведения о systemd
**systemd** — современная система инициализации и управления службами в Linux. Она заменяет старую систему SysVinit и управляет всеми процессами, службами, сокетами, устройствами и точками монтирования.

**Основные элементы systemd:**
- **Службы (services)** — процессы, запускаемые при старте системы.
- **Цели (targets)** — наборы служб, аналог уровней загрузки (runlevels).
- **Юниты (units)** — конфигурационные объекты (службы, таймеры, сокеты, монтирования и т. д.).

Файлы юнитов хранятся в:
- `/usr/lib/systemd/system/` — системные юниты (по умолчанию);
- `/etc/systemd/system/` — пользовательские и переопределенные юниты.

---

### 2. Управление службами с помощью systemctl

**Основные команды:**
```bash
sudo systemctl start ssh       # Запустить службу
sudo systemctl stop ssh        # Остановить службу
sudo systemctl restart ssh     # Перезапустить
sudo systemctl status ssh      # Проверить состояние
sudo systemctl enable ssh      # Включить автозапуск
sudo systemctl disable ssh     # Отключить автозапуск
```

**Просмотр всех активных служб:**
```bash
systemctl list-units --type=service
```

**Проверка состояния systemd:**
```bash
systemctl is-system-running
```

---

### 3. Анализ логов с journalctl
systemd использует двоичный журнал логов, управляемый службой **journald**.

**Основные команды:**
```bash
sudo journalctl -xe                  # Просмотр последних ошибок
sudo journalctl -u ssh.service       # Логи конкретной службы
sudo journalctl --since "1 hour ago" # Логи за последний час
sudo journalctl -p err               # Только ошибки
```

**Постоянное хранение логов:**
По умолчанию логи временные. Чтобы сделать их постоянными:
```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

---

### 4. Создание собственной службы systemd

#### 4.1. Пример скрипта
Создайте простой скрипт:
```bash
sudo nano /usr/local/bin/hello.sh
```

Содержимое:
```bash
#!/bin/bash
echo "Hello, systemd! $(date)" >> /var/log/hello.log
```

Сделайте исполняемым:
```bash
sudo chmod +x /usr/local/bin/hello.sh
```

#### 4.2. Создание юнита
Создайте файл службы:
```bash
sudo nano /etc/systemd/system/hello.service
```

Содержимое:
```ini
[Unit]
Description=Пример пользовательской службы

[Service]
ExecStart=/usr/local/bin/hello.sh
Type=oneshot

[Install]
WantedBy=multi-user.target
```

#### 4.3. Проверка
```bash
sudo systemctl daemon-reload
sudo systemctl start hello.service
sudo systemctl status hello.service
```

Проверка результата:
```bash
cat /var/log/hello.log
```

Для автозапуска при загрузке:
```bash
sudo systemctl enable hello.service
```

---

### 5. Работа с systemd-nspawn (ознакомительно)
**systemd-nspawn** — инструмент для создания легких контейнеров (аналог chroot, но с поддержкой namespace).

Создание контейнера:
```bash
sudo debootstrap stable /var/lib/machines/demo
sudo systemd-nspawn -D /var/lib/machines/demo
```
В случае с Astra репозиторий stable резила не найдется, он должен быть заменен на вашу версию системы, посмотреть ее можно выполнив команду `lsb_release -a`

Запуск контейнера как службы:
```bash
sudo systemctl start systemd-nspawn@demo
```

Выход из контейнера — `Ctrl+]`.

---

### 6. Изучение целей загрузки
**Цели (targets)** в systemd аналогичны runlevels из SysVinit.

| Цель | Назначение |
|------|-------------|
| `graphical.target` | Графический интерфейс (аналог runlevel 5) |
| `multi-user.target` | Консольный режим (аналог runlevel 3) |
| `rescue.target` | Однопользовательский режим |
| `emergency.target` | Минимальный аварийный режим |

**Просмотр текущей цели:**
```bash
systemctl get-default
```

**Изменение цели по умолчанию:**
```bash
sudo systemctl set-default multi-user.target
```

**Переключение цели без перезагрузки:**
```bash
sudo systemctl isolate rescue.target
```

---

## Практическое задание

1. **Управление службами:**
   - Просмотрите список активных служб.
   - Остановите и запустите службу `cron`.
   - Включите и отключите автозапуск SSH.

2. **Анализ логов:**
   - Посмотрите последние записи службы `ssh`.
   - Отфильтруйте ошибки за последний час.
   - Настройте постоянное хранение логов.

3. **Создание собственной службы:**
   - Напишите скрипт, который записывает текущее время в `/var/log/time.log`.
   - Создайте для него службу systemd и включите автозапуск.
   - Проверьте выполнение после перезапуска системы.

4. **Работа с целями:**
   - Определите текущую цель загрузки.
   - Переключитесь в `rescue.target`, затем вернитесь в обычный режим.
   - Измените цель по умолчанию на `multi-user.target`.

5. **(Дополнительно)** Ознакомьтесь с `systemd-nspawn` и запустите контейнер из минимального окружения.

---

## Контрольные вопросы
1. Чем systemd отличается от SysVinit?  
2. Какие типы юнитов поддерживает systemd?  
3. Как создать и активировать пользовательскую службу?  
4. Как просмотреть журнал конкретной службы?  
5. Что такое цели (targets) и как их изменить?  

---

## Дополнительная литература
- [Официальная документация systemd](https://systemd.io/)
- [Руководство по journalctl](https://man7.org/linux/man-pages/man1/journalctl.1.html)
- [systemd-nspawn — контейнеризация](https://wiki.archlinux.org/title/Systemd-nspawn)
