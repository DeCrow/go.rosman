# ROSMan - RouterOS Manager (MikroTik)

[English version](/readme.md) | [Русская версия](/docs/readme_ru.md)  | [Español](/docs/readme_es.md) | [Português](/docs/readme_pt.md) | [Bahasa Indonesia](/docs/readme_id.md) | [简体中文](/docs/readme_zh.md)

Автоматизированная система управления и резервного копирования устройств MikroTik на Go.

## Возможности

- **Централизованное управление пользователями и группами** - автоматическая синхронизация пользователей и групп доступа на всех устройствах
- **Управление расписаниями** - автоматическое развертывание и синхронизация scheduled tasks с RouterOS скриптами
- **Автоматическое резервное копирование** - создание backup файлов и export конфигураций по расписанию
- **SSH ключи** - автоматическая загрузка и импорт публичных SSH ключей для пользователей
- **Защита от удаления** - сохраняет указанных системных пользователей от случайного удаления
- **Гибкие интервалы выполнения** - поддержка различных интервалов: minutely, hourly, daily, weekly, monthly
- **Работа через API, SSH и SFTP** - использование MikroTik API для управления, SSH/SFTP для передачи файлов

## Требования

- Go 1.16+
- Доступ к MikroTik устройствам через:
    - API (порт 8728 по умолчанию)
    - SSH (порт 22 по умолчанию)
- Пользователь с полными правами на устройствах MikroTik

## Установка

```bash
git clone <repository-url>
cd go.rosman
go mod download
go build -o rosman main.go
```

## Структура проекта

```
go.rosman/
├── main.go                      # Точка входа
├── lib/
│   └── mikrotik/
│       └── mikrotik.go          # Основная библиотека
├── configs/
│   └── mikrotik/
│       ├── main.json            # Общие настройки путей
│       ├── hosts.json           # Конфигурация устройств
│       ├── users.json           # Пользователи для синхронизации
│       ├── groups.json          # Группы доступа
│       ├── schedules.json       # Расписания задач
│       └── tasks.json           # Интервалы выполнения
├── data/
│   ├── scripts/                 # RouterOS скрипты (.rsc)
│   └── keys/                    # SSH публичные ключи (.pub)
└── backup/                      # Директория для резервных копий
```

## Конфигурация

### hosts.json

Список управляемых устройств:

```json
[
  {
    "name": "Mikrotik 1",
    "ip": "172.24.0.1",
    "login": "admin",
    "pass": "password",
    "port_api": 8728,
    "port_ssh": 22,
    "backup_folder": "backup",
    "task_name": "hourly",
    "schedules_aliases": ["export_weekly", "backup_weekly"],
    "users_aliases": ["teamlead"],
    "users_allowed": ["admin"]
  }
]
```

**Параметры:**
- `name` - имя устройства
- `ip` - IP адрес
- `login/pass` - учетные данные для подключения
- `port_api/port_ssh` - порты API и SSH
- `backup_folder` - папка для бэкапов на устройстве
- `task_name` - интервал выполнения задачи (из tasks.json)
- `schedules_aliases` - список расписаний для развертывания
- `users_aliases` - список пользователей для создания
- `users_allowed` - список пользователей, защищенных от удаления

### users.json

Пользователи для синхронизации:

```json
[
  {
    "login": "teamlead.login",
    "pass": "password",
    "group": "full",
    "address": "",
    "comment": "Teamlead User",
    "alias": "teamlead",
    "key": "example.pub"
  }
]
```

**Параметры:**
- `alias` - псевдоним для ссылки в hosts.json
- `key` - имя файла публичного SSH ключа (из data/keys/)
- `group` - группа доступа (из groups.json)

### groups.json

Группы доступа с политиками:

```json
[
  {
    "name": "full",
    "skin": "default",
    "comment": "Admin group",
    "policy": "local,telnet,ssh,reboot,read,test,winbox,password,web,sniff,sensitive,api,romon,rest-api,ftp,write,policy"
  }
]
```

### schedules.json

Расписания с привязкой к RouterOS скриптам:

```json
[
  {
    "name": "backup_weekly",
    "disabled": "false",
    "start-date": "dec/07/2022",
    "start-time": "02:05:00",
    "interval": "7d 00:00:00",
    "policy": "ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon",
    "comment": "[API] weekly backup",
    "script": "backup_weekly.rsc",
    "alias": "backup_weekly"
  }
]
```

**Параметры:**
- `alias` - псевдоним для ссылки в hosts.json
- `script` - имя файла скрипта (из data/scripts/)

### tasks.json

Интервалы выполнения:

```json
[
  {
    "name": "hourly",
    "start": 13035600,
    "delay": 3600,
    "expired": 600,
    "alert": 0,
    "note": "Hourly. If unsuccessful, repeat every 10 minutes."
  }
]
```

**Параметры:**
- `delay` - основной интервал выполнения (секунды)
- `expired` - интервал повтора при ошибке (секунды)

### main.json

Общие настройки путей:

```json
[
  {
    "name": "dir_scripts",
    "value": "data/scripts/",
    "note": "Directory with RouterOS scripts"
  },
  {
    "name": "dir_ssh-pub-keys",
    "value": "data/keys/",
    "note": "Directory with public SSH keys"
  },
  {
    "name": "dir_backup",
    "value": "backup/[{host.ip}] {host.name}",
    "note": "Backup directory"
  }
]
```

## Использование

```bash
./rosman
```

После запуска программа:

1. Загружает конфигурацию из JSON файлов
2. Запускает отдельную горутину для каждого устройства
3. Для каждого устройства выполняет последовательность действий:
    - Удаляет пользователей не из списка `users_allowed`
    - Удаляет группы отсутствующие в конфигурации
    - Удаляет расписания отсутствующие в конфигурации
    - Создает/обновляет группы доступа
    - Создает/обновляет пользователей и загружает SSH ключи
    - Создает директорию для backup
    - Создает/обновляет расписания с RouterOS скриптами
    - Загружает backup файлы с устройства
4. Засыпает до следующего времени выполнения согласно `task_name`

## RouterOS скрипты

Проект включает примеры скриптов:

### backup_weekly.rsc
Создает резервную копию конфигурации (.backup файл):
```routeros
/system backup save name=backup/2024.12.23-020500;
```

### export_weekly.rsc
Экспортирует конфигурацию в текстовый формат (.rsc файл):
```routeros
/export show-sensitive terse file=backup/2024.12.23-021000;
```

### reboot_daily.rsc
Перезагружает устройство:
```routeros
/system/reboot
```

## Зависимости

- `gopkg.in/routeros.v2` - клиент MikroTik API
- `golang.org/x/crypto/ssh` - SSH клиент
- `github.com/pkg/sftp` - SFTP клиент

## Лицензия

Copyright (c) 2023 Denis Decrow

Лицензия разрешает использование, копирование, модификацию и распространение при условии указания оригинального автора. См. файл [LICENSE](/LICENSE) для деталей.