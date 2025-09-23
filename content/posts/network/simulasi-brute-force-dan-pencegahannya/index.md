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

Pada kesempatan ini kita akan membuat simulasi sederhana tentang cara kerja brute force. Setelah itu kita akan merancang firewall menggunakan Mikrotik untuk melindungi sistem dari metode serangan ini.

>- Simulasi dilakukan menggunakan GNS3
>- Brute force akan dieksekusi dari VM Kali Linux "Penjahat" menggunakan tool Hydra
>- Target brute force adalah VM Ubuntu "Korban"

## Topologi

[![Gambar 1. Topologi](http-topologi.png#center "Gambar 1. Topologi")](http-topologi.png)