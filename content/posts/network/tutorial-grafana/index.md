---
date: '2025-08-19T15:48:46+07:00'
draft: false
title: 'Tutorial Grafana'
summary: Tutorial instalasi Grafana dan cara menghubungkannya dengan Prometheus
author: ["Ilham Wahayu Yanre"]
cover:
  image: cover.png
  hiddenInList: true
aliases: ["/network"]
tags: ["Grafana","Prometheus","SNMP"]
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
grafana@grafana-ternoted:~$ sudo apt-get install -y apt-transport-https software-properties-common wget

# Import GPG key
grafana@grafana-ternoted:~$ sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

# Tambahkan repository untuk versi stable
grafana@grafana-ternoted:~$ echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

# Install Grafana OSS
grafana@grafana-ternoted:~$ sudo apt-get update
grafana@grafana-ternoted:~$ sudo apt-get install grafana

# Jalankan Grafana
grafana@grafana-ternoted:~$ sudo systemctl daemon-reload
grafana@grafana-ternoted:~$ sudo systemctl enable grafana-server.service
grafana@grafana-ternoted:~$ sudo systemctl start grafana-server
```

### Membuat Panel Visual

Sekarang buka Grafana melalui browser dengan mengakses `IP_server:3000`.
[![Grafana UI](http-grafana-ui.png#center "Grafana UI")](http-grafana-ui.png)

Default user & pass-nya adalah admin:admin. Setelah login silakan ganti password-nya terlebih dahulu dengan yang lebih aman.
Setelah login, klik **DATA SOURCES** kemudian pilih Prometheus.
[![Grafana Data Sources](http-data-source.png#center "Grafana Data Sources")](http-data-source.png)

Karena Grafana dan Prometheus kita install di server yang sama, maka URL-nya bisa kita isi dengan `http://localhost:9090`
[![Prometheus Connection](http-prometheus-connection.png#center "Prometheus Connection")](http-prometheus-connection.png)

Selanjutnya scroll ke bagian paling bawah, dan klik **Save & test**. Jika berhasil, maka hasilnya akan seperti gambar di bawah.
[![Save & test](http-save-test.png#center "Save & test")](http-save-test.png)

Sekarang Grafana dan Prometheus sudah terhubung. Selanjutnya kembali ke **Home** dan klik **Dashboards**.
Klik **Add visualization** dan pilih Prometheus sebagai data source-nya.
[![New panel](http-new-panel.png#center "New Panel")](http-new-panel.png)

Di sinilah kita akan membuat panel visual kita. Sebagai permulaan, kita akan membuat tampilan penggunaan CPU dengan menggunakan query "hrProcessorLoad".
[![Penggunaan CPU](http-cpu-usage.png#center "Penggunaan CPU")](http-cpu-usage.png)

Kita bisa mengubah tampilan panel ini dengan yang kita inginkan melalui panel **Visualization** pada bagian kanan.
Misalnya merubah visualization dari **Time Series** (gambar atas) ke mode **Gauge** (gambar bawah).
[![Mode Gauge](http-gauge.png#center "Mode Gauge")](http-gauge.png)

### Variables

Pada contoh di atas, kita menggunakan query `avg(hrProcessorLoad{instance="192.168.100.13"})` untuk menampilkan penggunaan CPU pada Router dengan IP tersebut.
Bagaimana jika kita memiliki banyak Router yang ingin kita monitor? Kita bisa membuat variable sehingga kita bisa memilih Router mana yang ingin kita lihat melalui menu dropdown. Sehingga kita tidak perlu membuat panel satu persatu secara manual.

Caranya klik **Back to dashboard** pada sudut kanan atas, lalu pilih **Settings**, kemudian pilih tab **Variables**, dan pilih **New variable**.

Sebagai contoh, variable akan kita buat seperti di bawah ini:
[![Variable](http-variable.png#center "Variable")](http-variable.png)

Selanjutnya kita kembali ke Dashboard Penggunaan CPU yang sudah dibuat sebelumnya, dan update query menjadi `avg(hrProcessorLoad{instance="$router_name"})`. Artinya Grafana akan membuat panel visual secara dinamis untuk setiap IP Router yang ada pada variable `$router_name`.
[![Dropdown](http-dropdown.png#center "Dropdown")](http-dropdown.png)

Karena pada tutorial ini kita hanya menambahkan 1 Router pada konfig Prometheus, maka IP Router yang ditampilkan pada dropdown di atas hanya 1.

## Penutup

Menggunakan kombinasi Grafana dan Prometheus, kita bisa merancang monitoring system tanpa perlu mengeluarkan biaya untuk membeli license/subscription, karena keduanya bersifat open source dan gratis. Kita bahkan bisa membuatnya sedetail yang kita butuhkan. Di bawah ini adalah contoh panel-panel visual yang dikembangkan saat pembuatan artikel ini untuk monitoring hardware, trafik bandwidth dan packets pada Router.

[![Router Monitoring System Menggunakan Grafana](http-grana-visual-panels.png#center "Router Monitoring System Menggunakan Grafana")](http-grana-visual-panels.png)