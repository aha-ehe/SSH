# Laporan Audit Keamanan Sistem

**Tanggal Audit:** 16 Februari 2026
**Target:** 36.88.105.236 (Hostname: `unira5`)
**Auditor:** Jules (AI Security Engineer)

## 1. Ringkasan Eksekutif

Sistem `unira5` (Debian 12) berada dalam kondisi **SANGAT KRITIS**. Ditemukan bukti kuat adanya kompromi sistem (backdoor), penggunaan alat penyerangan aktif (Bettercap, Metasploit), dan konfigurasi keamanan yang sangat lemah. Sistem ini tampaknya digunakan sebagai "jump box" atau mesin serangan, namun juga memiliki kerentanan yang membuatnya mudah diambil alih oleh pihak lain.

**Temuan Utama:**
-   **Backdoor Aktif:** Ditemukan file binary SUID root yang menyamar sebagai proses sistem.
-   **Akun Mencurigakan:** Akun layanan sistem memiliki password yang diset, yang mengindikasikan persistensi penyerang.
-   **Konfigurasi SSH Lemah:** Login root diizinkan dengan autentikasi password.
-   **Layanan Berisiko:** Webmin, VNC, dan RabbitMQ terekspos ke internet.
-   **Perangkat Lunak Usang:** Kernel dan PHP (versi EOL) membutuhkan pembaruan.

## 2. Informasi Sistem

-   **OS:** Debian GNU/Linux 12 (bookworm)
-   **Kernel:** Linux 6.1.0-41-amd64
-   **IP Address:** 36.88.105.236
-   **User Audit:** `unira5` (dengan akses `su -` ke root)

## 3. Temuan Kritis (Critical)

### 3.1. Backdoor Binary SUID
Ditemukan dua file mencurigakan di `/usr/bin/` dengan bit SUID root aktif. File ini menyamar sebagai komponen systemd tetapi bukan bagian dari paket standar.
-   `/usr/bin/.systemd-update` (SUID Root)
-   `/usr/bin/.system-task` (SUID Root)

**Rekomendasi:** Segera isolasi sistem. Jangan hanya menghapus file ini; sistem harus dianggap *compromised* total. Analisis forensik lebih lanjut diperlukan jika data penting ada di sana. Reinstall ulang sistem operasi sangat disarankan.

### 3.2. Akun Layanan dengan Password
Akun yang seharusnya *locked* (tidak bisa login) ditemukan memiliki hash password di `/etc/shadow`. Ini adalah teknik umum untuk *persistence* (mempertahankan akses).
-   `systemd-netconf`
-   `dbus-worker`
-   `dbus-service`
-   `ima`

**Rekomendasi:** Kunci akun-akun ini segera (`passwd -l <user>`) atau hapus jika tidak valid.

### 3.3. SSH Root Login Diizinkan
Konfigurasi SSH (`/etc/ssh/sshd_config`) mengizinkan login root langsung dengan password.
-   `PermitRootLogin yes`
-   `PasswordAuthentication yes`

**Rekomendasi:** Ubah `PermitRootLogin` menjadi `no` atau `prohibit-password` dan nonaktifkan `PasswordAuthentication` (gunakan SSH Key).

### 3.4. Alat Penyerangan Berjalan sebagai Root
Proses `bettercap` berjalan sebagai root (PID 735470). Bettercap adalah alat *Man-in-the-Middle* yang kuat. Keberadaannya dan status *running*-nya menunjukkan mesin ini sedang aktif melakukan aktivitas jaringan yang agresif.
Direktori `/root` juga penuh dengan skrip eksploitasi (`brute_ojs.py`, `dump_pma_228.py`, dll) dan hasil scan (`nikto_results.txt`).

**Rekomendasi:** Hentikan proses ini jika tidak diotorisasi.

## 4. Temuan Risiko Tinggi (High)

### 4.1. Layanan Terekspos (Publicly Accessible)
Port berikut terbuka untuk dunia luar (`0.0.0.0`):
-   **TCP 10000 (Webmin):** Target serangan umum. Versi yang terinstall (2.600) bukan yang terbaru (2.621 tersedia).
-   **TCP 5900 (VNC):** Jika password lemah, penyerang bisa melihat desktop secara grafis.
-   **TCP 5674 (RabbitMQ):** Pesan antrian bisa disadap atau dimanipulasi.
-   **TCP 8080 & 8081:** Layanan HTTP tambahan.

**Rekomendasi:** Gunakan Firewall (UFW/IPTables) untuk membatasi akses ke port ini hanya dari IP terpercaya (misal VPN atau IP Admin).

### 4.2. Perangkat Lunak Usang & EOL
-   **PHP:** Versi 7.2 dan 7.4 terinstall. Versi ini sudah End-of-Life (EOL) dan tidak lagi menerima patch keamanan.
-   **Kernel:** Versi 6.1.158 terinstall, pembaruan ke 6.1.162 tersedia.
-   **Webmin:** Versi usang.

**Rekomendasi:** Hapus versi PHP yang EOL, perbarui Kernel dan Webmin.

### 4.3. Kebijakan Password Lemah
-   `PASS_MAX_DAYS 99999` di `/etc/login.defs`: Password tidak pernah kadaluarsa.
-   Tidak ada modul `pam_pwquality` atau `pam_cracklib`: Pengguna bisa membuat password yang sangat lemah.

**Rekomendasi:** Konfigurasikan rotasi password (misal 90 hari) dan instal `libpam-pwquality` untuk menegakkan kompleksitas password.

## 5. Kesimpulan

Sistem ini sangat tidak aman dan kemungkinan besar sudah dikuasai oleh pihak ketiga (atau digunakan sebagai alat serangan oleh pemiliknya dengan cara yang sangat tidak aman). Prioritas utama adalah mengamankan akses root (menghapus backdoor SUID, mengunci akun layanan, membatasi SSH) dan membatasi eksposur jaringan (Firewall).

Jika ini adalah lingkungan produksi, **sangat disarankan untuk mem-backup data dan melakukan instalasi ulang bersih (clean install)** karena integritas sistem sudah tidak dapat dipercaya akibat adanya binary backdoor SUID.
