---
date: '2026-03-23T14:50:35+07:00'
draft: true
title: 'GTA V FiveM - Reverse Proxy'
summary: Penggunaan Reverse Proxy untuk meningkatkan keamanan server dari serangan DDoS
author: ["Ilham Wahayu Yanre"]
cover:
  image: cover.png
  hiddenInList: true
aliases: ["/fivem"]
series: ["FiveM"]
ShowBreadCrumbs: true
ShowToc: true
tocopen: true
---

Reverse proxy pada FiveM server dapat digunakan untuk menyembunyikan IP server utama, sebagai gateway alternatif "Gerbang", atau sebagai cached proxy "CDN". Ketiga fungsi tersebut dapat diaplikasikan pada 1 proxy server sekaligus atau terpisah-pisah. Namun berdasarkan pengalaman penulis, cara yang terbaik adalah dengan mengaplikasikannya pada proxy server yang berbeda-beda. Jadi 1 proxy server untuk menggantikan IP utama server pada server list, 1 proxy server sebagai gateway alternatif, dan 1 proxy server sebagai cached proxy. Tujuan akhirnya adalah sebagai layer keamanann tambahan server dari serangan DDoS.

## Requirement
>- Server dengan Ubuntu OS 22.04 (Umumnya menggunakan VPS)
>- Reachability antara proxy server dengan FiveM server
>- NGINX

## Reverse Proxy untuk Menyembunyikan IP Utama Server
Dalam hal ini ada 2 jenis proxy, yaitu connect proxy (HTTPS) dan server proxy (TCP/UDP). Connect proxy akan menggantikan IP server utama yang muncul pada server list, dan server proxy akan menjadi perantara antara player dan server ketika di dalam game. Jadi IP utama server pada case ini disembunyikan di balik proxy.
