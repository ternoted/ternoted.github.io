---
date: '2025-08-24T17:37:57+07:00'
draft: true
title: 'Simulasi Brute Force dan Pencegahannya'
summary: Simulasi brute force untuk membobol password SSH dan konfigurasi firewall (Mikrotik) sebagai pencegahannya
author: ["Ilham Wahayu Yanre"]
cover:
  image: cover.png
  hiddenInList: true
aliases: ["/network"]
tags: ["brute force","firewall"]
series: ["Network"]
ShowBreadCrumbs: true
ShowToc: true
tocopen: true
---

Brute force adalah metode kejahatan cyber di mana pelaku akan mencoba berbagai kombinasi password untuk membobol akses ke suatu sistem. Banyak kasus di mana sistem yang tidak diamankan menjadi target brute force yang berlangsung sampai berbulan-bulan atau lebih, sehingga pihak yang melancarkan brute force bisa mencoba sampai jutaan kombinasi password untuk mendapatkan password yang benar untuk masuk ke sistem tersebut.

Artikel ini membahas tentang simulasi sederhana dari brute force, dan konfigurasi firewall (Mikrotik) untuk pencegahannya.

>- Simulasi dilakukan menggunakan GNS3
>- Brute force dieksekusi dari VM Kali Linux "Penjahat" menggunakan tool Hydra
>- Target brute force adalah VM Ubuntu "Korban"

## Topologi

[![Gambar 1. Topologi](http-topologi.png#center "Gambar 1. Topologi")](http-topologi.png)

>- Untuk mempersingkat durasi brute force, password pada VM "Korban" hanya dibuat 3 digit angka. Kemudian pada VM "Penjahat", tool Hydra juga di-set untuk melakukan brute force menggunakan numeric 0-9, dan dibatasi kombinasi 3 digit saja
>- Sekali lagi, pada kasus nyata, brute force dapat berlangsung sangat lama tanpa disadari oleh target. Terutama target yang tidak ada pengamanan sama sekali

## Hydra

[![Gambar 2. Penggunaan Hydra](http-hydra-1.png#center "Gambar 2. Penggunaan Hydra")](http-hydra-1.png)

Command di atas, `hydra -l root -x 3:3:1 ssh://192.168.1.250 -V` akan mengeksekusi brute force ke IP 192.168.1.250 (VM "Korban").

>- `hydra` = tool yang digunakan
>- `-l root` = user yang di-brute force adalah `root`
>- `-x 3:3:1` = menentukan jenis kombinasi password yang akan dicoba. Formatnya adalah MIN-CHAR:MAX-CHAR:CHARSET(1=numeric).

[![Gambar 3. Eksekusi Hydra](http-hydra-2.png#center "Gambar 3. Eksekusi Hydra")](http-hydra-2.png)

Setelah command sebelumnya dieksekusi, hydra kemudian menjalankan proses brute force dan berhasil menemukan password yang bisa digunakan untuk masuk ke target, yaitu 918.

[![Gambar 4. Akses SSH dengan Password yang Ditemukan](http-akses-ssh.png#center "Gambar 4. Akses SSH dengan Password yang Ditemukan")](http-akses-ssh.png)

Akses SSH ke VM "Korban" menggunakan password yang ditemukan hydra berhasil. Bagaimana penampakan log login attempt pada VM ini?

[![Gambar 5. /var/log/auth.log](http-login-attempt-log.png#center "Gambar 5. /var/log/auth.log")](http-login-attempt-log.png)

[![Gambar 6. Total Percobaan Login](http-total-percobaan.png#center "Gambar 6. Total Percobaan Login")](http-total-percobaan.png)

Tercatat 1000an kali percobaan ketika hydra melancarkan brute force ke target ini.

## Firewall (Mikrotik)

Bagaimana proses TCP/IP yang terjadi ketika seseorang (client) mengakses SSH ke suatu sistem (server)? Pertama, client mengirim packet SYN ke port SSH server. Server kemudian membalas dengan SYN-ACK. Kemudian client membalas dengan ACK. Dengan begitu TCP connection menjadi established dan sesi SSH berlangsung di dalam koneksi TCP ini. Setelah itu client memasukkan password. Apabila client menginput password yang salah terlalu banyak, maka server akan men-terminate sesi SSH dan koneksi TCP dengan mengirim RST.

[![Gambar 7. TCP Handshake](http-tcp-handshake.png#center "Gambar 7. TCP Handshake")](http-tcp-handshake.png)

Sekarang bagaimana proses TCP/IP ketika hydra melancarkan brute force ke suatu target? Hydra membuat beberapa koneksi TCP ke port SSH target, sehingga ada banyak sesi SSH di saat yang sama, sehingga hydra bisa mencoba lebih dari 1 password sekaligus. Untuk setiap koneksi TCP di mana hydra mendeteksi password yang salah berdasarkan respon SSH/TCP dari target, hydra akan langsung menutup koneksi tersebut dan membuat koneksi baru untuk terus mencoba kombinasi password lainnya.

[![Gambar 8. Connection Reset](http-conn-reset.png#center "Gambar 8. Connection Reset")](http-conn-reset.png)

Berdasarkan 2 poin di atas, kita bisa membuat firewall yang akan mem-filter pola trafik brute force tersebut.

```bash
/ip firewall filter 

add action=accept chain=forward connection-state=established,related 

add action=add-src-to-address-list address-list=stage3_list address-list-timeout=6h chain=forward connection-state=new dst-port=22 protocol=tcp src-address-list=stage2_list
comment="IP yang ada dalam stage2_list yang kembali membuat koneksi baru TCP ke port 22 dalam jangka waktu 30 menit akan masuk ke list ini dan akan dihapus setelah 6 jam"

add action=add-src-to-address-list address-list=stage2_list address-list-timeout=30m chain=forward connection-state=new dst-port=22 protocol=tcp src-address-list=stage1_list
comment="IP yang ada dalam stage1_list yang kembali membuat koneksi baru TCP ke port 22 dalam jangka waktu 5 menit akan masuk ke list ini dan akan dihapus setelah 30 menit"

add action=add-src-to-address-list address-list=stage1_list address-list-timeout=5m chain=forward connection-state=new dst-port=22 protocol=tcp
comment="IP yang membuat koneksi baru TCP ke port 22 akan masuk ke list ini dan akan dihapus setelah 5 menit"

add action=drop chain=forward comment="Block IP yang ada dalam stage3_list" src-address-list=stage3_list 
```