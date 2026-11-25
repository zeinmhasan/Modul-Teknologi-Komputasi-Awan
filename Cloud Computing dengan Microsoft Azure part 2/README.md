# Modul 7 - PaaS dengan Azure Container Apps: Deploy Tanpa Server, HTTPS Otomatis, Revisions & Scale to Zero

## Daftar Isi

1. [Pendahuluan](#1-pendahuluan)
2. [Dari IaaS ke PaaS](#2-dari-iaas-ke-paas)
3. [Mengenal Azure Container Apps](#3-mengenal-azure-container-apps)
4. [Gambaran Praktikum](#4-gambaran-praktikum)
5. [Persiapan](#5-persiapan)
6. [Hands-on 1: Deploy Container App Pertama](#6-hands-on-1-deploy-container-app-pertama)
7. [Hands-on 2: Konfigurasi & Observasi](#7-hands-on-2-konfigurasi--observasi)
8. [Hands-on 3: Revisions, Rollback & Traffic Splitting](#8-hands-on-3-revisions-rollback--traffic-splitting)
9. [Hands-on 4: Scaling & Scale to Zero](#9-hands-on-4-scaling--scale-to-zero)
10. [Hands-on 5: Kerja Tim dengan Azure RBAC](#10-hands-on-5-kerja-tim-dengan-azure-rbac)
11. [Bonus: Semua Lewat Azure CLI](#11-bonus-semua-lewat-azure-cli)
12. [IaaS vs PaaS: Kapan Pakai yang Mana?](#12-iaas-vs-paas-kapan-pakai-yang-mana)
13. [Bersih-bersih Biaya](#13-bersih-bersih-biaya)
14. [Troubleshooting](#14-troubleshooting)

---

## 1. Pendahuluan

Di Modul 6 kita sudah berhasil men-deploy dashboard ke VM di Azure. Hebat! Tapi coba ingat lagi apa saja yang harus kita kerjakan **sendiri**:

1. Membuat VM, memilih ukuran, mengatur disk, dan auto-shutdown
2. Membuat SSH key, mengatur `~/.ssh/config`, dan memeriksa keamanan SSH
3. Meng-install Docker di VM
4. Membuka port di firewall (NSG)
5. Setiap update: `ssh` → `docker rm -f` → `docker run` (dan dashboard **mati sebentar** di antaranya)
6. Dashboard dibuka lewat `http://20.xx.xx.xx:8501`, **tanpa HTTPS** dan tanpa nama domain
7. VM **ditagih per jam** selama menyala, walaupun tidak ada satu pun yang membuka dashboard

Padahal yang sebenarnya kita inginkan cuma satu: **"tolong jalankan image Docker ini, dan berikan saya URL-nya."**

Itulah yang ditawarkan oleh **PaaS (Platform as a Service)**. Di modul ini kita memakai **Azure Container Apps**, layanan untuk menjalankan container **tanpa mengurus server**. Kita akan men-deploy **image yang sama persis** dengan Modul 6 (`dashboard-kopi:v1` dan `v2`), lalu membandingkan sendiri bedanya.

Di akhir modul, kita akan bisa:

- Menjelaskan perbedaan IaaS dan PaaS berdasarkan pengalaman langsung
- Men-deploy image Docker ke **Azure Container Apps** dengan **HTTPS otomatis**
- Mengubah konfigurasi tanpa build ulang image, membaca log, dan masuk ke console container
- Melakukan update tanpa downtime, **rollback**, dan **traffic splitting** (canary) dengan *revisions*
- Mengatur **autoscaling**, termasuk **scale to zero** supaya tidak ada biaya saat sepi
- Memberi akses tim dengan **Azure RBAC**, padanan "satu orang satu akun" dari Modul 6

> 💡 Tampilan Azure Portal cukup sering berubah. Kalau nama menu atau tombol sedikit berbeda dari modul ini, cari menu dengan nama yang mirip. Konsepnya tetap sama.

---

## 2. Dari IaaS ke PaaS

### 2.1 Kembali ke Analogi Pizza

Ingat tabel pizza di Modul 6?

| Model | Analogi Pizza | Modul |
| :---- | :------------ | :---- |
| **IaaS** | Sewa dapur lengkap, masak sendiri | Modul 6: Virtual Machine |
| **PaaS** | Pesan pizza, makan di rumah | **Modul 7: Container Apps** ← kita di sini |
| **SaaS** | Makan di restoran | Microsoft 365, Gmail |

Di Modul 6 kita "masak sendiri". Sekarang kita cukup **menyerahkan resep (image Docker)**, dan Azure yang memasak, menyajikan, bahkan menambah koki kalau pesanan sedang ramai.

### 2.2 Siapa Mengurus Apa?

| Pekerjaan | Modul 6 (VM / IaaS) | Modul 7 (Container Apps / PaaS) |
| :-------- | :------------------ | :------------------------------ |
| Memilih ukuran server & disk | 🧑 Kita | ☁️ Azure (kita cukup pilih CPU/RAM container) |
| Update OS & patch keamanan | 🧑 Kita | ☁️ Azure |
| Install Docker | 🧑 Kita | ☁️ Azure |
| SSH key & user Linux | 🧑 Kita | ☁️ Tidak perlu SSH sama sekali |
| Buka port di firewall (NSG) | 🧑 Kita | ☁️ Cukup centang *Ingress* |
| Sertifikat HTTPS | 🧑 Kita (belum kita kerjakan) | ☁️ **Otomatis** |
| Container mati → hidupkan lagi | 🧑 `--restart unless-stopped` | ☁️ Otomatis |
| Update tanpa downtime | 🧑 Susah (ada jeda `rm` → `run`) | ☁️ **Otomatis** |
| Rollback | 🧑 Manual lewat SSH | ☁️ Pindahkan traffic, beberapa klik |
| Tambah server saat ramai | 🧑 Buat VM baru + load balancer | ☁️ **Autoscaling** |
| Biaya saat tidak ada pengunjung | 💸 Tetap ditagih per jam | ✅ Bisa **nol** (scale to zero) |
| Kode aplikasi & image | 🧑 Kita | 🧑 Kita |

Yang tersisa untuk kita urus hanyalah **kode dan image**. Itulah inti PaaS.

> ⚠️ PaaS bukan berarti selalu lebih baik. Ada hal yang **tidak bisa** kita lakukan di PaaS, misalnya mengakses OS secara penuh. Kita bahas di [bagian 12](#12-iaas-vs-paas-kapan-pakai-yang-mana).

---

## 3. Mengenal Azure Container Apps

### 3.1 Apa itu Azure Container Apps?

**Azure Container Apps (ACA)** adalah layanan *serverless container*: kita memberikan image Docker, Azure yang menjalankannya. Di balik layar, ACA berjalan di atas **Kubernetes** dan **KEDA** (autoscaler), tetapi kita tidak perlu menyentuh Kubernetes sama sekali.

Padanannya di cloud lain:

| Provider | Layanan serupa |
| :------- | :------------- |
| **Azure** | Container Apps |
| **AWS** | App Runner / ECS Fargate |
| **GCP** | Cloud Run |

Kalau suatu saat kamu memakai Cloud Run atau App Runner, hampir semua konsep di modul ini tetap berlaku.

### 3.2 Istilah-istilah Penting

Supaya mudah diingat, kita pakai analogi yang nyambung dengan dashboard kita: **kedai kopi di dalam mal**.

```
Subscription "Azure for Students"
└── Resource Group "rg-modul7"
    ├── Container Apps Environment "cae-modul7"   ← MAL: gedung, jaringan & listrik bersama
    │   └── Container App "andi-dashboard"        ← KEDAI KOPI kita di dalam mal
    │       ├── Revision  andi-dashboard--v1       ← EDISI MENU v1 (tidak bisa diubah)
    │       │   └── Replica (0..n)                 ← BARISTA yang sedang bertugas
    │       └── Revision  andi-dashboard--v2       ← EDISI MENU v2
    └── Log Analytics workspace                    ← BUKU CATATAN harian mal
```

| Istilah | Penjelasan | Analogi Kedai Kopi |
| :------ | :--------- | :----------------- |
| **Environment** | "Wadah" bersama untuk satu atau beberapa Container App: jaringan, log, dan batas keamanan | **Mal**: banyak kedai berbagi gedung, listrik, dan satpam |
| **Container App** | Aplikasi kita. Punya URL sendiri | **Kedai kopi** kita |
| **Revision** | *Snapshot* versi aplikasi (image + konfigurasi). Sekali dibuat **tidak bisa diubah** (*immutable*). Setiap perubahan = revision baru | **Edisi menu**. Mau ganti menu? Cetak edisi baru, jangan coret edisi lama |
| **Replica** | Container yang **benar-benar sedang berjalan** untuk suatu revision. Bisa 0, 1, atau banyak | **Barista** yang sedang bertugas |
| **Ingress** | Pintu masuk dari internet ke aplikasi, lengkap dengan HTTPS | **Pintu kedai + satpam** |
| **Scaling** | Menambah/mengurangi replica secara otomatis | Menambah barista saat **antrean panjang**, memulangkan saat sepi |
| **Scale to zero** | Replica = 0 saat tidak ada request | Kedai **tutup** saat sepi, tapi langsung buka begitu ada pelanggan datang |
| **Cold start** | Jeda waktu saat replica pertama dinyalakan dari 0 | Waktu menunggu barista datang dan menyalakan mesin kopi |
| **Traffic splitting** | Membagi pengunjung ke beberapa revision (misal 80% / 20%) | Sebagian pelanggan diberi **menu baru untuk dicoba** dulu |

### 3.3 Revision: Kunci dari Update dan Rollback

Di Modul 6, "versi" hanya ada di **tag image** (`v1`, `v2`). Di Container Apps, versi juga dicatat sebagai **revision**, dan revision mencakup **image + konfigurasi** (environment variable, CPU/RAM, aturan scaling).

Tidak semua perubahan membuat revision baru:

| Jenis perubahan | Contoh | Membuat revision baru? |
| :-------------- | :----- | :--------------------- |
| **Revision-scope** | Ganti image/tag, ubah environment variable, ubah CPU/RAM, ubah aturan scaling | ✅ Ya |
| **Application-scope** | Ubah ingress, secrets, registry, revision mode | ❌ Tidak. Berlaku untuk semua revision |

### 3.4 Bagaimana Container Apps Ditagih?

Kita memakai **Consumption plan**, yaitu bayar sesuai pemakaian per detik. Yang ditagih:

- **vCPU-detik** dan **GiB-detik**: berapa CPU dan RAM yang dialokasikan, dikali berapa detik replica berjalan
- **Jumlah request** HTTP

Kabar baiknya, setiap subscription mendapat **kuota gratis setiap bulan**:

| Meter | Gratis per bulan |
| :---- | :--------------- |
| vCPU-detik | **180.000** |
| GiB-detik (memori) | **360.000** |
| Request | **2 juta** |

**Latihan hitung:** di modul ini kita akan memakai container berukuran **0,5 vCPU + 1 GiB RAM**.

```
vCPU   : 180.000 detik ÷ 0,5 vCPU = 360.000 detik = 100 jam
Memori : 360.000 detik ÷ 1 GiB    = 360.000 detik = 100 jam
```

Artinya, **satu replica bisa aktif ±100 jam per bulan secara gratis**. Bandingkan dengan VM yang menyala 24 jam × 30 hari = 720 jam ditagih penuh. Dengan **scale to zero**, replica hanya menyala saat ada yang membuka dashboard.

> 💡 Ada juga tarif **idle** (lebih murah) untuk replica yang menyala tapi tidak sedang bekerja. Ini berlaku kalau kita memaksa minimal 1 replica selalu hidup. Kita bahas di [bagian 9](#9-hands-on-4-scaling--scale-to-zero).
>
> ⚠️ Angka kuota gratis bisa berubah. Selalu cek halaman resminya: <https://azure.microsoft.com/pricing/details/container-apps/>

---

## 4. Gambaran Praktikum

### 4.1 Tools yang Digunakan

| Tool | Fungsi |
| :--- | :----- |
| **Azure Portal** | Membuat dan mengelola Container App |
| **Azure Container Apps** | Menjalankan container tanpa server |
| **Docker Hub** | Registry tempat image `dashboard-kopi:v1` dan `v2` dari Modul 6 |
| **Log Analytics** | Menyimpan log aplikasi |
| **Python** (di laptop) | Menjalankan skrip uji beban sederhana untuk melihat autoscaling |
| **ARM template** | File JSON untuk membuat Container Apps Environment jenis Standard |
| **Azure CLI** *(bonus)* | Mengelola Azure lewat perintah dari terminal laptop |

### 4.2 Arsitektur Sistem

```
 LAPTOP KAMU                              AZURE (Resource Group: rg-modul7)
 ┌─────────────────────┐                 ┌──────────────────────────────────────────────────┐
 │ image v1 & v2       │                 │ Container Apps Environment  (cae-modul7)         │
 │ (sudah dibuat di    │                 │ ┌──────────────────────────────────────────────┐ │
 │  Modul 6)           │                 │ │ Container App: andi-dashboard                │ │
 └─────────┬───────────┘                 │ │                                              │ │
           │ docker push (sudah)         │ │   Ingress (HTTPS otomatis)                   │ │
           ▼                             │ │        │ 80%                │ 20%             │ │
     ┌────────────┐   pull otomatis      │ │        ▼                    ▼                 │ │
     │ Docker Hub │ ─────────────────────┼►│   revision v1          revision v2           │ │
     └────────────┘                      │ │   [replica]            [replica][replica]    │ │
                                         │ └──────────────────────────────────────────────┘ │
                                         │ Log Analytics workspace (log)                    │
                                         └──────────────────────────────────────────────────┘
                                                       ▲
 BROWSER SIAPA SAJA ── https://andi-dashboard.<acak>.<region>.azurecontainerapps.io
```

Alurnya:

1. Kita membuat **Container Apps Environment** dan **Container App** yang menarik image `v1` dari Docker Hub.
2. Azure otomatis memberi **URL HTTPS**. Tidak ada SSH, tidak ada install Docker, tidak ada buka port.
3. Kita merilis `v2` sebagai **revision baru**, lalu bermain dengan **rollback** dan **traffic splitting**.
4. Kita mengatur **autoscaling** dan melihat replica bertambah saat diserbu request, lalu turun ke **nol** saat sepi.
5. Kita memberi akses ke teman lewat **Azure RBAC**.

> 💡 Praktikum ini tetap dikerjakan **berpasangan**, seperti Modul 6. Setiap orang membuat Container App di subscription masing-masing, lalu di Hands-on 5 kalian saling memberi akses.

---

## 5. Persiapan

### 5.1 Cek Kredit, Budget & Sisa Resource Modul 6

1. Buka **Education Hub** di portal dan cek sisa kredit.
2. Buka **Cost Management → Budgets**. Budget `budget-modul6` dari Modul 6 berlaku untuk seluruh subscription, jadi tetap melindungi kita di modul ini.
3. Buka **Resource groups**. Kalau `rg-modul6` masih ada dan sudah tidak dipakai, **hapus** dulu. Kalau masih dibutuhkan, minimal pastikan VM berstatus **Stopped (deallocated)**.

### 5.2 Cek Image di Docker Hub

Buka <https://hub.docker.com>, lalu buka repository `dashboard-kopi`. Pastikan:

- Tag **`v1`** dan **`v2`** ada
- Repository bersifat **Public** (default untuk akun gratis)
- Di tab *Tags*, arsitektur (OS/ARCH) tertulis **`linux/amd64`**

> ⚠️ Container Apps hanya menjalankan image **`linux/amd64`**. Kalau kamu memakai Mac M-series dan dulu lupa `--platform linux/amd64`, build ulang dulu:
>
> ```powershell
> docker build --platform linux/amd64 -t <username-dockerhub>/dashboard-kopi:v1 .
> docker push <username-dockerhub>/dashboard-kopi:v1
> ```

### 5.3 Daftarkan Resource Provider

Setiap layanan Azure punya **resource provider**. Di subscription baru, provider untuk Container Apps kadang belum aktif, dan pembuatan resource akan gagal dengan error `MissingSubscriptionRegistration`.

1. Di kolom pencarian portal, ketik **Subscriptions**, lalu buka **Azure for Students**.
2. Di menu kiri, pilih **Settings → Resource providers**.
3. Cari **`Microsoft.App`**. Kalau statusnya *NotRegistered*, pilih lalu klik **Register**.
4. Ulangi untuk **`Microsoft.OperationalInsights`** (untuk Log Analytics).
5. Tunggu sampai status berubah menjadi **Registered** (bisa beberapa menit, klik *Refresh*).

<!-- 📸 SCREENSHOT: halaman Resource providers dengan Microsoft.App berstatus Registered -->

### 5.4 Region

Gunakan **region yang sama** dengan yang sudah kamu catat di Modul 6 (bagian 5.2, daftar *Allowed resource deployment regions*). Region terdekat seperti *Southeast Asia* atau *East Asia* sudah mendukung Container Apps.

> 💡 Kalau region pilihanmu tidak muncul di daftar saat membuat Container App, pilih region lain yang **diizinkan** oleh policy subscription-mu.

### 5.5 Tools di Laptop

1. **Browser** (sebaiknya siapkan juga jendela **Incognito/Private** untuk Hands-on 3)
2. **Python 3** untuk skrip uji beban. Cek dengan:

   ```powershell
   python --version
   ```

   Skrip yang kita pakai hanya memakai library bawaan Python, jadi tidak perlu `pip install` apa pun.

---

## 6. Hands-on 1: Deploy Container App Pertama

### 6.1 Membuat Resource Group (dengan Tags)

1. Di kolom pencarian portal, ketik **Resource groups**, lalu klik **+ Create**.
2. Tab **Basics**:
   - **Subscription**: Azure for Students
   - **Resource group**: `rg-modul7`
   - **Region**: region yang sudah dicatat
3. Tab **Tags**, tambahkan:

   | Name | Value |
   | :--- | :---- |
   | `pemilik` | `andi` (namamu) |
   | `modul` | `7` |

4. Klik **Review + create → Create**.

> 💡 **Tags** adalah label bebas untuk resource. Di dunia kerja, tags dipakai untuk menjawab pertanyaan seperti *"resource ini punya siapa?"* dan *"biaya bulan ini paling banyak dari proyek apa?"*. Di **Cost Management**, biaya bisa difilter berdasarkan tag. Ini adalah padanan **MOTD** dari Modul 6: informasi pemilik yang menempel langsung di resource-nya.

### 6.2 Membuat Container Apps Environment (Standard)

Azure sekarang punya **dua jenis** Container Apps Environment:

| Jenis | Kelebihan | Kekurangan untuk praktikum ini |
| :---- | :-------- | :----------------------------- |
| **Standard** | Fitur lengkap: revision suffix, multiple revision, traffic splitting, label, custom scale rule, metrics | Butuh beberapa menit untuk dibuat |
| **Express** (preview) | Dibuat dalam hitungan detik, cold start sangat cepat | Hanya **single revision**, **tidak bisa** memakai revision suffix maupun traffic splitting, dan custom scale rule masih dalam pengembangan |

Hands-on 3 dan 4 **membutuhkan environment Standard**. Supaya pasti mendapat jenis yang benar, kita membuat environment dengan **template ARM**: sebuah file JSON yang menjelaskan resource apa saja yang ingin dibuat. Cukup lewat portal, tanpa install apa pun.

> 💡 Ini juga perkenalan pertama kita dengan **Infrastructure as Code (IaC)**: infrastruktur ditulis sebagai kode, sehingga bisa disimpan, dibagikan, dan dijalankan ulang dengan hasil yang sama persis.

1. Di kolom pencarian portal, ketik **Deploy a custom template**, lalu buka.
2. Klik **Build your own template in the editor**.
3. Hapus isi editor, lalu tempel template berikut:

   ```json
   {
     "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
     "contentVersion": "1.0.0.0",
     "parameters": {
       "location": {
         "type": "string",
         "defaultValue": "[resourceGroup().location]"
       }
     },
     "resources": [
       {
         "type": "Microsoft.OperationalInsights/workspaces",
         "apiVersion": "2022-10-01",
         "name": "log-modul7",
         "location": "[parameters('location')]",
         "properties": {
           "sku": { "name": "PerGB2018" },
           "retentionInDays": 30
         }
       },
       {
         "type": "Microsoft.App/managedEnvironments",
         "apiVersion": "2026-07-01",
         "name": "cae-modul7",
         "location": "[parameters('location')]",
         "dependsOn": [
           "[resourceId('Microsoft.OperationalInsights/workspaces', 'log-modul7')]"
         ],
         "properties": {
           "environmentMode": "WorkloadProfiles",
           "appLogsConfiguration": {
             "destination": "log-analytics",
             "logAnalyticsConfiguration": {
               "customerId": "[reference(resourceId('Microsoft.OperationalInsights/workspaces', 'log-modul7'), '2022-10-01').customerId]",
               "sharedKey": "[listKeys(resourceId('Microsoft.OperationalInsights/workspaces', 'log-modul7'), '2022-10-01').primarySharedKey]"
             }
           },
           "workloadProfiles": [
             { "name": "Consumption", "workloadProfileType": "Consumption" }
           ]
         }
       }
     ]
   }
   ```

   Template ini membuat dua resource: **Log Analytics workspace** `log-modul7` dan **Container Apps Environment** `cae-modul7` (jenis **Standard**, hanya dengan profile **Consumption**). Lokasinya otomatis mengikuti region resource group.

   Baris terpenting adalah `"environmentMode": "WorkloadProfiles"`. Kalau baris ini tidak ditulis, di beberapa region (misalnya *East Asia*) Azure akan membuat environment **Express** secara default.

4. Klik **Save**.
5. Di halaman **Basics**:
   - **Subscription**: Azure for Students
   - **Resource group**: `rg-modul7`
   - **Region** dan **Location**: biarkan (mengikuti resource group)
6. Klik **Review + create → Create**. Tunggu ±3–5 menit sampai muncul *Your deployment is complete*.
7. Buka **rg-modul7**. Sekarang ada `cae-modul7` dan `log-modul7`.
8. **Verifikasi jenis environment:** buka `cae-modul7`, lalu di halaman **Overview** klik **JSON View** (pojok kanan atas bagian *Essentials*). Di dropdown **API version**, pilih versi **paling baru** (paling atas), karena versi lama tidak menampilkan jenis environment. Tekan `Ctrl + F` dan cari **`environmentMode`**. Nilainya harus **`"WorkloadProfiles"`** (Standard) ✅. Kalau nilainya **`"Express"`**, environment tersebut tidak bisa dipakai untuk Hands-on 3 dan 4.

> ⚠️ Kalau sebelumnya sudah ada environment **Express** bernama `cae-modul7` di resource group yang sama, template **tidak** akan mengubahnya menjadi Standard. Hapus dulu Container App dan environment Express tersebut (atau hapus seluruh `rg-modul7` lalu buat ulang), baru deploy template.

> ⚠️ Jangan menambahkan workload profile **Dedicated** ke environment ini. Profile Dedicated ditagih per jam seperti VM, walaupun tidak ada aplikasi yang berjalan. Untuk praktikum ini **cukup Consumption**.

<details>
<summary><b>Alternatif: lewat Azure CLI</b></summary>

Kalau Azure CLI sudah terpasang di laptop ([bagian 11](#11-bonus-semua-lewat-azure-cli)), environment Standard juga bisa dibuat dengan:

```bash
az containerapp env create -n cae-modul7 -g rg-modul7 -l southeastasia
```

Ganti `southeastasia` dengan region milikmu (format huruf kecil tanpa spasi). Setelah itu tetap **verifikasi** `environmentMode` lewat JSON View seperti langkah 8.

</details>

### 6.3 Membuat Container App

1. Di kolom pencarian portal, ketik **Container Apps**, lalu klik **+ Create → Container App**.

2. Tab **Basics**:

   | Field | Isi |
   | :---- | :-- |
   | Subscription | Azure for Students |
   | Resource group | `rg-modul7` |
   | Container app name | `andi-dashboard` (prefix namamu, huruf kecil) |
   | Deployment source | **Container image** |
   | Region | **Sama** dengan region environment |
   | Container Apps environment | Pilih **`cae-modul7`** yang sudah dibuat di 6.2. **Jangan** klik *Create new* |

   > ⚠️ Kalau `cae-modul7` tidak muncul di daftar, pastikan region yang dipilih sama dengan region resource group. **Jangan** biarkan portal membuat environment baru secara otomatis, karena environment buatan portal bisa berjenis **Express**.

3. Tab **Container**:

   | Field | Isi |
   | :---- | :-- |
   | Use quickstart image | ❌ **Jangan dicentang** |
   | Name | `dashboard` |
   | Image source | **Docker Hub or other registries** |
   | Image type | **Public** |
   | Registry login server | `docker.io` |
   | Image and tag | `<username-dockerhub>/dashboard-kopi:v1` |
   | CPU and Memory | **0.5 CPU cores, 1 Gi memory** |
   | Environment variables | Name: `PEMILIK`, Source: *Manual entry*, Value: `Andi` |

   Perhatikan bahwa `-e PEMILIK=Andi` dari Modul 6 sekarang diisi lewat formulir. Konsepnya sama persis.

4. Tab **Ingress**:

   | Field | Isi |
   | :---- | :-- |
   | Ingress | ✅ **Enabled** |
   | Ingress traffic | **Accepting traffic from anywhere** |
   | Ingress type | **HTTP** |
   | Insecure connections | ❌ Jangan dicentang (HTTP otomatis dialihkan ke HTTPS) |
   | Target port | **`8501`** |

   **Target port** adalah port **di dalam container**, sama seperti angka kanan di `-p 8501:8501`. Bedanya, di sini kita **tidak perlu** memilih port di sisi "server", karena Azure yang mengurus pintu masuknya (port 443/HTTPS).

5. Klik **Review + create**, tunggu validasi lolos, lalu klik **Create**.

Deployment butuh **±1–2 menit**. Setelah selesai, klik **Go to resource**.

### 6.4 Buka Dashboard Lewat HTTPS

Di halaman **Overview**, cari **Application Url**. Bentuknya kira-kira seperti ini:

```
https://andi-dashboard.<kata-acak>.<region>.azurecontainerapps.io
```

Klik URL tersebut. 🎉 Dashboard kopi kita sudah online, **lengkap dengan HTTPS (gembok di address bar)**, tanpa satu pun perintah SSH!

<!-- 📸 SCREENSHOT: halaman Overview Container App (Application Url, status Running, environment) -->

<!-- 📸 SCREENSHOT: dashboard v1 terbuka lewat https://...azurecontainerapps.io (gembok terlihat) -->

Coba juga buka dengan `http://` (tanpa s). Browser akan otomatis dialihkan ke `https://`.

> 💡 **Refleksi:** hitung berapa langkah yang kita kerjakan di Modul 6 dari nol sampai dashboard online (buat VM, SSH key, config, install Docker, usermod, NSG, docker run). Bandingkan dengan langkah di bagian ini. Itulah yang dimaksud dengan "Azure yang mengurus platform-nya".

---

## 7. Hands-on 2: Konfigurasi & Observasi

Tidak ada SSH bukan berarti kita "buta". Container Apps menyediakan cara lain untuk mengintip apa yang terjadi di dalam aplikasi.

### 7.1 Mengubah Environment Variable (Tanpa Build Ulang)

Di Modul 6, untuk mengganti `PEMILIK` kita harus `docker rm -f` lalu `docker run` lagi. Sekarang:

1. Di menu kiri Container App, buka **App → Revisions and replicas**, lalu klik **+ Create new revision** di toolbar atas.
2. Di halaman pembuatan revision, isi **Name / Suffix**: `v1b`.
3. Di tabel **Container image**, klik container `dashboard`, lalu buka bagian/tab **Environment variables**.
4. Ubah nilai `PEMILIK` menjadi, misalnya, `Andi - Tim Data Kopi`.
5. Klik **Save**, lalu **Create**.

> 💡 **Cara cepat (tanpa suffix):** buka **App → Containers**, pilih tab **Environment variables**, ubah nilainya, lalu klik **Save as a new revision** di bagian bawah. Hasilnya sama, tetapi nama revision dibuat otomatis oleh Azure (berupa kode acak, misalnya `andi-dashboard--0000002`). Di modul ini kita memakai **Create new revision** supaya nama revision mudah dikenali.

Tunggu ±1 menit, lalu refresh dashboard. Caption sekarang berubah, padahal **image tidak di-build ulang**.

Sekarang kembali ke **App → Revisions and replicas**:

| Tab | Revision | Status | Traffic |
| :-- | :------- | :----- | :------ |
| **Active revisions** | `andi-dashboard--v1b` | Running | 100% |
| **Inactive revisions** | `andi-dashboard--v1` | Inactive | 0% |

Karena mengubah environment variable termasuk perubahan **revision-scope**, Azure membuat **revision baru** dan memindahkan traffic ke sana. Revision lama otomatis dinonaktifkan **setelah** revision baru siap, jadi **tidak ada downtime**.

> 💡 **Bagaimana dengan password atau API key?** Jangan ditulis sebagai environment variable biasa. Simpan di menu **Settings → Secrets**, lalu di environment variable pilih *Source*: **Reference a secret**. Nilainya tidak akan terlihat di portal, dan secret bisa diganti tanpa mengubah image.

### 7.2 Membaca Log (Pengganti `docker logs`)

1. Di menu kiri, pilih **Monitoring → Log stream**.
2. Pilih revision dan replica yang aktif.
3. Buka dashboard di tab lain, lalu klik-klik filter di sidebar.

Kita akan melihat log Streamlit secara langsung, sama seperti `docker logs -f andi-dashboard` di Modul 6.

<!-- 📸 SCREENSHOT: Log stream menampilkan log Streamlit -->

### 7.3 Masuk ke Dalam Container (Pengganti `ssh` + `docker exec`)

1. Di menu kiri, pilih **Monitoring → Console**.
2. Pilih revision, replica, dan container `dashboard`.
3. Pilih startup command **`/bin/sh`**, lalu **Connect**.

Coba beberapa perintah:

```sh
hostname              # nama replica (bukan nama VM!)
ls                    # isi folder /app: app.py, data_kopi.csv, ...
env | grep PEMILIK    # cek environment variable
cat /etc/os-release   # OS di dalam container (Debian, dari python:3.12-slim)
```

> ⚠️ Console hanya bisa dipakai kalau ada **replica yang sedang berjalan**. Kalau aplikasi sedang *scale to zero* (replica = 0), buka dulu URL dashboard supaya replica menyala.
>
> 💡 Perubahan apa pun yang kita lakukan di console (membuat file, menghapus file) **akan hilang** ketika replica dimatikan. Container itu **sekali pakai** (*ephemeral*). Data yang harus disimpan permanen harus diletakkan di luar container, misalnya di database atau Blob Storage.

### 7.4 Melihat Metrik

1. Di menu kiri, pilih **Monitoring → Metrics**.
2. Pada **Metric**, pilih **Requests**. Tambahkan metric kedua: **Replica Count**.
3. Atur rentang waktu ke **Last 30 minutes**.

Grafik ini akan sangat berguna di Hands-on 4.

---

## 8. Hands-on 3: Revisions, Rollback & Traffic Splitting

### 8.1 Revision Mode

Container Apps punya beberapa **revision mode**:

| Mode | Perilaku | Cocok untuk |
| :--- | :------- | :---------- |
| **Single** (default) | Hanya **satu** revision aktif. Revision baru otomatis menggantikan yang lama setelah siap | Aplikasi sederhana, update otomatis |
| **Multiple** | **Beberapa** revision bisa aktif bersamaan, dan kita yang mengatur pembagian traffic | Rollback cepat, canary, A/B testing, blue-green |
| **Labels** | Traffic diatur lewat **label** (misal `staging`, `prod`) | Alur rilis yang lebih kompleks |

Di bagian ini kita memakai mode **Multiple**.

### 8.2 Mengubah ke Multiple Revision Mode

1. Buka **App → Revisions and replicas**.
2. Klik **Deployment mode** di toolbar atas, pilih **Multiple**, lalu **Apply**.

Mengubah revision mode termasuk perubahan **application-scope**, jadi tidak membuat revision baru.

### 8.3 Merilis v2 sebagai Revision Baru

1. Masih di **Revisions and replicas**, klik **+ Create new revision**.
2. **Name / Suffix**: `v2`
3. Klik container `dashboard`, lalu ubah **Image and tag** menjadi `<username-dockerhub>/dashboard-kopi:v2`. Biarkan environment variable apa adanya.
4. Klik **Save**, lalu **Create**.

Kembali ke **Revisions and replicas**. Sekarang ada **dua revision aktif** sekaligus:

| Revision | Image | Traffic |
| :------- | :---- | :------ |
| `andi-dashboard--v1b` | `dashboard-kopi:v1` | ? |
| `andi-dashboard--v2` | `dashboard-kopi:v2` | ? |

Atur kolom **Traffic** secara eksplisit:

- `andi-dashboard--v1b`: **0**
- `andi-dashboard--v2`: **100**

Klik **Save**. Refresh dashboard: caption menunjukkan **Versi v2**, dan sidebar filter muncul.

<!-- 📸 SCREENSHOT: Revisions and replicas dengan dua revision aktif -->

### 8.4 Rollback dalam Beberapa Klik

**Simulasi:** ternyata v2 punya bug! Di Modul 6, kita harus SSH ke VM, `docker rm -f`, lalu `docker run` image lama, dan dashboard **mati sebentar** di antaranya.

Di Container Apps, revision v1b **masih hidup**. Rollback cukup dengan memindahkan traffic:

- `andi-dashboard--v1b`: **100**
- `andi-dashboard--v2`: **0**

Klik **Save**, lalu refresh dashboard. Caption kembali ke **Versi v1**. Tanpa SSH, tanpa downtime, dan tanpa menunggu container baru dinyalakan.

Setelah itu, kembalikan lagi ke v2 (**v1b: 0**, **v2: 100**).

| | Modul 6 (VM) | Modul 7 (Container Apps) |
| :-- | :----------- | :----------------------- |
| Cara rollback | SSH → `docker rm -f` → `docker run :v1` | Ubah angka traffic → Save |
| Downtime | Ada, beberapa detik | Tidak ada |
| Versi lama | Harus di-pull & dinyalakan ulang | Masih hidup, siap menerima traffic |

### 8.5 Canary Release: Uji Coba ke Sebagian Pengguna

Di dunia kerja, versi baru jarang langsung diberikan ke **semua** pengguna. Biasanya dicoba dulu ke sebagian kecil pengguna. Kalau aman, porsinya dinaikkan. Strategi ini disebut **canary release** (diambil dari kisah burung kenari yang dulu dibawa penambang untuk mendeteksi gas beracun lebih awal).

1. Atur traffic: `v1b` = **50**, `v2` = **50**. Klik **Save**.
2. Buka URL dashboard di jendela **Incognito/Private**. Catat versinya dari caption.
3. **Tutup** jendela incognito, buka lagi, lalu ulangi sampai **10 kali**.
4. Isi tabel:

   | Percobaan | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
   | :-------- | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
   | Versi | | | | | | | | | | |

5. Ubah menjadi `v1b` = **80**, `v2` = **20**, lalu ulangi percobaan. Apakah proporsinya mendekati 8 : 2?

> 💡 Hasilnya tidak akan **persis** 5 : 5 atau 8 : 2, karena pembagian traffic bersifat acak per koneksi. Makin banyak percobaan, makin mendekati proporsi yang diatur.

<!-- 📸 SCREENSHOT: pengaturan traffic 80/20 di Revisions and replicas -->

### 8.6 (Opsional) Label: URL Khusus untuk Satu Revision

Bagaimana kalau tim QA ingin menguji v2 **sebelum** ada pengguna yang diarahkan ke sana (traffic v2 = 0%)? Gunakan **label**.

1. Di **Revisions and replicas**, pada baris `andi-dashboard--v2`, isi kolom **Label** dengan `uji`, lalu **Save**.
2. Klik nama revision `andi-dashboard--v2`. Di panel detail, salin **Label URL**.
3. Buka Label URL tersebut. Kamu **selalu** mendapat v2, berapa pun pembagian traffic-nya.

Pola ini adalah dasar dari **blue-green deployment**: versi baru disiapkan dan diuji lewat URL khusus, lalu traffic dipindahkan 100% sekaligus setelah yakin.

### 8.7 Merapikan untuk Hands-on Berikutnya

1. Atur traffic: `v1b` = **0**, `v2` = **100**, lalu **Save**.
2. Pada baris `andi-dashboard--v1b`, klik **⋯ → Deactivate**.
3. Klik **Deployment mode**, lalu ubah kembali ke **Single**.

> 💡 Revision yang **inactive** tidak ditagih, tetapi tetap tersimpan dan bisa diaktifkan lagi kapan pun. Container Apps menyimpan hingga 100 revision inactive terakhir.

---

## 9. Hands-on 4: Scaling & Scale to Zero

### 9.1 Konsep Autoscaling di Container Apps

Setiap revision punya pengaturan:

| Pengaturan | Arti | Default |
| :--------- | :--- | :------ |
| **Min replicas** | Jumlah replica minimal. `0` = boleh *scale to zero* | 0 |
| **Max replicas** | Batas atas replica | 10 |
| **Scale rule** | Kapan replica ditambah | HTTP, 10 request bersamaan |

Untuk **HTTP scale rule**, Container Apps menghitung jumlah *concurrent requests* setiap **15 detik** (jumlah request dalam 15 detik terakhir ÷ 15). Kalau angkanya melewati batas, replica ditambah. Kalau traffic turun, replica dikurangi.

Saat tidak ada traffic sama sekali, replica terakhir dimatikan setelah masa tunggu (*cooldown*) sekitar **5 menit** (300 detik). Aplikasi menjadi **0 replica = 0 biaya compute**.

```
Request/detik   ▁▁▁▃▆█████▆▃▁▁▁▁▁▁▁▁▁▁▁▁
Replica         1 1 1 2 3 3 3 3 2 1 1 1 1 1 0 0 0 ...
                                  └── cooldown ──┘
```

### 9.2 Mengamati Scale to Zero & Cold Start

1. **Tutup semua tab** yang membuka dashboard (termasuk incognito).
2. Tunggu **±6–10 menit**. Sambil menunggu, baca bagian [9.6](#96-min-replicas-0-atau-1).
3. Buka **Monitoring → Metrics**, pilih **Replica Count**. Grafiknya seharusnya turun ke **0**.
4. Sekarang buka URL dashboard sambil menghitung dalam hati (atau pakai stopwatch HP). Berapa detik sampai dashboard muncul?

Jeda itu adalah **cold start**: Azure harus menyalakan replica baru, menjalankan container, dan menunggu Streamlit siap. Buka lagi (refresh) setelahnya, dan dashboard akan muncul jauh lebih cepat karena replica sudah menyala.

| Percobaan | Waktu tunggu |
| :-------- | :----------- |
| Pertama (dari 0 replica, *cold start*) | ... detik |
| Refresh berikutnya (replica sudah menyala) | ... detik |

### 9.3 Mengatur Scale Rule

Supaya autoscaling mudah terlihat di praktikum, kita pakai batas yang **sengaja dibuat kecil**.

1. Buka **App → Revisions and replicas**, lalu klik **+ Create new revision**.
2. **Name / Suffix**: `v2-scale`
3. Buka tab **Scale**:
   - **Min replicas**: `0`
   - **Max replicas**: `3`
4. Pada bagian scale rule, edit rule HTTP yang sudah ada (atau klik **Add**):
   - **Rule name**: `http-demo`
   - **Type**: **HTTP scaling**
   - **Concurrent requests**: `5`
5. Klik **Create**.

Karena aturan scaling termasuk perubahan **revision-scope**, Azure membuat revision baru `andi-dashboard--v2-scale`, dan karena mode-nya **Single**, traffic otomatis pindah ke sana.

> ⚠️ Max replicas sengaja dibatasi **3** supaya pemakaian kredit tetap kecil. Jangan mengisi angka besar.

### 9.4 Skrip Uji Beban

Di laptop, buat file `beban.py`:

```python
import sys
import time
import urllib.request
from concurrent.futures import ThreadPoolExecutor

URL = sys.argv[1]
DURASI = int(sys.argv[2]) if len(sys.argv) > 2 else 120  # detik
JUMLAH_WORKER = 20


def kirim_request_terus(_):
    """Satu worker mengirim request berulang-ulang sampai waktu habis."""
    berhasil, gagal = 0, 0
    selesai = time.time() + DURASI
    while time.time() < selesai:
        try:
            with urllib.request.urlopen(URL, timeout=10) as respons:
                respons.read()
            berhasil += 1
        except Exception:
            gagal += 1
    return berhasil, gagal


print(f"Mengirim request ke {URL} selama {DURASI} detik dengan {JUMLAH_WORKER} worker...")
with ThreadPoolExecutor(max_workers=JUMLAH_WORKER) as pool:
    hasil = list(pool.map(kirim_request_terus, range(JUMLAH_WORKER)))

total_berhasil = sum(h[0] for h in hasil)
total_gagal = sum(h[1] for h in hasil)
print(f"Selesai: {total_berhasil} berhasil, {total_gagal} gagal")
print(f"Rata-rata: {total_berhasil / DURASI:.1f} request/detik")
```

Jalankan (ganti dengan URL milikmu):

```powershell
python beban.py https://andi-dashboard.<kata-acak>.<region>.azurecontainerapps.io 120
```

> ⚠️ **Etika:** jalankan uji beban **hanya ke aplikasimu sendiri**. Membanjiri server milik orang lain dengan request tanpa izin sama saja dengan serangan DoS (*Denial of Service*).

### 9.5 Mengamati Replica Bertambah

Selama skrip berjalan, buka di portal:

- **App → Revisions and replicas** → tab **Replicas** (klik **Refresh** beberapa kali), atau
- **Monitoring → Metrics** → **Replica Count** dan **Requests**, rentang waktu **Last 30 minutes**

Replica seharusnya naik dari 0/1 menjadi **2 lalu 3** dalam waktu kurang dari satu menit. Setelah skrip selesai, tunggu beberapa menit. Replica akan turun lagi, dan akhirnya kembali ke **0**.

<!-- 📸 SCREENSHOT: grafik Metrics yang menunjukkan Replica Count naik ke 3 lalu turun ke 0 -->

> 💡 Bandingkan dengan **load balancer** di modul sebelumnya. Di sana kita menentukan sendiri berapa container di belakang load balancer. Di sini Azure menambah dan mengurangi container **otomatis** sesuai beban, dan load balancing-nya sudah termasuk di *ingress*.

### 9.6 Min Replicas: 0 atau 1?

| | `min replicas = 0` | `min replicas = 1` |
| :-- | :----------------- | :----------------- |
| Biaya saat sepi | **Nol** (tidak ada replica) | Ditagih tarif **idle** (lebih murah dari tarif aktif, tetapi tidak nol) |
| Cold start | **Ada**: pengunjung pertama menunggu beberapa detik | **Tidak ada**: selalu siap |
| Cocok untuk | Praktikum, demo, aplikasi internal yang jarang dibuka | Aplikasi yang pengunjungnya tidak boleh menunggu |

Tidak ada jawaban yang selalu benar. Ini adalah **trade-off** antara **biaya** dan **pengalaman pengguna**, dan memilihnya adalah bagian dari pekerjaan seorang engineer.

---

## 10. Hands-on 5: Kerja Tim dengan Azure RBAC

### 10.1 Dari User Linux ke Azure RBAC

Di Modul 6, aturan emasnya adalah **satu orang = satu akun = satu key**, dan kita mewujudkannya dengan user Linux. Di PaaS tidak ada user Linux, karena kita tidak pernah login ke server. Gantinya adalah **Azure RBAC (Role-Based Access Control)**: memberi **peran (role)** kepada **akun Microsoft** seseorang pada **cakupan (scope)** tertentu.

| Modul 6 (VM) | Modul 7 (Azure RBAC) |
| :----------- | :------------------- |
| User Linux `budi` | Akun Microsoft (email kampus) Budi |
| Memasang public key di `authorized_keys` | **Role assignment** |
| User biasa (hanya melihat-lihat) | Role **Reader** |
| User + group `docker` (bisa deploy, setara root) | Role **Contributor** |
| `azureuser` / pemilik VM | Role **Owner** |
| Akses ke satu VM | **Scope**: satu Resource Group |
| `last -n 10` (riwayat login) | **Activity log** (riwayat siapa mengubah apa) |
| `sudo userdel -r budi` | **Remove role assignment** |

Tiga role bawaan yang paling umum:

| Role | Bisa melihat | Bisa mengubah/membuat/menghapus | Bisa memberi akses ke orang lain |
| :--- | :----------: | :-----------------------------: | :------------------------------: |
| **Reader** | ✅ | ❌ | ❌ |
| **Contributor** | ✅ | ✅ | ❌ |
| **Owner** | ✅ | ✅ | ✅ |

Prinsipnya sama dengan peringatan group `docker` di Modul 6: **berikan hak sekecil mungkin yang masih cukup untuk bekerja** (*principle of least privilege*). Berikan akses di level **Resource Group**, bukan seluruh subscription.

### 10.2 Memberi Akses ke Teman

Kerjakan **berpasangan**. Contoh: **Andi** memberi akses ke **Budi**, lalu tukar peran.

> ⚠️ Langkah ini hanya bisa dilakukan kalau kalian berdua berada di **tenant (organisasi) yang sama**, misalnya sama-sama mendaftar Azure for Students dengan email kampus. Kalau akun teman tidak bisa ditemukan, lewati 10.2–10.3 dan kerjakan [10.4](#104-activity-log-siapa-melakukan-apa) saja.

**Andi** melakukan:

1. Buka **Resource groups → rg-modul7**.
2. Di menu kiri, pilih **Access control (IAM)**.
3. Klik **+ Add → Add role assignment**.
4. Tab **Role**: pilih **Reader**, lalu **Next**.
5. Tab **Members**: *Assign access to* **User, group, or service principal**, lalu klik **+ Select members** dan cari **email kampus Budi**.
6. Klik **Review + assign**.

### 10.3 Teman Mengecek Aksesnya

**Budi** login ke <https://portal.azure.com> dengan akunnya sendiri:

1. Buka **Resource groups**. Kalau `rg-modul7` milik Andi belum terlihat, ubah filter subscription menjadi **All subscriptions** (atau buka *Settings → Directories + subscriptions* dan pastikan subscription Andi tercentang).
2. Buka Container App `andi-dashboard`. Budi bisa melihat **Overview**, **Revisions**, dan **Metrics**.
3. Coba ubah traffic di **Revisions and replicas**, atau klik **Create new revision**. Seharusnya **ditolak**, karena role Reader tidak boleh mengubah apa pun.

<!-- 📸 SCREENSHOT: pesan error/akses ditolak saat Budi (Reader) mencoba mengubah sesuatu -->

**Andi** kemudian **mencabut** akses Budi:

1. **rg-modul7 → Access control (IAM) → Role assignments**
2. Centang baris Budi, lalu klik **Remove**.

Sama seperti `userdel` di Modul 6, Andi tidak perlu mengganti apa pun milik orang lain.

### 10.4 Activity Log: Siapa Melakukan Apa?

1. Buka **rg-modul7 → Activity log**.
2. Atur *Timespan* ke **Last 24 hours**.

Setiap perubahan tercatat lengkap dengan **siapa** yang melakukannya: membuat Container App, mengubah traffic, menambah role assignment, dan seterusnya. Klik salah satu entri untuk melihat detailnya.

Inilah alasan yang sama dengan `last -n 10` di Modul 6: kalau semua orang memakai satu akun bersama, catatan ini tidak ada artinya.

<!-- 📸 SCREENSHOT: Activity log rg-modul7 -->

---

## 11. Bonus: Semua Lewat Azure CLI

Semua yang kita lakukan lewat klik di portal juga bisa dilakukan lewat **perintah**. Ini penting, karena perintah bisa **disimpan, diulang, dan diotomatisasi** (misalnya oleh CI/CD di modul berikutnya).

### 11.1 Memasang Azure CLI di Laptop

Azure punya terminal di browser bernama **Cloud Shell** (ikon `>_` di portal), tetapi banyak tenant kampus **menonaktifkannya**. Karena itu kita pasang **Azure CLI** langsung di laptop. CLI ini juga akan berguna untuk modul CI/CD berikutnya.

Windows (PowerShell):

```powershell
winget install -e --id Microsoft.AzureCLI
```

macOS:

```bash
brew install azure-cli
```

Linux (Ubuntu/Debian): ikuti petunjuk di <https://learn.microsoft.com/cli/azure/install-azure-cli-linux>

**Tutup lalu buka lagi** terminal, kemudian login dan pasang extension Container Apps:

```powershell
az login
az account show -o table
az extension add --name containerapp --upgrade
```

`az login` akan membuka browser. Login dengan **email kampus**. Kalau diminta memilih subscription, pilih **Azure for Students**.

> 💡 Perintah di bawah ditulis dalam format **Bash** (variabel `RG=...`, baris disambung dengan `\`). Di **PowerShell**, tulis variabel sebagai `$RG = "rg-modul7"` dan sambung baris dengan backtick `` ` ``, atau tulis setiap perintah dalam satu baris.

### 11.2 Perintah-perintah Berguna

Simpan nama-nama ke variabel supaya tidak salah ketik:

```bash
RG=rg-modul7
APP=andi-dashboard
```

```bash
# Daftar Container App di resource group
az containerapp list -g $RG -o table

# Ambil URL aplikasi
az containerapp show -n $APP -g $RG --query properties.configuration.ingress.fqdn -o tsv

# Daftar revision (termasuk yang inactive)
az containerapp revision list -n $APP -g $RG --all -o table

# Lihat log secara langsung (Ctrl + C untuk berhenti)
az containerapp logs show -n $APP -g $RG --follow --tail 30

# Deploy image baru sebagai revision baru
az containerapp update -n $APP -g $RG \
  --image docker.io/<username-dockerhub>/dashboard-kopi:v2 \
  --revision-suffix v2-cli
```

Traffic splitting lewat CLI (membutuhkan mode **multiple**):

```bash
az containerapp revision set-mode -n $APP -g $RG --mode multiple

az containerapp ingress traffic set -n $APP -g $RG \
  --revision-weight $APP--v1b=80 $APP--v2-cli=20
```

> 💡 Perintah `az containerapp up` bahkan bisa membuat environment dan Container App sekaligus dalam **satu perintah**:
>
> ```bash
> az containerapp up -n andi-cli -g $RG --environment cae-modul7 \
>   --image docker.io/<username-dockerhub>/dashboard-kopi:v2 \
>   --ingress external --target-port 8501 \
>   --env-vars PEMILIK=Andi
> ```

---

## 12. IaaS vs PaaS: Kapan Pakai yang Mana?

Setelah mencoba keduanya, kita bisa menilai sendiri. Tidak ada yang "selalu lebih baik", yang ada adalah **lebih cocok untuk kasus tertentu**.

| Kebutuhan | Pilihan yang lebih cocok | Alasan |
| :-------- | :----------------------- | :----- |
| Dashboard / web app / API sederhana | **PaaS** | HTTPS, scaling, dan rollback sudah tersedia |
| Pengunjung naik-turun, sering sepi | **PaaS** | Scale to zero = hemat biaya |
| Tim kecil tanpa orang khusus infrastruktur | **PaaS** | Tidak perlu mengurus OS, patch, SSH |
| Training model lama (berjam-jam) dengan GPU | **IaaS** | Butuh kontrol penuh atas hardware dan proses jangka panjang |
| Butuh install software khusus di level OS, driver, atau akses root | **IaaS** | PaaS tidak memberi akses ke OS host |
| Menyimpan data di disk lokal server (misal database sendiri) | **IaaS** (atau database terkelola) | Container PaaS bersifat sekali pakai, disk-nya hilang saat replica mati |
| Beban berat dan stabil 24 jam sehari | **IaaS** (perlu dihitung) | Harga per jam VM bisa lebih murah daripada bayar per detik terus-menerus |
| Ingin belajar dasar server, jaringan, dan Linux | **IaaS** | Semua terlihat dan bisa diutak-atik |

> 💡 Di dunia kerja keduanya sering **dipakai bersamaan**: dashboard dan API berjalan di PaaS, sementara training model berat berjalan di VM dengan GPU.

---

## 13. Bersih-bersih Biaya

### 13.1 Apa Saja yang Masih Bisa Menagih?

| Resource | Ditagih saat sepi? |
| :------- | :----------------- |
| Container App (min replicas = 0) | ✅ Tidak, kalau replica sudah 0 |
| Container App (min replicas ≥ 1) | ⚠️ Ya, tarif idle |
| Container Apps Environment (Consumption) | ✅ Tidak ada biaya tetap |
| Log Analytics workspace | ⚠️ Ditagih per GB log yang masuk dan disimpan. Untuk praktikum ini sangat kecil, tapi tetap bukan nol |
| Workload profile **Dedicated** (kalau tidak sengaja dibuat) | ⚠️ **Ya, per jam** |

### 13.2 Hapus Semua Setelah Selesai

Setelah praktikum dan Soal Latihan selesai dinilai:

1. Buka **Resource groups → rg-modul7**.
2. Klik **Delete resource group**.
3. Ketik nama resource group untuk konfirmasi, lalu klik **Delete**.

Pastikan setelah beberapa menit tidak ada resource yang tersisa di menu **All resources**. Cek juga bahwa `rg-modul6` sudah terhapus.

### 13.3 Cek Sisa Kredit

Buka **Education Hub** dan bandingkan pemakaian kredit Modul 6 dan Modul 7. Mana yang lebih hemat?

---

## 14. Troubleshooting

| Masalah | Solusi |
| :------ | :----- |
| Error `MissingSubscriptionRegistration` atau *"subscription is not registered to use namespace Microsoft.App"* | Daftarkan resource provider `Microsoft.App` dan `Microsoft.OperationalInsights` (lihat [5.3](#53-daftarkan-resource-provider)) |
| Error *"Revision Suffix is not supported for container app ... on express environments"* (`ExpressRevisionSuffixNotSupported`) | Container App kamu berada di environment **Express**, yang hanya mendukung single revision tanpa suffix dan traffic splitting. Environment tidak bisa diubah dari Express ke Standard. Cara paling bersih: hapus seluruh `rg-modul7`, buat ulang resource group, deploy template ([6.2](#62-membuat-container-apps-environment-standard)), **verifikasi lewat JSON View** (API version terbaru) bahwa `environmentMode` bernilai `WorkloadProfiles`, lalu buat ulang Container App dan pilih environment itu |
| *"Cloud Shell has been disabled in directory ..."* | Admin tenant kampus menonaktifkan Cloud Shell, dan mahasiswa tidak bisa mengaktifkannya sendiri. Buat environment dengan template ([6.2](#62-membuat-container-apps-environment-standard)) dan pakai Azure CLI di laptop ([11.1](#111-memasang-azure-cli-di-laptop)) |
| Deploy template gagal (*"The template deployment failed"*) | Buka detail error di halaman deployment. Penyebab umum: region tidak diizinkan policy (`RequestDisallowedByAzure`), resource provider `Microsoft.App`/`Microsoft.OperationalInsights` belum *Registered* ([5.3](#53-daftarkan-resource-provider)), atau nama `cae-modul7` sudah dipakai di resource group yang sama |
| Error `RequestDisallowedByAzure` | Region tidak diizinkan untuk subscription-mu. Pakai region dari daftar policy (Modul 6 bagian 5.2) |
| Revision berstatus **Failed** / **Activation failed** | Buka **Log stream** atau klik revision → lihat *System logs*. Penyebab umum: nama image/tag salah, `Registry login server` bukan `docker.io`, atau target port bukan `8501` |
| Log berisi `exec format error` | Image di-build untuk Arm (Mac M-series). Build ulang dengan `--platform linux/amd64` lalu push lagi |
| Error `toomanyrequests` saat menarik image dari Docker Hub | Docker Hub membatasi jumlah pull tanpa login. Di Container App, buka **Settings → Registries**, tambahkan `docker.io` dengan username Docker Hub dan *access token*. Alternatif: pakai Azure Container Registry |
| Image Docker Hub **private** tidak bisa ditarik | Pilih *Image type*: **Private** dan isi kredensial Docker Hub, atau jadikan repository public |
| Dashboard lama sekali saat pertama dibuka | Itu **cold start** karena scale to zero. Normal. Kalau mengganggu, set **min replicas = 1** (ada biaya idle) |
| Streamlit hanya menampilkan "Please wait..." / tidak tersambung | Pastikan *Ingress type* **HTTP** dan *target port* **8501**. Coba hard refresh (`Ctrl + Shift + R`) atau jendela incognito |
| Kolom **Traffic** tidak bisa diubah | Revision mode masih **Single**. Ubah ke **Multiple** dulu ([8.2](#82-mengubah-ke-multiple-revision-mode)) |
| Error *revision suffix already exists* | Suffix revision harus unik. Pakai suffix lain, misal `v2b`, atau kosongkan agar dibuat otomatis |
| Selalu mendapat versi yang sama walaupun traffic 50/50 | Browser memakai ulang koneksi lama. Tutup jendela incognito sepenuhnya sebelum membuka lagi, lalu ulangi lebih banyak kali |
| **Console** tidak bisa tersambung | Tidak ada replica yang berjalan (scale to zero). Buka URL dashboard dulu, tunggu, lalu coba lagi |
| Replica tidak bertambah saat uji beban | Tunggu minimal 1 menit (dihitung tiap 15 detik). Pastikan max replicas > 1, revision yang benar menerima traffic, dan concurrent requests kecil (misal 5). Naikkan `DURASI` di skrip |
| Aplikasi dengan *file upload* atau gambar error saat replica > 1 | Aktifkan **Session affinity** (sticky session) di menu **Ingress**. Catatan: fitur ini membutuhkan revision mode **Single** |
| Teman (Reader) tidak menemukan resource group | Ubah filter subscription di portal menjadi **All subscriptions**, atau cek *Directories + subscriptions* |
| Teman tidak bisa ditemukan saat menambah role | Kalian berbeda tenant. Kerjakan bagian Activity log saja ([10.4](#104-activity-log-siapa-melakukan-apa)) |

---

## Soal Latihan

Pemilik kedai senang dengan dashboard dari Modul 6, tapi ada keluhan: *"Kenapa alamatnya angka-angka dan browser bilang 'Not secure'? Lalu kenapa tagihan server tetap jalan padahal malam hari tidak ada yang buka?"* Tugas tim kalian adalah memindahkan dashboard ke **PaaS**. Ketentuan:

1. Gunakan **image `v1` dan `v2` milikmu sendiri dari Soal Latihan Modul 6** (dataset buatanmu sendiri, bukan data kopi). Kalau belum ada, buat dulu sesuai ketentuan Modul 6.
2. Deploy `v1` ke **Azure Container Apps** di resource group `rg-modul7` (dengan tag `pemilik` dan `modul`), dengan nama Container App berprefix namamu dan environment variable pemilik.
3. Ubah environment variable pemilik **tanpa build ulang image**, dan tunjukkan bahwa revision baru terbentuk.
4. Rilis `v2` sebagai revision baru dalam mode **Multiple**, lakukan **rollback** ke v1 dengan traffic, lalu lakukan **canary 80/20** dan isi tabel 10 percobaan dari [8.5](#85-canary-release-uji-coba-ke-sebagian-pengguna).
5. Atur **scale rule** (min 0, max 3), jalankan skrip uji beban, dan tunjukkan replica naik lalu kembali ke **0**. Catat juga waktu **cold start**.
6. Beri teman akses **Reader** ke resource group-mu, minta teman mencoba mengubah sesuatu (harus ditolak), lalu **cabut** aksesnya. *(Kalau beda tenant, ganti dengan screenshot Activity log saja.)*
7. Kumpulkan **screenshot** berikut:
   - Halaman Overview Container App (Application Url dan status terlihat)
   - Dashboard v2 milikmu diakses lewat `https://...azurecontainerapps.io` (gembok HTTPS terlihat)
   - Halaman **Revisions and replicas** yang menunjukkan dua revision aktif dengan traffic **80/20**
   - Output **Log stream** atau **Console** (`env | grep PEMILIK`)
   - Grafik **Metrics → Replica Count** yang menunjukkan replica naik saat uji beban dan turun ke 0
   - Output skrip `beban.py`
   - Halaman **Access control (IAM)** dengan role assignment teman, **atau** halaman **Activity log**
8. Jawab pertanyaan refleksi berikut (masing-masing 2–4 kalimat):
   1. Sebutkan **tiga** hal yang kamu kerjakan sendiri di Modul 6 tetapi diurus oleh Azure di Modul 7.
   2. Dengan ukuran container yang kamu pakai, hitung berapa **jam replica aktif** per bulan yang masih masuk kuota gratis. Tunjukkan perhitungannya.
   3. Untuk dashboard pemilik kedai, kamu memilih **min replicas 0 atau 1**? Jelaskan trade-off-nya.
   4. Sebutkan **satu** kasus di mana kamu tetap akan memilih **VM** dibanding Container Apps, dan alasannya.
9. Setelah dinilai, **hapus Resource Group** dan screenshot halaman sisa kredit di Education Hub.

**Bonus (opsional):** lakukan langkah 4 (rilis v2 dan traffic splitting) sepenuhnya lewat **Azure CLI** di laptop, dan lampirkan screenshot perintah beserta output-nya.

---

## Done 🎉🎉

Pada modul ini kita telah:

1. Memahami perbedaan IaaS dan PaaS dari pengalaman langsung, bukan sekadar teori
2. Mengenal konsep Azure Container Apps: environment, container app, revision, replica, dan ingress
3. Memahami model biaya Consumption plan dan menghitung kuota gratisnya
4. Men-deploy image Docker dari Modul 6 ke Container Apps dengan **HTTPS otomatis**, tanpa SSH
5. Mengubah konfigurasi tanpa build ulang, membaca log, masuk ke console, dan melihat metrik
6. Melakukan update tanpa downtime, **rollback**, **canary release**, dan memahami **blue-green deployment**
7. Mengatur **autoscaling** dan **scale to zero**, serta memahami trade-off cold start vs biaya
8. Memberi dan mencabut akses tim dengan **Azure RBAC**, serta menelusuri perubahan lewat **Activity log**

Masih ada satu hal yang manual: setiap rilis versi baru, kita tetap harus **build → push → edit revision** sendiri. Di modul berikutnya, kita akan mengotomatiskan seluruh alur itu dengan **CI/CD (GitHub Actions)**: cukup `git push`, dan dashboard versi baru langsung online.

*Modul 7 — PaaS dengan Azure Container Apps*
*Cloud Computing · Zein M Hasan · Teknologi Informasi ITS*
