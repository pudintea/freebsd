Bagus. Untuk server production, login SSH dengan **public key authentication** jauh lebih aman daripada password.

Karena Anda menggunakan **PuTTYgen** (Windows), saya akan jelaskan dari awal sampai selesai.

---

# Tahap 1 — Buat SSH Key dengan PuTTYgen

Buka PuTTYgen.

## Pilih jenis key

Saya sarankan:

```text
Type of key:
Ed25519
```

Jika tidak ada Ed25519 (PuTTY lama), gunakan:

```text
RSA
4096 bits
```

Ed25519 lebih modern dan lebih cepat.

---

## Generate Key

Klik:

```text
Generate
```

Gerakkan mouse secara acak sampai selesai.

---

## Beri Passphrase

Isi:

```text
Key passphrase:
********
Confirm passphrase:
********
```

Ini melindungi private key jika laptop Anda hilang.

---

## Simpan Private Key

Klik:

```text
Save private key
```

Misalnya simpan sebagai:

```text
id_ed25519.ppk
```

Jangan pernah upload file ini ke server.

---

## Salin Public Key

Di bagian atas ada kotak:

```text
Public key for pasting into OpenSSH authorized_keys file:
```

Bentuknya seperti:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...
```

Salin seluruh baris ini.

---

# Tahap 2 — Upload Public Key ke Server

Login ke server seperti biasa (masih pakai password dulu).

Misalnya user:

```text
admin
```

---

Buat direktori SSH:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

---

Edit file:

```bash
ee ~/.ssh/authorized_keys
```

Paste public key tadi:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA....
```

Simpan.

---

Atur permission:

```bash
chmod 600 ~/.ssh/authorized_keys
```

---

Cek:

```bash
ls -la ~/.ssh
```

Harus kurang lebih:

```text
drwx------  .ssh
-rw-------  authorized_keys
```

---

# Tahap 3 — Test Login dengan Key

**JANGAN matikan password dulu.**

Buka jendela PuTTY baru.

---

Masuk ke:

```text
Connection
→ SSH
→ Auth
→ Credentials
```

Pilih:

```text
Browse
```

Arahkan ke:

```text
id_ed25519.ppk
```

---

Masukkan:

```text
Host Name:
IP_SERVER
```

Klik Open.

---

Jika berhasil:

```text
login as: admin
Authenticating with public key ...
```

Langsung masuk tanpa password akun.

Kalau masih gagal, jangan lanjut ke tahap berikutnya.

---

# Tahap 4 — Backup Konfigurasi SSH

Di server:

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

---

# Tahap 5 — Konfigurasi SSHD

Edit:

```bash
ee /etc/ssh/sshd_config
```

Cari dan ubah:

```text
PubkeyAuthentication yes
```

Pastikan tidak ada tanda #.

---

Pastikan:

```text
AuthorizedKeysFile .ssh/authorized_keys
```

---

Kemudian:

```text
PasswordAuthentication no
```

---

Dan:

```text
ChallengeResponseAuthentication no
```

atau

```text
KbdInteractiveAuthentication no
```

(tergantung versi OpenSSH)

---

Kemudian:

```text
PermitEmptyPasswords no
```

---

Untuk keamanan tambahan:

```text
PermitRootLogin no
```

Root tidak boleh login langsung.

---

Hasil akhirnya kira-kira:

```text
PubkeyAuthentication yes

PasswordAuthentication no

KbdInteractiveAuthentication no

ChallengeResponseAuthentication no

PermitRootLogin no

PermitEmptyPasswords no
```

---

# Tahap 6 — Validasi Konfigurasi

Sebelum restart:

```bash
sshd -t
```

Jika tidak ada output:

```text
OK
```

---

# Tahap 7 — Reload SSH

Jangan restart dulu.

Reload:

```bash
service sshd reload
```

---

# Tahap 8 — Test Lagi

Buka terminal PuTTY baru.

Login menggunakan key.

Pastikan berhasil.

---

# Tahap 9 — Verifikasi Password Sudah Mati

Dari Windows:

Coba login tanpa key.

Harus muncul:

```text
No supported authentication methods available
```

atau

```text
Permission denied (publickey)
```

Artinya password sudah benar-benar dinonaktifkan.

---

# Tahap 10 — Nonaktifkan Login Root (Sangat Direkomendasikan)

Buat user admin:

```bash
pw groupmod wheel -m admin
```

Install sudo:

```bash
pkg install sudo
```

Edit:

```bash
visudo
```

Pastikan ada:

```text
%wheel ALL=(ALL:ALL) ALL
```

Sekarang:

```bash
su -
```

atau:

```bash
sudo -i
```

untuk menjadi root.

---

# Struktur yang Direkomendasikan untuk Production

```text
root
 └─ login langsung: TIDAK

admin
 └─ login dengan SSH key: YA

password authentication
 └─ TIDAK

public key authentication
 └─ YA

sudo
 └─ YA
```

Dengan konfigurasi ini, seseorang yang mengetahui username server Anda tetap tidak bisa login tanpa memiliki private key `.ppk` yang sesuai. Ini adalah konfigurasi standar yang umum digunakan pada VPS dan server production modern.

## Pudin Saepudin
