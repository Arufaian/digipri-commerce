# Tutorial Lengkap: Laravel Sail dengan MySQL & phpMyAdmin

## Daftar Isi
1. [Pengantar](#pengantar)
2. [Instalasi & Setup](#instalasi--setup)
3. [Migrasi dari SQLite ke MySQL](#migrasi-dari-sqlite-ke-mysql)
4. [Common Commands](#common-commands)
5. [Development Workflow](#development-workflow)
6. [Debugging & Troubleshooting](#debugging--troubleshooting)
7. [phpMyAdmin Guide](#phpmyadmin-guide)
8. [Best Practices & Tips](#best-practices--tips)

---

## Pengantar

### Apa itu Docker dan Containers?

Docker adalah sebuah teknologi yang memungkinkan Anda menjalankan aplikasi dalam "kontainer" - bayangkan seperti kontainer pengiriman, tapi untuk aplikasi. Setiap kontainer berisi:

- **Aplikasi Anda** (Laravel)
- **Database** (MySQL)
- **Semua Dependencies** yang dibutuhkan
- **Semua Konfigurasi**

Keuntungan utama:
- ✅ Konsistensi: Kode berjalan sama di lokal, testing, dan production
- ✅ Isolasi: Tidak ada konflik dengan aplikasi lain di komputer Anda
- ✅ Easy Cleanup: Hapus container dan tidak ada jejak yang tertinggal
- ✅ Kolaborasi: Rekan kerja mendapat environment yang sama persis

### Apa itu Laravel Sail?

Laravel Sail adalah interface sederhana (command-line) yang membuat Docker mudah digunakan untuk Laravel developers. Sail menghilangkan kompleksitas Docker dan memberikan Anda perintah-perintah simple seperti `sail up`, `sail down`, `sail artisan migrate`.

**Tanpa Sail**, Anda harus menulis perintah Docker yang panjang dan kompleks. **Dengan Sail**, semuanya menjadi simple dan mudah diingat.

### Mengapa Menggunakan Sail?

1. **Tidak perlu install PHP, MySQL, Node lokal** - Semuanya di Docker
2. **Konsisten di semua developer** - Tidak ada "tapi di komputer saya berjalan baik"
3. **Easy cleanup** - Hapus container dan database dengan satu perintah
4. **Multiple projects** - Jalankan berbagai versi PHP/MySQL untuk proyek berbeda tanpa conflict
5. **Production-like** - Environment lokal mirip dengan production

### Persyaratan

Sebelum memulai, pastikan Anda memiliki:

- ✅ **Docker Desktop** (versi terbaru) - Download dari [docker.com](https://www.docker.com/products/docker-desktop)
- ✅ **Git** - Untuk version control
- ✅ **Laravel project** yang sudah ada atau ingin dibuat baru
- ✅ **Terminal/Command Prompt** yang nyaman digunakan
- ✅ **Minimal 4GB RAM** untuk Docker

**Cek instalasi Docker:**
```bash
docker --version
# Output: Docker version 27.x.x atau lebih baru
```

---

## Instalasi & Setup

### Step 1: Pastikan Docker Desktop Berjalan

Buka Docker Desktop dan tunggu sampai status menunjukkan "Docker is running" (biasanya simbol di taskbar/menu bar berubah menjadi biru).

### Step 2: Install Laravel Sail ke Project Existing

Jika project Anda sudah ada (seperti kasus Anda dengan project baru yang inisiasi), install Sail via Composer:

```bash
cd /path/to/your/project
composer require laravel/sail --dev
```

**Output yang diharapkan:**
```
Using version ^1.53 for laravel/sail
./composer.json has been updated
Running composer update laravel/sail
...
```

### Step 3: Jalankan Sail Installation Command

```bash
php artisan sail:install
```

Perintah ini akan:
- ✅ Membuat file `compose.yaml` (konfigurasi Docker untuk project Anda)
- ✅ Membuat folder `docker/` dengan Dockerfile dan konfigurasi
- ✅ Update `.env` dengan konfigurasi database dan services yang sesuai

**Output yang diharapkan:**
```
Publishing the Sail docker files...
Publishing the Sail configuration file...
```

### Step 4: Konfigurasi MySQL dan phpMyAdmin di compose.yaml

File `compose.yaml` sekarang ada di root project Anda. Kita perlu menambahkan service phpMyAdmin. Buka file tersebut dan temukan section `services`:

```yaml
services:
  laravel.test:
    # ...konfigurasi app
    
  mysql:
    # ...konfigurasi MySQL yang sudah ada
    
  # TAMBAHKAN SECTION INI untuk phpMyAdmin:
  phpmyadmin:
    image: phpmyadmin:latest
    container_name: sail-phpmyadmin
    environment:
      PMA_HOST: mysql
      PMA_USER: root
      PMA_PASSWORD: "${DB_PASSWORD}"
      PMA_PORT: 3306
    ports:
      - "8080:80"
    depends_on:
      - mysql
    networks:
      - sail
```

💡 **Penjelasan:**
- `PMA_HOST: mysql` - phpMyAdmin terhubung ke service MySQL
- `ports: - "8080:80"` - Akses phpMyAdmin di `http://localhost:8080`
- `PMA_PASSWORD: "${DB_PASSWORD}"` - Menggunakan password dari `.env`

### Step 5: Update .env untuk MySQL

Buka file `.env` Anda dan pastikan konfigurasi database sudah benar:

```env
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=password
```

💡 **Catatan penting:**
- `DB_HOST=mysql` bukan `localhost` - ini hostname service MySQL di Docker
- `DB_PASSWORD` bisa diubah sesuai keinginan (ingat untuk phpMyAdmin nanti)
- Database `laravel` akan dibuat otomatis saat container pertama kali start

### Step 6: Mulai Docker Containers

```bash
sail up -d
```

🔄 **Proses ini akan:**
1. Download Docker images (MySQL, PHP, Node, dll) - hanya pertama kali
2. Build image untuk Laravel app Anda
3. Start semua containers di background (`-d` = detached mode)

**Tunggu 1-2 menit** untuk semua services siap.

### Step 7: Verifikasi Setup Berhasil

Periksa apakah semua containers berjalan:

```bash
sail ps
```

**Output yang diharapkan:**
```
CONTAINER ID   IMAGE                    COMMAND                  STATUS
xxxxx          sail-8.5.3/app           "start-container"        Up 2 minutes
xxxxx          mysql:8                  "docker-entrypoint.sh"   Up 2 minutes
xxxxx          phpmyadmin:latest        "docker-php-entrypoi"    Up 2 minutes
xxxxx          redis:7-alpine           "redis-server --ap"      Up 2 minutes
```

### Step 8: Jalankan Migrations

```bash
sail artisan migrate
```

**Output yang diharapkan:**
```
INFO  Preparing database.

Creating migration table ..................... 11ms DONE

  2025_03_05_000000_create_users_table ........... 45ms DONE
  2025_03_05_000001_create_cache_table ........... 8ms DONE
  ...
```

### Step 9: Cek Aplikasi Berjalan

Buka browser dan kunjungi:
- **Aplikasi Laravel**: http://localhost
- **phpMyAdmin**: http://localhost:8080

Jika berhasil, Anda sudah ready untuk development! 🎉

---

## Migrasi dari SQLite ke MySQL

Ini adalah proses untuk mengganti database dari SQLite (default Laravel) ke MySQL di Sail.

### Prerequisites

Sebelum memulai, pastikan:
- ✅ Docker Desktop sedang berjalan
- ✅ Project Laravel sudah punya Laravel Sail installed (dari section sebelumnya)
- ✅ Anda sudah follow langkah setup di atas

### Step 1: Backup Data Lama (Best Practice)

Meskipun kasus Anda tidak ada existing data, ini adalah best practice:

```bash
# Jika ada file SQLite
cp database/database.sqlite database/database.sqlite.backup
```

### Step 2: Hapus Database SQLite

```bash
rm database/database.sqlite
```

File ini tidak perlu lagi karena kita akan menggunakan MySQL.

### Step 3: Update .env untuk MySQL

Pastikan file `.env` sudah konfigurasi MySQL (lihat Step 5 di Instalasi):

```env
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=password
```

### Step 4: Restart Docker Containers

```bash
# Stop containers yang sedang berjalan
sail down

# Start ulang dengan MySQL
sail up -d
```

Tunggu beberapa saat agar MySQL siap menerima koneksi.

### Step 5: Jalankan Migrations

```bash
sail artisan migrate
```

Perintah ini akan:
- ✅ Membuat semua tables di MySQL database `laravel`
- ✅ Menjalankan semua migration files

**Output yang diharapkan:**
```
INFO  Preparing database.

Creating migration table ..................... 11ms DONE

  2025_03_05_000000_create_users_table ........... 45ms DONE
  2025_03_05_000001_create_cache_table ........... 8ms DONE
  2025_03_05_000002_create_jobs_table ........... 22ms DONE
  ...
```

### Step 6: Verifikasi di phpMyAdmin

1. Buka http://localhost:8080
2. Login dengan:
   - **Username**: `root`
   - **Password**: (sesuai `DB_PASSWORD` di `.env`)
3. Pilih database `laravel` di sidebar kiri
4. Seharusnya Anda bisa melihat tables seperti `users`, `cache`, `jobs`, dll

### Step 7: Seed Data (Opsional)

Jika Anda punya seeders untuk populate data dummy:

```bash
sail artisan db:seed
```

**Selesai!** Database Anda sudah migrasi dari SQLite ke MySQL di Sail. 🎉

---

## Common Commands

Berikut adalah perintah-perintah yang paling sering digunakan saat development dengan Sail.

### Lifecycle Commands

**Mulai containers:**
```bash
sail up
```
Jalankan di foreground. Anda bisa melihat logs semua services. Tekan `Ctrl+C` untuk stop.

**Mulai containers di background:**
```bash
sail up -d
```
Cocok untuk saat development, bisa tetap menggunakan terminal untuk perintah lain.

**Lihat status containers:**
```bash
sail ps
```
Menampilkan semua containers yang sedang berjalan.

**Stop containers (tapi tetap ada):**
```bash
sail stop
```
Containers tetap ada, hanya dihentikan. Lebih cepat dari `down` saat mau resume.

**Stop dan hapus containers:**
```bash
sail down
```
Menghentikan dan menghapus containers. Data di database tetap tersimpan di Docker volumes.

**Restart containers:**
```bash
sail restart
```
Sama dengan `sail stop` kemudian `sail up -d`.

**Rebuild Docker images (setelah update dependencies):**
```bash
sail build --no-cache

sail up -d
```

### Artisan Commands

Semua Artisan commands dijalankan dengan prefix `sail`:

```bash
# Migrate database
sail artisan migrate

# Migrate dengan seed data
sail artisan migrate --seed

# Rollback migrasi terakhir
sail artisan migrate:rollback

# Reset semua migrations (hapus semua tables dan jalankan ulang)
sail artisan migrate:reset

# Fresh - reset dan migrate
sail artisan migrate:fresh

# Fresh dengan seed
sail artisan migrate:fresh --seed

# Tinker - interactive shell untuk test code
sail artisan tinker

# Make migration baru
sail artisan make:migration create_posts_table

# Make model baru
sail artisan make:model Post

# Make controller baru
sail artisan make:controller PostController

# Make factory
sail artisan make:factory PostFactory

# Make seeder
sail artisan make:seeder PostSeeder
```

### Database Commands

**Koneksi langsung ke MySQL CLI:**
```bash
sail mysql
```
Anda sekarang bisa mengetik SQL queries langsung:
```sql
USE laravel;
SHOW TABLES;
SELECT * FROM users;
EXIT;
```

**Seed database:**
```bash
sail artisan db:seed
```

**Seed dengan seeder tertentu:**
```bash
sail artisan db:seed --class=PostSeeder
```

### Node & NPM Commands

**Install dependencies:**
```bash
sail npm install
```

**Development server (Vite HMR):**
```bash
sail npm run dev
```
Jalankan di terminal terpisah. Ini akan auto-rebuild assets saat file berubah.

**Build production assets:**
```bash
sail npm run build
```

**Jalankan linter (Pint):**
```bash
sail artisan pint
```

### Composer Commands

**Install/update dependencies:**
```bash
sail composer install

sail composer update
```

**Require package baru:**
```bash
sail composer require package/name
```

**Require package development saja:**
```bash
sail composer require --dev package/name
```

### Testing Commands

**Jalankan semua tests:**
```bash
sail artisan test
```

**Jalankan test dengan filter nama:**
```bash
sail artisan test --filter=UserTest
```

**Jalankan dengan coverage report:**
```bash
sail artisan test --coverage
```

### Shell Access

**Masuk ke container app:**
```bash
sail shell
```
Sekarang Anda di dalam container. Anda bisa menjalankan bash commands apapun:
```bash
root@xxxxx:/var/www/html# ls -la
root@xxxxx:/var/www/html# php artisan tinker
root@xxxxx:/var/www/html# exit
```

### Logs

**Lihat logs semua services:**
```bash
sail logs
```

**Lihat logs service tertentu:**
```bash
sail logs mysql

sail logs redis
```

**Follow logs real-time:**
```bash
sail logs -f
```
Tekan `Ctrl+C` untuk stop.

---

## Development Workflow

Workflow ini adalah cara umum developer bekerja dengan Laravel Sail sehari-hari.

### Workflow Tipikal Satu Hari

**🌅 Pagi - Mulai Development:**

```bash
# Terminal 1 - Start Sail di background
sail up -d

# Terminal 2 - Start Vite dev server (untuk auto-rebuild assets)
sail npm run dev

# Terminal 3 - Siap untuk Artisan commands, migrations, dll
# (Anda bisa buka tab/window baru di terminal)
```

Sekarang Anda siap development:
- Edit files PHP di IDE/editor Anda
- Edit files JavaScript/CSS di IDE/editor Anda
- Vite otomatis rebuild dan browser refresh (Hot Module Reload)

**💻 Saat Development:**

1. **Membuat feature baru:**
   ```bash
   sail artisan make:model Post -m -c -r
   # Ini membuat Model Post, migration, dan ResourceController sekaligus
   ```

2. **Edit model dan migration:**
   - Buka file yang dibuat di editor Anda
   - Edit sesuai kebutuhan

3. **Jalankan migration:**
   ```bash
   sail artisan migrate
   ```

4. **Edit controller dan routes:**
   - Buka file controller dan routes di editor
   - Jika menggunakan Inertia/Vue/React, edit component juga

5. **Testing feature:**
   ```bash
   sail artisan tinker
   # Test logika di interactive shell
   
   # atau
   
   sail artisan test
   # Jalankan automated tests
   ```

6. **Lihat perubahan di browser:**
   - Refresh browser - Vite HMR otomatis refresh
   - Atau perubahan database bisa cek di phpMyAdmin (http://localhost:8080)

**🔧 Database Changes:**

Saat Anda perlu mengubah database schema:

```bash
# 1. Buat migration baru
sail artisan make:migration add_status_to_posts

# 2. Edit migration file di database/migrations/

# 3. Jalankan migration
sail artisan migrate

# 4. Verify di phpMyAdmin atau MySQL CLI
sail mysql
```

**🧪 Testing:**

```bash
# Jalankan semua tests
sail artisan test

# Atau dengan kompact output
sail artisan test --compact

# Test file tertentu
sail artisan test tests/Feature/PostTest.php

# Test dengan coverage
sail artisan test --coverage
```

**🌙 Akhir Hari - Cleanup:**

```bash
# Option 1: Stop containers (mereka tetap ada)
sail stop

# Option 2: Hapus semua containers (data di DB tetap aman)
sail down

# Besok pagi, cukup:
sail up -d
```

### Debugging Database Issues

**Lihat struktur tabel:**
```bash
sail mysql

# Di MySQL prompt:
DESCRIBE users;
# atau
SHOW COLUMNS FROM users;
```

**Lihat data:**
```bash
# Di MySQL prompt:
SELECT * FROM users;

SELECT COUNT(*) FROM posts;
```

**Lihat logs aplikasi:**
```bash
sail logs -f
# Ctrl+C untuk stop
```

**Lihat logs MySQL:**
```bash
sail logs mysql -f
```

### Tips Produktivitas

**💡 Tip 1: Setup Bash Alias (Optional)**

Jika Anda ingin mengetik `sail` tanpa `./vendor/bin/`, tambahkan alias:

```bash
# Buka ~/.bashrc atau ~/.zshrc
nano ~/.bashrc

# Tambahkan di akhir file:
alias sail=./vendor/bin/sail

# Simpan (Ctrl+O, Enter, Ctrl+X)

# Reload shell:
source ~/.bashrc

# Sekarang Anda bisa pakai:
sail artisan migrate
# Bukan:
./vendor/bin/sail artisan migrate
```

**💡 Tip 2: Monitor Multiple Terminals**

Simpan ini di berbagai terminal tabs:
1. `sail logs -f` - Monitor logs
2. `sail npm run dev` - Vite dev server
3. Open untuk mengetik Artisan commands

**💡 Tip 3: Quick Test Loop**

Saat development dengan TDD (Test-Driven Development):
```bash
# Terminal khusus untuk testing
sail artisan test --filter=UserTest --watch
```

**💡 Tip 4: Check Database Quickly**

Daripada buka phpMyAdmin, gunakan MySQL CLI:
```bash
sail mysql -e "SELECT * FROM users LIMIT 5;"
```

---

## Debugging & Troubleshooting

Bagian ini membantu Anda mengatasi masalah umum saat menggunakan Laravel Sail.

### Problem 1: "Cannot connect to Docker daemon"

**Gejala:**
```
Cannot connect to Docker daemon at unix:///var/run/docker.sock
```

**Solusi:**
1. ✅ Pastikan Docker Desktop sudah dibuka dan running
2. ✅ Cek status di taskbar/menu bar (seharusnya ada icon Docker)
3. ✅ Jika belum, buka Docker Desktop dan tunggu sampai "Docker is running"
4. ✅ Coba lagi: `sail up -d`

**Untuk Linux:**
```bash
# Docker daemon belum berjalan
sudo systemctl start docker

# Jika perlu, tambah user Anda ke docker group
sudo usermod -aG docker $USER
# Logout dan login ulang, atau:
newgrp docker
```

### Problem 2: "Port 3306 already in use"

**Gejala:**
```
Error starting userland proxy: listen tcp4 0.0.0.0:3306: bind: address already in use
```

**Solusi 1 - Kill existing MySQL:**
```bash
# Lihat process yang menggunakan port 3306
lsof -i :3306

# Kill process
kill -9 <PID>

# Coba sail up lagi
sail up -d
```

**Solusi 2 - Ubah port di compose.yaml:**
```yaml
mysql:
  ports:
    - "3307:3306"  # Ubah dari 3306 menjadi 3307
```

Lalu update `.env`:
```env
DB_PORT=3307
```

### Problem 3: "Port 8080 already in use" (phpMyAdmin)

**Gejala:**
```
Error starting userland proxy: listen tcp4 0.0.0.0:8080: bind: address already in use
```

**Solusi:**
Ubah port phpMyAdmin di `compose.yaml`:
```yaml
phpmyadmin:
  ports:
    - "8081:80"  # Ubah dari 8080 menjadi 8081
```

Akses phpMyAdmin di `http://localhost:8081`

### Problem 4: "Connection refused" dari application

**Gejala:**
```
SQLSTATE[HY000] [2002] Connection refused
```

**Solusi:**
1. ✅ Pastikan containers semua running: `sail ps`
2. ✅ Restart containers: `sail restart`
3. ✅ Cek `.env` DB_HOST adalah `mysql` bukan `localhost`
4. ✅ Tunggu MySQL siap (bisa butuh 10-15 detik):
   ```bash
   sail logs mysql
   # Tunggu sampai lihat: "ready for connections"
   ```

### Problem 5: "Migration failed" setelah setup baru

**Gejala:**
```
SQLSTATE[HY000]: General error: 1030 Got an error...
```

**Solusi:**
1. ✅ Reset database:
   ```bash
   sail artisan migrate:reset
   ```
2. ✅ Jalankan migrate fresh:
   ```bash
   sail artisan migrate:fresh
   ```
3. ✅ Jika tetap error, cek MySQL logs:
   ```bash
   sail logs mysql
   ```

### Problem 6: "Disk space" atau "Containers tidak bisa start"

**Gejala:**
```
Error response from daemon: no space left on device
```

**Solusi - Cleanup Docker:**
```bash
# Lihat usage
docker system df

# Hapus unused images, containers, networks
docker system prune

# Hapus juga unused volumes (WARNING: data akan hilang!)
docker system prune -a --volumes
```

### Problem 7: "npm run dev tidak auto-refresh browser"

**Gejala:**
Browser tidak refresh otomatis saat edit CSS/JS.

**Solusi:**
1. ✅ Pastikan `sail npm run dev` sedang berjalan
2. ✅ Cek browser console (F12) apakah ada error
3. ✅ Hard refresh browser: `Ctrl+Shift+R` atau `Cmd+Shift+R`
4. ✅ Jika masih tidak jalan:
   ```bash
   # Stop dan rebuild
   sail down
   sail build --no-cache
   sail up -d
   ```

### Problem 8: "Database password salah"

**Gejala:**
```
SQLSTATE[HY000] [1045] Access denied for user 'root'@'mysql'
```

**Solusi:**
1. ✅ Cek `.env` - pastikan `DB_PASSWORD` sesuai
2. ✅ Jika ubah password, restart containers:
   ```bash
   sail down
   sail up -d
   ```
3. ✅ Jika sudah ada data dan ingin ubah password, lebih kompleks - sebaiknya reset:
   ```bash
   sail down -v  # -v menghapus volumes (termasuk data!)
   sail up -d
   sail artisan migrate
   ```

### Viewing Logs untuk Debugging

**Lihat logs real-time semua services:**
```bash
sail logs -f
```

**Lihat logs service tertentu:**
```bash
sail logs mysql -f
sail logs redis -f
sail logs laravel.test -f  # logs aplikasi Anda
```

**Lihat N baris terakhir saja:**
```bash
sail logs --tail=50
```

**Lihat logs tanpa follow:**
```bash
sail logs
```

**Lihat logs sambil grep (cari text tertentu):**
```bash
sail logs | grep "error"
sail logs mysql | grep "ERROR"
```

### Quick Debug dengan phpMyAdmin

phpMyAdmin berguna untuk cepat inspect dan manipulasi data:

1. Buka http://localhost:8080
2. Login dengan root dan DB_PASSWORD dari `.env`
3. Klik database `laravel` di sidebar
4. Lihat tables dan data
5. Run custom SQL queries di tab "SQL"

Contoh query berguna:
```sql
-- Lihat struktur table
DESCRIBE users;

-- Lihat jumlah data
SELECT COUNT(*) FROM posts;

-- Lihat data dengan limit
SELECT * FROM users LIMIT 10;

-- Clear table
TRUNCATE TABLE posts;
```

### Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| `Connection refused` | MySQL belum ready | Tunggu beberapa saat, `sail restart` |
| `Table doesn't exist` | Migration belum jalan | `sail artisan migrate` |
| `UNIQUE constraint fails` | Duplicate data | `sail artisan migrate:fresh` |
| `Port already in use` | Port sudah dipakai service lain | Ubah di compose.yaml atau kill process |
| `Cannot find module` | npm packages belum install | `sail npm install` |
| `Class not found` | Autoload belum updated | `sail composer dump-autoload` |

---

## phpMyAdmin Guide

phpMyAdmin adalah web interface untuk manage MySQL database. Ini memudahkan untuk browse, edit, dan delete data tanpa command line.

### Akses phpMyAdmin

1. Pastikan `sail up -d` sudah dijalankan
2. Buka browser: http://localhost:8080
3. Login dengan:
   - **Server**: `mysql` (atau bisa `localhost` jika tidak jalan)
   - **Username**: `root`
   - **Password**: Sesuai `DB_PASSWORD` di `.env`

💡 **Tips:** Simpan bookmark ini untuk akses cepat di masa depan.

### Interface phpMyAdmin

Setelah login, Anda akan melihat:

**Sidebar kiri:**
- Daftar databases (termasuk `laravel` - database app Anda)
- Klik untuk expand dan lihat tables

**Panel utama:**
- Berbagai tab: Browse, Structure, Insert, Export, Import, SQL, dll

### Membuka Database dan Table

1. **Klik database `laravel`** di sidebar
2. **Anda akan melihat daftar tables**: `users`, `migrations`, `jobs`, `cache`, dll
3. **Klik table** untuk melihat data

### Browse Data

**Melihat data:**
1. Klik table (misal: `users`)
2. Tab "Browse" akan otomatis terbuka
3. Anda akan melihat semua records dengan pagination

**Columns di setiap table:**
- Biasanya ada: `id`, `name`, `email`, `created_at`, `updated_at`, dll

**Aksi di setiap row:**
- **Edit** (pensil icon) - Ubah data
- **Delete** (trash icon) - Hapus record
- **Copy** - Duplikat record

### Insert Data (Tambah Record Baru)

1. Klik table
2. Klik tab **"Insert"**
3. Isi form dengan data baru
4. Klik **"Insert"**

Contoh: Menambah user baru
- Name: "John Doe"
- Email: "john@example.com"
- Password: Hash bcrypt password (atau bisa dari app)
- Email verified: null
- Created at: Abaikan (MySQL auto-fill)

### Edit Data

1. Klik table
2. Tab "Browse" - Cari record
3. Klik **Edit** (pensil icon) di row
4. Edit fields yang diinginkan
5. Klik **"Save"**

### Delete Data

1. Klik table
2. Tab "Browse" - Cari record
3. Klik **Delete** (trash icon)
4. Confirm delete
5. Record akan dihapus

⚠️ **Warning:** Delete di phpMyAdmin tidak bisa di-undo!

### Lihat Struktur Table

1. Klik table
2. Klik tab **"Structure"**
3. Anda akan melihat:
   - Nama column
   - Type (VARCHAR, INT, DATETIME, TEXT, dll)
   - Null/Not Null
   - Key (PRIMARY, FOREIGN, dll)
   - Default value

Contoh struktur table `users`:
```
Column          | Type         | Null | Key | Default
id              | int          | NO   | PRI | NULL
name            | varchar(255) | NO   |     | NULL
email           | varchar(255) | NO   | UNI | NULL
password        | varchar(255) | NO   |     | NULL
email_verified  | datetime     | YES  |     | NULL
created_at      | datetime     | YES  |     | NULL
updated_at      | datetime     | YES  |     | NULL
```

### Run SQL Queries Custom

1. Klik tab **"SQL"** di top
2. Ketik SQL query Anda
3. Klik **"Go"**

Contoh queries berguna:

```sql
-- Lihat semua users
SELECT * FROM users;

-- Lihat users dengan email tertentu
SELECT * FROM users WHERE email = 'john@example.com';

-- Count berapa banyak users
SELECT COUNT(*) AS total_users FROM users;

-- Lihat users yang dibuat bulan ini
SELECT * FROM users WHERE MONTH(created_at) = MONTH(NOW());

-- Update data
UPDATE users SET name = 'Jane Doe' WHERE id = 1;

-- Delete data
DELETE FROM users WHERE id = 5;

-- Join tables
SELECT u.id, u.name, p.title 
FROM users u 
LEFT JOIN posts p ON u.id = p.user_id;
```

### Export Data

**Export 1 table:**
1. Klik table
2. Klik tab **"Export"**
3. Pilih format: SQL, CSV, JSON, XML, dll
4. Klik **"Go"**
5. File akan download

**Export database penuh:**
1. Klik database `laravel` (jangan tabel)
2. Tab "Export" otomatis muncul
3. Pilih format dan tables yang mau di-export
4. Klik **"Go"**

Format export:
- **SQL** - Bisa di-import balik ke database lain
- **CSV** - Buka dengan Excel/Spreadsheet
- **JSON** - Untuk API atau data exchange
- **XML** - Untuk dokumen terstruktur

### Import Data

**Import SQL file:**
1. Klik database `laravel`
2. Tab **"Import"**
3. Klik **"Choose File"** dan select file `.sql`
4. Klik **"Import"**

**Import CSV file:**
1. Tab "Import"
2. Pilih format "CSV"
3. Upload file
4. Map columns dengan table columns
5. Klik "Import"

### Table Operations

**Rename table:**
1. Klik table
2. Tab "Operations"
3. Section "Rename table"
4. Ketik nama baru
5. Klik "Go"

**Copy table:**
1. Klik table
2. Tab "Operations"
3. Section "Copy table to"
4. Isi nama tabel baru
5. Pilih struktur saja atau struktur + data
6. Klik "Go"

**Truncate table (hapus semua data):**
1. Klik table
2. Tab "Operations"
3. Klik **"Truncate table"**
4. Confirm

⚠️ **Warning:** Ini akan hapus SEMUA data table!

### Drop Table (Hapus Table)

⚠️ **DANGER - Hati-hati!**

1. Klik table
2. Tab "Operations"
3. Klik **"Drop table"**
4. Confirm

Ini akan hapus table selamanya (data hilang selamanya).

### Database Operations

**Create Database Baru:**
1. Di sidebar, ketik nama database baru
2. Pilih collation (utf8mb4_unicode_ci adalah standard)
3. Klik "Create"

**Backup Database:**
1. Klik database
2. Tab "Export"
3. Pilih semua tables
4. Format: SQL
5. Klik "Go" dan save file

Ini berguna untuk backup sebelum do besar-besaran changes!

---

## Best Practices & Tips

Bagian ini berisi tips dan best practices untuk bekerja dengan Laravel Sail secara efektif.

### Environment Variables

**Jangan hardcode credentials di kode!** Gunakan `.env`:

```php
// ❌ SALAH
$db_password = 'my_secret_password';

// ✅ BENAR
$db_password = config('database.connections.mysql.password');
// atau
$db_password = env('DB_PASSWORD');
```

**.env File Management:**
```bash
# .env harus di .gitignore (jangan commit)
# Setiap developer punya .env mereka sendiri

# Gunakan .env.example untuk template
cp .env.example .env

# Edit .env sesuai setup lokal mereka
```

**Secrets Management:**
- ✅ `.env` hanya untuk development (bisa publish)
- ✅ `.env.local` untuk sensitive values (jangan commit)
- ✅ Production: Gunakan secret manager atau env vars di server

### Docker Volume Management

Data di database tersimpan di Docker volumes, bukan di container. Ini berarti:

```bash
# Even jika container dihapus, data tetap aman
sail down
sail up -d
# Database Anda masih ada!

# Hanya menghapus data jika:
sail down -v  # -v = remove volumes (DESTRUCTIVE!)
```

**Backup volume:**
```bash
# Inspect volume
docker volume ls

# Backup data
docker run --rm -v sail_mysql:/data -v $(pwd):/backup \
  busybox tar czf /backup/mysql_backup.tar.gz -C /data .

# Restore
docker run --rm -v sail_mysql:/data -v $(pwd):/backup \
  busybox tar xzf /backup/mysql_backup.tar.gz -C /data
```

### Docker Cleanup

Secara berkala, cleanup Docker untuk hemat disk space:

```bash
# Lihat usage
docker system df

# Safe cleanup (remove unused images, containers, networks)
docker system prune

# Lebih aggressive cleanup
docker system prune -a

# Remove unused volumes (WARNING: DATA HILANG!)
docker system prune --volumes
```

### Working dengan Rekan Tim

**Share Docker setup:**
1. ✅ Commit `compose.yaml` ke git
2. ✅ Commit `.env.example` ke git
3. ❌ Jangan commit `.env` (private!)

**Setup untuk tim baru:**
```bash
git clone project
cp .env.example .env
sail up -d
sail artisan migrate
sail artisan db:seed
```

**Sync database changes:**
Saat rekan membuat migration baru:
```bash
git pull
sail artisan migrate
```

**Sync dependencies:**
Saat rekan add composer packages:
```bash
git pull
sail composer install
sail npm install
```

### Performance Tips

**1. Memory allocation untuk Docker:**
- Default Docker Desktop: 2GB
- Recommended untuk Laravel: 4GB (8GB lebih baik)
- Setting di Docker Desktop → Preferences → Resources

**2. Disable services yang tidak perlu:**
Jika Anda tidak perlu Redis, hapus dari `compose.yaml`:
```yaml
# Komentar atau hapus:
# redis:
#   image: redis:7-alpine
#   ...
```

**3. Run containers di background:**
```bash
sail up -d  # Better than: sail up
```

**4. Use volume mounts efficiently:**
```yaml
# ✅ GOOD - sync hanya folder yang perlu
volumes:
  - '.:/var/www/html'
  - '/var/www/html/node_modules'  # Exclude node_modules

# ❌ AVOID - sync file-by-file (lambat)
volumes:
  - './app:/var/www/html/app'
  - './routes:/var/www/html/routes'
  - '...'
```

### Local Development Tweaks

**Add aliases untuk commands yang sering:**
```bash
# ~/.bashrc atau ~/.zshrc
alias sail='./vendor/bin/sail'
alias sartisan='sail artisan'
alias snpm='sail npm'
```

Sekarang Anda bisa:
```bash
sartisan migrate
snpm run dev
```

**Create shell scripts untuk workflow:**
```bash
#!/bin/bash
# scripts/dev-setup.sh

sail up -d
sail artisan migrate:fresh --seed
sail npm run dev
```

Jalankan: `bash scripts/dev-setup.sh`

**IDE Integration:**
- **VS Code**: Install Docker extension untuk monitoring containers
- **PhpStorm**: Built-in Docker support, configure sebagai PHP interpreter

### Security Considerations

**Local development - tidak perlu super ketat, tapi:**

1. ✅ Jangan expose `.env` file
2. ✅ Jangan commit credentials ke git
3. ✅ Disable phpMyAdmin di production
4. ✅ Gunakan strong passwords untuk development
5. ✅ Regular `docker system prune` untuk cleanup
6. ✅ Update Docker Desktop regularly untuk security patches

**phpMyAdmin Security:**
```yaml
# Di compose.yaml, restrict access:
phpmyadmin:
  environment:
    PMA_HOST: mysql
    # Hanya allow dari localhost (saat di production)
```

### Troubleshooting Checklist

Sebelum bertanya di forum/issue, coba:

```bash
# 1. Cek containers status
sail ps

# 2. Check logs
sail logs -f

# 3. Restart containers
sail restart

# 4. Full rebuild (jika ada issue)
sail down
sail build --no-cache
sail up -d

# 5. Database reset (last resort)
sail artisan migrate:reset
sail artisan migrate
```

---

## Conclusion & Next Steps

Selamat! Anda sudah punya setup Laravel Sail yang lengkap dengan MySQL dan phpMyAdmin. 

### Apa yang bisa Anda lakukan sekarang:

✅ Mulai development dengan `sail up -d` dan `sail npm run dev`  
✅ Create models, controllers, migrations dengan `sail artisan make:xxx`  
✅ Monitor database di phpMyAdmin (http://localhost:8080)  
✅ Run tests dengan `sail artisan test`  
✅ Push changes ke git (jangan lupa: `.env` di `.gitignore`)  

### Helpful Resources:

- [Laravel Official Docs - Sail](https://laravel.com/docs/sail)
- [Docker Documentation](https://docs.docker.com)
- [MySQL Reference Manual](https://dev.mysql.com/doc)
- [phpMyAdmin Documentation](https://www.phpmyadmin.net)

### Quick Reference Sheet:

```bash
# Daily workflow
sail up -d                    # Start containers
sail npm run dev              # Vite HMR in separate terminal
sail artisan migrate          # Run migrations
sail artisan tinker           # Test code interactively
sail down                     # Stop containers

# Common tasks
sail artisan make:model Post -m -c -r    # Model + Migration + Controller
sail artisan migrate:fresh --seed        # Fresh start with seed
sail mysql                                # MySQL CLI
sail logs -f                             # Watch logs
sail shell                               # Access container

# Testing & debugging
sail artisan test             # Run tests
sail artisan test --filter=TestName
phpMyAdmin: http://localhost:8080
```

**Happy coding!** 🚀

---

**Terakhir diupdate:** 2026-03-05  
**Untuk:** Laravel 12 dengan Sail v1.53  
**Database:** MySQL 8.0  
**PHP:** 8.5.3
