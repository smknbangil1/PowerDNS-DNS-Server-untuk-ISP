Berikut adalah konfigurasi lengkap untuk implementasi **PowerDNS Recursor + Redis + PowerDNS Authoritative + MariaDB** tanpa load balancing.  

---

## **📌 Arsitektur & IP Address**  
```
[ISP Clients]  
      │  
      ▼  
┌──────────────────────────────────────────────────┐  
│ PowerDNS Recursor (10.10.10.53)                 │  
│ - Resolver utama & caching                      │  
│ - Terhubung ke Redis untuk cache tambahan       │  
│ - Forward domain blokir ke PowerDNS Authoritative │  
└──────────────────────────────────────────────────┘  
            │                      │  
            ▼                      ▼  
┌───────────────────┐        ┌──────────────────────────┐  
│ Redis Cache      │        │ PowerDNS Authoritative  │  
│ (10.10.10.54)   │        │ (10.10.10.55)           │  
│ - Caching cepat  │        │ - Filtering via MariaDB │  
└───────────────────┘        └──────────────────────────┘  
                                   │  
                                   ▼  
                  ┌──────────────────────────┐  
                  │ MariaDB (10.10.10.56)    │  
                  │ - Database daftar blokir │  
                  └──────────────────────────┘  
```
---

# **1️⃣ Konfigurasi PowerDNS Recursor (10.10.10.53)**
**File:** `/etc/powerdns/recursor.conf`
```ini
# Aktifkan multi-threading untuk menangani banyak query
threads=8

# Gunakan Redis untuk caching
redis-cache-ttl=300
redis-cache-connection="10.10.10.54:6379"

# Forward query ke PowerDNS Authoritative untuk domain yang diblokir
forward-zones-file=/etc/powerdns/blocked-zones.conf
```
**File:** `/etc/powerdns/blocked-zones.conf`  
(Domain yang diblokir akan ditangani oleh PowerDNS Authoritative di 10.10.10.55)
```
badwebsite.com=10.10.10.55
malware-site.net=10.10.10.55
ads.example.com=10.10.10.55
```
🚀 **Restart PowerDNS Recursor:**  
```bash
systemctl restart pdns-recursor
```

---

# **2️⃣ Konfigurasi Redis untuk Caching Cepat (10.10.10.54)**
**1. Install Redis**  
```bash
yum install redis -y
systemctl enable --now redis
```
**2. Ubah konfigurasi Redis (`/etc/redis.conf`)**  
```ini
bind 10.10.10.54
maxmemory 1gb
maxmemory-policy allkeys-lru
```
🚀 **Restart Redis:**  
```bash
systemctl restart redis
```

---

# **3️⃣ Konfigurasi PowerDNS Authoritative (10.10.10.55)**
**File:** `/etc/powerdns/pdns.conf`
```ini
launch=gmysql
gmysql-host=10.10.10.56
gmysql-user=powerdns
gmysql-password=yourpassword
gmysql-dbname=powerdns

# Aktifkan zona blokir
local-address=10.10.10.55
```
🚀 **Restart PowerDNS Authoritative:**  
```bash
systemctl restart pdns
```

---

# **4️⃣ Konfigurasi MariaDB (10.10.10.56) untuk Filtering**
**1. Install MariaDB:**  
```bash
yum install mariadb-server -y
systemctl enable --now mariadb
mysql_secure_installation
```
**2. Buat Database dan User untuk PowerDNS:**  
```sql
CREATE DATABASE powerdns;
GRANT ALL PRIVILEGES ON powerdns.* TO 'powerdns'@'10.10.10.%' IDENTIFIED BY 'yourpassword';
FLUSH PRIVILEGES;
```
**3. Buat Tabel untuk Filtering Domain:**  
```sql
USE powerdns;
CREATE TABLE domains (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(6) NOT NULL DEFAULT 'BLOCKED'
);
```
**4. Tambahkan Domain ke Daftar Blokir:**  
```sql
INSERT INTO domains (name, type) VALUES ('badwebsite.com', 'BLOCKED');
INSERT INTO domains (name, type) VALUES ('malware-site.net', 'BLOCKED');
INSERT INTO domains (name, type) VALUES ('ads.example.com', 'BLOCKED');
```
🚀 **Restart MariaDB:**  
```bash
systemctl restart mariadb
```

---

## **📌 Kesimpulan**
✅ **PowerDNS Recursor** (10.10.10.53) menangani query dari klien ISP.  
✅ **Redis** (10.10.10.54) menyimpan cache untuk mempercepat resolusi DNS.  
✅ **PowerDNS Authoritative** (10.10.10.55) memfilter domain yang diblokir.  
✅ **MariaDB** (10.10.10.56) menyimpan daftar domain yang difilter.  

🚀 **Mau saya buatkan skrip otomatis untuk deploy ini di server Anda?**
