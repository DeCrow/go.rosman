# ROSMan - RouterOS Manager (MikroTik)

[English version](readme.md) | [Русская версия](docs/readme_ru.md)  | [Español](docs/readme_es.md) | [Português](docs/readme_pt.md) | [Bahasa Indonesia](docs/readme_id.md) | [简体中文](docs/readme_zh.md)

Sistema automatizado de gerenciamento e backup de dispositivos MikroTik escrito em Go.

## Recursos

- **Gerenciamento centralizado de usuários e grupos** - sincronização automática de usuários e grupos de acesso em todos os dispositivos
- **Gerenciamento de agendamentos** - implantação e sincronização automática de tarefas agendadas com scripts RouterOS
- **Backups automáticos** - criação de arquivos de backup e exportações de configuração por agendamento
- **Chaves SSH** - upload e importação automática de chaves públicas SSH para usuários
- **Proteção contra exclusão** - mantém usuários do sistema especificados protegidos contra exclusão acidental
- **Intervalos de execução flexíveis** - suporte para vários intervalos: minutely, hourly, daily, weekly, monthly
- **Funciona via API, SSH e SFTP** - usa API do MikroTik para gerenciamento, SSH/SFTP para transferência de arquivos

## Requisitos

- Go 1.16+
- Acesso a dispositivos MikroTik via:
    - API (porta 8728 por padrão)
    - SSH (porta 22 por padrão)
- Usuário com privilégios completos nos dispositivos MikroTik

## Instalação

```bash
git clone <repository-url>
cd go.rosman
go mod download
go build -o rosman main.go
```

## Estrutura do Projeto

```
go.rosman/
├── main.go                      # Ponto de entrada
├── lib/
│   └── mikrotik/
│       └── mikrotik.go          # Biblioteca principal
├── configs/
│   └── mikrotik/
│       ├── main.json            # Configurações gerais de caminhos
│       ├── hosts.json           # Configuração de dispositivos
│       ├── users.json           # Usuários para sincronização
│       ├── groups.json          # Grupos de acesso
│       ├── schedules.json       # Agendamentos de tarefas
│       └── tasks.json           # Intervalos de execução
├── data/
│   ├── scripts/                 # Scripts RouterOS (.rsc)
│   └── keys/                    # Chaves públicas SSH (.pub)
└── backup/                      # Diretório de backups
```

## Configuração

### hosts.json

Lista de dispositivos gerenciados:

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

**Parâmetros:**
- `name` - nome do dispositivo
- `ip` - endereço IP
- `login/pass` - credenciais de conexão
- `port_api/port_ssh` - portas API e SSH
- `backup_folder` - pasta de backups no dispositivo
- `task_name` - intervalo de execução da tarefa (de tasks.json)
- `schedules_aliases` - lista de agendamentos para implantar
- `users_aliases` - lista de usuários para criar
- `users_allowed` - lista de usuários protegidos contra exclusão

### users.json

Usuários para sincronização:

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

**Parâmetros:**
- `alias` - alias para referência em hosts.json
- `key` - nome do arquivo de chave pública SSH (de data/keys/)
- `group` - grupo de acesso (de groups.json)

### groups.json

Grupos de acesso com políticas:

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

Agendamentos vinculados a scripts RouterOS:

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

**Parâmetros:**
- `alias` - alias para referência em hosts.json
- `script` - nome do arquivo de script (de data/scripts/)

### tasks.json

Intervalos de execução:

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

**Parâmetros:**
- `delay` - intervalo de execução principal (segundos)
- `expired` - intervalo de nova tentativa em caso de erro (segundos)

### main.json

Configurações gerais de caminhos:

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

Após a inicialização, o programa:

1. Carrega a configuração dos arquivos JSON
2. Inicia uma goroutine separada para cada dispositivo
3. Para cada dispositivo executa a seguinte sequência:
    - Remove usuários que não estão na lista `users_allowed`
    - Remove grupos que não estão presentes na configuração
    - Remove agendamentos que não estão presentes na configuração
    - Cria/atualiza grupos de acesso
    - Cria/atualiza usuários e faz upload de chaves SSH
    - Cria diretório de backups
    - Cria/atualiza agendamentos com scripts RouterOS
    - Baixa arquivos de backup do dispositivo
4. Dorme até o próximo tempo de execução de acordo com `task_name`

## Scripts RouterOS

O projeto inclui scripts de exemplo:

### backup_weekly.rsc
Cria backup de configuração (arquivo .backup):
```routeros
/system backup save name=backup/2024.12.23-020500;
```

### export_weekly.rsc
Exporta configuração para formato de texto (arquivo .rsc):
```routeros
/export show-sensitive terse file=backup/2024.12.23-021000;
```

### reboot_daily.rsc
Reinicia o dispositivo:
```routeros
/system/reboot
```

## Dependências

- `gopkg.in/routeros.v2` - Cliente API do MikroTik
- `golang.org/x/crypto/ssh` - Cliente SSH
- `github.com/pkg/sftp` - Cliente SFTP

## Licença

Copyright (c) 2023 Denis Decrow

A licença permite uso, cópia, modificação e distribuição desde que o autor original seja creditado. Veja o arquivo [LICENSE](LICENSE) para detalhes.