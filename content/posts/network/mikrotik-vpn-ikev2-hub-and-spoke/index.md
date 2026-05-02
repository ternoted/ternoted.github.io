---
date: '2026-05-03T00:49:51+07:00'
draft: true
title: 'Mikrotik VPN IPSec/IKEv2 (Hub-and-spoke)'
summary: Membahas konfigurasi VPN IKEv2 Hub-and-spoke untuk router dengan IP static dan dynamic
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

Hub-and-spoke adalah VPN yang topologi sederhananya terdiri dari 1 router hub (pusat) dan beberapa router spoke (cabang). Router hub berperan untuk menghubungkan LAN antar router spoke.

Konfigurasi pada artikel ini bisa diterapkan pada kondisi di mana router yang berperan menjadi hub memiliki IP static. Lalu router-router spoke bisa memiliki IP dynamic atau berada di belakang NAT.

Topologi hub-and-spoke pada artikel ini adalah pengembangan dari topologi site-to-site pada artikel [Mikrotik VPN IPSec/IKEv2 (Site-to-site)](https://ternoted.github.io/posts/network/mikrotik-vpn-ikev2-site-to-site/). Jadi yang akan dilakukan adalah menambahkan 2 router spoke pada topologi site-to-site tersebut.

## GNS3 Lab

[![](http-hub-and-spoke.png#center "Hub-and-spoke Topologi")](http-hub-and-spoke.png)