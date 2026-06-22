PF (*Packet Filter*) adalah firewall bawaan FreeBSD yang sangat cocok untuk server reverse proxy. Untuk server Nginx di belakang Cloudflare, tujuan utamanya biasanya:

* Mengizinkan SSH untuk administrasi.
* Mengizinkan HTTP/HTTPS.
* Memblokir koneksi yang tidak diperlukan.
* (Opsional) Membatasi akses ke origin server hanya dari IP Cloudflare.

Saya akan mulai dari konfigurasi yang aman dan mudah dipahami.

---

# 1. Backup konfigurasi dulu

```bash
cp /etc/pf.conf /etc/pf.conf.backup
```

---

# 2. Buat konfigurasi PF sederhana

Edit:

```bash
ee /etc/pf.conf
```

atau:

```bash
nano /etc/pf.conf
```

Isi dengan:

```pf
ext_if="vtnet0"

set skip on lo

block all

pass out keep state

pass in proto tcp to ($ext_if) port 22 keep state
pass in proto tcp to ($ext_if) port 80 keep state
pass in proto tcp to ($ext_if) port 443 keep state
```

**Catatan:** ganti `vtnet0` dengan nama interface server Anda.

Cek interface:

```bash
ifconfig
```

Contoh umum:

```text
vtnet0
em0
igc0
re0
```

---

# 3. Uji konfigurasi

Jangan langsung aktifkan sebelum test.

```bash
pfctl -nf /etc/pf.conf
```

Kalau benar:

```text
/etc/pf.conf: OK
```

---

# 4. Aktifkan PF

Tambahkan ke `/etc/rc.conf`:

```bash
sysrc pf_enable="YES"
sysrc pflog_enable="YES"
```

Start:

```bash
service pf start
service pflog start
```

Cek:

```bash
pfctl -s info
```

---

# 5. Lihat aturan yang aktif

```bash
pfctl -sr
```

---

# 6. Reload saat ada perubahan

Setelah mengedit `/etc/pf.conf`:

```bash
pfctl -f /etc/pf.conf
```

Tidak perlu reboot.

---

# 7. Lihat log firewall

```bash
tcpdump -n -e -ttt -i pflog0
```

Atau:

```bash
tcpdump -n -i pflog0
```

---

# 8. Konfigurasi yang lebih cocok untuk Nginx Reverse Proxy

Misalnya:

* SSH = 22
* HTTP = 80
* HTTPS = 443
* Semua lainnya ditolak

```pf
ext_if="vtnet0"

set block-policy drop
set skip on lo

scrub in all

block all
pass out keep state

pass in proto tcp to ($ext_if) port 22 keep state
pass in proto tcp to ($ext_if) port {80 443} keep state
```

Reload:

```bash
pfctl -f /etc/pf.conf
```

---

# 9. Batasi SSH hanya dari IP Anda (lebih aman)

Misalnya IP rumah/kantor:

```text
103.20.10.50
```

Maka:

```pf
pass in proto tcp from 103.20.10.50 to ($ext_if) port 22 keep state
```

Hapus rule SSH yang terbuka untuk semua.

---

# 10. Proteksi brute force SSH

Tambahkan:

```pf
table <ssh_bruteforce> persist

block in quick from <ssh_bruteforce>

pass in proto tcp to port 22 \
    keep state \
    (max-src-conn-rate 5/60, overload <ssh_bruteforce> flush)
```

Artinya:

* lebih dari 5 koneksi dalam 60 detik
* IP otomatis masuk blacklist PF

---

# 11. Batasi origin hanya untuk Cloudflare (sangat direkomendasikan)

Kalau semua trafik web harus lewat Cloudflare:

Buat file:

```bash
mkdir -p /usr/local/etc/pf
```

```bash
fetch -o /usr/local/etc/pf/cloudflare_ipv4.txt \
https://www.cloudflare.com/ips-v4
```

```bash
fetch -o /usr/local/etc/pf/cloudflare_ipv6.txt \
https://www.cloudflare.com/ips-v6
```

Lalu di `pf.conf`:

```pf
table <cloudflare_ipv4> persist file "/usr/local/etc/pf/cloudflare_ipv4.txt"
table <cloudflare_ipv6> persist file "/usr/local/etc/pf/cloudflare_ipv6.txt"

pass in proto tcp from <cloudflare_ipv4> to ($ext_if) port {80 443}
pass in proto tcp from <cloudflare_ipv6> to ($ext_if) port {80 443}
```

Sekarang:

* User → Cloudflare → Server = boleh
* User → IP server langsung = ditolak

Ini mencegah orang bypass Cloudflare.

---

# 12. Verifikasi

Lihat statistik:

```bash
pfctl -s info
```

Lihat state:

```bash
pfctl -s state
```

Lihat tabel Cloudflare:

```bash
pfctl -t cloudflare_ipv4 -T show
```

---

# Contoh Konfigurasi Production Sederhana

```pf
ext_if="vtnet0"

table <ssh_bruteforce> persist

set block-policy drop
set skip on lo

scrub in all

block all
pass out keep state

block in quick from <ssh_bruteforce>

pass in proto tcp to ($ext_if) port 22 \
    keep state \
    (max-src-conn-rate 5/60, overload <ssh_bruteforce> flush)

pass in proto tcp to ($ext_if) port {80 443} keep state
```

Ini sudah cukup baik untuk:

* FreeBSD VPS
* Nginx Reverse Proxy
* Website di belakang Cloudflare
* Server production kecil hingga menengah

Saran penting: saat pertama kali mengaktifkan PF melalui SSH, buka **dua sesi SSH**. Setelah menjalankan `pfctl -f /etc/pf.conf`, pastikan sesi kedua masih bisa login. Jika salah konfigurasi dan terkunci, Anda masih punya sesi pertama untuk memperbaikinya.


## Pudin Saepudin
