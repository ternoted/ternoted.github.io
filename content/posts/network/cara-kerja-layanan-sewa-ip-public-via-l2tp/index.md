---
date: '2026-07-15T11:54:44+07:00'
draft: false
title: 'Cara Kerja Layanan Sewa IP Public via L2TP'
summary: Pernah menyewa IP public static yang delivery-nya melalui L2TP, yang bisa digunakan untuk remote access ke server atau router yang hanya memiliki IP private/dynamic? Yuk cari tahu bagaimana cara kerjanya!
author: ["Ilham Wahayu Yanre"]
cover:
  image: cover.png
  hiddenInList: true
aliases: ["/network"]
tags: ["L2TP","Mikrotik","IP Public"]
series: ["Network"]
ShowBreadCrumbs: true
ShowToc: true
tocopen: true
ShowCodeCopyButtons: true
---

## Introduction

Ada beberapa cara penyedia layanan dalam menyewakan IP public, salah satunya melalui L2TP. Langkah penggunaannya pun cukup mudah bagi end-user. User cukup memasukkan IP, username, dan password L2TP yang diberikan oleh penyedia layanan pada L2TP client seperti server, router, atau PC. Setelah koneksi L2TP berhasil, perangkat user akan mendapat IP public yang kemudian dapat digunakan untuk mengakses perangkat tersebut dari internet.

## Lab

Pada artikel ini kita akan mencoba mereplikasi layanan tersebut dengan menggunakan VPS Mikrotik CHR sebagai L2TP server, dan VM CHR di dalam GNS3 sebagai L2TP client. Lalu kita akan memberi IP public pada VM CHR di dalam GNS3 tersebut melalui VPS Mikrotik CHR, sehingga nantinya VM tersebut dapat kita akses dari internet.

Ada beberapa syarat yang harus terpenuhi untuk konfigurasi ini:
>- VPS sebagai L2TP server harus menggunakan minimal /29 IP public, /30 tidak dapat digunakan.
>- Penyedia VPS harus mengizinkan lebih dari 1 atau semua IP pada /29 untuk digunakan. Jadi jika penyedia VPS me-limit hanya 1 IP per VPS yang dapat digunakan meskipun prefix yang diberikan adalah /24, maka konfigurasi ini tidak dapat diterapkan.

### L2TP Server

#### Set ARP pada interface WAN menjadi proxy-arp

[![](http-8.png#center "")](http-8.png)

| No. | Deskripsi |
|:-:|:-|
|1|Ini sangat vital dan harus berfungsi sebelum konfigurasi selanjutnya bisa berjalan. ```proxy-arp``` memungkinkan VPS CHR ini untuk menjawab ARP request untuk IP xxx.30.246.xx/29 dari router di atasnya. Jadi jika IP yang akan diberikan untuk VM CHR di GNS3 adalah xxx.30.246.78, maka VPS CHR akan menjawab ARP request untuk IP tersebut dan mengarahkannya ke VM CHR|

#### Pasang IP Address untuk Local Address L2TP

[![](http-6.png#center "")](http-6.png)

#### Interface L2TP Server Binding

[![](http-1.png#center "")](http-1.png)

| No. | Deskripsi |
|:-:|:-|
|2|Pilih L2TP Server Binding|
|3|User yang akan dibuat pada PPP -> Secrets|

#### Enable L2TP Server

[![](http-2.png#center "")](http-2.png)

| No. | Deskripsi |
|:-:|:-|
|4|IPSec digunakan apabila perangkat L2TP client mewajibkan IPSec (ex. Windows Server/PC, Android)|

#### Membuat user untuk L2TP client

[![](http-3.png#center "")](http-3.png)

| No. | Deskripsi |
|:-:|:-|
|2|Samakan dengan user pada interface L2TP Server Binding|
|3|Buat password agak strong dikit, 1234 misalnya|
|5|IP public yang akan diberikan untuk L2TP client (VM CHR di dalam GNS3)|

#### Membuat firewall NAT

[![](http-9.png#center "")](http-9.png)

NAT ini diperlukan agar VM CHR di GNS3 dapat diakses dari internet. Ini karena VM GNS3 memiliki jalur WAN-nya tersendiri dengan IP public yang berbeda. Tanpa NAT, maka trafik dengan source IP 'X' dari internet yang destination-nya IP 'Y' VM CHR GNS3 (assigned public IP via L2TP), akan kembali ke IP 'X' dengan menggunakan source IP 'Z' (VM CHR GNS3 WAN IP). Ibaratnya seperti Budi yang memanggil Badu, tapi Bude yang menjawab. Sedangkan dengan NAT, maka IP source dari trafik internet ke arah IP L2TP akan diganti menjadi IP dari local address L2TP VPS CHR, dan IP tersebut directly connected pada routing table VM CHR GNS3.

### L2TP Client

#### Pastikan VM dapat mengakses internet

[![](http-4.png#center "")](http-4.png)

[![](http-5.png#center "")](http-5.png)

#### Konfigurasi koneksi L2TP Client

[![](http-7.png#center "Membuat koneksi L2TP ke server")](http-7.png)

#### Cek IP address list untuk melihat apakah IP address dari L2TP sudah terpasang

[![](http-10.png#center "")](http-10.png)

#### Akses winbox VM CHR GNS3 dari luar jaringan / internet

[![](http-11.png#center "Winbox VM CHR GNS3 bisa diakses dari luar jaringan")](http-11.png)

## Kesimpulan

Kunci dari konfigurasi di atas adalah ARP proxy dan fungsi protokol L2TP untuk "meng-assign" IP address secara dinamic pada L2TP client. Hasil seperti ini juga dapat dicapai dengan protokol lain dengan konsep yang sama seperti WireGuard.