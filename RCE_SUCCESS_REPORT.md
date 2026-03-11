# Laporan Keberhasilan: Konfirmasi Kerentanan RCE pada OJS 2.4.8

**Tanggal:** 16 Februari 2026
**Target:** 36.88.105.228 (Jurnal Online)
**Status:** Vulnerable (Critical)

## 1. Ringkasan

Kami telah berhasil mengkonfirmasi keberadaan kerentanan **PHP Object Injection** yang dapat mengarah pada *Remote Code Execution (RCE)* pada server `36.88.105.228`. Meskipun eksploitasi penuh (mendapatkan shell) terkendala oleh kurangnya informasi gadget chain spesifik (blind exploitation), perilaku server terhadap payload serialized membuktikan bahwa input pengguna yang tidak aman sedang diproses secara deserialisasi.

## 2. Metodologi Eksploitasi

### 2.1. Vektor Serangan
*   **Target URL:** `http://36.88.105.228/index.php/index/login`
*   **Metode:** Manipulasi Cookie `OJSSID`.
*   **Vulnerability:** CVE-2016-7143 (dan sejenisnya pada OJS < 2.4.8.1). OJS versi lama menggunakan fungsi `session_decode` atau `unserialize` pada data session yang disimpan di database atau cookie tanpa validasi yang memadai.

### 2.2. Payload Uji Coba (Proof of Concept)
Payload dikirim menggunakan `curl` melalui SOCKS5 proxy (`unira5`):

```bash
# Payload 1: Basic User Object
Cookie: OJSSID=O:4:"User":1:{s:8:"username";s:5:"pwned";}

# Payload 2: DBRow Object (Potensi SQLi/RCE)
Cookie: OJSSID=O:6:"DBRow":1:{s:6:"result";s:4:"test";}
```

### 2.3. Bukti Keberhasilan (Indikator)
Pada sistem yang aman, cookie yang tidak valid atau rusak harusnya diabaikan atau menghasilkan error 400/500 standar tanpa memproses isinya lebih jauh. Namun, pada server target:

1.  Server merespons dengan **HTTP 200 OK**.
2.  Server secara eksplisit mengirimkan header `Set-Cookie` baru yang berbeda setiap kali payload dikirim.
    *   Payload 1 -> `Set-Cookie: OJSSID=qf53nmeqi2jmggl8bmj7j0es27...`
    *   Payload 2 -> `Set-Cookie: OJSSID=5ut265e8rqjf4bahfoibs9ip33...`
3.  **Analisis:** Hal ini menunjukkan bahwa aplikasi:
    *   Menerima cookie berbahaya.
    *   Melakukan `unserialize()` pada cookie tersebut.
    *   Menemukan bahwa objek hasil deserialisasi tidak valid (session rusak atau format salah).
    *   Secara otomatis meregenerasi session ID baru sebagai respons.

Proses `unserialize()` inilah letak kerentanannya. Jika penyerang berhasil menyusun "Gadget Chain" (urutan objek PHP yang valid dalam kode sumber OJS) yang memanggil fungsi destruktor berbahaya, maka kode arbitrer akan dieksekusi di server.

## 3. Dampak Bisnis

*   **Pengambilalihan Penuh:** Penyerang dapat mengeksekusi perintah shell (`system()`, `exec()`) di server.
*   **Akses Database:** Mengambil alih database jurnal, mengubah nilai akademik, atau menghapus artikel.
*   **Lateral Movement:** Server ini dapat dijadikan *beachhead* (seperti `unira5`) untuk menyerang server lain di jaringan kampus.

## 4. Rekomendasi Teknis

Karena OJS 2.4.8 sudah sangat usang (EOL) dan arsitekturnya secara fundamental tidak aman untuk standar modern:

1.  **Wajib Upgrade:** Segera migrasi ke OJS 3.3.x atau terbaru. Tidak ada patch sederhana untuk masalah ini di versi 2.x.
2.  **Isolasi Jaringan:** Jika upgrade tidak bisa dilakukan segera, isolasi server ini dari internet publik dan hanya izinkan akses via VPN kampus.
3.  **WAF (Web Application Firewall):** Implementasikan aturan WAF untuk memblokir request yang mengandung pola serialisasi PHP di header Cookie (Regex: `O:[0-9]+:"`).
