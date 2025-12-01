---
date: '2025-12-01T07:20:44+07:00'
draft: false
title: 'Simulasi Netgate TNSR di GNS3'
summary: Tutorial konfigurasi Netgate TNSR menggunakan GNS3
author: ["Ilham Wahayu Yanre"]
cover:
  image: cover.png
  hiddenInList: true
aliases: ["/network"]
tags: ["TNSR","Router"]
series: ["Router"]
ShowBreadCrumbs: true
ShowToc: true
tocopen: true
---

[TNSR](https://www.netgate.com/tnsr) (dibaca "tensor") adalah high-performance Software Router keluaran Netgate. High-performance maksudnya adalah TNSR didesain untuk meng-handle trafik packet dalam kecepatan tinggi hingga 100+ Gbps. TNSR dibuat di atas Linux Ubuntu, jadi untuk instalasi awalnya seperti kita sedang menginstall Ubuntu Server.

## Pre-requisites
>- GNS3 sudah terinstall dan berfungsi dengan normal (Tulisan ini tidak mencakup cara instalasi dan konfigurasi GNS3)
>- File .iso TNSR (dikarenakan TNSR bukan free software, maka cara mendapatkannya penulis serahkan kepada pembaca.){{< collapse >}}
Actually you can download it for free somewhere on Internet
{{</ collapse >}}
>- Naikkan spek GNS3 VM. Contoh, VCPU 12 dan RAM 12GB. Karena Qemu TNSR akan kita setup di dalam GNS3 VM.

## Installation

### Step 1. Buka GNS3 dan Buat Project Baru
[![Gambar 1. GNS3 New Project](http-gns3-new-project.png#center "GNS3 New Project")](http-gns3-new-project.png)

### Step 2. Masuk ke Edit -> Preferences -> Qemu VMs
[![Gambar 2. Edit -> Preferences -> Qemu VMs](http-edit-preferences-qeme-vms.png#center "Edit -> Preferences -> Qemu VMs")](http-edit-preferences-qeme-vms.png)

### Step 3. Pilih New -> Run this Qemu on the GNS3 VM
[![Gambar 3. Run this Qemu on the GNS3 VM](http-run-this-qemu-on-gns3-vm.png#center "Run this Qemu on the GNS3 VM")](http-run-this-qemu-on-gns3-vm.png)

### Step 4. Tentukan Nama Qemu VM
[![Gambar 4. Nama Qemu VM](http-vm-name.png#center "Nama Qemu VM")](http-vm-name.png)

### Step 5. Sesuaikan RAM size (4GB)
[![Gambar 5. Ukuran RAM](http-ram-vm.png#center "Ukuran RAM")](http-ram-vm.png)

### Step 6. Next Sampai Tab Disk Image, lalu pilih ```empty10G.qcow2```
[![Gambar 6. Disk Image](http-disk-image.png#center "Disk Image")](http-disk-image.png)
Ini adalah base tempat kita menginstall .iso TNSR nantinya. Jadi seperti kita menyediakan disk 10GB untuk menginstall .iso.

Kemudian pilih Finish.

### Step 7. Pilih Edit pada Qemu VM TNSR Yang Telah Dibuat
[![Gambar 7. Edit Qemu VM](http-edit-qemu-vm.png#center "Edit Qemu VM")](http-edit-qemu-vm.png)

Ganti vCPUs menjadi 4. Sesuaikan juga Category dan Symbol seperti gambar di atas.

### Step 8. Masukkan file .iso TNSR pada tab "CD/DVD".
[![Gambar 8. CD/DVD .iso](http-cd-dvd-iso.png#center "CD/DVD .iso")](http-cd-dvd-iso.png)
Tunggu proses upload sampai selesai.

### Step 9. Sesuaikan Konfigurasi Network Adapter
[![Gambar 9. Network](http-network.png#center "Network")](http-network.png)
Ganti Type menjadi vmxnet3. vmxnet3 adalah virtual adapter milik VMWare yang direkomendasikan oleh Netgate untuk VM TNSR. Di sini penulis menambahkan adapter menjadi 3. 1 untuk port mgmt, dan 2 untuk port yang akan digunakan TNSR

### Step 10. Tambahkan Qemu Options pada Tab Advanced
[![Gambar 10. Advanced](http-qemu-options.png#center "Advanced")](http-qemu-options.png)
Tambahkan line berikut pada bagian Options di bawah Additional Settings:

``` bash
-machine pc,accel=kvm -cpu qemu64,+ssse3,+sse4.1,+sse4.2 -vga virtio -usbdevice tablet -boot order=cd
```
Baris di atas perlu ditambahkan agar TNSR dapat membaca dan menggunakan network adapter pada VM.

Selanjutnya pilih OK. Pilih Apply pada halaman Preferences -> Qemu VMs

### Step 11. Tarik Template TNSR yang Sudah Jadi dan Jalankan, Lalu Buka Console
[![Gambar 11. Template TNSR](http-template-tnsr.png#center "Template TNSR")](http-template-tnsr.png)

### Step 12. Lanjutkan proses instalasi dengan mengikuti panduan di bawah:

[![](http-continue-in-rich.png#center)](http-continue-in-rich.png)
[![](http-english.png#center)](http-english.png)
[![](http-keyboard.png#center)](http-keyboard.png)
[![](http-instal-network.png#center "Pada bagian ini penulis memilih interface ens3 sebagai port mgmt. Jadi interface yang statusnya disabled (ens4 dan ens5) akan digunakan oleh TNSR untuk routing.")](http-instal-network.png)
[![](http-user.png#center "User ini digunakan untuk mengakses host dari TNSR, bukan TNSR itu sendiri")](http-user.png)
Kemudian pilih Done, dan tunggu proses instalasi selesai lalu pilih Reboot Now.

Setelah memilih Reboot Now, seharusnya VM tersebut reboot dan menampilkan halaman login. Akan tetapi yang penulis temui VM tersebut stuck. Oleh karena itu silakan klik kanan VM/node lalu pilih reload, kemudian buka console kembali. Setelah itu kita dapat melihat proses booting selama 1-3 menit sampai ke halaman login.

## Konfigurasi Topologi Sederhana Menggunakan TNSR

Selanjutnya kita membuat topologi IP ptp sederhana antara TNSR dengan Mikrotik CHR seperti berikut:
[![](http-topologi.png#center)](http-topologi.png)

### Step 1. Login ke TNSR

Ada 2 user pada Router TNSR. 1 User yang dibuat saat proses instalasi untuk mengakses host, dan 1 user TNSR untuk mengakses Router.
```bash
TNSR Default Login

username: tnsr
password: tnsr-default
```

### Step 2. Aktifkan interface yang akan digunakan oleh TNSR

```bash
tnsr tnsr# conf t
tnsr tnsr(config)# dataplane dpdk dev ?
  0000:00:03.0          Ethernet controller: VMware VMXNET3 Ethernet Controller (rev 01) ( Active Interface ens3 )
  0000:00:04.0          Ethernet controller: VMware VMXNET3 Ethernet Controller (rev 01)
  0000:00:05.0          Ethernet controller: VMware VMXNET3 Ethernet Controller (rev 01)
  default
tnsr tnsr(config)# dataplane dpdk dev
```

Interface dengan keterangan (Active Interface) menunjukkan bahwa interface tersebut digunakan oleh host. Jadi interface tersebut tidak bisa digunakan untuk routing oleh TNSR. Sedangkan 2 interface di bawahnya adalah ens4 dan ens5 yang statusnya disabled pada saat instalasi sebelumnya.

Kesimpulannya, agar interface dapat digunakan oleh TNSR, maka interface tersebut harus dalam keadaan mati/disabled dari sisi host.

```bash
tnsr tnsr(config)# dataplane dpdk dev 0000:00:04 network
Changes to dataplane startup settings require a dataplane restart to take effect.
tnsr tnsr(config)# dataplane dpdk dev 0000:00:05 network
Changes to dataplane startup settings require a dataplane restart to take effect.
tnsr tnsr(config)#
tnsr tnsr(config)# service dataplane restart
```
Command di atas mengaktifkan interface dengan kode 0000:00:04 dan 0000:00:05 agar dapat digunakan oleh TNSR. Lalu me-restart dataplane.

```bash
tnsr tnsr(config)# show interface
  <cr>
  GigabitEthernet0/4/0  Interface name
  GigabitEthernet0/5/0  Interface name
  access-list           Access Control List
  acl                   Access Control List
  bond                  Bond interfaces
  bonding               Bonding status of hardware interfaces
  bridge                Bridge Domain
  counters              Interface statistics
  ip                    IPv4
  ipv4                  IPv4
  ipv6                  IPv6
  lacp                  LACP details
  link                  Link status
  local0                Interface name
  mac-address           MAC address
  memif                 Memif interfaces
  rx-queues             Rx-queue / CPU
  subif                 Subinterface
  tap                   Tap objects
  vlan                  VLAN
  vtr                   VLAN Tag rewrite
tnsr tnsr(config)# show interface
```
Command `show interface` di atas menunjukkan GigabitEthernet0/4/0 (0000:00:04) dan GigabitEthernet0/5/0 (0000:00:05) sudah terdeteksi dan diambil alih oleh TNSR.

### Step 3. Pasang IP pada Interface
```bash
tnsr tnsr(config)# interface GigabitEthernet0/4/0
tnsr tnsr(config-interface)# ip address 10.0.0.1/30
tnsr tnsr(config-interface)# enable
tnsr tnsr(config)# show interface GigabitEthernet0/4/0
Interface: GigabitEthernet0/4/0
    Admin status: up
    Link up, link-speed 10 Gbps, full duplex
    Link MTU: 1500 bytes
    MAC address: 0c:bd:7f:56:00:01
    IPv4 MTU: 0 bytes
    IPv4 Route Table: ipv4-VRF:0
    IPv4 addresses:
        10.0.0.1/30

tnsr tnsr(config)# ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=2.41 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.835 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=1.06 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.609 ms
^C
--- 10.0.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3035ms
rtt min/avg/max/mdev = 0.609/1.229/2.414/0.702 ms
```

## BGP Menggunakan TNSR

```bash
tnsr tnsr(config)# route dynamic bgp
tnsr tnsr(config-frr-bgp)# enable
tnsr tnsr(config-frr-bgp)# server vrf default
tnsr tnsr(config-bgp)# as-number 64698
tnsr tnsr(config-bgp)# router-id 10.0.0.1
tnsr tnsr(config-bgp)# neighbor 10.0.0.2
tnsr tnsr(config-bgp-neighbor)# remote-as 64699
tnsr tnsr(config-bgp-neighbor)# enable
tnsr tnsr(config-bgp-neighbor)# exit
tnsr tnsr(config-bgp)# exit
tnsr tnsr(config-frr-bgp)# exit
tnsr tnsr# show route dynamic bgp neighbors 10.0.0.2
BGP neighbor is 10.0.0.2, remote AS 64699, local AS 64698, external link
  BGP version 4, remote router ID 10.0.0.2, local router ID 10.0.0.1
  BGP state = Established, up for 00:03:50
  Last read 00:00:50, Last write 00:00:50
  Hold time is 180, keepalive interval is 60 seconds
  Configured conditional advertisements interval is 60 seconds
```

## Penutup

TNSR mampu meng-handle full Internet prefix dan jutaan pps dengan hardware yang memadai. Sehingga sangat cocok untuk jaringan skala menengah ke atas. Dari pengalaman penulis, TNSR tidak pernah menjadi titik bottleneck di saat penggunaan trafik yang tinggi atau DDoS sekalipun. Configuration style-nya pun sangat terstruktur, sehingga orang-orang yang familiar dengan Cisco, Juniper, atau Aruba, akan bisa menguasainya dengan lebih mudah.