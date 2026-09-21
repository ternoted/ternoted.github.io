---
date: '2026-09-20T10:15:12+07:00'
draft: false
title: 'Replikasi Masalah Jaringan yang Disebabkan oleh Asymmetric Routing dan NAT'
summary: Ping bisa, blocking firewall tidak ada. Tapi kenapa koneksi TCP selalu reset? Tulisan ini adalah replikasi masalah jaringan yang disebabkan oleh routing yang tidak simetris dan konfigurasi NAT yang kurang tepat.
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

Beberapa waktu yang lalu penulis dihadapkan dengan masalah jaringan antara 2 node server (Server-A dan Server-B). Di mana Server-B ingin melakukan koneksi TCP (eg. SSH, HTTP) ke Server-A namun selalu mendapat respon `connection reset by peer`. Padahal sudah dipastikan tidak ada firewall pada kedua node ataupun router yang meng-filter koneksi antara kedua server. Hasil ping normal dan bahkan MTR menunjukkan koneksi dari Server-B ke Server-A hanya melewati 1 router (Router-B). Kesimpulannya masalah ini hanya terjadi pada koneksi yang melibatkan TCP.

```bash
# Server-B Terminal
ubuntu@server-b:~$ ssh 192.168.100.188
kex_exchange_identification: read: Connection reset by peer
Connection reset by 192.168.100.188 port 22
ubuntu@server-b:~$

# -------------------------------------------------------------------------------

                             My traceroute  [v0.95]
server-b (172.16.244.14) -> 192.168.100.188 (192.168.102026-09-20T07:53:09+0000
Keys:  Help   Display mode   Restart statistics   Order of fields   quit
                                       Packets               Pings
 Host                                Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. 172.16.244.1                      0.0%     9    3.5   3.5   2.6   4.3   0.6
 2. 192.168.100.188                   0.0%     9    7.1   6.6   5.3   8.5   1.1

```

## Topologi

[![](http-1.png#center "")](http-1.png)

Topologi ini membuat routing antara Server-B dengan Server-A menjadi tidak simetris. Karena trafik request dari Server-B akan pergi melewati gateway Router-B. Lalu dari Router-B langsung ke Server-A. Karena IP Server-A "directly connected" melalui vlan 100 pada Router-B. Sedangkan trafik reply dari Server-A akan melewati gateway Router-A, dan langsung ke Server-B melalui vlan 96.

Hal tersebut membuat `connection tracking` pada Router-A maupun Router-B hanya dapat melihat setengah dari koneksi TCP yang terjadi. Misalnya, Router-A tidak melihat packet "SYN" dari Server-B -> Server-A, dan hanya melihat "SYN-ACK" dari Server-A -> Server-B. Koneksi seperti ini dianggap `connection-state=invalid` oleh `connection tracking`. Jika ada rule firewall untuk `drop` trafik dengan `connection-state=invalid`, maka packet tersebut akan di-drop oleh Router-A. Akan tetapi pada kasus ini tidak ada rule firewall seperti itu pada kedua router.

## Konfigurasi Router

>- Konfigurasi Router-A
```yaml
[admin@Router-A] > expo
# 2026-09-20 03:53:22 by RouterOS 7.21.5
# system id = yPYu8B7TrBJ
#
/interface ethernet
set [ find default-name=ether1 ] disable-running-check=no
/interface vlan
add interface=ether1 name=vlan96 vlan-id=96
add interface=ether1 name=vlan100 vlan-id=100
/ip address
add address=172.16.244.12/24 interface=vlan96 network=172.16.244.0
add address=192.168.100.100/24 interface=vlan100 network=192.168.100.0
/ip dhcp-client
add interface=ether1
/ip firewall nat
add action=masquerade chain=srcnat out-interface=all-vlan
/system identity
set name=Router-A
```

>- Konfigurasi Router-B
```yaml
[admin@Router-B] > expo
# 2026-09-20 03:55:19 by RouterOS 7.21.5
# system id = 233eiPky7WK
#
/interface ethernet
set [ find default-name=ether1 ] disable-running-check=no
/interface vlan
add interface=ether1 name=vlan96 vlan-id=96
add interface=ether1 name=vlan100 vlan-id=100
/ip address
add address=172.16.244.1/24 interface=vlan96 network=172.16.244.0
add address=192.168.100.1/24 interface=vlan100 network=192.168.100.0
/ip dhcp-client
add interface=ether1
/system identity
set name=Router-B
```

## Troubleshooting

Berdasarkan informasi `connection reset by peer` yang diterima Server-B, maka penulis mengisolasi pengecekan pada Router-A. Karena trafik dari Server-A -> Server-B pasti melewati Router-A.

- Pada Router-A tidak ditemukan rule `drop connection-state=invalid`. Jadi harusnya ini bukan disebabkan oleh rule firewall. Tapi jika `connection tracking` dimatikan, yang artinya firewall dan NAT menjadi mati total, maka masalah koneksi reset TCP antara Server-B dan Server-A menghilang. Namun mematikan `connection tracking` tidak dapat menjadi solusi. Karena rule firewall dan NAT pada Router-A dibutuhkan dalam production.

- Sampai tahap ini penulis masih belum menaruh curiga pada NAT. Karena bagaimana mungkin NAT dapat me-reset koneksi TCP antara Server-A dan Server-B? Karena merasa buntu, maka penulis menjalankan `tcpdump` pada kedua server dan mengeksekusi `ssh 192.168.100.188` dari Server-B.

### `tcpdump` Server-A
```
1	0.000000	172.16.244.14	192.168.100.188	TCP	74	59928 → 22 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM TSval=628339403 TSecr=0 WS=128
2	0.001208	192.168.100.188	172.16.244.14	TCP	74	22 → 59928 [SYN, ACK] Seq=0 Ack=1 Win=65160 Len=0 MSS=1460 SACK_PERM TSval=2166238194 TSecr=628339403 WS=128
3	0.008152	172.16.244.14	192.168.100.188	TCP	66	59928 → 22 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=628339412 TSecr=2166238194
4	0.009718	172.16.244.14	192.168.100.188	SSHv2	108	Client: Protocol (SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.5)
5	0.010103	192.168.100.188	172.16.244.14	TCP	66	22 → 59928 [ACK] Seq=1 Ack=43 Win=65152 Len=0 TSval=2166238203 TSecr=628339413
6	0.012268	192.168.100.188	172.16.244.14	SSHv2	108	Server: Protocol (SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.5)
7	0.019095	172.16.244.14	192.168.100.188	TCP	54	59928 → 22 [RST] Seq=43 Win=0 Len=0
8	0.229131	172.16.244.14	192.168.100.188	TCP	108	[TCP Spurious Retransmission] 59928 → 22 [PSH, ACK] Seq=1 Ack=1 Win=64256 Len=42 TSval=628339632 TSecr=2166238194
9	0.229567	192.168.100.188	172.16.244.14	TCP	54	22 → 59928 [RST] Seq=1 Win=0 Len=0
```

### `tcpdump` Server-B
```
1	0.000000	172.16.244.14	192.168.100.188	TCP	74	59928 → 22 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM TSval=628339403 TSecr=0 WS=128
2	0.008040	192.168.100.188	172.16.244.14	TCP	74	22 → 59928 [SYN, ACK] Seq=0 Ack=1 Win=65160 Len=0 MSS=1460 SACK_PERM TSval=2166238194 TSecr=628339403 WS=128
3	0.008547	172.16.244.14	192.168.100.188	TCP	66	59928 → 22 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=628339412 TSecr=2166238194
4	0.009625	172.16.244.14	192.168.100.188	SSHv2	108	Client: Protocol (SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.5)
5	0.017278	172.16.244.12	172.16.244.14	TCP	66	22 → 59928 [ACK] Seq=1 Ack=1 Win=509 Len=0 TSval=2166238203 TSecr=628339413
6	0.018635	172.16.244.12	172.16.244.14	SSH	108	Server: Protocol (SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.5)
7	0.018756	172.16.244.14	172.16.244.12	TCP	54	59928 → 22 [RST] Seq=1 Win=0 Len=0
8	0.019165	172.16.244.14	172.16.244.12	TCP	54	59928 → 22 [RST] Seq=1 Win=0 Len=0
9	0.228203	172.16.244.14	192.168.100.188	TCP	108	[TCP Retransmission] 59928 → 22 [PSH, ACK] Seq=1 Ack=1 Win=64256 Len=42 TSval=628339632 TSecr=2166238194
10	0.235178	192.168.100.188	172.16.244.14	TCP	54	22 → 59928 [RST] Seq=1 Win=0 Len=0
```

#### Analisa `tcpdump` Server-B

Penulis bukan packet expert, tapi dari pcap di atas dapat langsung diketahui masalahnya di mana.

| Packet No. | Deskripsi |
|:-:|:-|
| 1-3 | TCP-Handshake. Di sini masih normal. src-address=172.16.244.14 dan dst-address=192.168.100.188 |
| 4 | Payload SSH langsung dikirim tepat setelah "ACK" |
| 5-6| Ini packet yang harusnya datang dari IP 192.168.100.188. Akan tetapi di sini IP source nya adalah 172.16.244.12. IP yang ada pada Router-A. Router-A memiliki rule NAT `/ip firewall nat add action=masquerade chain=srcnat out-interface=all-vlan`. Sehingga dipastikan Router-A me-masquerade packet ini sebelum diteruskan ke Server-B |
| 7-8 | Server-B tidak mengenali koneksi dengan IP source 172.16.244.14. Sedangkan aturannya, TCP akan merespon dengan RST untuk packet yang tidak ada socket-nya atau tidak dia kenali. Sehingga Server-B membalas dengan 2 RST (RESET) untuk packet 5 dan 6. Apa yang terjadi selanjutnya? Router-A meneruskan packet RST tersebut ke Server-A (packet nomor 7 pada pcap Server-A). Karena berdasarkan rule NAT pada Router-A, dia beranggapan bahwa packet tersebut adalah untuk Server-A. Server-A yang menerima packet RST kemudian menutup koneksi TCP tersebut |
| 9 | Server-B melakukan transmisi ulang. Karena dari sudut pandang dia, Server-A (192.168.100.188) belum membalas packet yang dia kirim sebelumnya. Akan tetapi dari sudut pandang Server-A, koneksi TCP tersebut sudah ditutup sehingga dia tidak lagi mengenali packet retransmisi yang Server-B kirimkan. Sehingga Server-A membalas dengan RST. Packet tersebut sampai ke Server-B, dan beranggapan bahwa Server-A yang me-reset koneksi SSH darinya (`connection reset by peer`) |

### Analisa Router-A

Router-A memiliki rule NAT. Namun kenapa rule tersebut baru aktif pada packet ke-5 pada `tcpdump` Server-B? Sedangkan pada saat TCP Handshake, Router-A tidak me-masquerade IP 192.168.100.188 ke 172.16.244.12.
>- Router-A hanya melihat packet "SYN-ACK" dari Server-A -> Server-B pada saat TCP Handshake. Router-A menganggap packet ini `connection-state=invalid`. Sedangkan packet invalid tidak akan diteruskan untuk diproses NAT, dan Router-A juga tidak memiliki rule firewall untuk men-drop packet invalid. Jadi Router-A meneruskan packet tersebut ke Server-B tanpa perubahan apa-apa.
>- Selanjutnya pada packet ke-5 (ACK dari Server-A), entah kenapa Router-A menganggap packet tersebut sebagai `connection-state=new`. Sehingga `connection tracking` menjadi aktif dan packet tersebut diproses oleh rule NAT yang merubah source IP 192.168.100.188 menjadi 172.16.244.12.

Untuk mendapatkan bukti, maka penulis menambahkan option `log=yes log-prefix=NAT-Debug` pada rule NAT Router-A. Lalu Server-B kembali me-request koneksi SSH ke Server-A.

```
[admin@Router-A] > ip firewall/nat/print
Flags: X - disabled, I - invalid; D - dynamic
 0    chain=srcnat action=masquerade out-interface=all-vlan log=yes
      log-prefix="NAT-Debug"

[admin@Router-A] > log print
 2026-09-21 03:29:07 firewall,info NAT-Debug srcnat: in:vlan100 out:vlan96, connection-state:new src-mac 0c:bb:f8:f6:00:00, proto TCP (ACK), 192.168.100.188:22->172.16.244.14:49700, len 52
```

`connection-state:new... proto TCP (ACK)...` pada log di atas benar menunjukkan bahwa rule NAT baru aktif pada packet ke-5 pada `tcpdump` `5	0.017278	172.16.244.12	172.16.244.14	TCP	66	22 → 59928 [ACK]`

Lalu untuk membuktikan bahwa Router-A menganggap packet "SYN-ACK" pada saat TCP Handshake sebagai `invalid`, penulis membuat rule firewall untuk men-drop packet invalid dan me-log nya. Kemudian dari Server-B mengeksekusi SSH ke Server-A.
```
[admin@Router-A] > ip firewall/filter/add action=drop chain=forward connection-state=invalid log=yes log-prefix=INVALID-Packet

[admin@Router-A] > log print
 2026-09-21 03:35:52 firewall,info INVALID-Packet forward: in:vlan100 out:vlan96, connection-state:invalid src-mac 0c:bb:f8:f6:00:00, proto TCP (SYN,ACK), 192.168.100.188:22->172.16.244.14:33210, len 60
 2026-09-21 03:35:53 firewall,info INVALID-Packet forward: in:vlan100 out:vlan96, connection-state:invalid src-mac 0c:bb:f8:f6:00:00, proto TCP (SYN,ACK), 192.168.100.188:22->172.16.244.14:33210, len 60
 2026-09-21 03:35:53 firewall,info INVALID-Packet forward: in:vlan100 out:vlan96, connection-state:invalid src-mac 0c:bb:f8:f6:00:00, proto TCP (SYN,ACK), 192.168.100.188:22->172.16.244.14:33210, len 60
 2026-09-21 03:35:54 firewall,info INVALID-Packet forward: in:vlan100 out:vlan96, connection-state:invalid src-mac 0c:bb:f8:f6:00:00, proto TCP (SYN,ACK), 192.168.100.188:22->172.16.244.14:33210, len 60
 2026-09-21 03:35:55 firewall,info INVALID-Packet forward: in:vlan100 out:vlan96, connection-state:invalid src-mac 0c:bb:f8:f6:00:00, proto TCP (SYN,ACK), 192.168.100.188:22->172.16.244.14:33210, len 60
 2026-09-21 03:35:56 firewall,info INVALID-Packet forward: in:vlan100 out:vlan96, connection-state:invalid src-mac 0c:bb:f8:f6:00:00, proto TCP (SYN,ACK), 192.168.100.188:22->172.16.244.14:33210, len 60

```

`connection-state:invalid... proto TCP (SYN,ACK)...` menunjukkan bahwa benar Router-A menganggap packet tersebut sebagai invalid dan men-drop nya karena ada rule drop. Sedangkan tanpa rule drop, Router-A tetap meneruskan packet tersebut. Tercatat beberapa log dengan info yang sama karena Server-B berkali-kali mentransmisikan SYN ke Server-A. Sedangkan jawaban SYN-ACK dari Server-A terus di-drop oleh Router-A.

## Solusi

### Opsi A

 Menyesuaikan rule NAT pada Router-A. Namun ini tidak mudah apalagi jika ada infrastruktur atau topologi yang sudah ada sebelumnya yang bergantung pada rule ini.

### Opsi B

Mengecualikan koneksi antara IP Server-A dan Server-B dari `connection tracking`. Jika dikecualikan, artinya koneksi antara kedua IP tidak akan di-_track_. Sehingga koneksi tidak akan menyentuh rule NAT. Karena NAT kaitannya adalah dengan `connection tracking`.
```
[admin@Router-A] > ip firewall/raw/add action=notrack chain=prerouting src-address=192.168.100.188 dst-address=172.16.244.14 log=yes log-prefix=NO_TRACK
[admin@Router-A] > ip firewall/raw/add action=notrack chain=prerouting src-address=172.16.244.14 dst-address=192.168.100.188 log=yes log-prefix=NO_TRACK

[admin@Router-A] > log/print    # Saat eksekusi SSH dari Server-B ke Server-A
 2026-09-21 03:52:49 firewall,info NO_TRACK prerouting: in:vlan100 out:(unknown 0), connection-state:invalid src-mac 0c:bb:f8:f6:00:00, proto TCP (SYN,ACK), 192.168.100.188:22->172.16.244.14:48858, len 60
 2026-09-21 03:52:49 firewall,info NO_TRACK prerouting: in:vlan100 out:(unknown 0), connection-state:invalid src-mac 0c:bb:f8:f6:00:00, proto TCP (ACK), 192.168.100.188:22->172.16.244.14:48858, len 52
```

Dengan begini koneksi antara kedua server di-_whitelist_ sehingga tidak akan diproses `connection tracking` dan rule NAT.

### Opsi C

Membuat static route di Router-A dan Router-B agar jalur trafik Server-A <> Server-B menjadi simetris.

### Opsi D

Membuat static route di Server-A agar gateway dengan tujuan Server-B menjadi Router-B. Server-A dan Router-B terhubung dengan VLAN 100 subnet 192.168.100.0/24. Sepertinya ini yang paling mudah karena tidak perlu merubah konfigurasi apapun pada Router.
```
# netplan Server-A
network:
  version: 2
  ethernets:
    ens3:
      match:
        macaddress: "0c:bb:f8:f6:00:00"
      dhcp4: true
      dhcp6: true
      set-name: "ens3"
      addresses:
        - 192.168.100.188/24
      routes:
        - to: default
          via: 192.168.100.100
        - to: 172.16.244.14         # IP Server-B
          via: 192.168.100.1        # IP Router-B
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

Dari case seperti ini penulis sering teringat tentang pentingnya memahami TCP/IP atau OSI Model sebagai kerangka berfikir agar dapat mengidentifikasi permasalahan teknis secara terstruktur.