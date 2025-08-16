---
date: '2025-08-15T23:05:48+07:00'
draft: true
title: 'SNMP Exporter'
summary: Tutorial install SNMP-Exporter dan cara penggunaannya untuk Prometheus
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

Artikel ini membahas tentang cara menginstall [SNMP Exporter](https://github.com/prometheus/snmp_exporter). Software ini digunakan untuk menarik data SNMP dan menerjemahkannya ke format yang dapat dibaca oleh Prometheus. Karena by-default Prometheus tidak dapat membaca data SNMP secara langsung.

Untuk tutorial ini dan tutorial lanjutan tentang Prometheus dan Grafana, server yang digunakan adalah Ubuntu 24.04.3 LTS.

>- Pastikan waktu pada server sudah akurat dan ```System clock synchronized: yes```. Cek menggunakan ```timedatectl```.
>- Tutorial ini dibuat dengan asumsi SNMP pada Router atau Switch bisa diakses dari server. Cek menggunakan ```snmpwalk```.

```bash
grafana@grafana-wahayu:~$ cd /tmp
grafana@grafana-wahayu:/tmp$ wget https://github.com/prometheus/snmp_exporter/releases/download/v0.29.0/snmp_exporter-0.29.0.linux-amd64.tar.gz
```