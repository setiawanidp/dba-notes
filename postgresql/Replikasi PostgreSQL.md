# Tutorial Replikasi PostgreSQL (untuk yang Sudah Terbiasa dengan MySQL)

Karena kamu sudah paham arsitektur internal PostgreSQL (WAL, MVCC, dll), tutorial ini akan sering membandingkan dengan cara kerja replikasi di MySQL supaya lebih cepat "klik".

---

## 1. Konsep Dasar: Replikasi Itu Apa?

Replikasi = menyalin data dari satu server database (**primary/master**) ke satu atau lebih server lain (**standby/replica**) secara terus-menerus, sehingga data di replica selalu mengikuti perubahan di primary.

Tujuannya biasanya:
- **High availability** — kalau primary mati, replica bisa di-*promote* jadi primary baru.
- **Load balancing untuk baca (read scaling)** — query `SELECT` bisa diarahkan ke replica supaya beban primary berkurang.
- **Backup tanpa mengganggu primary**.
- **Disaster recovery** — replica di lokasi/DC berbeda.

### Perbandingan singkat dengan MySQL

| Konsep | MySQL | PostgreSQL |
|---|---|---|
| Mekanisme dasar | Binary log (binlog) di-replay | WAL (Write-Ahead Log) dikirim & di-replay |
| Nama proses pengirim | Binlog dump thread | WAL sender |
| Nama proses penerima | Replica I/O thread + SQL thread | WAL receiver + startup process (recovery) |
| Replikasi statement-based | Ada (SBR) | Tidak ada — PostgreSQL selalu physical (byte-level) di replikasi built-in |
| Replikasi logical (mirip row-based) | Ada sejak MySQL 5.7 (RBR) | Ada, disebut **Logical Replication** (PG 10+) |

Karena kamu sudah paham WAL untuk *crash recovery*, kabar baiknya: **WAL itulah yang dikirim ke replica**. Jadi replikasi PostgreSQL secara alami memanfaatkan mekanisme yang sudah kamu pelajari — replica pada dasarnya terus-menerus "replay" WAL seolah-olah sedang recovery, tapi tidak pernah berhenti.

---

## 2. Dua Jenis Replikasi di PostgreSQL

Ini penting dipahami dari awal karena beda konsep dan beda kegunaan:

### A. Physical Replication (Streaming Replication)
- Menyalin **seluruh cluster** byte-per-byte lewat WAL.
- Replica adalah salinan identik dari primary (semua database, semua schema).
- Replica **read-only** — tidak bisa menerima `INSERT/UPDATE/DELETE`.
- Analogi MySQL: mirip *binlog replication* tapi levelnya lebih rendah (physical, bukan logical), dan replica-nya read-only total.

### B. Logical Replication
- Menyalin **perubahan data** (insert/update/delete) di level baris, bisa dipilih per-tabel atau per-database.
- Replica **bisa menulis data lain** yang tidak direplikasi (tidak read-only total).
- Bisa replikasi antar versi PostgreSQL yang berbeda, bahkan struktur tabel boleh sedikit beda.
- Analogi MySQL: ini yang paling mirip *row-based replication* (RBR) di MySQL — konsepnya publish/subscribe.

**Aturan main sederhana:** kalau butuh HA/failover penuh → pakai **Physical/Streaming Replication**. Kalau butuh replikasi selektif, migrasi versi, atau multi-master-ish → pakai **Logical Replication**.

Tutorial ini fokus ke **Streaming Replication** dulu karena itu yang paling umum dipakai untuk HA, lalu di bagian akhir kita bahas Logical Replication secara ringkas.

---

## 3. Cara Kerja Streaming Replication (Konsepnya Dulu)

1. Primary menjalankan proses **WAL sender** untuk setiap replica yang terhubung.
2. Replica menjalankan proses **WAL receiver** yang terus terhubung ke primary lewat koneksi replikasi (mirip koneksi biasa, tapi mode khusus).
3. Setiap kali ada perubahan data di primary, WAL record ditulis, lalu **langsung di-stream** ke replica — tidak menunggu di-flush ke disk dulu seperti replikasi berbasis file.
4. Replica menerima WAL, menulis ke WAL lokal, lalu proses **startup/recovery** me-*replay* WAL tersebut ke data file — sama persis seperti proses crash recovery, cuma di sini dia jalan terus tanpa henti.

Karena berbasis WAL yang sama dengan mekanisme crash recovery, ini disebut **physical replication** — bit demi bit, bukan menerjemahkan ulang query.

### Mode: Asynchronous vs Synchronous

| Mode | Cara kerja | Analogi MySQL |
|---|---|---|
| **Asynchronous** (default) | Primary commit transaksi tanpa menunggu replica konfirmasi terima WAL. Ada risiko *data loss* kecil jika primary crash sebelum WAL sampai ke replica. | Mirip *async replication* MySQL (default) |
| **Synchronous** | Primary menunggu minimal satu replica (`synchronous_standby_names`) konfirmasi WAL sudah diterima/ditulis sebelum commit dianggap sukses. Lebih aman, tapi latency naik. | Mirip *semi-sync replication* MySQL |

---

## 4. Praktik: Setup Streaming Replication (Primary → 1 Replica)

Anggap kita punya 2 cluster di 1 server:
- **Port Primary**: `5433`
- **Port Replica**: `5434`

### Langkah 1 — Konfigurasi di Primary

Edit `postgresql.conf` di server primary:

```conf
listen_addresses = '*'
wal_level = replica          # wajib, ini level minimum untuk streaming replication
max_wal_senders = 10         # jumlah maksimum proses WAL sender (jumlah replica + buffer)
wal_keep_size = 1GB          # seberapa banyak WAL disimpan agar replica tidak "ketinggalan"
hot_standby = on             # izinkan replica menerima query SELECT
```

> `wal_level = replica` ini setara dengan mengaktifkan binlog di MySQL — tanpa ini, WAL tidak menyimpan cukup informasi untuk replikasi.

<img width="823" height="339" alt="Screenshot_19" src="https://github.com/user-attachments/assets/4501592d-05c8-4962-b864-57121f8b9d68" />

Edit `pg_hba.conf` supaya replica boleh konek dengan mode replikasi:

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    replication     replikator      127.0.0.1/32         scram-sha-256
```

<img width="716" height="172" alt="Screenshot_18" src="https://github.com/user-attachments/assets/12122e9c-af78-48f0-9615-dc8a16b5b1de" />

Buat user khusus untuk replikasi (jangan pakai superuser biasa):

```sql
CREATE ROLE replikator WITH REPLICATION LOGIN PASSWORD 'password_kuat';
```

<img width="710" height="207" alt="Screenshot_20" src="https://github.com/user-attachments/assets/98511188-8cca-42b9-b6c4-74eb23fc01b2" />

Restart PostgreSQL di primary supaya `postgresql.conf` yang butuh restart (seperti `wal_level`, `max_wal_senders`) diterapkan:

```bash
sudo systemctl restart postgresql
```

### Langkah 2 — Ambil Base Backup dari Primary ke Replica

Ini seperti mengambil "snapshot awal" data sebelum replica mulai streaming WAL. Jalankan **di server replica**:

```bash
sudo systemctl stop postgresql          # pastikan data directory kosong/tidak dipakai
rm -rf /var/lib/postgresql/16/main/*    # kosongkan data directory replica (sesuaikan path & versi)

/usr/pgsql-15/bin/pg_basebackup -h 127.0.0.1 -p5433 -D /var/lib/postgresql/16/main \
  -U replikator -P -R -X stream -C -S replica_slot1
```

Penjelasan opsi-opsi penting:
- `-D` : lokasi data directory tujuan di replica.
- `-P` : tampilkan progress.
- `-R` : otomatis membuat file `standby.signal` dan konfigurasi koneksi (`primary_conninfo`) — ini pengganti `recovery.conf` di versi lama.
- `-X stream` : streaming WAL sekalian saat backup diambil, supaya tidak ada gap data.
- `-C -S replica_slot1` : membuat **replication slot** bernama `replica_slot1` di primary (dibahas di bagian 5).

> Ini kira-kira setara dengan `mysqldump`/`xtrabackup` + `CHANGE MASTER TO` di MySQL, tapi digabung dalam satu perintah.

### Langkah 3 — Nyalakan Replica

```bash
sudo chown -R postgres:postgres /var/lib/postgresql/16/main
sudo systemctl start postgresql
```

Cek file `standby.signal` ada di data directory — keberadaan file ini yang membuat PostgreSQL tahu dirinya harus jalan sebagai **standby**, bukan primary.

### Langkah 4 — Verifikasi Replikasi Berjalan

Di **primary**, cek proses WAL sender aktif:

```sql
SELECT client_addr, state, sync_state, replay_lag
FROM pg_stat_replication;
```

Di **replica**, cek statusnya:

```sql
SELECT pg_is_in_recovery();   -- harus TRUE, artinya ini memang standby
SELECT status FROM pg_stat_wal_receiver;
```

Coba insert data di primary, lalu `SELECT` di replica — datanya harus muncul dalam hitungan detik (atau instan kalau sync).

---

## 5. Replication Slot — Konsep Penting yang Tidak Ada di MySQL

Di MySQL, kalau replica putus koneksi terlalu lama, binlog lama bisa saja sudah di-*purge*, dan kamu harus setup ulang dari nol.

PostgreSQL punya solusi bawaan: **replication slot**. Ini adalah "penanda" di primary yang memastikan WAL **tidak dihapus** sampai replica yang terhubung ke slot itu benar-benar sudah menerimanya — meskipun replica sedang offline.

```sql
-- Lihat slot yang ada
SELECT slot_name, active, restart_lsn FROM pg_replication_slots;

-- Hapus slot yang sudah tidak dipakai (PENTING dilakukan, kalau tidak WAL bisa menumpuk terus dan memenuhi disk)
SELECT pg_drop_replication_slot('replica_slot1');
```

⚠️ **Peringatan**: kalau replica mati permanen tapi slot-nya tidak dihapus, primary akan terus menyimpan WAL tanpa batas sampai disk penuh. Ini beda dengan `wal_keep_size` yang punya batas atas otomatis.

---

## 6. Mengaktifkan Synchronous Replication (Opsional)

Kalau butuh jaminan zero data loss, di `postgresql.conf` primary:

```conf
synchronous_standby_names = 'replica_slot1'
```

Ganti nilai `synchronous_commit` sesuai kebutuhan trade-off latency vs durability:

```conf
synchronous_commit = on   # default 'on' untuk async; ubah ke sesuai kebutuhan
```

---

## 7. Failover: Promosi Replica Jadi Primary

Kalau primary mati dan kamu perlu replica jadi primary baru:

```bash
pg_ctl promote -D /var/lib/postgresql/16/main
```

atau lewat SQL (PG 12+):

```sql
SELECT pg_promote();
```

Setelah promote, file `standby.signal` otomatis dihapus dan server mulai menerima tulisan (`INSERT/UPDATE/DELETE`) seperti primary biasa.

> Catatan: PostgreSQL **tidak punya failover otomatis bawaan** seperti MySQL Group Replication atau InnoDB Cluster. Untuk failover otomatis, biasanya pakai tools tambahan seperti **Patroni**, **repmgr**, atau **pg_auto_failover**.

---

## 8. Ringkas: Logical Replication

Kalau nanti kamu butuh replikasi selektif (misal cuma 1-2 tabel, atau ke versi PostgreSQL berbeda), konsepnya publish/subscribe:

**Di primary (jadi "publisher"):**
```sql
CREATE PUBLICATION pub_produk FOR TABLE produk, kategori;
```

**Di replica (jadi "subscriber"):**
```sql
CREATE SUBSCRIPTION sub_produk
CONNECTION 'host=192.168.1.10 dbname=tokoku user=replikator password=xxx'
PUBLICATION pub_produk;
```

Bedanya dengan streaming replication: replica tetap bisa menulis data lain, dan tabel yang direplikasi harus sudah ada strukturnya (schema) di sisi subscriber — tidak dibuat otomatis.

---

## 9. Ringkasan Perbandingan Akhir

| Aspek | MySQL | PostgreSQL |
|---|---|---|
| Log yang dikirim | Binlog | WAL |
| Replica read-only? | Bisa read-only, bisa juga writable (tergantung config) | Physical replica: selalu read-only. Logical: bisa writable untuk tabel lain |
| Failover otomatis bawaan | Ya (Group Replication/InnoDB Cluster) | Tidak, butuh tools tambahan (Patroni/repmgr) |
| Selective replication | Perlu filter di config | Native lewat Logical Replication |
| Slot/pencegah log terhapus | Tidak ada konsep persis sama | Replication slot |
