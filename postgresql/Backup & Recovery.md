# Backup & Recovery PostgreSQL 15
### Untuk DBA MySQL yang Belajar PostgreSQL

---

## 1. Gambaran Umum

Di PostgreSQL, backup terbagi menjadi 3 pendekatan besar. Sebagai perbandingan dengan dunia MySQL yang sudah Anda kuasai:

| Jenis Backup | Tool PostgreSQL | Analogi di MySQL |
|---|---|---|
| Logical backup | `pg_dump` / `pg_dumpall` | `mysqldump` |
| Physical backup (full) | `pg_basebackup` | `xtrabackup` / `mariabackup` |
| Point-in-Time Recovery (PITR) | WAL archiving + `pg_basebackup` | Binary log (binlog) + full backup |

Perbedaan konsep penting:
- MySQL punya **binlog** untuk replikasi sekaligus PITR. PostgreSQL punya **WAL (Write-Ahead Log)** yang perannya mirip, tapi WAL sudah Anda pelajari sebelumnya di materi arsitektur internal — di sinilah WAL benar-benar dipakai secara praktis.
- `pg_dump` menghasilkan backup **logis** (berupa perintah SQL atau format custom), sama seperti `mysqldump`. Cocok untuk migrasi, restore parsial, atau database kecil-menengah.
- `pg_basebackup` menghasilkan backup **fisik** (copy file data mentah), setara `xtrabackup`. Lebih cepat untuk database besar, tapi restore-nya harus utuh satu cluster.
- PITR di PostgreSQL **mewajibkan** kombinasi physical base backup + WAL archive. Anda tidak bisa PITR hanya dari `pg_dump`.

---

## 2. pg_dump — Logical Backup

### 2.1 Konsep

`pg_dump` membackup **satu database** (bukan seluruh cluster). Untuk membackup semua database sekaligus (termasuk role dan tablespace), gunakan `pg_dumpall`.

Format output yang tersedia:

| Format | Flag | Karakteristik |
|---|---|---|
| Plain SQL | `-F p` (default) | Teks SQL biasa, bisa dibaca & diedit, restore pakai `psql` |
| Custom | `-F c` | Format biner terkompresi, bisa restore parsial, restore pakai `pg_restore` |
| Directory | `-F d` | Seperti custom tapi per-file per-tabel, mendukung **paralel dump & restore** |
| Tar | `-F t` | Arsip tar, restore pakai `pg_restore` |

**Rekomendasi:** gunakan format `custom` atau `directory` untuk produksi karena mendukung kompresi dan restore selektif — mirip alasan Anda mungkin memilih `mysqldump` dengan opsi tertentu, tapi di sini lebih fleksibel.

### 2.2 Tutorial Praktik

**a. Dump satu database (format custom)**
```bash
pg_dump -U postgres -d toko_online -F c -f /backup/toko_online.dump
```

**b. Dump dengan kompresi maksimal & paralel (format directory)**
```bash
pg_dump -U postgres -d toko_online -F d -j 4 -f /backup/toko_online_dir
```
`-j 4` artinya 4 proses paralel — hanya didukung format `directory`.

**c. Dump plain SQL (mudah dibaca, cocok untuk versi kecil)**
```bash
pg_dump -U postgres -d toko_online -F p -f /backup/toko_online.sql
```

**d. Dump hanya schema (struktur tanpa data)**
```bash
pg_dump -U postgres -d toko_online --schema-only -f /backup/schema_only.sql
```

**e. Dump hanya data (tanpa struktur)**
```bash
pg_dump -U postgres -d toko_online --data-only -f /backup/data_only.sql
```

**f. Dump seluruh cluster (semua database + role) dengan pg_dumpall**
```bash
pg_dumpall -U postgres -f /backup/full_cluster.sql
```
Berguna khusus untuk membackup **global objects** (role, tablespace) yang tidak ikut ter-backup oleh `pg_dump` biasa.

### 2.3 Restore

**a. Restore dari format plain SQL**
```bash
psql -U postgres -d toko_online_baru -f /backup/toko_online.sql
```

**b. Restore dari format custom/directory (pakai pg_restore)**
```bash
pg_restore -U postgres -d toko_online_baru /backup/toko_online.dump
```

**c. Restore paralel (format custom/directory saja)**
```bash
pg_restore -U postgres -d toko_online_baru -j 4 /backup/toko_online_dir
```

**d. Restore selektif — hanya satu tabel**
```bash
pg_restore -U postgres -d toko_online_baru -t pelanggan /backup/toko_online.dump
```
Ini keunggulan format custom/directory dibanding plain SQL: Anda bisa memilih objek mana yang mau direstore, mirip `mysqldump` dengan opsi `--tables` tapi lebih fleksibel karena dilakukan saat restore, bukan saat dump.

> **Catatan:** database tujuan restore (`toko_online_baru`) harus sudah dibuat lebih dulu dengan `CREATE DATABASE`, kecuali Anda menambahkan flag `-C` pada `pg_restore`/`psql` agar database dibuat otomatis dari dump.

---

## 3. pg_basebackup — Physical Backup

### 3.1 Konsep

`pg_basebackup` menyalin **seluruh direktori data cluster** (`PGDATA`) dalam kondisi konsisten, termasuk seluruh database, role, dan tablespace — tanpa perlu mematikan server (hot backup), mirip cara kerja `xtrabackup` di MySQL.

Backup ini adalah **fondasi wajib** untuk PITR di langkah berikutnya.

### 3.2 Prasyarat

1. Buat user replikasi (atau gunakan superuser) dengan hak `REPLICATION`:
```sql
CREATE ROLE replikator WITH REPLICATION LOGIN PASSWORD 'passwordkuat';
```

2. Izinkan koneksi replikasi di `pg_hba.conf`:
```
host    replication     replikator      127.0.0.1/32            scram-sha-256
```

3. Reload konfigurasi:
```bash
psql -U postgres -c "SELECT pg_reload_conf();"
```

### 3.3 Tutorial Praktik

**a. Backup dasar, format plain (direktori langsung bisa dipakai sebagai PGDATA baru)**
```bash
pg_basebackup -h 127.0.0.1 -U replikator -D /backup/basebackup_$(date +%F) -P -Fp -Xs
```
Penjelasan flag:
- `-D` : direktori tujuan
- `-P` : tampilkan progress
- `-F p` : format plain (folder biasa, siap pakai)
- `-X s` : sertakan WAL yang dibutuhkan secara **streaming** selama backup berjalan (`s` = stream), memastikan backup konsisten

**b. Backup format tar terkompresi (hemat storage)**
```bash
pg_basebackup -h 127.0.0.1 -U replikator -D /backup/basebackup_tar -Ft -z -P -Xs
```
- `-F t` : format tar
- `-z` : kompresi gzip

**c. Backup dengan checkpoint cepat (fast checkpoint)**
```bash
pg_basebackup -h 127.0.0.1 -U replikator -D /backup/basebackup_fast -P -Fp -Xs -c fast
```
`-c fast` memaksa checkpoint segera (lebih membebani I/O sesaat) alih-alih menunggu checkpoint terjadwal — berguna agar proses backup tidak menunggu lama di awal.

### 3.4 Restore dari pg_basebackup (tanpa PITR)

```bash
systemctl stop postgresql
rm -rf /var/lib/postgresql/15/main/*
cp -R /backup/basebackup_2026-08-19/* /var/lib/postgresql/15/main/
chown -R postgres:postgres /var/lib/postgresql/15/main
systemctl start postgresql
```
Ini adalah restore penuh ke titik waktu backup dibuat — setara restore `xtrabackup` di MySQL tanpa replay binlog tambahan.

---

## 4. PITR (Point-In-Time Recovery) dengan WAL Archiving

### 4.1 Konsep

PITR memungkinkan Anda memulihkan database ke **detik tertentu** di masa lalu (misal: "sebelum tabel `pesanan` ter-DROP jam 14:32"), bukan hanya ke titik backup terakhir.

Caranya: gabungkan **base backup** (pg_basebackup) + **WAL archive** (rekaman semua perubahan setelah base backup dibuat) → PostgreSQL akan replay WAL sampai target waktu yang Anda tentukan.

Analogi MySQL: full backup (xtrabackup) + replay binlog sampai posisi/timestamp tertentu menggunakan `mysqlbinlog`.

### 4.2 Konfigurasi WAL Archiving

Edit `postgresql.conf`:
```conf
wal_level = replica              # minimal 'replica', default di PG15 sudah replica
archive_mode = on
archive_command = 'cp %p /backup/wal_archive/%f'
archive_timeout = 300            # paksa switch WAL tiap 300 detik meski belum penuh (opsional)
```
Penjelasan:
- `%p` : path lengkap file WAL sumber
- `%f` : nama file WAL saja
- `archive_command` di atas contoh sederhana (copy lokal). Di produksi biasanya diganti `rsync`, `scp`, atau tool seperti `pgBackRest`/`WAL-G` ke storage terpisah (mis. S3).

Buat direktori arsip & restart PostgreSQL:
```bash
mkdir -p /backup/wal_archive
chown postgres:postgres /backup/wal_archive
systemctl restart postgresql
```

Verifikasi archiving berjalan:
```sql
SELECT pg_switch_wal();  -- paksa switch WAL segera
SELECT * FROM pg_stat_archiver;
```
Kolom `archived_count` harus terus bertambah dan `last_failed_time` idealnya kosong.

### 4.3 Tutorial Lengkap: Simulasi PITR

**Langkah 1 — Ambil base backup** (setelah WAL archiving aktif)
```bash
pg_basebackup -h 127.0.0.1 -U replikator -D /backup/pitr_base -P -Fp -Xs
```

**Langkah 2 — Catat waktu sekarang sebagai acuan, lalu lakukan aktivitas normal**
```sql
SELECT now();  -- misal hasilnya: 2026-08-19 10:00:00
INSERT INTO pesanan VALUES (...);
```

**Langkah 3 — Simulasikan insiden** (misal DROP TABLE tidak sengaja)
```sql
SELECT now();  -- catat: 2026-08-19 10:15:00 (waktu sebelum insiden, ini target recovery kita)
DROP TABLE pesanan;  -- insiden terjadi setelah ini, misal jam 10:16:00
```

**Langkah 4 — Hentikan server, kosongkan PGDATA, copy base backup**
```bash
systemctl stop postgresql
rm -rf /var/lib/postgresql/15/main/*
cp -R /backup/pitr_base/* /var/lib/postgresql/15/main/
chown -R postgres:postgres /var/lib/postgresql/15/main
```

**Langkah 5 — Buat file sinyal recovery & konfigurasi target**

Di PostgreSQL 12 ke atas (termasuk PG15), gunakan file `postgresql.auto.conf` atau `postgresql.conf` untuk parameter recovery, ditambah file penanda `recovery.signal` (bukan lagi `recovery.conf` seperti versi lama):

```bash
touch /var/lib/postgresql/15/main/recovery.signal
```

Tambahkan ke `postgresql.conf`:
```conf
restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_time = '2026-08-19 10:15:00'
recovery_target_action = 'promote'   # otomatis promote setelah target tercapai
```
Penjelasan:
- `restore_command` : kebalikan dari `archive_command`, memberi tahu PostgreSQL cara **mengambil kembali** file WAL dari arsip saat replay
- `recovery_target_time` : titik waktu tujuan recovery (persis sebelum insiden)
- `recovery_target_action` : `promote` (langsung jadi server normal), `pause` (berhenti, biar Anda cek dulu), atau `shutdown`

**Langkah 6 — Jalankan server, biarkan proses recovery berjalan**
```bash
systemctl start postgresql
tail -f /var/log/postgresql/postgresql-15-main.log
```
Anda akan melihat log semacam:
```
starting point-in-time recovery to 2026-08-19 10:15:00
...
recovery stopping before commit of transaction ...
database system is ready to accept connections
```

**Langkah 7 — Verifikasi**
```sql
SELECT * FROM pesanan;   -- tabel harus kembali ada, insiden ter-undo
SELECT pg_is_in_recovery();  -- harus 'f' (false) jika sudah promote
```

### 4.4 Opsi Target Recovery Lain

Selain `recovery_target_time`, PostgreSQL 15 mendukung target berikut (pilih salah satu):

| Parameter | Kegunaan |
|---|---|
| `recovery_target_time` | Recovery sampai timestamp tertentu |
| `recovery_target_xid` | Recovery sampai transaction ID tertentu |
| `recovery_target_lsn` | Recovery sampai posisi WAL (LSN) tertentu |
| `recovery_target_name` | Recovery sampai restore point bernama (dibuat via `pg_create_restore_point()`) |
| `recovery_target_immediate` | Recovery berhenti secepat mungkin di titik konsisten pertama |

Contoh membuat restore point bernama sebelum melakukan perubahan berisiko (praktik yang sangat direkomendasikan):
```sql
SELECT pg_create_restore_point('sebelum_migrasi_v2');
```
Lalu saat recovery:
```conf
recovery_target_name = 'sebelum_migrasi_v2'
```

---

## 5. Ringkasan Perbandingan Strategi

| Skenario | Solusi yang Tepat |
|---|---|
| Migrasi database ke server lain | `pg_dump` / `pg_restore` |
| Backup rutin database kecil-menengah | `pg_dump` format custom/directory |
| Backup cluster besar, butuh cepat | `pg_basebackup` |
| Butuh restore ke detik tertentu (recovery dari human error) | Base backup + WAL archiving (PITR) |
| Disaster recovery skala produksi | PITR + tool orkestrasi seperti **pgBackRest** atau **WAL-G** (otomatisasi archive, retensi, kompresi, backup incremental) |

> **Catatan lanjutan:** Di produksi, mengelola `archive_command` manual seperti contoh di atas jarang dipakai langsung. Tool seperti **pgBackRest** sangat populer karena menangani backup incremental/differential, retensi otomatis, kompresi, enkripsi, dan verifikasi — mirip peran `xtrabackup` + scripting tambahan di ekosistem MySQL. Ini bisa jadi topik lanjutan setelah Anda menguasai mekanisme dasar di atas.
