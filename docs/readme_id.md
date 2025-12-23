# ROSMan - RouterOS Manager (MikroTik)

[English version](/readme.md) | [Русская версия](/docs/readme_ru.md)  | [Español](/docs/readme_es.md) | [Português](/docs/readme_pt.md) | [Bahasa Indonesia](/docs/readme_id.md) | [简体中文](/docs/readme_zh.md)

Sistem otomatis untuk manajemen dan backup perangkat MikroTik yang ditulis dalam Go.

## Fitur

- **Manajemen pengguna dan grup terpusat** - sinkronisasi otomatis pengguna dan grup akses di semua perangkat
- **Manajemen jadwal** - deployment dan sinkronisasi otomatis tugas terjadwal dengan skrip RouterOS
- **Backup otomatis** - pembuatan file backup dan ekspor konfigurasi sesuai jadwal
- **Kunci SSH** - upload dan impor otomatis kunci publik SSH untuk pengguna
- **Perlindungan penghapusan** - menjaga pengguna sistem tertentu dari penghapusan yang tidak disengaja
- **Interval eksekusi fleksibel** - dukungan untuk berbagai interval: minutely, hourly, daily, weekly, monthly
- **Bekerja melalui API, SSH dan SFTP** - menggunakan API MikroTik untuk manajemen, SSH/SFTP untuk transfer file

## Persyaratan

- Go 1.16+
- Akses ke perangkat MikroTik melalui:
    - API (port 8728 secara default)
    - SSH (port 22 secara default)
- Pengguna dengan hak akses penuh pada perangkat MikroTik

## Instalasi

```bash
git clone <repository-url>
cd go.rosman
go mod download
go build -o rosman main.go
```

## Struktur Proyek

```
go.rosman/
├── main.go                      # Entry point
├── lib/
│   └── mikrotik/
│       └── mikrotik.go          # Library utama
├── configs/
│   └── mikrotik/
│       ├── main.json            # Pengaturan path umum
│       ├── hosts.json           # Konfigurasi perangkat
│       ├── users.json           # Pengguna untuk sinkronisasi
│       ├── groups.json          # Grup akses
│       ├── schedules.json       # Jadwal tugas
│       └── tasks.json           # Interval eksekusi
├── data/
│   ├── scripts/                 # Skrip RouterOS (.rsc)
│   └── keys/                    # Kunci publik SSH (.pub)
└── backup/                      # Direktori backup
```

## Konfigurasi

### hosts.json

Daftar perangkat yang dikelola:

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

**Parameter:**
- `name` - nama perangkat
- `ip` - alamat IP
- `login/pass` - kredensial koneksi
- `port_api/port_ssh` - port API dan SSH
- `backup_folder` - folder backup di perangkat
- `task_name` - interval eksekusi tugas (dari tasks.json)
- `schedules_aliases` - daftar jadwal untuk di-deploy
- `users_aliases` - daftar pengguna untuk dibuat
- `users_allowed` - daftar pengguna yang dilindungi dari penghapusan

### users.json

Pengguna untuk sinkronisasi:

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

**Parameter:**
- `alias` - alias untuk referensi di hosts.json
- `key` - nama file kunci publik SSH (dari data/keys/)
- `group` - grup akses (dari groups.json)

### groups.json

Grup akses dengan kebijakan:

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

Jadwal yang terhubung dengan skrip RouterOS:

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

**Parameter:**
- `alias` - alias untuk referensi di hosts.json
- `script` - nama file skrip (dari data/scripts/)

### tasks.json

Interval eksekusi:

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

**Parameter:**
- `delay` - interval eksekusi utama (detik)
- `expired` - interval coba ulang saat error (detik)

### main.json

Pengaturan path umum:

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

## Penggunaan

```bash
./rosman
```

Setelah startup, program akan:

1. Memuat konfigurasi dari file JSON
2. Memulai goroutine terpisah untuk setiap perangkat
3. Untuk setiap perangkat melakukan urutan berikut:
    - Menghapus pengguna yang tidak ada dalam daftar `users_allowed`
    - Menghapus grup yang tidak ada dalam konfigurasi
    - Menghapus jadwal yang tidak ada dalam konfigurasi
    - Membuat/memperbarui grup akses
    - Membuat/memperbarui pengguna dan meng-upload kunci SSH
    - Membuat direktori backup
    - Membuat/memperbarui jadwal dengan skrip RouterOS
    - Mengunduh file backup dari perangkat
4. Tidur hingga waktu eksekusi berikutnya sesuai dengan `task_name`

## Skrip RouterOS

Proyek ini mencakup contoh skrip:

### backup_weekly.rsc
Membuat backup konfigurasi (file .backup):
```routeros
/system backup save name=backup/2024.12.23-020500;
```

### export_weekly.rsc
Mengekspor konfigurasi ke format teks (file .rsc):
```routeros
/export show-sensitive terse file=backup/2024.12.23-021000;
```

### reboot_daily.rsc
Me-reboot perangkat:
```routeros
/system/reboot
```

## Dependensi

- `gopkg.in/routeros.v2` - Klien API MikroTik
- `golang.org/x/crypto/ssh` - Klien SSH
- `github.com/pkg/sftp` - Klien SFTP

## Lisensi

Copyright (c) 2023 Denis Decrow

Lisensi mengizinkan penggunaan, penyalinan, modifikasi, dan distribusi dengan syarat penulis asli disebutkan. Lihat file [LICENSE](/LICENSE) untuk detail.