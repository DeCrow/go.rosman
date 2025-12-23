# ROSMan - RouterOS Manager (MikroTik)

[English version](readme.md) | [Русская версия](docs/readme_ru.md)  | [Español](docs/readme_es.md) | [Português](docs/readme_pt.md) | [Bahasa Indonesia](docs/readme_id.md) | [简体中文](docs/readme_zh.md)

Sistema automatizado de gestión y respaldo de dispositivos MikroTik escrito en Go.

## Características

- **Gestión centralizada de usuarios y grupos** - sincronización automática de usuarios y grupos de acceso en todos los dispositivos
- **Gestión de tareas programadas** - despliegue y sincronización automática de tareas programadas con scripts RouterOS
- **Respaldos automáticos** - creación de archivos de respaldo y exportaciones de configuración según calendario
- **Claves SSH** - carga e importación automática de claves públicas SSH para usuarios
- **Protección contra eliminación** - mantiene a salvo usuarios del sistema especificados de eliminación accidental
- **Intervalos de ejecución flexibles** - soporte para varios intervalos: minutely, hourly, daily, weekly, monthly
- **Funciona vía API, SSH y SFTP** - usa API de MikroTik para gestión, SSH/SFTP para transferencia de archivos

## Requisitos

- Go 1.16+
- Acceso a dispositivos MikroTik vía:
    - API (puerto 8728 por defecto)
    - SSH (puerto 22 por defecto)
- Usuario con privilegios completos en dispositivos MikroTik

## Instalación

```bash
git clone <repository-url>
cd go.rosman
go mod download
go build -o rosman main.go
```

## Estructura del Proyecto

```
go.rosman/
├── main.go                      # Punto de entrada
├── lib/
│   └── mikrotik/
│       └── mikrotik.go          # Biblioteca principal
├── configs/
│   └── mikrotik/
│       ├── main.json            # Configuración general de rutas
│       ├── hosts.json           # Configuración de dispositivos
│       ├── users.json           # Usuarios para sincronización
│       ├── groups.json          # Grupos de acceso
│       ├── schedules.json       # Tareas programadas
│       └── tasks.json           # Intervalos de ejecución
├── data/
│   ├── scripts/                 # Scripts RouterOS (.rsc)
│   └── keys/                    # Claves públicas SSH (.pub)
└── backup/                      # Directorio de respaldos
```

## Configuración

### hosts.json

Lista de dispositivos gestionados:

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

**Parámetros:**
- `name` - nombre del dispositivo
- `ip` - dirección IP
- `login/pass` - credenciales de conexión
- `port_api/port_ssh` - puertos API y SSH
- `backup_folder` - carpeta de respaldos en el dispositivo
- `task_name` - intervalo de ejecución de tareas (de tasks.json)
- `schedules_aliases` - lista de tareas programadas para desplegar
- `users_aliases` - lista de usuarios para crear
- `users_allowed` - lista de usuarios protegidos contra eliminación

### users.json

Usuarios para sincronización:

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

**Parámetros:**
- `alias` - alias para referencia en hosts.json
- `key` - nombre de archivo de clave pública SSH (de data/keys/)
- `group` - grupo de acceso (de groups.json)

### groups.json

Grupos de acceso con políticas:

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

Tareas programadas vinculadas a scripts RouterOS:

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

**Parámetros:**
- `alias` - alias para referencia en hosts.json
- `script` - nombre de archivo de script (de data/scripts/)

### tasks.json

Intervalos de ejecución:

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

**Parámetros:**
- `delay` - intervalo de ejecución principal (segundos)
- `expired` - intervalo de reintento en caso de error (segundos)

### main.json

Configuración general de rutas:

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

## Uso

```bash
./rosman
```

Después del inicio, el programa:

1. Carga la configuración desde archivos JSON
2. Inicia una goroutine separada para cada dispositivo
3. Para cada dispositivo realiza la siguiente secuencia:
    - Elimina usuarios que no están en la lista `users_allowed`
    - Elimina grupos que no están presentes en la configuración
    - Elimina tareas programadas que no están presentes en la configuración
    - Crea/actualiza grupos de acceso
    - Crea/actualiza usuarios y carga claves SSH
    - Crea directorio de respaldos
    - Crea/actualiza tareas programadas con scripts RouterOS
    - Descarga archivos de respaldo del dispositivo
4. Duerme hasta el próximo tiempo de ejecución según `task_name`

## Scripts RouterOS

El proyecto incluye scripts de ejemplo:

### backup_weekly.rsc
Crea respaldo de configuración (archivo .backup):
```routeros
/system backup save name=backup/2024.12.23-020500;
```

### export_weekly.rsc
Exporta configuración a formato de texto (archivo .rsc):
```routeros
/export show-sensitive terse file=backup/2024.12.23-021000;
```

### reboot_daily.rsc
Reinicia el dispositivo:
```routeros
/system/reboot
```

## Dependencias

- `gopkg.in/routeros.v2` - Cliente API de MikroTik
- `golang.org/x/crypto/ssh` - Cliente SSH
- `github.com/pkg/sftp` - Cliente SFTP

## Licencia

Copyright (c) 2023 Denis Decrow

La licencia permite uso, copia, modificación y distribución siempre que se acredite al autor original. Ver archivo [LICENSE](LICENSE) para detalles.