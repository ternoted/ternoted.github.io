---
date: '2026-01-12T10:34:06+07:00'
draft: true
title: 'Prometheus Alertmanager - Instalasi dan Penggunaan'
summary: Tutorial instalasi Prometheus Alertmanager dan penggunaannya
author: ["Ilham Wahayu Yanre"]
cover:
  image: cover.png
  hiddenInList: true
aliases: ["/network"]
tags: ["Grafana","Prometheus","SNMP"]
series: ["Network Monitoring System"]
ShowBreadCrumbs: true
ShowToc: true
tocopen: true
---

Ini adalah artikel terakhir dari series Network Monitoring System yang membahas tentang [Prometheus Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) untuk system alerting/alarm pada sistem jaringan.

## Pre-requisites
>- **Prometheus** dan **SNMP Exporter** sudah terinstall dan berfungsi dengan benar

Prometheus tidak mengirimkan alert  ke receiver (eg. Telegram, webhook, etc) secara langsung. Akan tetapi Prometheus memiliki alert rules yang apabila ter-trigger, Prometheus akan mengirim alert tersebut ke Alertmanager. Barulah selanjutnya Alertmanager mengelola alert tersebut. Misalnya langsung dikirim ke receiver, ditunda pengirimannya, atau di-silence.

## Instalasi Alertmanager

Untuk proses instalasinya sangat mirip dengan cara menginstall Prometheus yang telah dibahas pada artikel sebelumnya. File dapat diunduh di [Prometheus Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/). Proses instalasinya dapat mengikuti [cara install Prometheus](https://ternoted.github.io/posts/network/prometheus/) pada artikel sebelumnya.

Untuk `systemd service` dapat menggunakan contoh ini
```bash
[Unit]
Description=Prometheus Alertmanager
After=network-online.target

[Service]
User=grafana
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/prometheus/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager
Restart=always

[Install]
WantedBy=multi-user.target
```

Jika berhasil, `sudo systemctl status alertmanager` seharusnya menunjukkan enabled dan active (running). Selanjutnya Web UI dapat diakses melalui `IP_server:9093`.

### Menghubungkan Prometheus dengan Alertmanager
Tambahkan baris di bawah pada file prometheus.yml, atau pastikan targets nya sama apabila barisnya sudah ada. Setelah itu reload Prometheus.
```bash
# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
           - localhost:9093
```

### Konfigurasi File `alertmanager.yml`

Sebagai contoh kita akan menggunakan Telegram sebagai receiver. Bot Telegram dapat dibuat menggunakan chat Telegram @BotFather. Jangan lupa menyimpan Bot token-nya. Setelah itu kirim chat ke Bot yang telah dibuat. Lalu ambil Chat ID melalui browser dengan mengakses url https://api.telegram.org/bot<bot_token>/getUpdates. Bot token adalah kode rahasia yang memberi akses Alertmanager untuk menggunakan Bot dalam mengirim chat, dan Chat ID adalah target yang akan dikirimi chat/notifikasi oleh Bot tersebut. Jadi jika Bot ingin digunakan di dalam group, maka Chat ID yang diambil adalah ID dari chat group tersebut.