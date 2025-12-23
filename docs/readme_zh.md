# ROSMan - RouterOS Manager (MikroTik)

[English version](readme.md) | [Русская версия](docs/readme_ru.md)  | [Español](docs/readme_es.md) | [Português](docs/readme_pt.md) | [Bahasa Indonesia](docs/readme_id.md) | [简体中文](docs/readme_zh.md)

使用 Go 语言编写的 MikroTik 设备自动化管理和备份系统。

## 功能特性

- **集中式用户和组管理** - 自动同步所有设备上的用户和访问组
- **计划任务管理** - 使用 RouterOS 脚本自动部署和同步计划任务
- **自动备份** - 按计划创建备份文件和配置导出
- **SSH 密钥** - 自动上传和导入用户的 SSH 公钥
- **删除保护** - 保护指定的系统用户免遭意外删除
- **灵活的执行间隔** - 支持多种间隔：每分钟、每小时、每天、每周、每月
- **通过 API、SSH 和 SFTP 工作** - 使用 MikroTik API 进行管理，SSH/SFTP 进行文件传输

## 系统要求

- Go 1.16+
- 通过以下方式访问 MikroTik 设备：
    - API（默认端口 8728）
    - SSH（默认端口 22）
- 在 MikroTik 设备上具有完全权限的用户

## 安装

```bash
git clone <repository-url>
cd go.rosman
go mod download
go build -o rosman main.go
```

## 项目结构

```
go.rosman/
├── main.go                      # 入口点
├── lib/
│   └── mikrotik/
│       └── mikrotik.go          # 主库
├── configs/
│   └── mikrotik/
│       ├── main.json            # 通用路径设置
│       ├── hosts.json           # 设备配置
│       ├── users.json           # 同步用户
│       ├── groups.json          # 访问组
│       ├── schedules.json       # 任务计划
│       └── tasks.json           # 执行间隔
├── data/
│   ├── scripts/                 # RouterOS 脚本 (.rsc)
│   └── keys/                    # SSH 公钥 (.pub)
└── backup/                      # 备份目录
```

## 配置

### hosts.json

管理设备列表：

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

**参数：**
- `name` - 设备名称
- `ip` - IP 地址
- `login/pass` - 连接凭据
- `port_api/port_ssh` - API 和 SSH 端口
- `backup_folder` - 设备上的备份文件夹
- `task_name` - 任务执行间隔（来自 tasks.json）
- `schedules_aliases` - 要部署的计划列表
- `users_aliases` - 要创建的用户列表
- `users_allowed` - 受保护不被删除的用户列表

### users.json

同步用户：

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

**参数：**
- `alias` - 在 hosts.json 中引用的别名
- `key` - SSH 公钥文件名（来自 data/keys/）
- `group` - 访问组（来自 groups.json）

### groups.json

具有策略的访问组：

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

链接到 RouterOS 脚本的计划：

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

**参数：**
- `alias` - 在 hosts.json 中引用的别名
- `script` - 脚本文件名（来自 data/scripts/）

### tasks.json

执行间隔：

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

**参数：**
- `delay` - 主要执行间隔（秒）
- `expired` - 错误时的重试间隔（秒）

### main.json

通用路径设置：

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

## 使用方法

```bash
./rosman
```

启动后，程序将：

1. 从 JSON 文件加载配置
2. 为每个设备启动单独的协程
3. 对于每个设备执行以下操作序列：
    - 删除不在 `users_allowed` 列表中的用户
    - 删除配置中不存在的组
    - 删除配置中不存在的计划
    - 创建/更新访问组
    - 创建/更新用户并上传 SSH 密钥
    - 创建备份目录
    - 使用 RouterOS 脚本创建/更新计划
    - 从设备下载备份文件
4. 根据 `task_name` 休眠直到下次执行时间

## RouterOS 脚本

项目包含示例脚本：

### backup_weekly.rsc
创建配置备份（.backup 文件）：
```routeros
/system backup save name=backup/2024.12.23-020500;
```

### export_weekly.rsc
导出配置到文本格式（.rsc 文件）：
```routeros
/export show-sensitive terse file=backup/2024.12.23-021000;
```

### reboot_daily.rsc
重启设备：
```routeros
/system/reboot
```

## 依赖项

- `gopkg.in/routeros.v2` - MikroTik API 客户端
- `golang.org/x/crypto/ssh` - SSH 客户端
- `github.com/pkg/sftp` - SFTP 客户端

## 许可证

版权所有 (c) 2023 Denis Decrow

该许可证允许使用、复制、修改和分发，前提是注明原作者。详见 [LICENSE](LICENSE) 文件。