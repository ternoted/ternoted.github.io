---
date: '2026-10-02T10:06:28+07:00'
draft: false
title: 'High Availability (HA), Hyperconverged (HCI), Hyper-V Cluster with StarWind VSAN FREE'
summary: Membangun infrastruktur Hyper-V Cluster menggunakan 2 perangkat Windows Server 2022 dan software StarWind VSAN FREE untuk solusi Storage Area Network
author: ["Ilham Wahayu Yanre"]
cover:
  image: cover.png
  hiddenInList: true
aliases: ["/infrastructure"]
tags: ["StarWind","Hyper-V"]
series: ["Infrastructure"]
ShowBreadCrumbs: true
ShowToc: true
tocopen: true
ShowCodeCopyButtons: true
---

## Introduction

>- High Availability (HA). Sistem yang dirancang untuk terus berjalan dengan downtime seminimal mungkin ketika terjadi blackout dengan mengandalkan redudansi pada komponen.
>- Hyperconverged (HCI). Infrastruktur IT yang menggabungkan komputasi, storage, dan network pada 1 perangkat seperti server.
>- StarWind VSAN FREE. Software-defined Solution (SDS) yang dapat menggantikan perangkat Storage Area Network (SAN) dengan cara me-mirror storage yang ada pada server menjadi Shared Storage. Lisensi FREE berlaku selama 1 tahun (bisa diperpanjang) dan bisa digunakan untuk production dengan limitasi tidak ada GUI untuk konfigurasi. Konfigurasi dapat dilakukan melalui Powershell.

Artikel ini mencakup:
- Mempersiapkan server dan topologi
- Konfigurasi StarWind VSAN FREE pada 2 node Windows Server 2022 untuk membuat Shared Storage
- Mengkonfigurasi Windows Server Failover Cluster (WSFC) untuk Hyper-V dan menggunakan Shared Storage yang dibuat melalui StarWind VSAN FREE

## Server dan Topologi

[![](http-1.png#center "Gambaran Topologi Windows Server Hyper-V Failover Cluster Menggunakan StarWind VSAN sebagai solusi Shared Storage")](http-1.png)

Cluster yang akan dibuat terdiri dari 2 server/node. Kedua node harus berada pada domain AD yang sama. Jadi dibutuhkan 1 server tambahan sebagai Domain Controller.

### Storage Preparation

Dengan pertimbangan redundancy, gunakan 2 SSD 256GB RAID1 untuk OS. Lalu gunakan 2 keping 5TB NVMe yang juga di-RAID1 dan partition style GPT untuk Storage Pool yang akan diproses oleh StarWind. *But in this economy? making RAID1 out of 2 pieces of 5TB NVMe? Really?*

### Network Interface

Masing-masing server harus memiliki minimal 3 NIC. 1 digunakan untuk Management, 1 untuk iSCSI/Heartbeat, dan 1 lagi untuk Sync/Replication. Link Replication digunakan untuk sinkronisasi dan replikasi data secara real-time, jadi dibutuhkan NIC dengan performa dan kapasitas tinggi (ex. SFP 10Gbps atau lebih). Link Heartbeat digunakan untuk komunikasi iSCSI antar node dan health monitoring.

### Domain Controller

| Name |
dc01 172.16.0.10

Install **Active Directory Domain Services** dan **DNS Server** melalui Server Manager, lalu konfigurasi server menjadi Domain Controller.

### Windows Server Node 1

| NIC | IP |
|:-|:-|
| NIC1 Management | 172.16.0.2/24 |
| NIC2 iSCSI/Heartbeat | 172.16.10.1/24 |
| NIC3 Sync/Replication | 172.16.20.1/24 |

>- Install **Hyper-V**, **Failover Clustering**, **Multipath I/O** melalui Server Manager. Join-kan server ke domain yang dibuat sebelumnya
>- Setelah ketiganya berhasil di-install dan server sudah join ke domain, buka **iSCSI Initiator** (bisa diketik pada search box pada sudut kiri bawah) dan pilih Yes jika ada pop up dengan keterangan iSCSI belum berjalan dan perlu di-set untuk auto start
>- Selanjutnya buka **MPIO** (bisa diketik pada search box pada sudut kiri bawah), lalu buka tab **Discover Multi-Paths**, dan centang **Add support for iSCSI devices**, lalu klik Add
>- Restart server


### Windows Server Node 2

| NIC | IP |
|:-|:-|
| NIC1 Management | 172.16.0.3/24 |
| NIC2 iSCSI/Heartbeat | 172.16.10.2/24 |
| NIC3 Sync/Replication | 172.16.20.2/24 |

>- Konfigurasi persis dengan Node 1. Yang berbeda hanya IP

### Jumbo Frame

Powershell script di bawah ini akan mengaktifkan jumbo frame dengan menaikkan MTU dari default 1500 ke 9000 atau 9014 (merujuk ke manual NIC yang digunakan untuk value maksimalnya) untuk NIC2 dan NIC3. Tujuannya adalah meningkatkan throughput pada NIC karena semakin besar kapasitas MTU dalam 1 packet, maka akan semakin efisien performa jaringannya.

Script di bawah ini mengasumsikan Adapter Name untuk NIC2 memuat karakter "iSCSI", dan NIC3 memuat karakter "Sync". 

```powershell
$iSCSIs = (Get-NetAdapter -Name "*iSCSI*").Name   # Sesuaikan dengan nama adapter
$Syncs = (Get-NetAdapter -Name "*Sync*").Name     # Sesuaikan dengan nama adapter
foreach ($iSCSI in $iSCSIs) {
Set-NetAdapterAdvancedProperty -Name “$iSCSI” -RegistryKeyword “*JumboPacket” -Registryvalue 9014
Get-NetAdapterAdvancedProperty -Name "$iSCSI" -RegistryKeyword “*JumboPacket”
}
foreach ($Sync in $Syncs) {
Set-NetAdapterAdvancedProperty -Name “$Sync” -RegistryKeyword “*JumboPacket” -Registryvalue 9014
Get-NetAdapterAdvancedProperty -Name "$Sync" -RegistryKeyword “*JumboPacket”
}
```

## Instalasi dan Konfigurasi StarWind VSAN FREE

### Download StarWind VSAN FREE

StarWind VSAN FREE bisa di-download dengan mendaftarkan email corporate melalui https://www.starwindsoftware.com/starwind-virtual-san-free#download. Download page tersebut tidak menerima email dengan domain *@gmail.com dan yang semisalnya. Jadi jika tidak memiliki email corporate, maka bisa dicoba berkirim email ke sales@starwind.com untuk meminta versi FREE nya.

Setelah itu StarWind akan mengirim email berisi link download, file lisensi, penjelasan tentang VSAN FREE, dan pilihan instalasi atau deploy-nya.
- Pilihan pertama adalah Controller Virtual Machine (CVM)
- Pilihan kedua adalah Windows native application (starwind-v8.exe). Inilah yang penulis pilih karena bisa langsung di-install pada host. Sedangkan CVM harus di-install sebagai VM dengan step-step yang lebih kompleks

[![](http-2.png#center "")](http-2.png)

### Install StarWind VSAN FREE

>- Copy dan install starwind-v8.exe dan file license ke node-1 dan node-2
>- Uncheck SMI-S Agent, karena tidak dibutuhkan di sini
>- Pilih Next sampai halaman license, dan pilih "Thank you, I do have a key already", lalu pilih file license
>- Next dan Install sampai selesai

### Membuat HA Device

[![](http-3.png#center "Pop up yang pertama kali muncul saat menjalankan StarWind VSAN FREE")](http-3.png)

>- Pilih Choose path... dan arahkan ke folder atau disk yang ingin dijadikan Storage Pool

[![](http-4.png#center "GUI StarWind VSAN FREE")](http-4.png)

Konfigurasi selanjutnya tidak dapat dilakukan melalui GUI karena batasan lisensi FREE. Sample Powershell script disediakan oleh StarWind pada directory
`C:\Program Files\StarWind Software\StarWind\StarWindX\Samples\powershell`.

[![](http-5.png#center "Sample konfigurasi melalui Powershell")](http-5.png)

Membuat HA Device menggunakan `CreateHA_2.ps1` hanya perlu dilakukan di node-1. Script ini akan otomatis mengkonfigurasi HA Device pada node-2 juga.

Selanjutnya copy folder powershell ke tempat lain untuk di-edit dan dijalankan. Lalu buka file `CreateHA_2.ps1` menggunakan Powershell ISE atau text editor lainnya.

[![](http-6.png#center "Windows Powershell ISE")](http-6.png)

```powershell
## file CreateHA_2.ps1 yang sudah disesuaikan untuk project

param($addr="172.16.0.2", $port=3261, $user="root", $password="starwind", # IP Address node-1 NIC1 Management
	$addr2="172.16.0.3", $port2=$port, $user2=$user, $password2=$password,  # IP Address node-2 NIC1 Management
#common
	$initMethod="Clear",
	$size=10000,  # Size Storage Pool dalam satuan MB
	$sectorSize=4096, # 4096 value yang direkomendasikan untuk Hyper-V
	$failover=0,  # 0 = Heartbeat Failover Strategy, 1 = Node Majority Failover Strategy
	$bmpType=1,
	$bmpStrategy=1,
    [Uint32][ValidateRange(0, 4294967294)]
	$maxUnmap=128,
#primary node
	$imagePath="My computer\D\StarWind",  # Directory folder dan disk yang ingin dijadikan Storage Pool pada node-1
	$imageName="masterImg22",
	[bool][ValidateSet($false, $true)]
	$createImage=$true,
	$storageName="",
	$targetAlias="targetha22",
	$poolName="pool1",
	$syncSessionCount=1,
	[bool][ValidateSet($false, $true)]
	$aluaOptimized=$true,
	$cacheMode="none",
	$cacheSize=0,
	$syncInterface="#p2=172.16.20.2:3260" -f $addr2,  # IP node-2 NIC3 - Replication
	$hbInterface="#p2=172.16.10.2:3260",  # IP node-2 NIC2 - Heartbeat
	[bool][ValidateSet($false, $true)]
	$createTarget=$true,
	$bmpFolderPath="",
#secondary node
	$imagePath2="My computer\D\StarWind", # Directory folder dan disk yang ingin dijadikan Storage Pool pada node-2
	$imageName2="partnerImg23",
	[bool][ValidateSet($false, $true)]
	$createImage2=$true,
	$storageName2="",
	$targetAlias2="partnerha23",
	$poolName2="pool1",
	$syncSessionCount2=1,
	[bool][ValidateSet($false, $true)]
	$aluaOptimized2=$false,
	$cacheMode2=$cacheMode,
	$cacheSize2=$cacheSize,
	$syncInterface2="#p1=172.16.20.1:3260" -f $addr,  # IP node-1 NIC3 - Replication
	$hbInterface2="#p1=172.16.10.1:3260", # IP node-1 NIC2 - Heartbeat
	[bool][ValidateSet($false, $true)]
	$createTarget2=$true,
	$bmpFolderPath2=""
	)
	
Import-Module StarWindX

try
{
	Enable-SWXLog

	$server = New-SWServer -host $addr -port $port -user $user -password $password

	$server.Connect()

	$firstNode = new-Object Node

	$firstNode.HostName = $addr
	$firstNode.HostPort = $port
	$firstNode.Login = $user
	$firstNode.Password = $password
	$firstNode.ImagePath = $imagePath
	$firstNode.ImageName = $imageName
	$firstNode.Size = $size
	$firstNode.CreateImage = $createImage
	$firstNode.StorageName = $storageName
	$firstNode.TargetAlias = $targetAlias
	$firstNode.SyncInterface = $syncInterface
	$firstNode.HBInterface = $hbInterface
	$firstNode.PoolName = $poolName
	$firstNode.SyncSessionCount = $syncSessionCount
	$firstNode.ALUAOptimized = $aluaOptimized
	$firstNode.CacheMode = $cacheMode
	$firstNode.CacheSize = $cacheSize
	$firstNode.FailoverStrategy = $failover
	$firstNode.CreateTarget = $createTarget
	$firstNode.BitmapStoreType = $bmpType
	$firstNode.BitmapStrategy = $bmpStrategy
	$firstNode.BitmapFolderPath = $bmpFolderPath
	$firstNode.MaxUnmap = $maxUnmap
    
	#
	# device sector size. Possible values: 512 or 4096(May be incompatible with some clients!) bytes. 
	#
	$firstNode.SectorSize = $sectorSize
    
	$secondNode = new-Object Node

	$secondNode.HostName = $addr2
	$secondNode.HostPort = $port2
	$secondNode.Login = $user2
	$secondNode.Password = $password2
	$secondNode.ImagePath = $imagePath2
	$secondNode.ImageName = $imageName2
	$secondNode.CreateImage = $createImage2
	$secondNode.StorageName = $storageName2
	$secondNode.TargetAlias = $targetAlias2
	$secondNode.SyncInterface = $syncInterface2
	$secondNode.HBInterface = $hbInterface2
	$secondNode.SyncSessionCount = $syncSessionCount2
	$secondNode.ALUAOptimized = $aluaOptimized2
	$secondNode.CacheMode = $cacheMode2
	$secondNode.CacheSize = $cacheSize2
	$secondNode.FailoverStrategy = $failover
	$secondNode.CreateTarget = $createTarget2
	$secondNode.BitmapFolderPath = $bmpFolderPath2
	$secondNode.MaxUnmap = $maxUnmap
        
	$device = Add-HADevice -server $server -firstNode $firstNode -secondNode $secondNode -initMethod $initMethod
	if(!$device) { return }
    
	while ($device.SyncStatus -ne [SwHaSyncStatus]::SW_HA_SYNC_STATUS_SYNC)
	{
		$syncPercent = $device.GetPropertyValue("ha_synch_percent")
	        Write-Host "Synchronizing: $($syncPercent)%" -foreground yellow

		Start-Sleep -m 2000

		$device.Refresh()
	}
}
catch
{
	Write-Host $_ -foreground red 
}
finally
{
	$server.Disconnect()
}

```

Save dan eksekusi file `CreateHA_2.ps1` yang sudah disesuaikan. Jika berhasil maka outputnya adalah `Synchronizing 0-99%`.
[![](http-7.png#center "Synchronizing")](http-7.png)

[![](http-8.png#center "Tampilan GUI StarWind setelah berhasil membuat HA Device")](http-8.png)

### Konfigurasi iSCSI

#### Node-1

##### Koneksi ke Loopback
1. Buka **iSCSI Initiator**
2. Pilih Tab **Discovery**
3. Pilih **Discover Portal**
4. Masukkan IP `127.0.0.1`
5. Pilih **Advanced**
6. Pilih **Local Adapter** Microsoft iSCSI Initiator
7. Pilih OK
8. Selanjutnya pilih Tab **Targets**. Target yang sudah dimasukkan akan muncul di bawah **Discovered Targets** dengan status Inactive
9. Pilih target tersebut lalu **Connect**
10. Centang **Enable multi-path**
11. Pilih **Advanced**
12. Pilih **Local Adapter** Microsoft iSCSI Initiator
13. Pilih **Target Portal IP** 127.0.0.1 / 3260
14. Pilih OK

##### Koneksi ke Node-2
1. Kembali ke Tab **Discovery**
2. Pilih **Discover Portal**
3. Masukkan IP node-2 `(172.16.10.2)`
4. Pilih **Advanced**
5. Pilih **Local Adapter** Microsoft iSCSI Initiator
6. Pilih **Initiator IP** `172.16.10.1`
7. Kembali ke Tab **Targets**. Nama dengan IP mgmt node-2 akan muncul pada **Discovered Targets**
8. Pilih target tersebut lalu **Connect**
9. Centang **Enable multi-path**
10. Pilih **Advanced**
11. Pilih **Local Adapter** Microsoft iSCSI Initiator
12. Pilih **Initiator IP** `172.16.10.1`
13. Pilih **Target Portal** `172.16.10.2`

#### Node-2
Ulangi langkah yang sama pada Node-2. Hanya IP-nya saja yang dibalikkan.

### Menghubungkan Disk ke Server

Langkah ini hanya perlu dilakukan di node-1.

1. Buka Disk Management. Storage yang dibuat oleh StarWind akan muncul dengan status offline
2. Jadikan online, dan konfigurasi Disk sebagai GPT, dan buat menjadi partisi NTFS

## Windows Server Failover Cluster (Hyper-V)

Di sinilah konfigurasi untuk menghubungkan Failover Cluster dengan Shared Storage yang telah dibuat melalui StarWind.
1. Buka **Failover Cluster Manager**
2. Pilih **Validate Configuration** pada bagian kanan
3. Pada halaman **Select Servers or a Cluster**, masukkan node-1 dan node-2
4. Jalankan validasi dengan mode all tests dan tunggu hingga selesai

>- Failover Cluster siap dikonfigurasi jika tidak ada fatal error pada hasil validasi

### Buat Failover Cluster

1. Pilih **Create Cluster...** pada bagian kanan
2. Masukkan node-1 dan node-2 pada halaman **Select Servers**
[![](http-9.png#center "")](http-9.png)
3. Buat nama cluster dan berikan IP Address
[![](http-10.png#center "")](http-10.png)
4. Jangan centang *Add all eligible storage to the cluster.*
[![](http-11.png#center "")](http-11.png)
5. Next sampai finish

### Menambahkan Disk ke Cluster Storage

1. Expand nama cluster pada bagian kiri
2. Pilih Storage
3. Pilih Disks
4. Pilih **Add Disk** pada bagian kanan
5. Pilih Disk yang dibuat melalui StarWind
[![](http-12.png#center "")](http-12.png)
6. Pilih OK

[![](http-13.png#center "Disk akan tampil sebagai Available Storage")](http-13.png)

7. Pilih **Add to Cluster Shared Volumes**

[![](http-14.png#center "Disk akan tampil sebagai Cluster Shared Volume")](http-14.png)

Shared Volume tersebut akan muncul pada node-1 dan node-2 dengan directory `C:\ClusterStorage\Volume1`. Replikasi bisa dites dengan membuat file atau folder pada directory tersebut pada node-1. File/folder yang dibuat akan muncul juga pada directory yang sama pada node-2. Begitupun sebaliknya.

### Membuat File Share Witness

Witness dibutuhkan untuk cluster dengan 2 node. Sangat dibutuhkan untuk menghindari skenario split-brain. Split-brain dapat terjadi ketika 1 node kehilangan komunikasi dengan node lainnya pada cluster. Kedua node yang sama-sama terisolasi kemudian berpikir bahwa node yang lain down sehingga keduanya "naik" sebagai "Master" dan tetap menerima data dan melakukan write pada disk. Akhirnya data pada disk menjadi tumpang tindih dan corrupt total. Akan tetapi dengan kehadiran witness, maka ketika 2 node saling putus komunikasi, lalu node pertama yang bisa berkomunikasi dengan Witness duluan akan naik menjadi "Master", sedangkan node lainnya akan dikunci dan dianggap offline. Sehingga data hanya akan diterima dan di-write oleh node pertama.

File Share Witness harus dibuat di server lain pada domain yang sama. Caranya cukup dengan membuat folder pada server tersebut, lalu konfigurasi file sharing folder tersebut dengan cluster dan berikan full access.

[![](http-15.png#center "Klik kanan nama cluster -> More Actions -> Configure Cluster Quorum Settings")](http-15.png)

Selanjutnya pilih **Select the quorum witness** -> **Configure a file share witness**. pada halaman **File Share Path**, arahkan ke server dan folder yang telah disiapkan.

[![](http-16.png#center "")](http-16.png)

Sampai tahap ini Failover Cluster sudah bisa digunakan.

## Failover Tests

Pengetesan bisa dilakukan dengan membuat VM melalui Failover Cluster Manager pada salah satu node. Lalu node tersebut dimatikan secara proper melalui shutdown, atau cabut kabel jaringan untuk melihat proses failover-nya.

Dari pengetesan yang penulis lakukan, ketika node yang menjadi host VM dimatikan melalui shutdown, maka VM pada node tersebut akan pindah ke node lain yang online melalui live migration, tanpa downtime. Live migration tidak bisa dilakukan jika node-1 dan node-2 menggunakan CPU yang berbeda generasi atau vendor, misal node-1 menggunakan Intel dan node-2 menggunakan AMD.

[![](http-17.png#center "Cluster Manager pada node-1. Pada skenario ini VM sedang berada pada node-2. Kemudian node-2 di-shutdown. Status VM langsung berubah menjadi Live Migrating ke node-1")](http-17.png)

Selanjutnya penulis mencoba menyimulasikan node yang menjadi host VM mengalami power cut dan atau kabel jaringan terputus. Live migration tidak terjadi pada skenario ini. Akan tetapi cluster mengkategorikan node tersebut sebagai "isolated", dan status VM yang berada pada node tersebut berubah menjadi "unmonitored" selama kurang lebih 5 menit.

[![](http-18.png#center "")](http-18.png)
[![](http-19.png#center "")](http-19.png)
[![](http-20.png#center "")](http-20.png)

Apabila setelah 5 menitan status node masih "isolated", maka node tersebut akan ditendang dari cluster, dan VM pada node tersebut akan di-restart pada node yang masih online. Setelah penulis telusuri, ternyata mekanisme isolation tersebut dibuat oleh Microsoft untuk menjaga kestabilan cluster. Apabila node terputus dari cluster dan tidak diketahui apakah node tersebut down atau masih up, atau hanya kabel jaringannya yang lepas, maka node akan diisolasi selama kurang lebih 5 menit. Apabila dalam kurun waktu 5 menit node kembali online atau recovered, maka cluster akan mengembalikan status node menjadi online. Akan tetapi jika 5 menit berlalu dan node masih terisolasi, maka node tersebut akan dikeluarkan dari cluster.

*Sekian*

*Source: https://www.starwindsoftware.com/resource-library/starwind-virtual-san-for-hyper-v-2-node-hyperconverged-scenario-with-windows-server-2016/*