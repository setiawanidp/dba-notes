<div align="center"> 
   
# Installation Postgresql 15 di Linux centOS 

</div>

1. Buka browser lalu pergi ke website postgresql cari halaman download. Pada kasus ini saya menggunakan link download berikut https://download.postgresql.org/pub/repos/yum/15/redhat/rhel-9-x86_64/    

   Kemudian cari file dengan nama-nama berikut:
    - postgresql15-15.15-1PGDG.rhel9.x86_64.rpm
    - postgresql15-libs-15.15-1PGDG.rhel9.x86_64.rpm
    - postgresql15-server-15.15-1PGDG.rhel9.x86_64.rpm
2. Pindahkan ke server linux centos yang akan di instal postgresql, misalnya di folder /postgresql/instaler
3. Masuk dengan akses yang sudah diberi akses sudo atau menggunakan root
    
    ```
    cd /postgresql/instaler
    ```
    
    Ketik perintah berikut untuk instal ketiga rpm yang sudah di copy tadi
    
    ```
    sudo yum localinstall *.rpm --disablerepo="*"
    ```
    
    Ketika sudah di instal rpm akan ada binary di path `/usr/pgsql-15/bin`
    
4. Jalankan perintah inisialisasi berikut
    
    ```
    sudo /usr/pgsql-15/bin/postgresql-15-setup initdb 
    ```
   Setetelah di inisialisasi maka postgresql akan membuat struktur datadir di directory `/var/lib/pgsql/15/data`
   
    >***untuk menjalankan perintah inisialisasi secara manual dan menginstal datadir di directory yang berbeda dapat dilakukan dengan perintah berikut***
    >```
    >cp -r /var/lib/pgsql/15/data /postgresql/database/
    >mv /postgresql/database/data/ /postgresql/database/pgdata_b/
    >sudo chown -R postgres:postgres /postgresql/database/pgdata_b/
    >sudo chmod 700 /postgresql/database/pgdata_b/
    >sudo su - postgres
    >/usr/pgsql-15/bin/initdb -D /postgresql/database/pgdata_b/ -U postgres
    >```
    >![!image.png](https://github.com/setiawanidp/dba-notes/blob/main/postgresql/images/inisialisasi%20postgresql%20manual.png)

   Masuk ke dalam directory `/postgresql/database/pgdata_b/` edit file `postgresql.conf` dibagian `#port = 5432` hilangkan tanda #
    
6. Jika sudah berhasil jalankan perintah berikut untuk memulai service postgresql
    
    ```
    sudo systemctl start postgresql-15
    sudo systemctl enable postgresql-15
    ```
    
    >***start service dengan inisialisasi manual***
    >
    >```
    >/usr/pgsql-15/bin/pg_ctl start -D /postgresql/database/pgdata_b/ -l /postgresql/database/pgdata_b/log-b.log
    >```
    >
    >![!image.png](https://github.com/setiawanidp/dba-notes/blob/main/postgresql/images/start%20postgresql%20manual.png)
    
7. Cek service database dengan menjalankan perintah berikut
    
    ```
    sudo systemctl status postgresql-15
    ```
    
    Jika tampil seperti gambar berikut, service database sudah sukses berjalan
   
   ![cekservice.png](https://github.com/setiawanidp/dba-notes/blob/main/postgresql/images/cek%20service%20postgresql%20auto.png)
