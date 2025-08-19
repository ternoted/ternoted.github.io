---
date: '2025-08-19T15:48:46+07:00'
draft: true
title: 'Tutorial Grafana'
summary: Tutorial instalasi Grafana dan menghubungkannya dengan Prometheus
author: ["Ilham Wahayu Yanre"]
cover:
  image: cover.png
  hiddenInList: true
aliases: ["/network"]
series: ["Grafana"]
ShowBreadCrumbs: true
ShowToc: true
tocopen: true
---

Artikel ini adalah lanjutan dari artikel sebelumnya, [Tutorial Prometheus](https://ternoted.github.io/posts/network/prometheus/). Artikel ini membahas tentang tutorial instalasi Grafana, cara menghubungkannya dengan Prometheus, dan cara membuat visual panel.

## Pre-requisites
>- **Prometheus** dan **SNMP Exporter** sudah terinstall dan berfungsi dengan benar

## Instalasi Grafana
```bash
# Install packages yang dibutuhkan
grafana@grafana-wahayu:~$ sudo apt-get install -y apt-transport-https software-properties-common wget

# Import GPG key
grafana@grafana-wahayu:~$ sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

# Tambahkan repository untuk versi stable
grafana@grafana-wahayu:~$ echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

# Install Grafana OSS
grafana@grafana-wahayu:~$ sudo apt-get update
grafana@grafana-wahayu:~$ sudo apt-get install grafana

# Jalankan Grafana
grafana@grafana-wahayu:~$ sudo systemctl daemon-reload
grafana@grafana-wahayu:~$ sudo systemctl enable grafana-server.service
grafana@grafana-wahayu:~$ sudo systemctl start grafana-server
```