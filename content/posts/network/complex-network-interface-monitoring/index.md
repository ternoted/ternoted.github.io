---
date: '2026-08-21T10:40:05+07:00'
draft: false
title: 'Complex Network Interface Monitoring (Grafana & Prometheus)'
summary: Interface monitoring menggunakan SNMP yang mencakup statistik interface (bandwidth, packets, interface status) dan SFP diagnostic (Rx/Tx Power, Voltage, dll)
author: ["Ilham Wahayu Yanre"]
cover:
  image: cover.png
  hiddenInList: true
aliases: ["/network"]
tags: ["Grafana","Prometheus","SNMP"]
series: ["Network"]
ShowBreadCrumbs: true
ShowToc: true
tocopen: true
ShowCodeCopyButtons: true
---

## Introduction

>- Sistem monitoring pada artikel ini dibangun untuk perangkat switch berbasis Comware 7 dan Mikrotik CRS
>- Artikel ini dapat diaplikasikan untuk semua vendor selama diketahui MIB yang bertanggung jawab untuk meng-expose data SNMP interface (IF-MIB) dan SFP (HH3C-TRANSCEIVER-INFO-MIB untuk Comware 7 dan mikrotik-mib untuk Mikrotik ROS), sebagai contoh
>- Semua data yang ditampilkan pada panel monitoring adalah dari SNMP yang ditarik dari perangkat switch menggunakan Prometheus SNMP Exporter
>- Software yang digunakan adalah Grafana Enterprise untuk visualisasi, Prometheus sebagai database (TSDB), dan SNMP Exporter untuk menerjemahkan data SNMP ke HTTP yang dapat diproses oleh Prometheus
>- Semua software dijalankan menggunakan Docker Compose pada WSL2 Ubuntu 26.04 pada Windows 11

## Docker
Instalasi Grafana dan Prometheus menggunakan Docker dapat merujuk pada masing-masing dokumentasi software: https://hub.docker.com/r/grafana/grafana dan https://hub.docker.com/r/prom/prometheus. Sedangkan SNMP Exporter pada artikel ini di-install dengan membuat Docker image secara langsung dari GitHub repo-nya. Dikarenakan Generator snmp.yml yang tersedia pada Github repo berbeda versi dan tidak kompatibel dengan installer versi release pada saat artikel ini dibuat. Generator tersebut diperlukan untuk membuat file snmp.yml custom yang dibuat khusus untuk Comware 7 dan Mikrotik ROS 7. File snmp.yml digunakan oleh SNMP Exporter untuk menarik data SNMP dari perangkat.

### SNMP Exporter Docker Image
Clone repo https://github.com/prometheus/snmp_exporter

```bash
git clone https://github.com/prometheus/snmp_exporter.git
```

Install tools yang diperlukan untuk membuat Docker image

```bash
sudo apt-get install unzip build-essential libsnmp-dev curl golang-go git # Debian-based distros
```

Buat binary snmp_exporter

```bash
make build
```

Buat directory ``` .build/linux-amd64 ``` dan pindahkan file binary snmp_exporter sebelumnya ke dir tersebut

```bash
mkdir -p .build/linux-amd64
cp snmp_exporter .build/linux-amd64/snmp_exporter
```

Buat file Docker image

```bash
docker build -t snmp-exporter:local .
```

Setelah selesai, pastikan snmp-exporter:local sudah tersedia sebagai docker image

```bash
docker images | grep snmp-exporter
```

### Compose
Buat directory untuk file config Docker compose, Prometheus, dan SNMP Exporter

```bash
mkdir -p docker/mon-stack #compose
mkdir -p docker/prometheus
mkdir -p docker/snmp-exporter
```
Buat file ```compose.yml```
```bash
nano docker/mon-stack/compose.yml
```

>- [File compose.yml](https://github.com/ternoted/sfp-interface-monitoring/blob/main/docker/mon-stack/compose.yml)

File compose.yml di atas akan menginstall dan menjalankan image SNMP Exporter, Prometheus, dan Grafana pada Docker

Jalankan ```compose.yml```

```bash
docker compose up -d
```
Jika command dijalankan di luar directory file compose.yml, gunakan

```bash
docker compose -f ~/docker/mon-stack/compose.yml up -d
```

Pastikan ketiga container sudah berjalan

```bash
docker ps

CONTAINER ID   IMAGE                               COMMAND                  CREATED      STATUS          PORTS                                         NAMES
c380cec30bff   grafana/grafana-enterprise:latest   "/run.sh"                5 days ago   Up 27 minutes   0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp   mon-stack-grafana-enterprise-1
75e6209c8b65   prom/prometheus:latest              "/bin/prometheus --c…"   5 days ago   Up 27 minutes   0.0.0.0:9090->9090/tcp, [::]:9090->9090/tcp   mon-stack-prometheus-1
6ce5f7c8f341   snmp-exporter:local                 "/bin/snmp_exporter …"   5 days ago   Up 27 minutes   0.0.0.0:9116->9116/tcp, [::]:9116->9116/tcp   mon-stack-snmp-exporter-1
```

## SNMP Exporter
Proteksi `basic_auth` dapat digunakan untuk mengamankan akses SNMP Exporter. Credential user dan password bcrypt tersedia dalam bentuk file config `web.yml`.

Password dalam bentuk bcrypt bisa dibuat menggunakan `apache2-utils` yang dapat di-install pada host. Kemudian jalankan command `htpasswd -nBC 10 "nama_user"`. Prompt password akan muncul, lalu nama_user:hash_password akan tampil setelah selesai.

`nama_user: hash_password` di-paste pada file config `web.yml` kemudian diaktifkan dengan menambahkan baris command `- "--web.config.file=/etc/snmp_exporter/web.yml"` pada `compose.yml`.

Selanjutnya adalah meng-generate file snmp.yml menggunakan Generator. Masuk ke folder /generator pada snmp_exporter yang di-clone sebelumnya dan jalankan ```make generate``` untuk membuat file binary generator

```bash
cd snmp_exporter.git/generator/
make generate
```
Di bawah ini adalah generator.yml yang digunakan untuk menarik data SNMP dari perangkat Comware 7 dan Mikrotik ROS

>- [File generator.yml](https://github.com/ternoted/sfp-interface-monitoring/blob/main/snmp_exporter/generator/generator_univ.yml)

Generator dapat dijalankan dengan command

```bash
./generator generate -g generator.yml -m mibs_univ/ -o /tmp/snmp.yml
```
| Opsi | Deskripsi |
|-|-|
|`-g generator.yml` | file generator.yml yang digunakan|
|`-m mibs_univ/` |  folder yang berisi file-file MIB yang dibutuhkan untuk meng-generate snmp.yml menggunakan generator.yml di atas|
|`-o /tmp/snmp.yml` | output file snmp.yml|

File-file MIB yang dibutuhkan adalah:
```bash
~/snmp_exporter.git/generator$ ls mibs_univ/
HH3C-OID-MIB.mib               IF-MIB                                SNMPv2-MIB  mikrotik.mib
HH3C-TRANSCEIVER-INFO-MIB.mib  INET-ADDRESS-MIB.mib                  SNMPv2-SMI  mikrotik.mib:Zone.Identifier
IANAifType-MIB                 INET-ADDRESS-MIB.mib:Zone.Identifier  SNMPv2-TC
```
Di bawah ini adalah output file `snmp.yml`.

>- [File snmp.yml](https://github.com/ternoted/sfp-interface-monitoring/blob/main/docker/snmp_exporter/snmp.yml)

Selanjutnya file snmp.yml yang baru dapat dipindah ke directory config SNMP Exporter

```bash
cp /tmp/snmp.yml ~/docker/snmp_exporter/snmp.yml
```

Restart container snmp-exporter
```bash
docker compose -f ~/docker/mon-stack/compose.yml restart snmp-exporter
```

UI SNMP Exporter dapat diakses via browser melalui `localhost:9116`. Pengetesan dapat dilakukan dengan memasukkan IP target perangkat dan nama module pada snmp.yml.

## Prometheus

`basic_auth` juga dapat digunakan untuk Prometheus dengan cara yang sama. Prometheus menggunakan file config `prometheus.yml` yang berisi parameter scraping dan target perangkat jaringan.

>- [File prometheus.yml](https://github.com/ternoted/sfp-interface-monitoring/blob/main/docker/prometheus/prometheus.yml)

`metric_relabel_configs` digunakan untuk mengganti nama OID `mtxrOpticalName` dari `mikrotik-mib` menjadi `ifName` agar sama dengan yang dari `IF-MIB`. Ini karena IF-MIB dan mikrotik-mib menggunakan indexing yang berbeda, sedangkan HH3C-TRANSCEIVER-INFO-MIB menggunakan indexing yang sama dengan IF-MIB. Ini diperlukan untuk Outer Join pada panel table Grafana karena semua query akan digabungkan dengan ifName sebagai identifier (primary key).

UI Prometheus dapat diakses melalui `localhost:9090`

## Grafana
UI Grafana dapat langsung diakses melalui `localhost:3000`. Data source ditambahkan dengan memasukkan URL `http://prometheus:9090`. Container Prometheus reachable dari Container Grafana melalui internal DNS Docker.

>- [File Grafana Dashboard JSON](https://github.com/ternoted/sfp-interface-monitoring/blob/main/docker/grafana-dashboard.json)

File .json di atas dapat langsung di-paste pada Grafana Dashboard Setting untuk membuat variable dashboard dan panel table secara instant, dengan catatan perangkat yang menjadi target adalah Comware 7 atau Mikrotik ROS. Penampakan variable dan panel adalah seperti berikut:

[![](http-3.png#center "Dashboard Variable")](http-3.png)

[![](http-1.png#center "Perangkat HPE yang menggunakan OS Comware 7")](http-1.png)
[![](http-2.png#center "Perangkat Mikrotik yang menggunakan ROS 7")](http-2.png)