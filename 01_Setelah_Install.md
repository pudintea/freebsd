# Setelah selesai Install FreeBSD
* [Link Dokumentasi](https://docs.freebsd.org/en/books/handbook/book/#ports)

## Repository
```
mkdir -p /usr/local/etc/pkg/repos
```
```
ls /usr/local/etc/pkg/repos
```
```
touch /usr/local/etc/pkg/repos/FreeBSD.conf
```
```
echo 'FreeBSD-ports: { url: "pkg+https://pkg.FreeBSD.org/${ABI}/latest" }' > /usr/local/etc/pkg/repos/FreeBSD.conf
```
```
pkg
```

Then run this command to update the local package repositories catalogues for the Latest branch:
```
pkg update -f
```
```
pkg upgrade
```

## Install Editor Nano
```
pkg install nano
```

## Pudin Saepudin
