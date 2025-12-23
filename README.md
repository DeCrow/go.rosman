# ROSMan - RouterOS Manager (MikroTik)

[English version](/readme.md) | [Русская версия](/docs/readme_ru.md)  | [Español](/docs/readme_es.md) | [Português](/docs/readme_pt.md) | [Bahasa Indonesia](/docs/readme_id.md) | [简体中文](/docs/readme_zh.md)

Automated management and backup system for MikroTik devices written in Go.

## Features

- **Centralized user and group management** - automatic synchronization of users and access groups across all devices
- **Schedule management** - automatic deployment and synchronization of scheduled tasks with RouterOS scripts
- **Automatic backups** - creation of backup files and configuration exports on schedule
- **SSH keys** - automatic upload and import of public SSH keys for users
- **Deletion protection** - keeps specified system users safe from accidental deletion
- **Flexible execution intervals** - support for various intervals: minutely, hourly, daily, weekly, monthly
- **Works via API, SSH and SFTP** - uses MikroTik API for management, SSH/SFTP for file transfer

## Requirements

- Go 1.16+
- Access to MikroTik devices via:
    - API (port 8728 by default)
    - SSH (port 22 by default)
- User with full privileges on MikroTik devices

## Installation

```bash
git clone <repository-url>
cd go.rosman
go mod download
go build -o rosman main.go
```

## Project Structure

```
go.rosman/
├── main.go                      # Entry point
├── lib/
│   └── mikrotik/
│       └── mikrotik.go          # Main library
├── configs/
│   └── mikrotik/
│       ├── main.json            # General path settings
│       ├── hosts.json           # Device configuration
│       ├── users.json           # Users for synchronization
│       ├── groups.json          # Access groups
│       ├── schedules.json       # Task schedules
│       └── tasks.json           # Execution intervals
├── data/
│   ├── scripts/                 # RouterOS scripts (.rsc)
│   └── keys/                    # SSH public keys (.pub)
└── backup/                      # Backup directory
```

## Configuration

### hosts.json

List of managed devices:

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

**Parameters:**
- `name` - device name
- `ip` - IP address
- `login/pass` - connection credentials
- `port_api/port_ssh` - API and SSH ports
- `backup_folder` - backup folder on device
- `task_name` - task execution interval (from tasks.json)
- `schedules_aliases` - list of schedules to deploy
- `users_aliases` - list of users to create
- `users_allowed` - list of users protected from deletion

### users.json

Users for synchronization:

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

**Parameters:**
- `alias` - alias for reference in hosts.json
- `key` - public SSH key filename (from data/keys/)
- `group` - access group (from groups.json)

### groups.json

Access groups with policies:

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

Schedules linked to RouterOS scripts:

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

**Parameters:**
- `alias` - alias for reference in hosts.json
- `script` - script filename (from data/scripts/)

### tasks.json

Execution intervals:

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

**Parameters:**
- `delay` - main execution interval (seconds)
- `expired` - retry interval on error (seconds)

### main.json

General path settings:

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

## Usage

```bash
./rosman
```

After startup, the program:

1. Loads configuration from JSON files
2. Starts a separate goroutine for each device
3. For each device performs the following sequence:
    - Removes users not in the `users_allowed` list
    - Removes groups not present in configuration
    - Removes schedules not present in configuration
    - Creates/updates access groups
    - Creates/updates users and uploads SSH keys
    - Creates backup directory
    - Creates/updates schedules with RouterOS scripts
    - Downloads backup files from device
4. Sleeps until next execution time according to `task_name`

## RouterOS Scripts

The project includes example scripts:

### backup_weekly.rsc
Creates configuration backup (.backup file):
```routeros
/system backup save name=backup/2024.12.23-020500;
```

### export_weekly.rsc
Exports configuration to text format (.rsc file):
```routeros
/export show-sensitive terse file=backup/2024.12.23-021000;
```

### reboot_daily.rsc
Reboots the device:
```routeros
/system/reboot
```

## Dependencies

- `gopkg.in/routeros.v2` - MikroTik API client
- `golang.org/x/crypto/ssh` - SSH client
- `github.com/pkg/sftp` - SFTP client

## License

Copyright (c) 2023 Denis Decrow

The license permits use, copying, modification and distribution provided the original author is credited. See [LICENSE](LICENSE) file for details.