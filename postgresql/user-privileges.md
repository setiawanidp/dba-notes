# PostgreSQL: Konsep User Privileges (Roles & Grants)
 
> Lanjutan dari materi cluster/database/schema — sekarang membahas bagaimana akses dikontrol di PostgreSQL, dan bagaimana ini berbeda cukup mendasar dari model user MySQL.
 
---
 
## 1. Perbedaan Fundamental: Role vs User@Host
 
Di MySQL, identitas akses adalah pasangan **`user@host`** — `budi@localhost` dan `budi@'%'` dianggap dua user yang berbeda sepenuhnya, masing-masing bisa punya privilege sendiri-sendiri.
 
Di PostgreSQL, tidak ada konsep host di dalam identitas user. Yang ada hanyalah **role**. Satu nama role = satu identitas, titik. Pembatasan "boleh connect dari mana" diatur terpisah, di file `pg_hba.conf`, bukan bagian dari nama usernya.
 
Poin krusial lain: **di PostgreSQL, "user" dan "role" pada dasarnya adalah objek yang sama.**
 
- `CREATE ROLE nama;` → dibuat, tapi defaultnya **tidak bisa login**.
- `CREATE USER nama;` → sebenarnya cuma shortcut dari `CREATE ROLE nama LOGIN;`
Role juga bisa berperan sebagai **grup**. Kamu bisa membuat role tanpa hak login yang isinya kumpulan privilege, lalu role lain (user asli) "menjadi anggota" role tersebut untuk mewarisi privilege-nya. Ini menggantikan konsep GRANT ke banyak user satu-satu seperti sering dilakukan di MySQL.
 
**Analogi untuk DBA MySQL:**
 
| MySQL | PostgreSQL |
|---|---|
| `user@host` sebagai identitas unik | Role saja (tanpa host) sebagai identitas |
| Batasan host: bagian dari nama user | Batasan koneksi: `pg_hba.conf`, terpisah dari nama role |
| Tidak ada "role" murni sebagai grup (sebelum MySQL 8) | Role bisa jadi grup/kumpulan privilege sejak awal |
| `GRANT ... TO user1, user2, user3` | Buat 1 role grup, lalu `GRANT role_grup TO user1, user2, user3` |
 
---
 
## 2. Role Attributes (Melekat pada Role, Bukan per-Database)
 
Saat membuat role, ada sejumlah **attribute** global yang menentukan kemampuan level-cluster-nya:
 
```sql
CREATE ROLE nama_role
    LOGIN                   -- boleh login (kalau tidak ada, jadi grup saja)
    PASSWORD 'rahasia'
    SUPERUSER | NOSUPERUSER -- akses penuh ke seluruh cluster (mirip root MySQL)
    CREATEDB   | NOCREATEDB -- boleh membuat database
    CREATEROLE | NOCREATEROLE -- boleh membuat role lain
    INHERIT    | NOINHERIT  -- otomatis mewarisi privilege dari role yang diikuti
    CONNECTION LIMIT 10     -- batas jumlah koneksi bersamaan
    VALID UNTIL '2027-01-01'; -- kedaluwarsa akun
```
 
Ini berbeda dari MySQL, di mana kemampuan seperti "boleh bikin database" atau "boleh bikin user" itu sendiri adalah **privilege biasa** (`CREATE`, `CREATE USER`) yang di-`GRANT` seperti privilege lain. Di PostgreSQL, hal-hal itu dipisahkan jadi *attribute* bawaan role, bukan privilege yang di-grant ke objek.
 
---
 
## 3. Privilege ke Objek: GRANT / REVOKE
 
Setelah role dibuat, privilege ke objek (table, schema, database, dll.) diberikan lewat `GRANT`, konsepnya mirip MySQL tapi granularity dan targetnya beda level.
 
### Privilege tingkat Database
 
```sql
GRANT CONNECT ON DATABASE toko_online TO budi;
GRANT CREATE ON DATABASE toko_online TO budi;   -- boleh bikin schema baru di db ini
```
 
### Privilege tingkat Schema
 
```sql
GRANT USAGE ON SCHEMA sales TO budi;   -- boleh "melihat" objek di schema (wajib, sebelum akses tabel)
GRANT CREATE ON SCHEMA sales TO budi;  -- boleh bikin objek baru di schema ini
```
 
> Catatan penting: `USAGE` pada schema itu **prasyarat**. Tanpa `USAGE` di schema-nya, privilege apa pun di tabel dalam schema itu tidak akan berfungsi — beda dari MySQL yang tidak mengenal level namespace di antara database dan tabel.
 
### Privilege tingkat Table
 
```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON sales.orders TO budi;
GRANT ALL PRIVILEGES ON sales.orders TO budi;
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO budi;  -- semua tabel yang sudah ada saat ini
```
 
### Privilege tingkat kolom
 
```sql
GRANT SELECT (id, nama), UPDATE (nama) ON sales.pelanggan TO budi;
```
 
### Mencabut privilege
 
```sql
REVOKE INSERT ON sales.orders FROM budi;
REVOKE ALL PRIVILEGES ON sales.orders FROM budi;
```
 
---
 
## 4. Default Privileges (Objek yang Belum Ada)
 
Ini konsep yang **tidak ada padanannya di MySQL**. `GRANT ... ON ALL TABLES` di atas hanya berlaku untuk tabel yang **sudah ada saat perintah dijalankan**. Untuk tabel yang **akan dibuat nanti**, perlu diatur terpisah:
 
```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA sales
    GRANT SELECT ON TABLES TO budi;
```
 
Dengan ini, setiap kali ada tabel baru dibuat di schema `sales` (oleh role yang menjalankan `ALTER DEFAULT PRIVILEGES` ini), role `budi` otomatis dapat `SELECT` tanpa perlu `GRANT` ulang manual.
 
---
 
## 5. Role Membership (Pengganti "Grup" di MySQL)
 
```sql
-- Buat role sebagai "grup"
CREATE ROLE app_readonly NOLOGIN;
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA sales GRANT SELECT ON TABLES TO app_readonly;
 
-- Jadikan user anggota grup tersebut
GRANT app_readonly TO budi;
GRANT app_readonly TO citra;
 
-- Melepas keanggotaan
REVOKE app_readonly FROM budi;
```
 
Kalau role diberi atribut `INHERIT` (default-nya begitu sejak PostgreSQL 16 untuk role baru), privilege dari grup otomatis aktif tanpa perlu langkah tambahan. Kalau `NOINHERIT`, anggota harus eksplisit `SET ROLE app_readonly;` dulu untuk memakai privilege grup itu.
 
---
 
## 6. Ownership
 
Role yang membuat suatu objek (database, schema, tabel, dll.) otomatis jadi **owner**-nya, dan owner selalu punya privilege penuh ke objek tersebut tanpa perlu `GRANT` eksplisit — mirip prinsip di MySQL bahwa pembuat objek biasanya diberi hak penuh, tapi di PostgreSQL ini konsep formal (`OWNER`) yang bisa dipindahkan:
 
```sql
ALTER TABLE sales.orders OWNER TO budi;
ALTER SCHEMA sales OWNER TO budi;
```
 
---
 
## 7. Command Bantu untuk Inspeksi (psql)
 
```sql
\du                         -- daftar semua role + attribute-nya
\du+                        -- lebih detail
\dp sales.orders            -- privilege pada tabel tertentu (mirip SHOW GRANTS)
\z sales.orders             -- alias dari \dp
 
-- Setara SHOW GRANTS FOR user di MySQL:
SELECT grantee, privilege_type, table_schema, table_name
FROM information_schema.role_table_grants
WHERE grantee = 'budi';
```
 
---
 
## 8. Ringkasan Perbandingan MySQL vs PostgreSQL
 
| Konsep | MySQL | PostgreSQL |
|---|---|---|
| Identitas | `user@host` | Role (tanpa host) |
| Pembatasan asal koneksi | Bagian dari nama user | `pg_hba.conf`, terpisah |
| Grup privilege | Role (MySQL 8+), agak terpisah dari user | Role itu sendiri = user atau grup, konsepnya menyatu sejak awal |
| Hak "buat database/user" | Privilege biasa (`GRANT CREATE, CREATE USER`) | Attribute role (`CREATEDB`, `CREATEROLE`) |
| Level privilege granularity | Global, DB, tabel, kolom | Global (attribute), database, schema, tabel, kolom |
| Privilege ke objek yang belum ada | Tidak ada mekanisme bawaan | `ALTER DEFAULT PRIVILEGES` |
| Melihat privilege user | `SHOW GRANTS FOR user` | `\dp`, `\du`, atau query `information_schema` |
| Superuser | `root`/user dengan `ALL PRIVILEGES` + `GRANT OPTION` | Attribute `SUPERUSER` pada role |
