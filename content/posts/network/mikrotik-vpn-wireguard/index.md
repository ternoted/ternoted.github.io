---
date: '2026-07-06T09:50:36+07:00'
draft: false
title: 'Mikrotik VPN WireGuard dengan Split Tunnel'
summary: Membahas konfigurasi WireGuard untuk membuat VPN Split Tunnel
author: ["Ilham Wahayu Yanre"]
cover:
  image: cover.png
  hiddenInList: true
aliases: ["/network"]
tags: ["VPN","Mikrotik"]
series: ["Network"]
ShowBreadCrumbs: true
ShowToc: true
tocopen: true
ShowCodeCopyButtons: true
---


## Introduction

Split Tunneling pada VPN adalah mode routing di mana hanya trafik-trafik dengan tujuan tertentu saja yang dilewatkan melalui interface VPN. Berbeda dengan VPN Full Tunnel di mana semua trafik internet akan melewati interface VPN.

Beberapa kondisi yang membutuhkan VPN Split Tunnel adalah jika resource & bandwidth pada VPN server terbatas, atau bandwidth ISP dari VPN client lebih bagus dari VPN server. Split Tunnel tidak digunakan untuk pengguna yang ingin menjaga anonymity-nya di internet.


## WireGuard

Adalah protokol VPN yang paling simpel dan mudah untuk digunakan. Konfigurasi pada client bisa dilakukan cukup dengan copy-paste konfig, atau dengan scan QR melalui smartphone. Namun alasan utama kenapa WireGuard digunakan untuk use case ini adalah karena kemudahannya dalam mengkonfigurasi split tunnel itu sendiri.

## Contoh Kasus

Budi adalah seorang karyawan IT _no-life_ yang sangat workaholic dan ingin selalu terhubung ke server -server production yang ada di kantornya 24/7. Masalahnya server-server tersebut hanya dapat diakses dari jaringan LAN kantor (192.168.200.0/24). Budi membutuhkan VPN yang dapat menghubungkannya ke LAN kantor. Akan tetapi Budi tidak ingin semua trafik internet-nya melewati VPN server, karena dikhawatirkan hobi streaming video edukasi Budi dapat membebani bandwidth jaringan internet kantor itu sendiri. Oleh karena itu Budi yang budiman akan mengkonfigurasi WireGuard dengan split tunneling pada router Mikrotik kantor yang menggunakan IP public static, agar dapat terhubung ke server-server tersebut dari jaringan luar kantor selamanya.

## Konfigurasi

Konfigurasi WireGuard hampir seluruhnya dilakukan di sisi VPN server/Mikrotik. Sedangkan PC/smartphone sebagai VPN client hanya perlu menginstall aplikasinya lalu salin/scan QR konfig dari VPN server.

### WireGuard Interface

[![](http-1.png#center "Private Key dan Public Key akan di-generate otomatis setelah menekan Apply")](http-1.png)

### IP Address

[![](http-2.png#center "Beri IP Address pada interface WireGuard. IP ini digunakan untuk point-to-point dengan client VPN")](http-2.png)

### WireGuard Peers

[![](http-3.png#center "")](http-3.png)

| No. | Deskripsi |
|:-:|:-|
|1|Membuka tab WireGuard Peers|
|2|Nama peer. 1 peer untuk 1 perangkat VPN client|
|3|Private key pilih "auto"|
|4|IP yang akan digunakan oleh WireGuard pada perangkat VPN client|

Nomor 5 dan selanjutnya adalah untuk meng-generate konfigurasi yang akan digunakan pada perangkat VPN client.

| No. | Deskripsi |
|:-:|:-|
|5|IP untuk perangkat VPN client|
|6,7|DNS yang akan digunakan client. Jika kosong, maka client tidak memiliki DNS dan tidak bisa resolve domain/internetan|
|8|IP public / WAN pada router VPN server|
|9|Split Tunnel. Artinya untuk trafik yang tujuannya ada di 192.168.200.0/24, maka akan melewati VPN WireGuard. Lebih dari 1 subnet bisa dimasukkan di sini dengan menekan panah bawah pada bagian kanan. Jika subnet yang dimasukkan adalah 0.0.0.0/0, maka VPN ini menjadi Full Tunnel|
|10|Setelah mengklik "Apply", Public Key, Private Key, dan Config untuk perangkat VPN client akan di-generate secara otomatis|

[![](http-4.png#center "Setelah mengklik Apply")](http-4.png)

Jika jendela di atas terus di-scroll sampai ke bawah, maka akan muncul QR Code yang bisa di-scan melalui aplikasi WireGuard pada smartphone untuk menginstall konfigurasi peering dengan VPN server.

[![](http-5.png#center "Konfigurasi dalam bentuk QR Code")](http-5.png)

### Instalasi dan Konfigurasi WireGuard pada PC

Selanjutnya download [WireGuard](https://www.wireguard.com/install/) dan install.

[![](http-6.png#center "Di sini konfigurasi bisa di-import atau di-paste secara manual melalui Add empty tunnel...")](http-6.png)


[![](http-7.png#center "Paste Client Config yang ada pada jendela WireGuard Peer, lalu save")](http-7.png)

[![](http-8.png#center "Klik Activate")](http-8.png)

Selanjutnya buka Windows Terminal atau CMD dan jalankan command `route print 192.168.200.*`. Jika subnet tersebut muncul dengan Interface=IP ptp WireGuard (di sini menggunakan 10.0.0.2), maka WireGuard aktif dan normal.

```bash
IPv4 Route Table
===========================================================================
Active Routes:
Network Destination        Netmask          Gateway       Interface  Metric
    192.168.200.0    255.255.255.0         On-link          10.0.0.2      5
  192.168.200.255  255.255.255.255         On-link          10.0.0.2    261
===========================================================================
Persistent Routes:
  None
```

Selanjutnya tes ping,

```bash
PS C:\Users\xxXDestroyerBudiXxx> ping 192.168.200.10 -t

Pinging 192.168.200.10 with 32 bytes of data:
Reply from 192.168.200.10: bytes=32 time=4ms TTL=64
Reply from 192.168.200.10: bytes=32 time=1ms TTL=64
Reply from 192.168.200.10: bytes=32 time=1ms TTL=64
Reply from 192.168.200.10: bytes=32 time=2ms TTL=64
Reply from 192.168.200.10: bytes=32 time=1ms TTL=64

Ping statistics for 192.168.200.10:
    Packets: Sent = 5, Received = 5, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 1ms, Maximum = 4ms, Average = 1ms
Control-C
PS C:\Users\xxXDestroyerBudiXxx>
```

WireGuard dengan Split Tunnel sudah jalan pada PC Budi.

### Konfigurasi WireGuard pada Smartphone

Tidak cukup sampai di situ, Budi ingin bisa mengakses server-server kesayangannya dari smartphone. Agar ketika ia terjaga pukul 03.00 AM, dia bisa langsung mengecek server-nya melalui HP tanpa perlu beranjak dari tempat tidur. Ia pun menginstall WireGuard melalui Playstore/Appstore.


[![](http-9.png#center "Buat Peer baru. PENTING! Jangan meng-copy Peer yang sudah ada untuk membuat Peer baru. Karena hal ini menyebabkan Peer yang di-copy menjadi corrupt")](http-9.png)

[![](http-10.jpeg#center "Pilih Scan from QR Code. Lalu arahkan kamera ke QR Code pada jendela WireGuard Peer sebelumnya")](http-10.jpeg)

[![](http-11.jpeg#center "Selanjutnya beri nama, lalu aktifkan")](http-11.jpeg)

Tes ping,

[![](http-12.jpeg#center "Ping dari smartphone ke LAN/server Budi di kantor")](http-12.jpeg)

### Konfigurasi WireGuard Site-to-site

Pada contoh ini 1 router memiliki 1 IP public, dan 1 router berada di belakang NAT. Jadi koneksi WireGuard akan dimulai oleh router NAT, dan router public akan bersifat sebagai responder.

WireGuard interface dan peer pada router NAT akan dibuat berdasarkan Client Config yang di-generate oleh WireGuard router public berikut:

```config
[Interface]
ListenPort = 51820
PrivateKey = 2DuXXXXXXXXXXXXXXXXXXXXXXXXXXXXXm8XU=
Address = 10.0.0.4/24
DNS = 1.1.1.1, 8.8.8.8

[Peer]
PublicKey = AX+XXXXXXXXXXXXXXXXXXXXXXXXXXXXXsS8Ec=
AllowedIPs = 10.0.0.1/32, 192.168.200.0/24, 1.1.1.1/32, 8.8.8.8/32
Endpoint = 202.xx.xxx.xx:13257
PersistentKeepalive = 25
```

[![](http-13.png#center "[Interface] PrivateKey pada Client Config di-paste ke WireGuard interface Private Key")](http-13.png)

[![](http-14.png#center "[Peer] PublicKey pada Client Config di-paste ke Peer Public Key. [Peer] AllowedIPs dimasukkan menjadi Peer Allowed Address")](http-14.png)

Selanjutnya tambahkan IP Address 10.0.0.4/24 ke interface WireGuard. Tidak seperti WireGuard Client pada smartphone dan PC, IP dan route pada router harus ditambahkan secara manual

```/ip address add address=10.0.0.4/24 interface=WG network=10.0.0.0```

Tambahkan static route

```/ip route add dst-address=192.168.200.0/24 gateway=10.0.0.1```

Tambahkan Firewall NAT

```/ip firewall nat add action=src-nat chain=srcnat out-interface=WG src-address-list=LAN to-addresses=10.0.0.4```


Selesai.

Berhasil.

Hore.