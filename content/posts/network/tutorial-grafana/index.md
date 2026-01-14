---
date: '2025-08-19T15:48:46+07:00'
draft: false
title: 'Grafana - Instalasi dan Penggunaan'
summary: Tutorial instalasi Grafana dan cara menghubungkannya dengan Prometheus
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

Artikel ini adalah lanjutan dari series Network Monitoring System, dan artikel sebelumnya, [Tutorial Prometheus](https://ternoted.github.io/posts/network/prometheus/). Artikel ini membahas tentang tutorial instalasi Grafana, cara menghubungkannya dengan Prometheus, dan cara membuat visual panel.

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

`generator.yml` yang digunakan
```bash
auths:
  public_v2:
    version: 2
    community: public
modules:
  mikrotik:
    walk:
      - sysDescr
      - sysUpTime
      - hrSystemDate
      - hrProcessorLoad
      - mtxrGaugeTable
      - mtxrLicVersion
      - mtxrOpticalTableEntry
      - ifIndex
      - ifOperStatus
      - ifXTable
    lookups:
      - source_indexes: [mtxrGaugeIndex]
        lookup: mtxrGaugeName
      - source_indexes: [mtxrGaugeIndex]
        lookup: mtxrGaugeUnit
        drop_source_indexes: true

      - source_indexes: [mtxrOpticalIndex]
        lookup: mtxrOpticalName
        drop_source_indexes: true

      - source_indexes: [ifIndex]
        lookup: ifName
      - source_indexes: [ifIndex]
        lookup: ifOperStatus

    overrides:
      ignore_true: &ignore
        ignore: true
      mtxrGaugeUnit: *ignore
      mtxrGaugeName: *ignore
      mtxrOpticalName: *ignore
      mtxrOpticalRxLoss: *ignore
      mtxrOpticalTxFault: *ignore
      mtxrOpticalWavelength: *ignore
      mtxrOpticalSupplyVoltage: *ignore
      mtxrOpticalTxBiasCurrent: *ignore
      mtxrOpticalVendorSerial: *ignore
      ifName: *ignore
      ifLinkUpDownTrapEnable: *ignore
      ifHighSpeed: *ignore
      ifPromiscuousMode: *ignore
      ifAlias: *ignore
      ifCounterDiscontinuityTime: *ignore
#      mtxrGaugeUnit:
#        type: EnumAsInfo
```

Hasil `snmp.yml`
```bash
# WARNING: This file was auto-generated using snmp_exporter generator, manual changes will be lost.
auths:
  public_v2:
    community: public
    security_level: noAuthNoPriv
    auth_protocol: MD5
    priv_protocol: DES
    version: 2
modules:
  mikrotik:
    walk:
    - 1.3.6.1.2.1.2.2.1.8
    - 1.3.6.1.2.1.25.3.3.1.2
    - 1.3.6.1.2.1.31.1.1
    - 1.3.6.1.4.1.14988.1.1.19.1.1
    - 1.3.6.1.4.1.14988.1.1.3.100
    - 1.3.6.1.4.1.25461.2.1.2.3.11.1.1
    get:
    - 1.3.6.1.2.1.1.1.0
    - 1.3.6.1.2.1.1.3.0
    - 1.3.6.1.2.1.25.1.2.0
    - 1.3.6.1.4.1.14988.1.1.4.4.0
    metrics:
    - name: sysDescr
      oid: 1.3.6.1.2.1.1.1
      type: DisplayString
      help: A textual description of the entity - 1.3.6.1.2.1.1.1
    - name: sysUpTime
      oid: 1.3.6.1.2.1.1.3
      type: gauge
      help: The time (in hundredths of a second) since the network management portion
        of the system was last re-initialized. - 1.3.6.1.2.1.1.3
    - name: ifOperStatus
      oid: 1.3.6.1.2.1.2.2.1.8
      type: gauge
      help: The current operational state of the interface - 1.3.6.1.2.1.2.2.1.8
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
      enum_values:
        1: up
        2: down
        3: testing
        4: unknown
        5: dormant
        6: notPresent
        7: lowerLayerDown
    - name: hrSystemDate
      oid: 1.3.6.1.2.1.25.1.2
      type: DateAndTime
      help: The host's notion of the local date and time of day. - 1.3.6.1.2.1.25.1.2
    - name: hrProcessorLoad
      oid: 1.3.6.1.2.1.25.3.3.1.2
      type: gauge
      help: The average, over the last minute, of the percentage of time that this
        processor was not idle - 1.3.6.1.2.1.25.3.3.1.2
      indexes:
      - labelname: hrDeviceIndex
        type: gauge
    - name: ifInMulticastPkts
      oid: 1.3.6.1.2.1.31.1.1.1.2
      type: counter
      help: The number of packets, delivered by this sub-layer to a higher (sub-)layer,
        which were addressed to a multicast address at this sub-layer - 1.3.6.1.2.1.31.1.1.1.2
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
    - name: ifInBroadcastPkts
      oid: 1.3.6.1.2.1.31.1.1.1.3
      type: counter
      help: The number of packets, delivered by this sub-layer to a higher (sub-)layer,
        which were addressed to a broadcast address at this sub-layer - 1.3.6.1.2.1.31.1.1.1.3
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
    - name: ifOutMulticastPkts
      oid: 1.3.6.1.2.1.31.1.1.1.4
      type: counter
      help: The total number of packets that higher-level protocols requested be transmitted,
        and which were addressed to a multicast address at this sub-layer, including
        those that were discarded or not sent - 1.3.6.1.2.1.31.1.1.1.4
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
    - name: ifOutBroadcastPkts
      oid: 1.3.6.1.2.1.31.1.1.1.5
      type: counter
      help: The total number of packets that higher-level protocols requested be transmitted,
        and which were addressed to a broadcast address at this sub-layer, including
        those that were discarded or not sent - 1.3.6.1.2.1.31.1.1.1.5
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
    - name: ifHCInOctets
      oid: 1.3.6.1.2.1.31.1.1.1.6
      type: counter
      help: The total number of octets received on the interface, including framing
        characters - 1.3.6.1.2.1.31.1.1.1.6
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
    - name: ifHCInUcastPkts
      oid: 1.3.6.1.2.1.31.1.1.1.7
      type: counter
      help: The number of packets, delivered by this sub-layer to a higher (sub-)layer,
        which were not addressed to a multicast or broadcast address at this sub-layer
        - 1.3.6.1.2.1.31.1.1.1.7
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
    - name: ifHCInMulticastPkts
      oid: 1.3.6.1.2.1.31.1.1.1.8
      type: counter
      help: The number of packets, delivered by this sub-layer to a higher (sub-)layer,
        which were addressed to a multicast address at this sub-layer - 1.3.6.1.2.1.31.1.1.1.8
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
    - name: ifHCInBroadcastPkts
      oid: 1.3.6.1.2.1.31.1.1.1.9
      type: counter
      help: The number of packets, delivered by this sub-layer to a higher (sub-)layer,
        which were addressed to a broadcast address at this sub-layer - 1.3.6.1.2.1.31.1.1.1.9
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
    - name: ifHCOutOctets
      oid: 1.3.6.1.2.1.31.1.1.1.10
      type: counter
      help: The total number of octets transmitted out of the interface, including
        framing characters - 1.3.6.1.2.1.31.1.1.1.10
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
    - name: ifHCOutUcastPkts
      oid: 1.3.6.1.2.1.31.1.1.1.11
      type: counter
      help: The total number of packets that higher-level protocols requested be transmitted,
        and which were not addressed to a multicast or broadcast address at this sub-layer,
        including those that were discarded or not sent - 1.3.6.1.2.1.31.1.1.1.11
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
    - name: ifHCOutMulticastPkts
      oid: 1.3.6.1.2.1.31.1.1.1.12
      type: counter
      help: The total number of packets that higher-level protocols requested be transmitted,
        and which were addressed to a multicast address at this sub-layer, including
        those that were discarded or not sent - 1.3.6.1.2.1.31.1.1.1.12
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
    - name: ifHCOutBroadcastPkts
      oid: 1.3.6.1.2.1.31.1.1.1.13
      type: counter
      help: The total number of packets that higher-level protocols requested be transmitted,
        and which were addressed to a broadcast address at this sub-layer, including
        those that were discarded or not sent - 1.3.6.1.2.1.31.1.1.1.13
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
    - name: ifConnectorPresent
      oid: 1.3.6.1.2.1.31.1.1.1.17
      type: gauge
      help: This object has the value 'true(1)' if the interface sublayer has a physical
        connector and the value 'false(2)' otherwise. - 1.3.6.1.2.1.31.1.1.1.17
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
      enum_values:
        1: "true"
        2: "false"
    - name: mtxrOpticalIndex
      oid: 1.3.6.1.4.1.14988.1.1.19.1.1.1
      type: gauge
      help: ' - 1.3.6.1.4.1.14988.1.1.19.1.1.1'
      indexes:
      - labelname: mtxrOpticalIndex
        type: gauge
      lookups:
      - labels:
        - mtxrOpticalIndex
        labelname: mtxrOpticalName
        oid: 1.3.6.1.4.1.14988.1.1.19.1.1.2
        type: DisplayString
      - labels: []
        labelname: mtxrOpticalIndex
    - name: mtxrOpticalTemperature
      oid: 1.3.6.1.4.1.14988.1.1.19.1.1.6
      type: gauge
      help: ' - 1.3.6.1.4.1.14988.1.1.19.1.1.6'
      indexes:
      - labelname: mtxrOpticalIndex
        type: gauge
      lookups:
      - labels:
        - mtxrOpticalIndex
        labelname: mtxrOpticalName
        oid: 1.3.6.1.4.1.14988.1.1.19.1.1.2
        type: DisplayString
      - labels: []
        labelname: mtxrOpticalIndex
    - name: mtxrOpticalTxPower
      oid: 1.3.6.1.4.1.14988.1.1.19.1.1.9
      type: gauge
      help: ' - 1.3.6.1.4.1.14988.1.1.19.1.1.9'
      indexes:
      - labelname: mtxrOpticalIndex
        type: gauge
      lookups:
      - labels:
        - mtxrOpticalIndex
        labelname: mtxrOpticalName
        oid: 1.3.6.1.4.1.14988.1.1.19.1.1.2
        type: DisplayString
      - labels: []
        labelname: mtxrOpticalIndex
    - name: mtxrOpticalRxPower
      oid: 1.3.6.1.4.1.14988.1.1.19.1.1.10
      type: gauge
      help: ' - 1.3.6.1.4.1.14988.1.1.19.1.1.10'
      indexes:
      - labelname: mtxrOpticalIndex
        type: gauge
      lookups:
      - labels:
        - mtxrOpticalIndex
        labelname: mtxrOpticalName
        oid: 1.3.6.1.4.1.14988.1.1.19.1.1.2
        type: DisplayString
      - labels: []
        labelname: mtxrOpticalIndex
    - name: mtxrOpticalVendorName
      oid: 1.3.6.1.4.1.14988.1.1.19.1.1.11
      type: DisplayString
      help: ' - 1.3.6.1.4.1.14988.1.1.19.1.1.11'
      indexes:
      - labelname: mtxrOpticalIndex
        type: gauge
      lookups:
      - labels:
        - mtxrOpticalIndex
        labelname: mtxrOpticalName
        oid: 1.3.6.1.4.1.14988.1.1.19.1.1.2
        type: DisplayString
      - labels: []
        labelname: mtxrOpticalIndex
    - name: mtxrGaugeIndex
      oid: 1.3.6.1.4.1.14988.1.1.3.100.1.1
      type: gauge
      help: ' - 1.3.6.1.4.1.14988.1.1.3.100.1.1'
      indexes:
      - labelname: mtxrGaugeIndex
        type: gauge
      lookups:
      - labels:
        - mtxrGaugeIndex
        labelname: mtxrGaugeName
        oid: 1.3.6.1.4.1.14988.1.1.3.100.1.2
        type: DisplayString
      - labels:
        - mtxrGaugeIndex
        labelname: mtxrGaugeUnit
        oid: 1.3.6.1.4.1.14988.1.1.3.100.1.4
        type: gauge
      - labels: []
        labelname: mtxrGaugeIndex
    - name: mtxrGaugeValue
      oid: 1.3.6.1.4.1.14988.1.1.3.100.1.3
      type: gauge
      help: ' - 1.3.6.1.4.1.14988.1.1.3.100.1.3'
      indexes:
      - labelname: mtxrGaugeIndex
        type: gauge
      lookups:
      - labels:
        - mtxrGaugeIndex
        labelname: mtxrGaugeName
        oid: 1.3.6.1.4.1.14988.1.1.3.100.1.2
        type: DisplayString
      - labels:
        - mtxrGaugeIndex
        labelname: mtxrGaugeUnit
        oid: 1.3.6.1.4.1.14988.1.1.3.100.1.4
        type: gauge
      - labels: []
        labelname: mtxrGaugeIndex
    - name: mtxrLicVersion
      oid: 1.3.6.1.4.1.14988.1.1.4.4
      type: DisplayString
      help: software version - 1.3.6.1.4.1.14988.1.1.4.4
    - name: ifIndex
      oid: 1.3.6.1.4.1.25461.2.1.2.3.11.1.1
      type: gauge
      help: Index of the interface - 1.3.6.1.4.1.25461.2.1.2.3.11.1.1
      indexes:
      - labelname: ifIndex
        type: gauge
      lookups:
      - labels:
        - ifIndex
        labelname: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
      - labels:
        - ifIndex
        labelname: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge

```