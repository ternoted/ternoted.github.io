---
date: '2025-08-05T20:56:21+07:00'
draft: false
title: 'Cara Kerja Traceroute'
summary: Membahas cara kerja traceroute dan beberapa miskonsepsi dalam menganalisa hasil traceroute
author: ["Ilham Wahayu Yanre"]
cover:
  image: cover.png
  hiddenInList: true
aliases: ["/network"]
series: ["Network"]
ShowBreadCrumbs: true
ShowToc: true
tocopen: true
---

Internet itu seperti jalan raya yang menghubungkan satu tempat ke tempat lainnya. Kita bisa menggunakan Google Map untuk memantau jalur ke suatu tempat, dan menganalisa di titik-titik mana saja terjadi kemacetan dan lain sebagainya. Sedangkan untuk menganalisa jalur internet, maka kita menggunakan traceroute. Traceroute digunakan untuk memetakan setiap Router yang dilewati packet untuk sampai ke perangkat tujuan dengan memanfaatkan TTL (Time to Live). Sehingga kita bisa melakukan troubleshooting dengan menganalisa hasil pemetaan tersebut.

## Apa Itu TTL (Time to Live)?

Di dalam IP packet terdapat field TTL yang fungsinya adalah mencegah packet dari berkelana tanpa henti dari Router ke Router tanpa pernah mencapai tujuannya, atau yang disebut Looping. Nilai minimum TTL adalah 1, dan maksimum 255. Jika packet memiliki TTL 32, maka packet tersebut bisa melompat dari Router ke Router sebanyak 32 kali untuk mencapai tujuannya. Setiap Router yang dilewati akan mengurangi 1 nilai pada TTL. Jika setelah 32 kali packet tersebut tidak sampai ke tujuan alias TTL mencapai 0, maka packet tersebut akan di drop. Lalu Router yang men-drop packet tersebut akan mengirimkan pesan ICMP "TTL Exceeded" ke pengirim.

## Cara Kerja Traceroute

Traceroute memetakan rute jaringan dengan cara mengirim packet yang dimulai dengan TTL 1, dan packet selanjutnya TTL n+1. Ketika packet diterima oleh Router pertama, TTL berkurang 1 sehingga menjadi 0. Router mengirim ICMP TTL Exceeded ke Pengirim yang memuat informasi IP Router dan jarak tempuh packet dari Pengirim-Router-Pengirim (RTT). Selanjutnya Traceroute kembali mengirim packet dengan TTL 2. Router pertama menerima packet tersebut dan mengurangi TTL 1. Router kedua menerima packet tersebut dan juga mengurangi TTL 1 sehingga menjadi 0. Router kedua mengirim ICMP TTL Exceeded ke Pengirim. Begitu seterusnya sampai packet tiba di tujuan. Dengan cara ini traceroute bisa memetakan setiap Router yang dilewati packet hingga sampai di tujuan.

![Alt text](cara-kerja.png "Cara Kerja Traceroute")