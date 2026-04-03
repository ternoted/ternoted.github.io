---
date: '2026-03-30T08:14:19+07:00'
draft: false
title: 'IKEv2/IPSec VPN (Hub-and-Spoke) dengan IP Static'
summary: Lab VPN IKEv2/IPSec Hub-and-spoke menggunakan Router Netgate TNSR, Cisco IOSv, dan Mikrotik CHR pada GNS3
author: ["Ilham Wahayu Yanre"]
cover:
  image: cover.png
  hiddenInList: true
aliases: ["/network"]
series: ["Network"]
ShowBreadCrumbs: true
ShowToc: true
tocopen: true
---

## Introduction

VPN Hub-and-spoke adalah topologi VPN di mana jaringan kantor-kantor cabang (spokes) terhubung ke kantor pusat (hub). Sehingga LAN dari semua jaringan cabang dapat saling terhubung melalui jaringan pusat.

Artikel ini ditulis dengan asumsi Router Hub dan Spoke sama-sama menggunakan Static IP Public.

Selanjutnya router pusat akan disebut "Hub" dan router cabang akan disebut "Spoke".

## GNS3 Lab
[![](http-topologi.png#center "Topologi")](http-topologi.png)

Tujuan utama dari lab ini adalah menghubungkan ketiga LAN dari router-router di atas menggunakan IKEv2/IPSec VPN. Router yang digunakan adalah Netgate TNSR sebagai Hub, lalu Cisco IOSv dan Mikrotik CHR sebagai Spoke. Sehingga LAN Cisco ke LAN Mikrotik akan melewati Netgate TNSR.

Perlu diketahui bahwa VPN pada TNSR dan Cisco sama-sama menggunakan route-based, sedangkan Mikrotik menggunakan policy-based. Sehingga menghubungkan VPN Mikrotik dengan TNSR membutuhkan pendekatan yang berbeda.

Silakan ikuti artikel [Netgate TNSR - Simulasi Router Menggunakan GNS3](https://ternoted.github.io/posts/network/netgate-tnsr/) untuk cara install TNSR pada GNS3.

### Netgate TNSR
```bash
tunnel ipip 0                                           # Tunnel ke Cisco
    source ipv4 address 104.18.27.120
    destination ipv4 address 104.18.27.130
exit
tunnel ipip 1                                           # Tunnel ke Mikrotik
    source ipv4 address 104.18.27.120
    destination ipv4 address 104.18.27.140
exit

ipsec tunnel 0                                          # Parameter IPSec ke Cisco
    enable
    crypto config-type ike
    crypto ike
        version 2
        lifetime 28800
        proposal 1
            encryption aes256
            integrity sha256
            group modp2048
        exit
        identity local
            type address
            value 104.18.27.120
        exit
        identity remote
            type address
            value 104.18.27.130
        exit
        authentication local
            round 1
                psk 123abc
            exit
        exit
        authentication remote
            round 1
                psk 123abc
            exit
        exit
        child 1
            lifetime 3600
            proposal 1
                encryption aes256
                integrity sha256
                group modp2048
            exit
        exit
    exit
exit

ipsec tunnel 1                                          # Parameter IPSec ke Mikrotik
    enable
    crypto config-type ike
    crypto ike
        version 2
        lifetime 28800
        proposal 1
            encryption aes256
            integrity sha256
            group modp2048
        exit
        identity local
            type address
            value 104.18.27.120
        exit
        identity remote
            type address
            value 104.18.27.140
        exit
        authentication local
            round 1
                psk 123abc
            exit
        exit
        authentication remote
            round 1
                psk 123abc
            exit
        exit
        child 1
            lifetime 3600
            proposal 1
                encryption aes256
                integrity sha256
                group modp2048
            exit
        exit
    exit
exit

interface GE0                                           # WAN Interface
    enable
    ip address 104.18.27.120/24
exit
interface GE1                                           # LAN INterface
    enable
    ip address 192.168.0.1/24
exit
interface ipip0                                         # Tunnel Interface ke Cisco
    enable
    mtu 1400
    ip address 10.10.0.1/30
exit
interface ipip1                                         # Tunnel Interface ke Mikrotik. Mikrotik tidak mengetahui interface atau IP ptp ini karena Mikrotik menggunakan policy-based.
    enable                                              # Bahkan lawan IP ini (10.10.1.2/30) tidak ada di Mikrotik. Interface dan IP di sini hanya digunakan secara internal oleh TNSR
    mtu 1400                                            # untuk meneruskan trafik IPSec ke Mikrotik.
    ip address 10.10.1.1/30
exit

route table ipv4-VRF:0                                  # Static Routing ke LAN Cisco dan LAN Mikrotik
    id 0
    route 172.16.0.0/24
        next-hop 0 via 10.10.0.2
    exit
    route 192.168.255.0/24
        next-hop 0 via 10.10.1.2
    exit
exit
```

Jika terdapat banyak spoke yang harus terhubung ke Hub, maka sebaiknya menggunakan 1 Tunnel dan Interface saja yang di-set menjadi point-to-multipoint, dari pada membuat 1 Tunnel & Interface untuk masing-masing spoke. Caranya dengan menghilangkan ```destination ipv4 address a.b.c.d```. Lalu menggunakan command ```tunnel next-hops ipipX``` untuk menambahkan IP spoke.

### Cisco IOSv
```bash
crypto ikev2 proposal SPOKE1-PROP                                     # Parameter IPSec
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy SPOKE1-POL
 proposal SPOKE1-PROP

crypto ikev2 keyring SPOKE1-KEYRING
 peer HUB
  address 104.18.27.120
  pre-shared-key 123abc
 !

crypto ikev2 profile SPOKE1-PROF
 match identity remote address 104.18.27.120 255.255.255.255
 identity local address 104.18.27.130
 authentication remote pre-share
 authentication local pre-share
 keyring local SPOKE1-KEYRING

crypto ipsec transform-set SPOKE1-TS esp-aes 256 esp-sha256-hmac
 mode tunnel

crypto ipsec profile SPOKE1-IPSEC
 set transform-set SPOKE1-TS
 set ikev2-profile SPOKE1-PROF

interface Tunnel0                                                     # Tunnel Interface ke TNSR
 ip address 10.10.0.2 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel mode ipsec ipv4
 tunnel destination 104.18.27.120
 tunnel protection ipsec profile SPOKE1-IPSEC

interface GigabitEthernet0/0                                          # WAN Interface
 ip address 104.18.27.130 255.255.255.0
 no shutdown

interface GigabitEthernet0/1                                          # LAN Interface
 ip address 172.16.0.1 255.255.255.0
 no shutdown

ip route 192.168.0.0 255.255.255.0 Tunnel0
ip route 192.168.255.0 255.255.255.0 Tunnel0
```

### Mikrotik CHR
Di sini yang pendekatannya berbeda. Mikrotik menggunakan policy-based sehingga trafik IPSec tidak mengalir melewati Tunnel Interface, akan tetapi dienkripsi apabila trafik tersebut memenuhi persyaratan *ip/ipsec/policy*.

```bash
/ip address
add address=104.18.27.140/24 interface=ether1 network=104.18.27.0                                     # WAN Interface
add address=192.168.255.1/24 interface=ether2 network=192.168.255.0                                   # LAN Interface

/ip ipsec profile                                                                                     # Parameter IPSec
add name=SPOKE2-PROF dh-group=modp2048 enc-algorithm=aes-256 hash-algorithm=sha256

/ip ipsec peer
add address=104.18.27.120/32 exchange-mode=ike2 name=HUB profile=SPOKE2-PROF

/ip ipsec proposal
add auth-algorithms=sha256 enc-algorithms=aes-256-cbc name=SPOKE2-PROP pfs-group=modp2048

/ip ipsec identity
add peer=HUB secret=123abc

/ip ipsec policy                                                                                      # Level=unique diperlukan jika policy lebih dari 1 untuk peer yang sama
add dst-address=192.168.0.0/24 level=unique peer=HUB proposal=SPOKE2-PROP \
    src-address=192.168.255.0/24 tunnel=yes
add dst-address=172.16.0.0/24 level=unique peer=HUB proposal=SPOKE2-PROP \
    src-address=192.168.255.0/24 tunnel=yes

/ip route
add dst-address=172.16.0.0/24 gateway=104.18.27.120
add dst-address=192.168.0.0/24 gateway=104.18.27.120
```

#### Quick Test

Tes ping dari LAN Cisco ke LAN Mikrotik
[![](http-test-ping.png#center "Quick Test")](http-test-ping.png)

## Troubleshooting
>- Pastikan IP ptp reachable dari kedua sisi.
>- Pada TNSR, gunakan command ```show ipsec tunnel verbose``` untuk menganalisa output IPSec.
>- Pada Cisco, command ```show configuration``` dapat memberikan informasi jika ada yang kurang/mismatch pada konfigurasi IPSec. lalu command ```debug crypto ...``` juga sangat bermanfaat untuk mengetahui error yang terjadi pada IPSec atau IKEv2.
>- Pada Mikrotik, command ```ip/ipsec/policy print detail``` akan memperlihatkan status Phase 2. Gunakan juga ```ip/ipsec/active-peers print detail``` untuk melihat status peering.


## Penutup

Artikel ini dapat dijadikan referensi apabila teman-teman ingin membuat topologi VPN Hub-and-spoke dengan syarat semua Router menggunakan IP Static. Sekarang bagaimana jika hanya Router Hub saja yang menggunakan IP Static, sedangkan Router Spoke menggunakan IP Dynamic atau bahkan berada di belakang NAT? InsyaAllah akan dibahas pada tulisan selanjutnya. Ditunggu ya!