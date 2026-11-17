# Modul 6 - Cloud Computing dengan Microsoft Azure: Sewa VM, SSH, Deploy Docker & Kerja Tim

## Daftar Isi

1. [Pendahuluan](#1-pendahuluan)
2. [Mengenal Cloud & Azure](#2-mengenal-cloud--azure)
3. [Apa Saja yang Ditawarkan Azure](#3-apa-saja-yang-ditawarkan-azure)
4. [Gambaran Praktikum](#4-gambaran-praktikum)
5. [Persiapan](#5-persiapan)
6. [Hands-on 1: Sewa VM](#6-hands-on-1-sewa-vm)
7. [Hands-on 2: SSH yang Nyaman & Aman](#7-hands-on-2-ssh-yang-nyaman--aman)
8. [Hands-on 3: Deploy dengan Docker](#8-hands-on-3-deploy-dengan-docker)
9. [Hands-on 4: Kerja Tim di Satu Server](#9-hands-on-4-kerja-tim-di-satu-server)
10. [Bersih-bersih Biaya](#10-bersih-bersih-biaya)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Pendahuluan

Di modul-modul sebelumnya, semua praktikum berjalan di laptop sendiri: Docker Desktop, container, sampai load balancer. Masalahnya, laptop punya batas:

1. Bagaimana kalau dataset atau model terlalu berat untuk laptop kita?
2. Bagaimana caranya teman satu tim bisa membuka hasil kerja kita, tanpa harus "di laptopku jalan kok"?
3. Bagaimana kalau satu server dipakai **ramai-ramai** oleh satu tim, tanpa saling ganggu dan tanpa berbagi password?

Jawabannya adalah menyewa komputer di **cloud**. Pada modul ini kita akan menggunakan layanan cloud sungguhan, yaitu **Microsoft Azure**, dengan kredit gratis dari program **Azure for Students**.

Di akhir modul, kita akan bisa:

- Menyewa sebuah server (VM) di Azure
- Masuk ke server dengan **SSH key** secara aman
- Men-deploy aplikasi data sederhana memakai **Docker** dengan cara yang profesional (registry + tag versi)
- Memberi akses server ke teman satu tim dan memahami **aturan kerja tim** di satu server

---

## 2. Mengenal Cloud & Azure

### 2.1 Apa itu Cloud Provider?

**Cloud provider** adalah perusahaan yang menyewakan sumber daya komputasi (server, storage, database, GPU, dan lain-lain) lewat internet. Kita tidak perlu membeli hardware, cukup **sewa sesuai pemakaian** dan bayar per jam/per GB.

Tiga cloud provider terbesar saat ini:

| Provider | Perusahaan | Contoh layanan VM |
| :------- | :--------- | :---------------- |
| **Microsoft Azure** | Microsoft | Azure Virtual Machines |
| **Amazon Web Services (AWS)** | Amazon | Amazon EC2 |
| **Google Cloud Platform (GCP)** | Google | Compute Engine |

Konsep yang kita pelajari di Azure hampir semuanya punya padanan di AWS dan GCP. Bedanya kebanyakan hanya di nama dan tampilan.

### 2.2 IaaS, PaaS, SaaS

Layanan cloud biasanya dibagi menjadi tiga model berdasarkan **seberapa banyak yang diurus oleh kita** dan **seberapa banyak yang diurus oleh provider**.

Analogi paling terkenal: **makan pizza**.

| Model | Analogi Pizza | Yang kita urus | Contoh di Azure |
| :---- | :------------ | :------------- | :-------------- |
| **IaaS** (Infrastructure as a Service) | Sewa dapur lengkap, masak sendiri | OS, install software, aplikasi, data | **Virtual Machines** |
| **PaaS** (Platform as a Service) | Pesan pizza, makan di rumah | Kode aplikasi dan data saja | App Service, Container Apps, Azure SQL |
| **SaaS** (Software as a Service) | Makan di restoran | Tinggal pakai | Microsoft 365, Outlook |

Pada modul ini kita fokus ke **IaaS (Virtual Machine)**, karena dengan VM kita belajar dasar yang paling penting: server, jaringan, SSH, dan deployment. Setelah paham IaaS, layanan PaaS akan terasa jauh lebih mudah.

### 2.3 Istilah Dasar di Azure

Sebelum membuka portal, kenali dulu "struktur folder" di Azure:

```
Tenant (akun organisasi, misal akun kampus)
└── Subscription  "Azure for Students"      ← dompet: semua biaya ditagihkan ke sini
    └── Resource Group  "rg-modul6"          ← folder proyek
        ├── Virtual Machine   "vm-modul6"
        ├── Disk              (hard disk VM)
        ├── Public IP         (alamat IP publik)
        ├── Network Security Group (firewall)
        └── Virtual Network   (jaringan privat)
```

| Istilah | Penjelasan | Analogi |
| :------ | :--------- | :------ |
| **Subscription** | Wadah tagihan. Semua resource yang dibuat memotong kredit dari sini | Dompet |
| **Resource Group** | Kumpulan resource untuk satu proyek | Folder proyek |
| **Resource** | Satu layanan yang kita buat (VM, disk, IP, dll) | Barang di dalam folder |
| **Region** | Lokasi data center fisik (misal *Southeast Asia* = Singapura) | Kota tempat gudang |

> 💡 **Tips penting:** satu VM ternyata terdiri dari **beberapa resource** (disk, IP, network, firewall). Karena itu, simpan semua yang berhubungan dengan satu proyek di **satu Resource Group**. Ketika proyek selesai, cukup **hapus Resource Group-nya**, dan semua isinya ikut terhapus. Tidak ada "sisa" yang diam-diam memotong kredit.

### 2.4 Bagaimana Azure Menagih Biaya?

Azure menggunakan model **pay-as-you-go**, yaitu bayar sesuai pemakaian:

- **VM** ditagih **per jam selama menyala**. VM yang lupa dimatikan = kredit terus berkurang.
- **Disk** tetap ditagih walaupun VM sedang mati, karena datanya tetap disimpan.
- **Public IP** juga bisa ditagih walaupun VM mati.

Artinya, **kebiasaan bersih-bersih** sama pentingnya dengan kemampuan membuat resource. Kita akan membahasnya di [bagian 10](#10-bersih-bersih-biaya).

---

## 3. Apa Saja yang Ditawarkan Azure

### 3.1 Layanan Azure yang Berguna untuk Data Science

Azure punya lebih dari 200 layanan. Kita tidak perlu hafal semuanya. Berikut beberapa yang paling relevan untuk anak data science:

| Kategori | Layanan Azure | Kegunaan untuk Data Science | Padanan AWS / GCP |
| :------- | :------------ | :-------------------------- | :---------------- |
| **Compute** | Virtual Machines | Server sendiri untuk notebook, training, atau deploy dashboard | EC2 / Compute Engine |
| **Compute** | App Service | Deploy web app/API tanpa mengurus server | Elastic Beanstalk / App Engine |
| **Compute** | Container Apps | Menjalankan container tanpa mengurus server | App Runner / Cloud Run |
| **Storage** | Blob Storage | Menyimpan dataset besar (CSV, Parquet, gambar) | S3 / Cloud Storage |
| **Database** | Azure Database for PostgreSQL | Database relasional terkelola | RDS / Cloud SQL |
| **Database** | Cosmos DB | Database NoSQL | DynamoDB / Firestore |
| **Container** | Container Registry (ACR) | Menyimpan image Docker secara privat | ECR / Artifact Registry |
| **AI / ML** | Azure Machine Learning | Training, tracking eksperimen, dan deploy model ML | SageMaker / Vertex AI |
| **AI / ML** | Azure AI Services | API siap pakai: OCR, speech, translator, dll | AWS AI Services / Cloud AI APIs |

> 💡 Pada modul ini kita memakai **Docker Hub** sebagai registry karena lebih sederhana dan sudah dikenal dari modul Docker. Di dunia kerja yang memakai Azure, perannya biasanya digantikan oleh **Azure Container Registry (ACR)**. Konsepnya sama persis: *build → push → pull*.

### 3.2 Azure for Students

**Azure for Students** adalah program Microsoft untuk mahasiswa:

- **Kredit USD 100** yang bisa dipakai selama **12 bulan**, dan bisa diperpanjang tiap tahun selama masih mahasiswa
- **Tidak perlu kartu kredit**, cukup daftar dengan **email kampus**
- Akses ke sejumlah **layanan gratis**, termasuk beberapa ukuran VM kecil seperti **B1s**

> ⚠️ Ketentuan kredit dan layanan gratis bisa berubah. Selalu cek halaman resminya: <https://azure.microsoft.com/free/students>
>
> ⚠️ Kalau kredit habis dan subscription tidak di-upgrade ke pay-as-you-go, subscription akan **dinonaktifkan** beserta seluruh resource di dalamnya.

---

## 4. Gambaran Praktikum

### 4.1 Tools yang Digunakan

| Tool | Fungsi |
| :--- | :----- |
| **Azure Portal** | Website untuk membuat dan mengelola resource Azure |
| **Ubuntu Server (VM)** | Server yang kita sewa di Azure |
| **OpenSSH** | Masuk (remote login) ke server dari laptop |
| **Docker** | Membungkus aplikasi agar bisa jalan sama persis di laptop dan di server |
| **Docker Hub** | Registry tempat menyimpan dan berbagi image Docker |
| **Streamlit** | Library Python untuk membuat dashboard data dengan cepat |

### 4.2 Arsitektur Sistem

```
 LAPTOP KAMU                                         AZURE (Resource Group: rg-modul6)
 ┌───────────────────────┐                          ┌──────────────────────────────────────┐
 │ app.py + Dockerfile   │                          │  VM Ubuntu  (vm-modul6)              │
 │                       │   1. docker push         │                                      │
 │ docker build ─────────┼──────────► Docker Hub ───┼─► 2. docker pull                     │
 │                       │                          │                                      │
 │ ssh / scp ────────────┼──── port 22 (SSH) ──────►│  ┌─ andi-dashboard  :8501 ─┐         │
 └───────────────────────┘                          │  └─ budi-dashboard  :8502 ─┘         │
                                                    │           ▲                          │
 LAPTOP TEMAN                                       │           │  Network Security Group  │
 ┌───────────────────────┐                          │           │  (firewall: 22, 8501-8510)│
 │ ssh budi@IP-VM ───────┼──── port 22 (SSH) ──────►│                                      │
 └───────────────────────┘                          └──────────────────────────────────────┘
                                                                ▲
 BROWSER SIAPA SAJA ──── http://IP-VM:8501 ─────────────────────┘
```

Alurnya:

1. Kita membuat **VM** di Azure, lalu masuk dengan **SSH key**.
2. Kita membuat aplikasi dashboard di laptop, membungkusnya jadi **image Docker**, dan **push** ke Docker Hub.
3. Di VM, kita cukup **pull** image tersebut lalu menjalankannya. Dashboard bisa dibuka dari browser mana pun.
4. Kita memberi akses VM ke **satu teman**. Teman login dengan akunnya sendiri dan men-deploy dashboard miliknya di port berbeda.

> 💡 Praktikum ini dikerjakan **berpasangan**. Setiap orang tetap membuat VM sendiri, lalu di Hands-on 4 kalian saling memberi akses ke VM masing-masing.

---

## 5. Persiapan

### 5.1 Aktivasi Azure for Students

1. Buka <https://azure.microsoft.com/free/students> lalu klik **Start free**.
2. Login dengan **email kampus**, lalu ikuti verifikasi status mahasiswa.
3. Setelah selesai, buka **Azure Portal**: <https://portal.azure.com>
4. Di kolom pencarian atas, ketik **Subscriptions**. Pastikan ada subscription bernama **Azure for Students** dengan status **Active**.

<img width="1916" height="905" alt="Screenshot 2026-11-17 212331" src="https://github.com/user-attachments/assets/110148bb-0f8a-4f60-9e19-5bb490f9b2c5" />


**Cek sisa kredit:** di kolom pencarian portal, ketik **Education**, lalu buka **Education Hub**. Halaman *Overview* menampilkan pemakaian dan sisa kredit.

### 5.2 Cek Region yang Diizinkan

Subscription Azure for Students **dibatasi hanya boleh membuat resource di beberapa region tertentu**. Daftarnya bisa berbeda untuk setiap orang. Kalau kita memilih region di luar daftar, pembuatan VM akan gagal dengan error `RequestDisallowedByAzure`.

Cara mengeceknya:

1. Di kolom pencarian portal, ketik **Policy**, lalu buka.
2. Pilih menu **Authoring → Assignments**.
3. Klik assignment bernama **Allowed resource deployment regions**.
4. Lihat bagian **Parameters**. Itulah daftar region yang boleh kita pakai.

<img width="1880" height="362" alt="Screenshot 2026-11-17 212617" src="https://github.com/user-attachments/assets/e636122a-bb32-4cd9-84b2-b54758f8c654" />


Pilih **satu** region dari daftar tersebut, sebaiknya yang terdekat dengan Indonesia (misalnya *Southeast Asia*, *East Asia*, *Indonesia Central*, atau *Malaysia West* jika tersedia), lalu **catat**. Semua resource di modul ini akan dibuat di region tersebut.

### 5.3 Pasang Budget Alert

Sebelum membuat resource apa pun, pasang **alarm biaya** dulu supaya tidak ada kejutan:

1. Di kolom pencarian portal, ketik **Cost Management**, lalu buka.
2. Pilih menu **Budgets → + Add**.
3. Pastikan *Scope* adalah subscription **Azure for Students**.
4. Isi:
   - **Name**: `budget-modul6`
   - **Reset period**: Monthly
   - **Amount**: `10` (USD)
5. Klik **Next**, lalu tambahkan alert:
   - **Type**: Actual, **% of budget**: `50`
   - **Type**: Actual, **% of budget**: `90`
   - **Alert recipients (email)**: email kamu
6. Klik **Create**.

Sekarang kamu akan dapat email kalau pemakaian bulan ini sudah mencapai USD 5 dan USD 9.

<img width="1540" height="336" alt="Screenshot 2026-11-17 213047" src="https://github.com/user-attachments/assets/d377134f-4388-4dd9-b0b1-a21e1c591525" />

> 💡 Untuk subscription yang baru dibuat, fitur Cost Management kadang baru bisa dipakai setelah **±48 jam**. Kalau menu Budgets belum bisa dipakai, lanjutkan dulu praktikumnya dan pasang budget di lain waktu.

### 5.4 Tools di Laptop

1. **Docker Desktop**, sudah dijelaskan di modul sebelumnya ;)

   ```powershell
   docker --version
   ```

2. **OpenSSH Client**. Sudah bawaan Windows 10/11, macOS, dan Linux. Cek dengan:

   ```powershell
   ssh -V
   ```

3. **Akun Docker Hub**. Daftar gratis di <https://hub.docker.com> dan catat **username**-nya. Di modul ini, username Docker Hub ditulis sebagai `<username-dockerhub>`.

---

## 6. Hands-on 1: Sewa VM

### 6.1 Konsep SSH Key

Ada dua cara masuk ke server lewat SSH:

- **Password**: mudah, tapi bisa ditebak, dibocorkan, dan dicoba berulang-ulang oleh bot (*brute force*). Server publik di internet **setiap hari** dicoba login oleh bot dari seluruh dunia.
- **SSH key**: jauh lebih aman, dan inilah standar di dunia kerja.

SSH key selalu berpasangan:

| File | Analogi | Boleh dibagikan? |
| :--- | :------ | :--------------- |
| **Public key** (`id_ed25519.pub`) | **Gembok** | ✅ Boleh. Dipasang di server mana pun yang ingin kita masuki |
| **Private key** (`id_ed25519`) | **Kunci** | ❌ **TIDAK PERNAH**. Hanya disimpan di laptop kita |

Kita memasang **gembok** (public key) di server. Hanya orang yang punya **kunci** yang cocok (private key) yang bisa masuk.

### 6.2 Membuat SSH Key

Jalankan di **PowerShell** (Windows) atau **Terminal** (macOS/Linux):

```powershell
ssh-keygen -t ed25519 -C "namakamu@laptop"
```

- Saat ditanya lokasi file, tekan **Enter** untuk memakai lokasi default.
- Saat ditanya *passphrase*, boleh diisi (lebih aman) atau dikosongkan untuk praktikum ini.

Dua file akan muncul di folder `.ssh`:

```
C:\Users\<nama>\.ssh\id_ed25519       ← private key (kunci)
C:\Users\<nama>\.ssh\id_ed25519.pub   ← public key (gembok)
```

Tampilkan isi **public key** (yang akan kita tempel ke Azure):

```powershell
Get-Content ~\.ssh\id_ed25519.pub
```

macOS/Linux:

```bash
cat ~/.ssh/id_ed25519.pub
```

Outputnya satu baris panjang seperti ini:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...xyz namakamu@laptop
```

### 6.3 Membuat Resource Group

1. Di kolom pencarian portal, ketik **Resource groups**, lalu klik **+ Create**.
2. Isi:
   - **Subscription**: Azure for Students
   - **Resource group**: `rg-modul6`
   - **Region**: region yang sudah dicatat di [5.2](#52-cek-region-yang-diizinkan)
3. Klik **Review + create → Create**.

### 6.4 Membuat Virtual Machine

1. Di kolom pencarian portal, ketik **Virtual machines**, lalu klik **+ Create → Azure virtual machine**.

2. Tab **Basics**:

   | Field | Isi |
   | :---- | :-- |
   | Subscription | Azure for Students |
   | Resource group | `rg-modul6` |
   | Virtual machine name | `vm-modul6` |
   | Region | **Sama** dengan region Resource Group |
   | Availability options | No infrastructure redundancy required |
   | Image | **Ubuntu Server 24.04 LTS - x64 Gen2** |
   | VM architecture | **x64** |
   | Size | **Standard_B1s** (1 vCPU, 1 GiB RAM). Klik *See all sizes* jika tidak muncul |
   | Authentication type | **SSH public key** |
   | Username | `azureuser` |
   | SSH public key source | **Use existing public key** |
   | Stored Key / SSH public key | Tempel isi `id_ed25519.pub` dari langkah 6.2 |
   | Public inbound ports | Allow selected ports |
   | Select inbound ports | **SSH (22)** |

   > ⚠️ **Jangan pilih ukuran berbasis Arm** (seri yang ada huruf **p**, misalnya B2pts v2). Image Docker yang kita build untuk x64 tidak akan bisa jalan di sana.

3. Tab **Disks**: ubah *OS disk type* menjadi **Standard SSD** (lebih hemat).

4. Tab **Networking**: biarkan default, tetapi **centang** opsi **Delete public IP and NIC when VM is deleted**.

5. Tab **Management**: aktifkan **auto-shutdown** supaya VM otomatis mati kalau kita lupa.
   - **Enable auto-shutdown**: ✅
   - **Shutdown time**: misal `22:00`
   - **Time zone**: (UTC+07:00) Bangkok, Hanoi, Jakarta

6. Klik **Review + create**. Tunggu validasi lolos, lalu klik **Create**.

Setelah deployment selesai (±1–2 menit), klik **Go to resource**. Di halaman **Overview**, catat **Public IP address**. Di modul ini alamat tersebut ditulis sebagai `<IP-VM>`.

<img width="1193" height="817" alt="Screenshot 2026-11-17 222211" src="https://github.com/user-attachments/assets/ccac410e-500e-46dc-a32d-dc2135a35e0d" />


### 6.5 Login Pertama ke VM

```powershell
ssh azureuser@<IP-VM>
```

Pada login pertama akan muncul pertanyaan seperti ini:

```
The authenticity of host '20.xx.xx.xx' can't be established.
ED25519 key fingerprint is SHA256:....
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Ketik `yes`. Ini adalah SSH yang "berkenalan" dengan server. Sidik jari server disimpan di file `known_hosts`, supaya kalau suatu saat ada server palsu yang menyamar dengan IP yang sama, SSH akan memberi peringatan.

Kalau berhasil, prompt berubah menjadi:

```
azureuser@vm-modul6:~$
```

Coba beberapa perintah untuk berkenalan dengan server baru kita:

```bash
hostname          # nama server
lsb_release -a    # versi Ubuntu
nproc             # jumlah CPU
free -h           # RAM
df -h /           # sisa disk
```

<img width="621" height="255" alt="Screenshot 2026-11-17 222521" src="https://github.com/user-attachments/assets/4eb31615-cec0-4e67-99f4-d078a6857afb" />


---

## 7. Hands-on 2: SSH yang Nyaman & Aman

### 7.1 Shortcut dengan `~/.ssh/config`

Mengetik `ssh azureuser@20.xx.xx.xx` setiap kali itu melelahkan dan rawan salah. Kita bisa membuat **shortcut** di file config SSH.

Keluar dulu dari VM (`exit`), lalu di laptop buat file `config` di folder `.ssh`.

Windows (PowerShell):

```powershell
New-Item -ItemType File -Force ~\.ssh\config
notepad ~\.ssh\config
```

macOS/Linux:

```bash
nano ~/.ssh/config
```

Isi dengan:

```
Host modul6
    HostName <IP-VM>
    User azureuser
    IdentityFile ~/.ssh/id_ed25519
```

> ⚠️ File harus bernama `config` **tanpa ekstensi**. Pastikan Notepad tidak diam-diam menyimpannya sebagai `config.txt`.

Sekarang cukup ketik:

```powershell
ssh modul6
```

### 7.2 Mengirim File dengan `scp`

`scp` (secure copy) adalah cara mengirim file ke/dari server lewat SSH. Shortcut `modul6` juga bisa dipakai di sini.

Buat file contoh di laptop lalu kirim ke VM:

```powershell
"Halo dari laptop!" | Out-File -Encoding utf8 halo.txt
scp halo.txt modul6:~/
```

Cek di VM:

```powershell
ssh modul6 "cat ~/halo.txt"
```

Sebaliknya, mengambil file **dari** VM ke laptop:

```powershell
scp modul6:~/halo.txt .\halo-dari-server.txt
```

### 7.3 Cek Keamanan SSH di Server

Masuk ke VM, lalu cek konfigurasi SSH server yang sedang aktif:

```bash
sudo sshd -T | grep -Ei "passwordauthentication|permitrootlogin|pubkeyauthentication"
```

Output yang **diharapkan**:

```
pubkeyauthentication yes
passwordauthentication no
permitrootlogin without-password
```

| Setting | Arti |
| :------ | :--- |
| `pubkeyauthentication yes` | Login dengan SSH key diizinkan |
| `passwordauthentication no` | Login dengan password **ditolak**, jadi bot brute force tidak ada gunanya |
| `permitrootlogin without-password` | User `root` tidak bisa login dengan password |

Karena VM dibuat dengan opsi *SSH public key*, Azure sudah mengaturnya untuk kita. Di server yang kita setup sendiri (bukan dari image cloud), dua setting ini **wajib** diperiksa dan diubah manual di `/etc/ssh/sshd_config`.

Penasaran seberapa sering server publik dicoba login oleh bot? Lihat log-nya:

```bash
sudo journalctl -u ssh --since "1 hour ago" | grep -i "invalid user" | tail -n 10
```

---

## 8. Hands-on 3: Deploy dengan Docker

Kita akan membuat **dashboard penjualan kedai kopi** sederhana dengan Streamlit, membungkusnya jadi image Docker, lalu men-deploy-nya ke VM.

### 8.1 Menyiapkan Folder Project (di laptop)

Struktur akhir project:

```
modul6/
├── app.py
├── data_kopi.csv
├── requirements.txt
├── Dockerfile
└── .dockerignore
```

Buat cepat lewat PowerShell:

```powershell
mkdir modul6
cd modul6
New-Item -ItemType File app.py, data_kopi.csv, requirements.txt, Dockerfile, .dockerignore
```

### 8.2 Dataset

Isi `data_kopi.csv`:

```csv
bulan,menu,terjual,harga
2026-07,Americano,120,18000
2026-07,Latte,150,25000
2026-07,Es Kopi Susu,210,20000
2026-08,Americano,135,18000
2026-08,Latte,160,25000
2026-08,Es Kopi Susu,245,20000
2026-09,Americano,128,18000
2026-09,Latte,172,25000
2026-09,Es Kopi Susu,268,20000
2026-10,Americano,140,18000
2026-10,Latte,181,25000
2026-10,Es Kopi Susu,300,20000
```

### 8.3 Aplikasi Dashboard (v1)

Isi `app.py`:

```python
import os

import pandas as pd
import streamlit as st

VERSI = "v1"
PEMILIK = os.getenv("PEMILIK", "Anonim")


def rupiah(angka):
    return "Rp " + f"{angka:,}".replace(",", ".")


st.set_page_config(page_title="Dashboard Kedai Kopi", page_icon="☕")
st.title("☕ Dashboard Penjualan Kedai Kopi")
st.caption(f"Versi {VERSI} · di-deploy oleh {PEMILIK}")

df = pd.read_csv("data_kopi.csv")
df["pendapatan"] = df["terjual"] * df["harga"]

kolom1, kolom2 = st.columns(2)
kolom1.metric("Total cup terjual", f"{df['terjual'].sum():,}".replace(",", "."))
kolom2.metric("Total pendapatan", rupiah(int(df["pendapatan"].sum())))

st.subheader("Cup terjual per bulan")
per_bulan = df.pivot_table(index="bulan", columns="menu", values="terjual", aggfunc="sum")
st.line_chart(per_bulan)

st.subheader("Data mentah")
st.dataframe(df)
```

Perhatikan dua hal:

- `VERSI` ditulis di kode, sehingga kita bisa melihat versi mana yang sedang berjalan di server.
- `PEMILIK` dibaca dari **environment variable**. Nilainya tidak ditulis di kode, tetapi diberikan saat container dijalankan. Ini cara standar untuk memberi konfigurasi ke container tanpa mengubah image.

### 8.4 requirements.txt

```
streamlit>=1.40,<2
pandas>=2.2,<3
```

> 💡 Membatasi versi (`>=1.40,<2`) membuat build lebih bisa diprediksi. Aplikasi tidak tiba-tiba rusak karena library naik ke versi mayor yang baru.

### 8.5 Dockerfile & .dockerignore

Isi `Dockerfile`:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install library dulu, baru salin kode.
# Kalau hanya app.py yang berubah, layer install tidak perlu diulang (build lebih cepat).
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8501

CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

Isi `.dockerignore`:

```
.venv/
__pycache__/
.git/
.env
*.pem
id_*
```

`.dockerignore` mencegah file yang tidak perlu (atau **rahasia**, seperti private key dan file `.env`) ikut masuk ke dalam image. Ingat: image yang di-push ke Docker Hub publik bisa di-download **siapa saja**.

### 8.6 Build & Tes di Laptop

```powershell
docker build --platform linux/amd64 -t <username-dockerhub>/dashboard-kopi:v1 .
```

| Bagian | Arti |
| :----- | :--- |
| `--platform linux/amd64` | Build untuk arsitektur x64 seperti VM kita. **Wajib** untuk pengguna Mac M1/M2/M3/M4 |
| `<username-dockerhub>/dashboard-kopi` | Nama image: `username/nama-repo` |
| `:v1` | **Tag versi** |

Tes jalankan di laptop:

```powershell
docker run --rm -p 8501:8501 -e PEMILIK=Andi <username-dockerhub>/dashboard-kopi:v1
```

Buka <http://localhost:8501>. Kalau dashboard muncul, tekan `Ctrl + C` untuk menghentikan.

<img width="695" height="490" alt="Screenshot 2026-11-17 224344" src="https://github.com/user-attachments/assets/4504c4a1-eefa-4f96-895a-327239f37186" />


### 8.7 Push ke Docker Hub

```powershell
docker login
docker push <username-dockerhub>/dashboard-kopi:v1
```

Buka <https://hub.docker.com>. Repository `dashboard-kopi` dengan tag `v1` sekarang sudah ada.

> 💡 **Kenapa tidak build langsung di server?** VM kita hanya punya RAM 1 GiB, dan build image bisa berat. Di dunia kerja, alurnya hampir selalu: **build di satu tempat (laptop/CI) → push ke registry → server cukup pull**. Server produksi tidak perlu punya source code, hanya image yang sudah jadi.

### 8.8 Install Docker di VM

Masuk ke VM:

```powershell
ssh modul6
```

Install Docker dengan script resmi:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Masukkan user kita ke group `docker` supaya tidak perlu `sudo` setiap kali menjalankan perintah docker:

```bash
sudo usermod -aG docker $USER
exit
```

Login lagi (supaya group baru terbaca), lalu tes:

```powershell
ssh modul6
```

```bash
docker run --rm hello-world
```

### 8.9 Buka Port di Firewall Azure (NSG)

Secara default hanya port 22 yang terbuka. Kita buka port untuk dashboard. Sekalian buka satu **rentang port** supaya teman nanti juga bisa memakai port lain.

1. Di portal, buka VM `vm-modul6`, lalu pilih menu **Networking** (atau **Network settings**).
2. Klik **Create port rule → Inbound port rule**.
3. Isi:
   - **Destination port ranges**: `8501-8510`
   - **Protocol**: TCP
   - **Action**: Allow
   - **Name**: `Allow-Dashboard-8501-8510`
4. Klik **Add**.

<img width="799" height="654" alt="Screenshot 2026-11-17 225432" src="https://github.com/user-attachments/assets/b368c096-70b7-4a6d-90aa-b33e9f600ec7" />


### 8.10 Pull & Run di VM

Di dalam VM:

```bash
docker run -d \
  --name andi-dashboard \
  --restart unless-stopped \
  -p 8501:8501 \
  -e PEMILIK=Andi \
  <username-dockerhub>/dashboard-kopi:v1
```

| Opsi | Arti |
| :--- | :--- |
| `-d` | Jalan di background |
| `--name andi-dashboard` | Nama container dengan **prefix nama sendiri** (penting untuk kerja tim, lihat bagian 9) |
| `--restart unless-stopped` | Container otomatis hidup lagi kalau VM restart |
| `-p 8501:8501` | Port VM 8501 → port container 8501 |
| `-e PEMILIK=Andi` | Mengisi environment variable |

Cek container:

```bash
docker ps
```

Buka di browser (dari laptop atau HP): `http://<IP-VM>:8501`

🎉 Dashboard kita sudah online di cloud dan bisa dibuka siapa saja!


### 8.11 Update ke v2 & Rollback

Di sinilah **tag versi** terasa manfaatnya. Kita tambahkan fitur baru: filter menu dan grafik pendapatan.

Di laptop, ubah `app.py` menjadi:

```python
import os

import pandas as pd
import streamlit as st

VERSI = "v2"
PEMILIK = os.getenv("PEMILIK", "Anonim")


def rupiah(angka):
    return "Rp " + f"{angka:,}".replace(",", ".")


st.set_page_config(page_title="Dashboard Kedai Kopi", page_icon="☕")
st.title("☕ Dashboard Penjualan Kedai Kopi")
st.caption(f"Versi {VERSI} · di-deploy oleh {PEMILIK}")

df = pd.read_csv("data_kopi.csv")
df["pendapatan"] = df["terjual"] * df["harga"]

# Fitur baru v2: filter menu di sidebar
semua_menu = sorted(df["menu"].unique())
pilihan = st.sidebar.multiselect("Pilih menu", semua_menu, default=semua_menu)
df = df[df["menu"].isin(pilihan)]

kolom1, kolom2 = st.columns(2)
kolom1.metric("Total cup terjual", f"{df['terjual'].sum():,}".replace(",", "."))
kolom2.metric("Total pendapatan", rupiah(int(df["pendapatan"].sum())))

st.subheader("Cup terjual per bulan")
per_bulan = df.pivot_table(index="bulan", columns="menu", values="terjual", aggfunc="sum")
st.line_chart(per_bulan)

# Fitur baru v2: pendapatan per menu
st.subheader("Pendapatan per menu")
st.bar_chart(df.groupby("menu")["pendapatan"].sum())

st.subheader("Data mentah")
st.dataframe(df)
```

Build dan push dengan **tag baru**:

```powershell
docker build --platform linux/amd64 -t <username-dockerhub>/dashboard-kopi:v2 .
docker push <username-dockerhub>/dashboard-kopi:v2
```

Di VM, ganti container lama dengan versi baru:

```bash
docker rm -f andi-dashboard

docker run -d --name andi-dashboard --restart unless-stopped \
  -p 8501:8501 -e PEMILIK=Andi \
  <username-dockerhub>/dashboard-kopi:v2
```

Refresh browser. Caption sekarang menunjukkan **Versi v2**, dan ada sidebar filter.

**Simulasi:** ternyata ada bug di v2 dan kita harus kembali ke versi lama secepatnya. Karena v1 masih tersimpan di registry, **rollback** cukup satu langkah:

```bash
docker rm -f andi-dashboard

docker run -d --name andi-dashboard --restart unless-stopped \
  -p 8501:8501 -e PEMILIK=Andi \
  <username-dockerhub>/dashboard-kopi:v1
```

Setelah itu, kembalikan lagi ke `v2` untuk dipakai di bagian selanjutnya.

> 💡 **Kenapa jangan hanya memakai tag `latest`?** Kalau semua versi di-push sebagai `latest`, kita tidak pernah tahu persis versi mana yang sedang berjalan, dan tidak bisa rollback ke versi sebelumnya karena `latest` sudah tertimpa. Tag versi = riwayat yang bisa dipercaya.

<img width="1200" height="988" alt="Screenshot 2026-11-17 230233" src="https://github.com/user-attachments/assets/667ab055-a5e0-4492-badf-995abbba286b" />


---

## 9. Hands-on 4: Kerja Tim di Satu Server

Di dunia kerja, satu server hampir selalu dipakai oleh **banyak orang**. Kesalahan paling umum (dan paling berbahaya) adalah **satu akun dipakai bersama**, misalnya semua orang login sebagai `azureuser` dengan private key yang dibagikan lewat grup chat. Akibatnya:

- Tidak ada yang tahu **siapa** melakukan **apa**
- Kalau satu orang keluar dari tim, aksesnya **tidak bisa dicabut** tanpa mengganti key semua orang
- Private key yang beredar di chat = private key yang **bocor**

Aturan emasnya: **satu orang = satu akun = satu key.**

Kerjakan bagian ini **berpasangan**. Sebagai contoh, **Andi** adalah pemilik VM dan **Budi** adalah teman yang diberi akses. Setelah selesai, **tukar peran**: Budi memberi akses ke VM miliknya untuk Andi.

### 9.1 Teman Mengirim Public Key

**Budi** (di laptopnya) menampilkan **public key**-nya:

```powershell
Get-Content ~\.ssh\id_ed25519.pub
```

Budi lalu mengirim **satu baris itu** ke Andi lewat chat. Ini aman, karena yang dikirim adalah **gembok**, bukan kunci.

> ⚠️ Kalau ada teman yang mengirim file **tanpa** `.pub` (private key), itu salah. Minta dia membuat key baru, karena key lama sudah dianggap bocor.

### 9.2 Pemilik VM Membuat Akun untuk Teman

**Andi** login ke VM miliknya (`ssh modul6`), lalu:

```bash
# 1. Buat user baru (nama user huruf kecil)
sudo useradd -m -s /bin/bash budi

# 2. Siapkan folder .ssh milik budi
sudo mkdir -p /home/budi/.ssh

# 3. Pasang public key Budi (ganti dengan baris yang dikirim Budi)
echo "ssh-ed25519 AAAAC3Nza...xyz budi@laptop" | sudo tee /home/budi/.ssh/authorized_keys

# 4. Atur kepemilikan dan permission (SSH menolak login kalau permission terlalu longgar)
sudo chown -R budi:budi /home/budi/.ssh
sudo chmod 700 /home/budi/.ssh
sudo chmod 600 /home/budi/.ssh/authorized_keys

# 5. Izinkan budi menjalankan docker
sudo usermod -aG docker budi
```

Cek user yang ada di server:

```bash
ls /home
id budi
```

> ⚠️ **Hati-hati dengan group `docker`.** Anggota group `docker` bisa menjalankan container dengan akses penuh ke server, dan secara praktis **setara dengan root**. Berikan hanya ke orang yang memang perlu men-deploy container. Untuk teman yang hanya perlu melihat-lihat atau mengambil file, cukup buat user biasa **tanpa** langkah 5.

### 9.3 Teman Login dengan Akunnya Sendiri

**Budi** (di laptopnya) menambahkan shortcut di `~/.ssh/config`:

```
Host vm-andi
    HostName <IP-VM-ANDI>
    User budi
    IdentityFile ~/.ssh/id_ed25519
```

Lalu login:

```powershell
ssh vm-andi
```

Prompt akan menunjukkan `budi@vm-modul6:~$`.

<img width="736" height="219" alt="Screenshot 2026-11-17 231553" src="https://github.com/user-attachments/assets/a13bb4ce-78d6-481c-8acd-eb7e05aa2d43" />


### 9.4 Teman Men-deploy Dashboard-nya Sendiri

Budi sudah punya image sendiri di Docker Hub dari Hands-on 3. Di VM Andi, Budi menjalankan dashboard miliknya di **port yang berbeda**:

```bash
docker run -d --name budi-dashboard --restart unless-stopped \
  -p 8502:8501 -e PEMILIK=Budi \
  <username-dockerhub-budi>/dashboard-kopi:v2
```

Perhatikan `-p 8502:8501`: port **8502** di VM diteruskan ke port 8501 di dalam container. Dua container boleh memakai port yang sama **di dalam container**, tetapi port di **VM** harus berbeda.

Cek:

```bash
docker ps
```

```
NAMES             IMAGE                                 PORTS
budi-dashboard    budi123/dashboard-kopi:v2             0.0.0.0:8502->8501/tcp
andi-dashboard    andi456/dashboard-kopi:v2             0.0.0.0:8501->8501/tcp
```

Sekarang ada dua dashboard di satu server:

- `http://<IP-VM-ANDI>:8501` → dashboard Andi
- `http://<IP-VM-ANDI>:8502` → dashboard Budi

<img width="1333" height="276" alt="Screenshot 2026-11-17 231826" src="https://github.com/user-attachments/assets/7cd52c76-dac3-4bdb-aebf-248d2945fdc3" />


### 9.5 Siapa Saja yang Ada di Server?

Karena setiap orang punya akun sendiri, semua aktivitas login **tercatat atas nama masing-masing**:

```bash
who            # siapa yang sedang login sekarang
last -n 10     # riwayat 10 login terakhir
```

Contoh output `last`:

```
budi     pts/1    114.xx.xx.xx    Mon Nov 16 20:15   still logged in
azureuser pts/0   36.xx.xx.xx     Mon Nov 16 19:50   still logged in
```

Kalau semua orang login sebagai `azureuser`, informasi ini tidak ada artinya.

### 9.6 Papan Pengumuman Server (MOTD)

Supaya semua orang tahu aturan dan pembagian port, tulis **papan pengumuman** yang otomatis muncul setiap kali ada yang login. **Andi** menjalankan:

```bash
sudo tee /etc/motd > /dev/null << 'EOF'
==========================================================
  SERVER vm-modul6  |  Pemilik: Andi
----------------------------------------------------------
  ATURAN:
  1. Satu orang satu akun. Jangan pinjam akun orang lain.
  2. Nama container wajib pakai prefix nama: <nama>-xxx
  3. Jangan stop/hapus container milik orang lain.
  4. Proses lama (training, dll) jalankan di dalam tmux.

  PEMBAGIAN PORT:
  8501  andi-dashboard
  8502  budi-dashboard
  8503-8510  kosong (lapor ke Andi sebelum pakai)
==========================================================
EOF
```

Logout lalu login lagi. Pengumuman akan muncul di atas prompt.

<img width="675" height="402" alt="Screenshot 2026-11-17 232103" src="https://github.com/user-attachments/assets/1c44e2f0-e93d-4eff-92a0-920510b35c57" />


### 9.7 Proses Lama dengan `tmux`

Anak data science sering menjalankan proses yang lama, misalnya training model atau scraping. Masalahnya, **kalau koneksi SSH putus, proses ikut mati.**

Solusinya adalah `tmux`: "layar virtual" di server yang tetap hidup walaupun kita logout.

```bash
# buat sesi baru bernama "training"
tmux new -s training
```

Di dalam tmux, jalankan proses yang lama, misalnya simulasi berikut:

```bash
for i in $(seq 1 300); do echo "epoch $i"; sleep 2; done
```

Sekarang **lepaskan** (*detach*) sesi tanpa mematikan proses: tekan `Ctrl + b`, lepas, lalu tekan `d`.

Kita boleh logout, menutup laptop, lalu besok login lagi:

```bash
tmux ls                 # lihat daftar sesi
tmux attach -t training # masuk lagi ke sesi
```

Proses masih berjalan. Untuk keluar dan menutup sesi, hentikan proses (`Ctrl + c`) lalu ketik `exit`.

### 9.8 Mencabut Akses Teman

Ketika ada anggota yang keluar dari tim, **cabut aksesnya**. Inilah keuntungan satu orang satu akun: yang lain tidak terganggu sama sekali.

Andi menjalankan:

```bash
docker rm -f budi-dashboard        # hentikan container milik budi (setelah koordinasi)
sudo userdel -r budi               # hapus user beserta home folder-nya
```

Coba Budi login lagi. Aksesnya sudah **ditolak**:

```
budi@20.xx.xx.xx: Permission denied (publickey).
```

> 💡 Untuk praktikum ini, lakukan langkah 9.8 **setelah** semua screenshot untuk Soal Latihan selesai diambil.

### 9.9 Rangkuman Aturan Kerja Tim

| ✅ Lakukan | ❌ Jangan |
| :-------- | :-------- |
| Satu orang satu akun, satu SSH key | Berbagi akun `azureuser` atau private key |
| Kirim **public key** (`.pub`) ke pemilik server | Mengirim private key lewat chat/email |
| Beri prefix nama pada container (`budi-xxx`) | Memberi nama container generik (`app`, `test`) |
| Catat pembagian port di MOTD/README server | Asal pakai port tanpa bilang-bilang |
| Hanya stop/hapus container milik sendiri | Menghapus container orang lain tanpa izin |
| Jalankan proses lama di `tmux` | Menjalankan training lalu menutup laptop |
| Beri group `docker`/`sudo` hanya yang perlu | Memberi semua orang akses root "biar gampang" |
| Cabut akses anggota yang sudah keluar | Membiarkan akun lama tetap aktif |

---

## 10. Bersih-bersih Biaya

### 10.1 Stop vs Deallocate

| Cara | Status VM | Biaya compute (CPU/RAM) | Biaya disk |
| :--- | :-------- | :---------------------- | :--------- |
| `sudo shutdown now` dari **dalam** VM | Stopped | ⚠️ **Tetap ditagih** | Ditagih |
| Tombol **Stop** di Azure Portal | **Stopped (deallocated)** | ✅ Tidak ditagih | Ditagih |
| Auto-shutdown (yang kita pasang di 6.4) | Stopped (deallocated) | ✅ Tidak ditagih | Ditagih |
| **Delete Resource Group** | Hilang | ✅ Tidak ditagih | ✅ Tidak ditagih |

> ⚠️ Mematikan VM dari dalam OS **tidak** menghentikan tagihan. Selalu matikan dari **Portal** dan pastikan statusnya **Stopped (deallocated)**.

> 💡 Setelah VM di-*deallocate* lalu dinyalakan lagi, **Public IP bisa berubah** (kecuali IP-nya statis). Kalau tidak bisa SSH, cek IP terbaru di halaman Overview, lalu perbarui `HostName` di `~/.ssh/config`.

### 10.2 Hapus Semua Setelah Selesai

Setelah praktikum dan Soal Latihan selesai dinilai, hapus semuanya:

1. Buka **Resource groups → rg-modul6**.
2. Klik **Delete resource group**.
3. Ketik nama resource group untuk konfirmasi, lalu klik **Delete**.

Pastikan setelah beberapa menit tidak ada resource yang tersisa di menu **All resources**.

### 10.3 Cek Sisa Kredit

Buka **Education Hub** di portal. Biasakan mengecek sisa kredit setiap selesai praktikum.

<img width="1918" height="910" alt="Screenshot 2026-11-17 234604" src="https://github.com/user-attachments/assets/8dcb0a6d-f4e3-4a18-9594-e4e484e6da36" />


---

## 11. Troubleshooting

| Masalah | Solusi |
| :------ | :----- |
| Membuat VM gagal dengan `RequestDisallowedByAzure` | Region yang dipilih tidak diizinkan untuk subscription kamu. Cek daftarnya di [5.2](#52-cek-region-yang-diizinkan), lalu buat **Resource Group dan VM** di region tersebut |
| Ukuran `Standard_B1s` tidak muncul / "not available" | Klik **See all sizes** dan cari `B1s`. Kalau tetap tidak ada, coba `Standard_B2ats_v2` atau region lain yang diizinkan. Kalau ada error terkait *Security type*, ubah menjadi **Standard** |
| `ssh: connect to host ... port 22: Connection timed out` | VM sedang mati (cek status di portal), IP berubah setelah VM dinyalakan ulang, atau rule SSH (22) tidak ada di NSG |
| `Permission denied (publickey)` | Public key yang dipasang tidak cocok dengan private key di laptop. Untuk akun teman: cek isi `authorized_keys` dan permission (`700` untuk `.ssh`, `600` untuk `authorized_keys`, pemilik harus user tersebut) |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | Biasanya karena VM dibuat ulang dengan IP yang sama. Hapus entri lama: `ssh-keygen -R <IP-VM>` |
| `~/.ssh/config` tidak terbaca di Windows | Pastikan nama file `config`, bukan `config.txt`. Cek dengan `dir ~\.ssh` |
| `permission denied while trying to connect to the Docker daemon socket` | User belum masuk group `docker`, atau belum logout-login setelah `usermod -aG docker` |
| Container langsung berhenti, `docker logs` berisi `exec format error` | Image di-build untuk Arm (Mac M-series) tetapi VM x64. Build ulang dengan `--platform linux/amd64`, atau VM yang dipilih berbasis Arm |
| `docker push` ditolak (`denied: requested access to the resource is denied`) | Belum `docker login`, atau nama image tidak diawali username Docker Hub kamu sendiri |
| Dashboard tidak bisa dibuka di `http://<IP-VM>:8501` | Cek `docker ps` (container jalan?), cek rule port `8501-8510` di NSG, dan pastikan memakai `http://`, bukan `https://` |
| `Bind for 0.0.0.0:8501 failed: port is already allocated` | Port sudah dipakai container lain. Pakai port lain di sisi VM, misal `-p 8503:8501`, lalu catat di MOTD |
| Menu Budgets belum bisa dipakai | Subscription baru butuh waktu hingga ±48 jam sebelum Cost Management aktif sepenuhnya |

---

## Soal Latihan

Kamu dan satu teman berperan sebagai tim data science sebuah kedai kopi. Pemilik kedai ingin bisa membuka dashboard penjualan dari HP, dan tim harus bisa bekerja di server yang sama tanpa saling ganggu. Ketentuan:

1. Buat **dataset sendiri** (minimal 12 baris, tema bebas, **bukan** data kopi dari modul ini) dan aplikasi Streamlit yang menampilkan:
   - minimal **1 metric**
   - minimal **1 grafik**
   - caption berisi **versi** dan **nama pemilik** (dari environment variable)
2. Build image dan push ke Docker Hub dengan **dua tag versi**: `v1` dan `v2`. `v2` harus punya minimal satu fitur tambahan yang terlihat di browser.
3. Deploy **v2** ke VM milikmu di Azure dengan nama container berprefix namamu dan opsi `--restart unless-stopped`.
4. Lakukan **rollback** ke `v1`, lalu kembali ke `v2`.
5. Beri akses VM-mu kepada teman dengan **akun dan SSH key miliknya sendiri**. Teman harus men-deploy dashboard miliknya di VM-mu pada port yang berbeda.
6. Tulis **MOTD** berisi aturan server dan pembagian port.
7. Kumpulkan **screenshot** berikut:
   - Halaman Overview VM (region, size, dan status terlihat)
   - Isi `~/.ssh/config` di laptopmu
   - Halaman Docker Hub yang menunjukkan tag `v1` dan `v2`
   - Dashboard v2 milikmu diakses lewat `http://<IP-VM>:<port>`
   - Output `docker ps` di VM-mu yang menunjukkan **dua** container (milikmu dan milik teman)
   - Output `last -n 10` yang menunjukkan login akun temanmu
   - MOTD yang muncul saat login
8. Setelah dinilai, **hapus Resource Group** dan screenshot halaman sisa kredit di Education Hub.

---

## Done 🎉🎉

Pada modul ini kita telah:

1. Memahami konsep cloud provider, IaaS/PaaS/SaaS, dan struktur Subscription → Resource Group → Resource di Azure
2. Mengenal layanan-layanan Azure yang berguna untuk data science
3. Mengaktifkan Azure for Students dan memasang budget alert agar kredit aman
4. Menyewa VM Ubuntu di Azure dan masuk menggunakan **SSH key**
5. Membuat shortcut SSH, mengirim file dengan `scp`, dan memeriksa keamanan SSH server
6. Men-deploy dashboard Streamlit dengan alur profesional: **build → push ke registry → pull di server**, lengkap dengan tag versi dan rollback
7. Bekerja dalam tim di satu server: akun terpisah, pembagian port, MOTD, `tmux`, dan mencabut akses
8. Membersihkan resource agar kredit tidak terbuang

*Modul 6 — Cloud Computing dengan Microsoft Azure*
*Cloud Computing · Zein M Hasan · Teknologi Informasi ITS*
