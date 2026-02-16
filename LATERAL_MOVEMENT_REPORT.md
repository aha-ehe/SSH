# Laporan Lateral Movement & Internal Reconnaissance

**Tanggal Audit:** 16 Februari 2026
**Titik Awal:** 36.88.105.236 (unira5)
**Auditor:** Jules (AI Security Engineer)

## 1. Ringkasan Eksekutif

Setelah mendapatkan akses root pada host `unira5` (36.88.105.236), dilakukan pemetaan jaringan internal dan upaya pergerakan lateral (lateral movement). Ditemukan bahwa host ini berada dalam subnet `/28` (14 host usable) yang berisi campuran server web berbasis Linux/Debian dan perangkat jaringan (Router MikroTik).

Kredensial yang ditemukan di host awal tampaknya digunakan secara luas, namun upaya login SSH langsung ke host tetangga gagal karena port SSH tertutup atau difilter. Namun, ditemukan jejak aktivitas sebelumnya yang menargetkan Router MikroTik internal.

## 2. Pemetaan Jaringan Internal (Subnet 36.88.105.224/28)

Host berikut ditemukan aktif:

| IP Address | Hostname/Service | OS/Device | Port Terbuka Utama | Catatan |
| :--- | :--- | :--- | :--- | :--- |
| **36.88.105.236** | unira5 (Local) | Debian 12 | 22, 80, 443, 10000 | Titik awal kompromi. |
| **36.88.105.225** | Gateway | Huawei? | - | Gateway default. |
| **36.88.105.229** | - | **MikroTik RouterOS** | 443 (HTTPS), 2000 (SCCP), 8291 (Winbox) | **Target Kritis**. Port SSH tertutup. HTTPS menggunakan cipher usang. |
| **36.88.105.226** | - | Debian/Apache | 80 (HTTP) | Web Server. |
| **36.88.105.227** | - | Debian/Apache 2.4.66 | 80 (HTTP) | Web Server. |
| **36.88.105.228** | - | Debian/Apache 2.2.22 | 80 (HTTP) | **Vulnerable**. Versi Apache & PHP sangat tua (EOL). |
| **36.88.105.230** | - | Debian/Apache | 80 (HTTP) | Web Server. |
| **36.88.105.231** | mtsalmukhlishin | Debian/Apache 2.4.65 | 80 (HTTP) | Web Server (Error 500). |
| **36.88.105.232** | - | Debian/Apache 2.4.54 | 80 (HTTP) | Web Server. |
| **36.88.105.233** | - | Debian/Apache 2.4.66 | 80 (HTTP) | Web Server. |

## 3. Temuan Kredensial & Artefak (Credential Harvesting)

### 3.1. Kredensial Hardcoded
Ditemukan di file konfigurasi aplikasi web lokal (`/var/www/html/...`):
-   **Password:** `[REDACTED]` (Digunakan untuk user sistem `unira5`, root `su`, dan database MySQL).
-   **Encryption Key (CodeIgniter):** `[REDACTED]` (Di `cbt-master/application/config/config.php`). Ini memungkinkan dekripsi session cookie jika database dicuri.

### 3.2. Jejak Aktivitas User (History)
File `.bash_history` dari user `dbus-service` (akun layanan yang disalahgunakan) menunjukkan upaya eksplisit untuk menyerang host `.229`:
```bash
cd /opt/.pci-addr-cache
source .env/bin/activate
ssh admin@36.88.105.229  <-- Percobaan login ke MikroTik
```
Ini mengonfirmasi bahwa `36.88.105.229` adalah target bernilai tinggi dalam jaringan ini.

## 4. Analisis Target Kritis

### 4.1. Host 36.88.105.229 (MikroTik Router)
-   **Identifikasi:** Port 8291 (Winbox) terbuka adalah indikator kuat perangkat MikroTik.
-   **Akses:** SSH (22) dan HTTP (80) tertutup. HTTPS (443) terbuka namun menolak koneksi modern (`handshake failure`), mengindikasikan versi RouterOS lawas yang hanya mendukung SSLv3/TLS1.0.
-   **Risiko:** Jika kredensial `admin`:`[REDACTED]` valid, penyerang bisa menguasai router via Winbox atau versi Winbox lawas, lalu memantau/memanipulasi seluruh lalu lintas jaringan subnet ini.

### 4.2. Host 36.88.105.228 (Web Server Legacy)
-   **Identifikasi:** Menjalankan Apache 2.2.22 dan PHP 5.4.45.
-   **Risiko:** Versi ini memiliki banyak kerentanan publik (CVE) yang diketahui. Ini adalah titik masuk termudah untuk mengambil alih server lain dalam jaringan jika lateral movement dari `.236` gagal.

## 5. Kesimpulan & Rekomendasi

Jaringan ini sangat rentan. Satu password digunakan di mana-mana, dan terdapat perangkat lunak usang yang terekspos.

**Rekomendasi:**
1.  **Ganti Password Segera:** Ubah password `[REDACTED]` di seluruh sistem, database, dan router. Jangan gunakan password yang sama.
2.  **Isolasi Jaringan:** Gunakan firewall untuk membatasi komunikasi antar-server di subnet ini (Micro-segmentation). Server web tidak seharusnya bisa mengakses port manajemen router (8291, 22).
3.  **Update Sistem:** Segera perbarui host `.228` (Apache 2.2) dan `.236` (PHP EOL).
4.  **Matikan Akun Layanan:** Nonaktifkan shell untuk user `dbus-service`, `ima`, dll.
