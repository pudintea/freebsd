Kalau tujuan Anda adalah **FreeBSD + Nginx sebagai reverse proxy** di depan aplikasi, dan SSL ingin dikelola melalui **Cloudflare**, saya sarankan tetap menggunakan sertifikat di server dengan mode **Full (strict)**. Ini lebih aman daripada mode **Full** biasa karena Cloudflare juga memverifikasi sertifikat origin.

## Arsitektur

```text
Internet
    │
    ▼
Cloudflare
    │ HTTPS
    ▼
Nginx (FreeBSD)
    │
    ├── App 1 : localhost:3000
    ├── App 2 : localhost:8080
    └── App 3 : localhost:5000
```

---

# 1. Update sistem

```bash
freebsd-update fetch
freebsd-update install

pkg update
pkg upgrade
```

---

# 2. Install Nginx

```bash
pkg install nginx
```

Cek lokasi konfigurasi:

```bash
whereis nginx
```

Biasanya:

```text
/usr/local/etc/nginx/nginx.conf
```

---

# 3. Aktifkan Nginx saat boot

```bash
sysrc nginx_enable="YES"
```

Start:

```bash
service nginx start
```

Cek:

```bash
sockstat -4 -l
```

Harus terlihat port 80.

---

# 4. Buka DNS di Cloudflare

Misalnya domain:

```text
example.com
```

Buat record:

```text
Type: A
Name: @
IPv4: IP_SERVER_ANDA
Proxy Status: Proxied (awan oranye)
```

Dan:

```text
Type: A
Name: www
IPv4: IP_SERVER_ANDA
Proxy Status: Proxied
```

---

# 5. Buat Cloudflare Origin Certificate

Masuk ke:

[Cloudflare Dashboard](https://dash.cloudflare.com?utm_source=chatgpt.com)

Lalu:

```text
SSL/TLS
↓
Origin Server
↓
Create Certificate
```

Pilih:

```text
RSA 2048
Validity 15 years
```

Cloudflare akan memberikan:

* Origin Certificate
* Private Key

---

# 6. Simpan sertifikat di FreeBSD

Buat direktori:

```bash
mkdir -p /usr/local/etc/nginx/ssl
chmod 700 /usr/local/etc/nginx/ssl
```

Simpan certificate:

```bash
nano /usr/local/etc/nginx/ssl/cloudflare.crt
```

Paste sertifikat.

Simpan private key:

```bash
nano /usr/local/etc/nginx/ssl/cloudflare.key
```

Paste private key.

Atur permission:

```bash
chmod 600 /usr/local/etc/nginx/ssl/*
```

---

# 7. Konfigurasi Reverse Proxy

Edit:

```bash
nano /usr/local/etc/nginx/nginx.conf
```

Contoh sederhana:

```nginx
events {
    worker_connections 1024;
}

http {

    server {
        listen 80;
        server_name example.com www.example.com;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name example.com www.example.com;

        ssl_certificate     /usr/local/etc/nginx/ssl/cloudflare.crt;
        ssl_certificate_key /usr/local/etc/nginx/ssl/cloudflare.key;

        location / {
            proxy_pass http://127.0.0.1:3000;

            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

---

# 8. Test konfigurasi

```bash
nginx -t
```

Kalau muncul:

```text
syntax is ok
test is successful
```

reload:

```bash
service nginx reload
```

atau

```bash
service nginx restart
```

---

# 9. Atur SSL Mode di Cloudflare

Masuk:

```text
SSL/TLS
↓
Overview
```

Pilih:

```text
Full (strict)
```

Jangan pilih:

```text
Flexible
```

karena sering menyebabkan redirect loop dan koneksi backend tidak terenkripsi.

---

# 10. Firewall

Jika memakai PF:

```pf
pass in proto tcp to port {80,443}
```

Reload:

```bash
pfctl -f /etc/pf.conf
```

---

# 11. Validasi

Dari server:

```bash
fetch -o - https://example.com
```

Atau dari PC:

```bash
curl -I https://example.com
```

---

## Jika aplikasi Anda berjalan di Docker/Jail

Misalnya:

* Node.js → `127.0.0.1:3000`
* Laravel → `127.0.0.1:8000`
* Go → `127.0.0.1:8080`

cukup ganti bagian:

```nginx
proxy_pass http://127.0.0.1:3000;
```

ke port aplikasi yang sesuai.

Untuk server production, saya juga menyarankan langkah tambahan setelah Nginx berjalan:

1. Install dan konfigurasi PF firewall.
2. Batasi akses SSH hanya dengan key.
3. Pasang fail2ban atau alternatif FreeBSD.
4. Konfigurasikan log rotation.
5. Aktifkan backup konfigurasi `/usr/local/etc/nginx`.
6. Izinkan hanya IP Cloudflare ke port 443 agar origin tidak bisa diakses langsung dari internet. Ini sangat berguna jika semua trafik memang harus melewati Cloudflare.


## Pudin Saepudin
