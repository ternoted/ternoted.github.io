---
date: '2026-04-17T10:09:02+07:00'
draft: false
title: 'Mikrotik VPN IPSec/IKEv2 (Site-to-site)'
summary: Membahas konfigurasi VPN IKEv2 Site-to-site dengan IP static dan dynamic
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

Site-to-site adalah VPN yang menghubungkan LAN dari 2 router di lokasi yang berbeda (Site A - Site B).
Artikel ini akan membahas bagaimana cara menghubungkan 2 router menggunakan VPN IKEv2 dengan metode autentikasi PSK (Pre-shared Key). Router pada Site A adalah Mikrotik-Hub, dan pada Site B Mikrotik-Spoke1. Penamaannya seperti itu karena nantinya topologi Site-to-site ini akan dikembangkan menjadi topologi Hub-and-spoke.

Semua konfigurasi akan disimulasikan pada GNS3 menggunakan Mikrotik CHR 7.20.4

## Site-to-site
[![](http-site-to-site.png#center "Site-to-site Topologi")](http-site-to-site.png)

| Mikrotik-Hub | Site A |
|:-|:-|
|LAN|192.168.50.0/24|
|WAN|50.0.0.50/24|
|VPN IP|172.16.0.1/24|

| Mikrotik-Spoke1 | Site B |
|:-|:-|
|LAN|192.168.150.0/24|
|WAN|150.0.0.150/24|
|VPN IP|172.16.0.2/24|

### Mikrotik-Hub Configuration (Site A)
- Memasang IP address yang diperlukan.
```bash
/system identity set name=Mikrotik-Hub
/ip address add address=50.0.0.50/24 interface=ether1 network=50.0.0.0              # WAN
/ip address add address=192.168.50.1/24 interface=ether2 network=192.168.50.0       # LAN
/ip address add address=172.16.0.1/24 interface=ether1 network=172.16.0.0           # VPN
/ip route add gateway=50.0.0.1
/ip dns set servers=1.1.1.1
```
- Konfigurasi IPSec

```bash
/ip ipsec mode-config add address=172.16.0.2 name=to-spoke1 split-include=172.16.0.0/24,192.168.50.0/24
```
| Property | Description |
|:---|:---|
| `address=172.16.0.2` | Ini adalah IP yang akan diberikan oleh Mikrotik-Hub ke Mikrotik-Spoke1 secara dynamic. Diambil dari IP 172.16.0.0/24 yang sudah dipasang pada `ether1`. IP ini menjadi IP ptp VPN antar kedua router |
| `split-include=172.16.0.0/24,192.168.50.0/24` | Ini untuk split tunnel. Mikrotik-Spoke1 akan membuat dynamic policy berdasarkan konfig ini dengan `src-addr="address"` dan `dst-addr="split-include"`. Lalu Mikrotik-Hub akan me-mirror dynamic policy tersebut, sehingga menjadi `src-addr="split-include"` dan `dst-addr="address"`. Jadi trafik antar kedua subnet akan dienkripsi melalui VPN.

```bash
/ip ipsec profile add dh-group=modp2048 enc-algorithm=aes-256 hash-algorithm=sha256 name=hub-n-spoke-prof
```
| Property | Description |
|:---|:---|
| `/ip ipsec profile` | Ini adalah paramater Phase 1 dari IKEv2. Parameter ini harus sama pada kedua router |

```bash
/ip ipsec peer add exchange-mode=ike2 name=to-spokes passive=yes profile=hub-n-spoke-prof send-initial-contact=no
```
| Property | Description |
|:---|:---|
|`passive=yes` | Mikrotik-Hub tidak akan memulai "initiate" koneksi VPN ke Mikrotik-Spoke1. Akan tetapi hanya menunggu request dari Mikrotik-Spoke1. Dengan begitu, Mikrotik-Hub tidak perlu mengetahui IP dari Mikrotik-Spoke1. Akan tetapi Mikrotik-Spoke1 perlu mengetahui IP Mikrotik-Hub. Ini cocok untuk kondisi di mana Mikrotik-Hub memiliki IP static, dan Mikrotik-Spoke1 memiliki IP dynamic |

```bash
/ip ipsec proposal add auth-algorithms=sha256 enc-algorithms=aes-256-cbc name=hub-n-nspoke-prop
/port set 0 name=serial0
```
| Property | Description |
|:---|:---|
| `/ip ipsec proposal` | Parameter Phase 2 IKEv2. Harus sama pada kedua router |

```bash
/ip ipsec identity add generate-policy=port-strict mode-config=to-spoke1 peer=to-spokes remote-id=key-id:spoke1 secret=123abc
```
| Property | Description |
|:---|:---|
| `generate-policy=port-strict` | Membuat dynamic policy yang telah ditentukan pada `mode-config` |
| `remote-id=key-id:spoke1` | Merupakan matcher atau identifier apabila router yang berpartisipasi dalam VPN lebih dari 2. Tidak dibutuhkan pada topologi site-to-site, tapi dibutuhkan pada topologi hub-and-spoke. Value `remote-id` di sini harus sama dengan value `my-id` yang ada pada Mikrotik-Spoke1 |
| `secret` | Seperti password atau PSK (Pre-shared Key). Harus sama pada kedua router. Gunakan kombinasi yang lebih kompleks dan panjang. Atau gunakan autentikasi dengan Certificate untuk kemanan yang lebih tinggi |

```bash
/ip ipsec policy set 0 proposal=hub-n-nspoke-prop
```
| Property | Description |
|:---|:---|
| `set 0 proposal=hub-n-nspoke-prop` | Meng-edit template pada `/ip ipsec policy` agar menggunakan proposal yang telah dibuat sebelumnya |

Konfigurasi IPSec pada Mikrotik-Hub selesai sampai tahap ini.

- Selanjutnya konfigurasi static route ke arah LAN Mikrotik-Spoke1.
```bash
/ip route add dst-address=192.168.150.0/24 gateway=172.16.0.2
```
- Konfigurasi NAT

```bash
/ip firewall nat add action=masquerade chain=srcnat out-interface=ether1
```
NAT ini berfungsi untuk WAN dan VPN. Untuk route ke 0.0.0.0/0 (Internet), IP router akan di-masquerade ke IP WAN. Sedangkan route ke 192.168.150.0/24 (LAN Mikrotik-Spoke1), IP router akan di-masquerade ke IP VPN. Dengan begitu tidak perlu membuat NAT untuk VPN secara spesifik. Karena IP WAN dan IP VPN sama-sama berada di `ether1`. Jika IP WAN dan IP VPN berada di interface yang berbeda, maka diperlukan rule NAT tambahan.

---

### Mikrotik-Spoke1 Configuration (Site B)
- Memasang IP address yang diperlukan.
```bash
/system identity set name=Mikrotik-Spoke1
/ip address add address=150.0.0.150/24 interface=ether1 network=150.0.0.0         # WAN
/ip address add address=192.168.150.1/24 interface=ether2 network=192.168.150.0   # LAN
/ip route add gateway=150.0.0.1

```

- Konfigurasi IPSec

```bash
/ip ipsec profile add dh-group=modp2048 enc-algorithm=aes-256 hash-algorithm=sha256 name=hub-n-spoke-prof
```
```bash
/ip ipsec peer add address=50.0.0.50/32 exchange-mode=ike2 name=to-hub profile=hub-n-spoke-prof
```
| Property | Description |
|:---|:---|
| `address=50.0.0.50/32` | IP address Mikrotik-Hub dimasukkan di sini karena Mikrotik-Spoke1 sebagai initiator harus mengetahui IP dari Mikrotik-Hub |
```bash
/ip ipsec proposal add auth-algorithms=sha256 enc-algorithms=aes-256-cbc name=hub-n-spoke-prop
```
```bash
/ip ipsec identity add generate-policy=port-strict mode-config=request-only my-id=key-id:spoke1 peer=to-hub secret=123abc
```
| Property | Description |
|:---|:---|
| `my-id=key-id:spoke1` | matcher yang harus sama dengan `remote-id` pada Mikrotik-Hub |
```bash
/ip ipsec policy set 0 proposal=hub-n-spoke-prop
/ip ipsec policy add dst-address=172.16.0.1/32 level=unique peer=to-hub proposal=hub-n-spoke-prop src-address=192.168.150.0/24 tunnel=yes
```
Tidak seperti Mikrotik-Hub yang semua policy-nya dibuat secara dynamic, pada Mikrotik-Spoke1 harus dibuat 1 policy secara manual. `dst-address=172.16.0.1/32` merupakan IP VPN pada Mikrotik-Hub, dan `src-address=192.168.150.0/24` adalah LAN Mikrotik-Spoke1. Policy ini akan di-mirror oleh Mikrotik-Hub menjadi `src-addr=172.16.0.1/32` dan `dst-addr=192.168.150.0/24`. Ini diperlukan agar LAN Mikrotik-Hub bisa berkomunikasi dengan LAN Mikrotik-Spoke1. Karena dynamic policy yang dibuat oleh Mikrotik-Hub secara otomatis hanya memungkinkan komunikasi dari LAN Mikrotik-Spoke1 ke LAN Mikrotik-Hub saja.

- Konfigurasi static route ke arah LAN Mikrotik-Hub
```bash
/ip route add dst-address=192.168.50.0/24 gateway=172.16.0.1
```
- Konfigurasi NAT
```bash
/ip firewall nat add action=masquerade chain=srcnat out-interface=ether1
```

## Troubleshooting

Sampai di sini konfigurasi pada kedua router sudah selesai. Selanjutnya beberapa pengecekan bisa dilakukan untuk memastikan VPN sudah berjalan dengan benar.

>- `ip ipsec/active-peers/print detail`
```bash
[admin@Mikrotik-Hub] > ip ipsec/active-peers/print detail
Flags: R - responder; N - natt-peer
 0 R  id="spoke1" local-address=50.0.0.50 port=4500 remote-address=150.0.0.150
      port=4500 state=established side=responder dynamic-address=172.16.0.2
      uptime=5h9m14s last-seen=3s ph2-total=3 spii="341c6d6330c59831"
      spir="16c05dd1993caee0" spii="341c6d6330c59831" spir="16c05dd1993caee0"
```
`ph2-total` di atas menunjukkan total policy yang ada pada `/ip ipsec policy`. Jika `ph2-total` tidak ada atau nilainya kurang dari jumlah policy yang seharusnya, maka Phase 2 dari IKE gagal terjadi. Artinya ada yang salah dengan konfigurasi `proposal`.

>- `ip ipsec/installed-sa/print detail`
```bash
[admin@Mikrotik-Hub] > ip ipsec/installed-sa/print detail
Flags: S - seen-traffic; H - hw-aead; A - AH, E - ESP
 0   E spi=0xE841441 src-address=150.0.0.150 dst-address=50.0.0.50 state=mature
       auth-algorithm=sha256 enc-algorithm=aes-cbc enc-key-size=256
       auth-key="61a63f8976fa4d3fae084cb927cc835b0a0c74a1b1d2fc6630198d1f150199
         03"
       enc-key="642751166df786565ca2e16a10ce4d7d647864354f4eca50490d58bde647ad9
        b"
       add-lifetime=24m23s/30m29s replay=128

 1   E spi=0xE2829EA src-address=50.0.0.50 dst-address=150.0.0.150 state=mature
       auth-algorithm=sha256 enc-algorithm=aes-cbc enc-key-size=256
       auth-key="c249a95e4afe58ac2e35ff971753d1246cc9055f99b31488c398ecf0d9441d
         a0"
       enc-key="3a53c91aaef317fb33ea80020286ec1abe07289564b70730f3045fb34ddf858
        8"
       add-lifetime=24m23s/30m29s replay=128

 2   E spi=0x65B1A81 src-address=150.0.0.150 dst-address=50.0.0.50 state=mature
       auth-algorithm=sha256 enc-algorithm=aes-cbc enc-key-size=256
       auth-key="57d8251dad5fa17cc1ec18e7975d3b6ee976b3adcf807a022c9092d162fab4
         14"
       enc-key="c7935f7a6fe7b7c0195eadb1341c09ca95dc15624cbb1a562ec59ce2cc9f170
        f"
       add-lifetime=24m21s/30m27s replay=128

 3   E spi=0xFF2BA1C src-address=50.0.0.50 dst-address=150.0.0.150 state=mature
       auth-algorithm=sha256 enc-algorithm=aes-cbc enc-key-size=256
       auth-key="6cda81bddcbea63241a7e9ba131ff459bcb35705ec3b88422b84c414d25b6d
         d1"
       enc-key="ed27dfe3fe815608470ca9ffe5805609bf4a8a2f51beef38abdfcd070cdad00
        9"
       add-lifetime=24m21s/30m27s replay=128

 4   E spi=0x73F97FA src-address=150.0.0.150 dst-address=50.0.0.50 state=mature
       auth-algorithm=sha256 enc-algorithm=aes-cbc enc-key-size=256
       auth-key="ccd04268bcaee75a5d35a1c84b99eea5bedb751973440e3ad79bc0e970ab10
         84"
       enc-key="cdfc7e0479fc25f5dc296f760746c3ad93efd20de2a5829af1572e0ffb46e6f
        8"
       add-lifetime=24m9s/30m12s replay=128

 5   E spi=0x7166EEF src-address=50.0.0.50 dst-address=150.0.0.150 state=mature
       auth-algorithm=sha256 enc-algorithm=aes-cbc enc-key-size=256
       auth-key="55b2aa1c9c8293eaf81d5049dd1c8a25400c796560be6ada64e394b1db9c7f
         79"
       enc-key="1990ca47d70ee872ab1cd936c6eb42dc46d4666a4faf64fe4beb85569e8aca3
        0"
       add-lifetime=24m9s/30m12s replay=128
```

Jika Phase 2 gagal karena miskonfigurasi proposal, maka output dari command di atas juga akan kosong. Dari output di atas kita juga bisa memastikan apakah parameter Phase 2 sudah sesuai dengan yang kita inginkan. Perhatikan `auth-algorithm` dan `enc-algorithm` sudah sama dengan proposal yang dibuat sebelumnya.

>- `ip ipsec/policy/print detail`
```bash
[admin@Mikrotik-Hub] > ip ipsec/policy/print detail
Flags: T - template; B - backup;
X - disabled, D - dynamic, I - invalid, A - active; * - default
 0 T  * group=default src-address=::/0 dst-address=::/0 protocol=all
        proposal=hub-n-nspoke-prop template=yes

 1   D  peer=to-spokes tunnel=yes src-address=172.16.0.1/32 src-port=any
        dst-address=192.168.150.0/24 dst-port=any protocol=all action=encrypt
        level=unique ipsec-protocols=esp sa-src-address=50.0.0.50
        sa-dst-address=150.0.0.150 proposal=hub-n-nspoke-prop ph2-count=1
        ph2-state=established

 2   D  peer=to-spokes tunnel=yes src-address=172.16.0.0/24 src-port=any
        dst-address=172.16.0.2/32 dst-port=any protocol=all action=encrypt
        level=unique ipsec-protocols=esp sa-src-address=50.0.0.50
        sa-dst-address=150.0.0.150 proposal=hub-n-nspoke-prop ph2-count=1
        ph2-state=established

 3   D  peer=to-spokes tunnel=yes src-address=192.168.50.0/24 src-port=any
        dst-address=172.16.0.2/32 dst-port=any protocol=all action=encrypt
        level=unique ipsec-protocols=esp sa-src-address=50.0.0.50
        sa-dst-address=150.0.0.150 proposal=hub-n-nspoke-prop ph2-count=1
        ph2-state=established
```
Policy nomor 1 adalah yang dibuat secara manual dari Mikrotik-Spoke1, dan policy nomor 2 dan 3 adalah policy yang dibuat secara dynamic berdasarkan `split-include` pada `mode-config`. Output command yang sama pada Mikrotik-Spoke1 adalah kebalikan dari output di atas:

```bash
[admin@Mikrotik-Spoke1] > ip ipsec/policy/print detail
Flags: T - template; B - backup;
X - disabled, D - dynamic, I - invalid, A - active; * - default
 0 T  * group=default src-address=::/0 dst-address=::/0 protocol=all
        proposal=hub-n-spoke-prop template=yes

 1   D  peer=to-hub tunnel=yes src-address=172.16.0.2/32 src-port=any
        dst-address=172.16.0.0/24 dst-port=any protocol=all action=encrypt
        level=unique ipsec-protocols=esp sa-src-address=150.0.0.150
        sa-dst-address=50.0.0.50 proposal=hub-n-spoke-prop ph2-count=1
        ph2-state=established

 2   D  peer=to-hub tunnel=yes src-address=172.16.0.2/32 src-port=any
        dst-address=192.168.50.0/24 dst-port=any protocol=all action=encrypt
        level=unique ipsec-protocols=esp sa-src-address=150.0.0.150
        sa-dst-address=50.0.0.50 proposal=hub-n-spoke-prop ph2-count=1
        ph2-state=established

 3   A  peer=to-hub tunnel=yes src-address=192.168.150.0/24 src-port=any
        dst-address=172.16.0.1/32 dst-port=any protocol=all action=encrypt
        level=unique ipsec-protocols=esp sa-src-address=150.0.0.150
        sa-dst-address=50.0.0.50 proposal=hub-n-spoke-prop ph2-count=1
        ph2-state=established
```

Policy nomor 3 adalah yang dibuat secara manual dengan flag `A - active`. Router harus mengetahui IP dari router lawan untuk membuat policy secara manual. Jadi policy manual tidak akan jalan dengan flag `A` jika dibuat di Mikrotik-Hub, karena Mikrotik-Hub tidak mengetahui IP Mikrotik-Spoke1.

>- `tool/torch ether1`
```bash
[admin@Mikrotik-Spoke1] > tool/torch ether1
Columns: MAC-PROTOCOL, IP-PROTOCOL, DSCP, SRC-ADDRESS, DST-ADDRESS, TX, RX, TX>
MA  IP-PROTOCOL  DSCP  SRC-ADDRESS    DST-ADDRESS  TX       RX       T  R
ip  icmp            0  192.168.50.50  172.16.0.2   0bps     784bps   0  1
ip  ipsec-esp       0  50.0.0.50      150.0.0.150  1360bps  1360bps  1  1
```

Torch di atas diambil dari Mikrotik-Spoke1 ketika PC Spoke1-LAN (192.168.150.150) melakukan ping ke PC Hub-LAN (192.168.50.50). Ada 2 entry pada hasil torch di atas. Pertama ICMP dengan `src-addr=192.168.50.50` `dst-addr=172.16.0.2`, ini adalah packet icmp reply dari PC Hub-LAN yang di-decrypt ketika sampai ke ether1, sehingga packet-nya bisa dibaca. IP 172.16.0.2 adalah IP NAT dari ether1 Mikrotik-Spoke1. Sedangkan full packet nya berada di dalam packet ipsec-esp pada entry kedua. Packet icmp-request yang biasanya muncul, tidak muncul di sini karena sudah di-encrypt terlebih dahulu oleh IPSec.

>- Wireshark

Di bawah ini adalah hasil pcap menggunakan Wireshark pada ether1 pada saat ping dari Spoke1-LAN masih berlangsung ke Hub-LAN.
[![](http-wireshark.png#center "PCAP Wireshark")](http-wireshark.png)
Wireshark tidak membaca ada packet ICMP karena semua trafik antar router dienkripsi. Sehingga yang muncul hanya `ESP`.

>- Simulasi IP dynamic pada Mikrotik-Spoke1

```bash
[admin@Mikrotik-Hub] > ip ipsec/active-peers/print
Flags: R - RESPONDER
Columns: ID, STATE, UPTIME, PH2-TOTAL, REMOTE-ADDRESS, DYNAMIC-ADDRESS
#   ID      STATE        UPTIME  PH2-TOTAL  REMOTE-ADDRESS  DYNAMIC-ADDRESS
0 R spoke1  established  3h8m2s          3  150.0.0.150     172.16.0.2
```

Dari Mikrotik-Hub, IP Mikrotik-Spoke1 yang terdeteksi adalah 150.0.0.150. Selanjutnya IP WAN pada Mikrotik-Spoke1 akan diganti menjadi 150.0.0.200 untuk menyimulasikan perubahan IP secara dynamic.

```bash
[admin@Mikrotik-Spoke1] > ip address/set 0 address=150.0.0.200
[admin@Mikrotik-Spoke1] > ip address/print
Flags: D - DYNAMIC
Columns: ADDRESS, NETWORK, INTERFACE
#   ADDRESS           NETWORK        INTERFACE
0   150.0.0.200/24    150.0.0.0      ether1
1   192.168.150.1/24  192.168.150.0  ether2
2 D 172.16.0.2/24     172.16.0.0     ether1

```

```bash
[admin@Mikrotik-Hub] > ip ipsec/active-peers/print interval=1
Flags: R - RESPONDER
Columns: ID, STATE, UPTIME, PH2-TOTAL, REMOTE-ADDRESS, DYNAMIC-ADDRESS
#   ID      STATE        UPTIME  PH2-TOTAL  REMOTE-ADDRESS  DYNAMIC-ADDRESS
2 R spoke1  established  9s              3  150.0.0.200     172.16.0.2
```

Ketika IP WAN pada Mikrotik-Spoke1 diganti, VPN sempat drop dan kemudian up lagi dengan IP yang berbeda. Koneksi drop sesaat ini tidak dapat dihindari karena kedua router harus membuat koneksi/peer VPN baru.

## Penutup

Sekian penjelasan dari konfigurasi VPN IKEv2 site-to-site, di mana 1 router menggunakan IP static, dan 1 router menggunakan IP dynamic. Artikel selanjutnya akan membahas tentang topologi VPN hub-and-spoke, yang pada dasarnya hanya menambahkan beberapa router spoke pada topologi site-to-site ini.