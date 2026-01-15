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

  Isi `alertmanager.yml` dengan contoh berikut:
  ```bash
global:
  resolve_timeout: 1m

route:
  receiver: 'telegram'
  group_by: ['alertname']
  group_wait: 5s
  group_interval: 1m
  repeat_interval: 6h

receivers:
  - name: 'telegram'
    telegram_configs:
      - bot_token: 'xxxxxx'
        chat_id: xxxxxx
        parse_mode: 'HTML'
        message: '{{ template "telegram.default.message" . }}'

  ```

  ### Alert Rules

  Sebagai contoh kita akan membuat alert rule untuk penggunaan CPU di atas 80%.
  Pada contoh ini kita membuat file rule di `/etc/prometheus/rules/cpu_alerts.yml` dengan isi:

  ```bash
groups:
  - name: cpu_alerts
    rules:
      - alert: High CPU Usage
        expr: avg by (router,instance) (hrProcessorLoad{job="router_mikrotik"}) > 80
        for: 15s
        labels:
          severity: warning
        annotations:
          sensor_value: "CPU {{ printf \"%.2f\" $value }}%"
          grafana: "link_ke_grafana_panel"
  ```

  Selanjutnya pada file `prometheus.yml`, di bawah garis
  ```bash
# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
           - localhost:9093
  ```

  Tambahkan baris berikut
  ```bash
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
   - "/etc/prometheus/rules/cpu_alerts.yml"
  ```

  Selanjutnya reload Prometheus dan Alertmanager. Untuk pengetesan alert, kita bisa merubah threshold value >80 menjadi lebih kecil dari penggunaan CPU perangkat yang sedang dimonitor. Sehingga alert rule akan ter-trigger sehingga alert akan dikirim ke receiver. Karena kita tidak menggunakan style/format untuk receiver Telegram pada alertmanager.yml, maka alert yang akan kita terima kurang lebih seperti ini:

  [![](http-alert-example.png#center "Triggered Alert")](http-alert-example.png)

  Format pesan alert bisa didesain agar hasilnya terlihat lebih rapi menggunakan Markdown atau HTML, namun tidak di-cover dalam artikel ini karena penulis ngantuk.

  ## Penutup
  Menggunakan kombinasi Prometheus + Alertmanager + Grafana, kita dapat membuat Network Monitoring & Alerting System tanpa mengeluarkan biaya untuk software karena semuanya gratis. Hanya saja dibutuhkan pengetahuan teknis yang cukup kompleks, terutama terkait SNMP. 