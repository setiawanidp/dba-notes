# PostgreSQL: Konsep Cluster, Database, dan Schema

> Materi lanjutan setelah instalasi PostgreSQL — fokus pada perbedaan mendasar arsitektur PostgreSQL vs MySQL yang sering membingungkan DBA MySQL saat pindah ke PostgreSQL.

---

## 1. Kenapa Ini Penting Dipahami Lebih Dulu

Di MySQL, istilah **database** dan **schema** adalah dua nama untuk hal yang sama. Satu `CREATE DATABASE` = satu `CREATE SCHEMA`. Tidak ada level di atasnya selain instance MySQL itu sendiri.

Di PostgreSQL, ada **tiga level hierarki**, bukan dua:

```
Cluster (satu instance PostgreSQL, satu proses postmaster, satu data directory)
 └── Database (terisolasi penuh satu sama lain)
      └── Schema (namespace di dalam satu database)
           └── Table, View, Function, dll.
```

Memahami hierarki ini adalah kunci untuk memahami hampir semua hal lain di PostgreSQL: bagaimana koneksi bekerja, bagaimana permission diatur, bagaimana backup/restore dilakukan, dan kenapa kamu tidak bisa melakukan `JOIN` lintas database seperti di MySQL.

---

## 2. Cluster

**Definisi:** Cluster adalah satu instance PostgreSQL yang dijalankan oleh satu proses server (`postgres`, sering disebut *postmaster*), melayani satu **data directory**, dan mendengarkan pada satu port (default `5432`).

Poin penting:

- Satu cluster bisa berisi **banyak database** di dalamnya.
- Semua database dalam satu cluster berbagi:
  - Konfigurasi server (`postgresql.conf`, `pg_hba.conf`)
  - Proses background (WAL writer, autovacuum launcher, checkpointer, dll.)
  - Global objects: **role/user** dan **tablespace**
- Kamu membuat cluster baru dengan perintah `initdb`, yang menghasilkan data directory baru (isinya termasuk database bawaan: `postgres`, `template0`, `template1`).

**Analogi untuk DBA MySQL:** Cluster PostgreSQL ≈ satu instance `mysqld` yang sedang berjalan. Bedanya, di MySQL "database" langsung setara dengan schema, sementara di PostgreSQL satu instance (cluster) bisa punya banyak database yang benar-benar terpisah.

**Command dasar terkait cluster:**

```bash
# Membuat cluster baru (data directory baru)
initdb -D /path/to/data --username=postgres

# Menjalankan/menghentikan cluster
pg_ctl -D /path/to/data start
pg_ctl -D /path/to/data stop

# Melihat status cluster
pg_ctl -D /path/to/data status
```

Satu server fisik bisa menjalankan **lebih dari satu cluster** sekaligus, asalkan setiap cluster punya data directory dan port yang berbeda. Ini biasa dipakai untuk menjalankan beberapa versi PostgreSQL (misalnya 15 dan 16) berdampingan.

---

## 3. Database

**Definisi:** Database adalah unit isolasi utama di dalam satu cluster. Setiap koneksi client ke PostgreSQL **harus menentukan satu database** — tidak ada koneksi "netral" yang bisa melihat semua database sekaligus.

Karakteristik penting:

- **Isolasi total**: Query di satu database tidak bisa langsung mengakses tabel di database lain dalam satu koneksi. Tidak ada `JOIN` lintas database (berbeda dengan MySQL, di mana kamu bisa `SELECT * FROM db1.table JOIN db2.table`).
  - Solusi lintas database di PostgreSQL: `dblink` atau `postgres_fdw` (Foreign Data Wrapper) — bukan fitur bawaan langsung seperti MySQL.
- Setiap database punya `pg_catalog` sendiri, encoding sendiri, collation sendiri (bisa berbeda antar database dalam satu cluster).
- Database dibuat dari sebuah **template** (default: `template1`). Kamu bisa membuat template kustom sendiri.

**Command dasar:**

```sql
-- Membuat database
CREATE DATABASE toko_online
    OWNER = app_user
    ENCODING = 'UTF8'
    TEMPLATE = template0;

-- Melihat semua database di cluster
\l              -- di psql
-- atau
SELECT datname FROM pg_database;

-- Pindah/connect ke database lain
\c toko_online  -- di psql

-- Menghapus database
DROP DATABASE toko_online;
```

**Analogi untuk DBA MySQL:** Bayangkan tiap "database" di PostgreSQL seperti instance MySQL yang terpisah, tapi berbagi satu server fisik dan satu daftar user/role. Mindset "satu koneksi = satu database, titik" ini yang paling sering bikin DBA MySQL kaget di awal.

---

## 4. Schema

**Definisi:** Schema adalah **namespace di dalam satu database** yang mengelompokkan objek (table, view, function, sequence, index, dll.). Inilah level yang tidak punya padanan eksplisit di MySQL.

Karakteristik penting:

- Satu database bisa punya **banyak schema**.
- Secara default, setiap database baru punya satu schema bernama `public`. Inilah kenapa banyak tutorial pemula tidak pernah menyebut kata "schema" — semua tabel mereka otomatis masuk ke `public` tanpa disadari.
- Objek di schema berbeda **dalam database yang sama** BISA di-`JOIN` langsung, karena masih dalam satu koneksi/database yang sama.
- Digunakan untuk:
  - Multi-tenancy (satu schema per klien/tenant, dalam satu database)
  - Memisahkan objek per aplikasi/modul (`sales`, `inventory`, `reporting`) tanpa perlu database terpisah
  - Mengatur permission secara granular per kelompok tabel

**Command dasar:**

```sql
-- Membuat schema
CREATE SCHEMA sales;
CREATE SCHEMA inventory AUTHORIZATION app_user;

-- Membuat tabel di schema tertentu
CREATE TABLE sales.orders (id serial PRIMARY KEY, total numeric);

-- Mengakses tabel dengan nama lengkap (qualified name)
SELECT * FROM sales.orders;

-- Melihat semua schema
\dn             -- di psql
-- atau
SELECT schema_name FROM information_schema.schemata;

-- Mengatur search_path (urutan schema yang dicari kalau nama tabel tidak diberi prefix)
SET search_path TO sales, public;
SELECT * FROM orders;  -- otomatis mencari di 'sales' dulu, baru 'public'

-- Menghapus schema (dan semua isinya kalau pakai CASCADE)
DROP SCHEMA sales CASCADE;
```

**Konsep `search_path`** ini penting dan tidak ada padanannya di MySQL — ini menentukan schema mana yang dicek lebih dulu ketika kamu menulis nama tabel tanpa prefix schema.

---

## 5. Ringkasan Perbandingan MySQL vs PostgreSQL

| Konsep | MySQL | PostgreSQL |
|---|---|---|
| Instance server | `mysqld` | Cluster (`postmaster`) |
| "Database" | = schema (satu level) | Level isolasi tertinggi di dalam cluster |
| Namespace di dalam database | Tidak ada level ini | Schema |
| JOIN lintas "database" | Bisa (`db1.tbl JOIN db2.tbl`) | Tidak bisa langsung (perlu FDW/dblink) |
| JOIN lintas schema (dalam 1 database) | N/A | Bisa langsung |
| User/role | Per instance, bisa dibatasi per database | Global di level cluster, permission diatur per database/schema |
| Satu koneksi bisa lihat semua "database"? | Ya (dengan `USE` untuk pindah) | Tidak — satu koneksi terikat satu database |
