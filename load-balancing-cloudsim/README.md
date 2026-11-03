# Modul 5 - Load Balancing dengan NGINX & Load Testing dengan Locust

## Daftar Isi

1. [Pendahuluan](#1-pendahuluan)
2. [Konsep Load Balancing](#2-konsep-load-balancing)
3. [Tools yang Digunakan](#3-tools-yang-digunakan)
4. [Implementasi Load Balancer Sederhana](#4-implementasi-load-balancer-sederhana)
5. [Menjalankan & Menguji Load Balancer](#5-menjalankan--menguji-load-balancer)
6. [Setup MongoDB & Uji Endpoint Database](#6-setup-mongodb--uji-endpoint-database)
7. [Load Testing dengan Locust](#7-load-testing-dengan-locust)
8. [Eksperimen Algoritma: Least Connection](#8-eksperimen-algoritma-least-connection)
9. [Bonus: Uji Ketahanan (Failover)](#9-bonus-uji-ketahanan-failover)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Pendahuluan

Dalam sistem berbasis komputasi awan, aplikasi tidak hanya berjalan di satu server, tetapi tersebar di banyak node, virtual machine, atau container yang bekerja secara bersamaan. Setiap permintaan dari pengguna perlu diarahkan ke salah satu dari sekian banyak resource yang tersedia.

Semakin tinggi jumlah pengguna dan permintaan, maka muncul tantangan utama:

1. Bagaimana cara kita menjaga ketersediaan layanan ketika salah satu server mengalami kegagalan?
2. Bagaimana kita memastikan tidak ada satu server pun yang kelebihan beban sementara server lain menganggur?
3. Bagaimana kita dapat mendistribusikan beban kerja secara efisien agar performa sistem tetap optimal?

Untuk menjawab tantangan tersebut, digunakanlah sebuah konsep yang disebut dengan **load balancing**.

---

## 2. Konsep Load Balancing

### 2.1 Apa itu Load Balancing?

Load balancing adalah sebuah mekanisme yang mengatur distribusi traffic atau beban komputasi ke beberapa server secara merata dan cerdas.

![Gambaran Load Balancing](img/load-balancing-visualized.png)

Contoh penyedia layanan cloud load balancing:

- Amazon Web Services (AWS) menggunakan **Elastic Load Balancing (ELB)**
- Google Cloud Platform (GCP) menggunakan **Google Cloud Load Balancing**
- Microsoft Azure menggunakan **Azure Load Balancer**

### 2.2 Load Balancing Algorithms

Sama seperti berbagai sistem di dunia IT, load balancing juga memiliki beberapa algoritma. Setiap algoritma memiliki logika masing-masing dalam menentukan server mana yang paling tepat untuk menerima traffic pada saat itu.

![Algorithms](img/algorithms.gif)

Load balancing dapat dibagi menjadi 2 kelompok:

1. **Static** — keputusan tidak melihat kondisi server saat ini
   - **Round robin**
     Request dibagi secara bergiliran (rotasi): 1 → A, 2 → B, 3 → C, 4 → A, dst.
   - **Weighted round robin**
     Seperti round robin, tetapi tiap server punya bobot. Server yang mampu menangani traffic lebih besar diberi weight lebih besar.
   - **IP hash**
     IP address client diubah menjadi hash. Berdasarkan hash tersebut, client yang sama selalu diarahkan ke server yang sama (berguna untuk *session*).

2. **Dynamic** — keputusan melihat kondisi server saat ini
   - **Least connection**
     Request dikirim ke server yang sedang memiliki koneksi aktif paling sedikit (asumsi seluruh kekuatan proses server sama).
   - **Weighted response time**
     Melihat rata-rata waktu respons tiap server, sekaligus jumlah koneksi pada server.
   - **Resource-based**
     Melihat sumber daya (mis. CPU, RAM) pada server. Memerlukan *agent* yang memonitor sumber daya server.

Dan masih banyak lagi!

---

## 3. Tools yang Digunakan

| Tool               | Fungsi                                                                                     |
| :----------------- | :----------------------------------------------------------------------------------------- |
| **Docker Compose** | Menjalankan seluruh service (3 aplikasi, NGINX, MongoDB, Mongo Express) sekaligus          |
| **FastAPI**        | Framework Python untuk membuat aplikasi backend (API)                                      |
| **MongoDB**        | Database yang dipakai bersama oleh ketiga aplikasi                                         |
| **NGINX**          | Load balancer — pintu masuk utama yang membagi request ke 3 aplikasi backend               |
| **Locust**         | Tool load testing — mensimulasikan banyak user sekaligus untuk mengukur performa sistem    |

### 3.1 NGINX

NGINX adalah program _open-source_ berkinerja tinggi yang sering digunakan untuk _web serving_, _reverse proxying_, _caching_, dan **load balancing**. Pada praktikum ini, NGINX bertindak sebagai pintu masuk utama yang membagi _request_ API dari _client_ menuju tiga aplikasi _backend_ yang berbeda.

### 3.2 Locust

Locust adalah tool _load testing_ berbasis Python. Kita cukup menulis perilaku user dalam file Python, lalu Locust akan mensimulasikan puluhan hingga ribuan user yang mengirim request secara bersamaan, lengkap dengan statistik dan grafik real-time.

### 3.3 Arsitektur Sistem

```
                                        ┌──► app  (Server A) :8000 ──┐
Client ──► localhost:80 ──► NGINX ──────┼──► app2 (Server B) :8000 ──┼──► MongoDB :27017
                                        └──► app3 (Server C) :8000 ──┘        ▲
                                                                              │
                                                       Mongo Express :8082 ───┘
```

---

## 4. Implementasi Load Balancer Sederhana

### 4.0 Persiapan

1. **Docker Desktop** (sudah termasuk Docker Compose) — sudah dijelaskan di modul sebelumnya ;)

   Cek instalasi:

   ```powershell
   docker --version
   docker compose version
   ```

2. **Library Python** — FastAPI, Uvicorn, PyMongo, dan Pydantic akan otomatis diinstall **di dalam container** lewat `requirements.txt`. Jadi tidak wajib diinstall di laptop (kecuali untuk keperluan autocomplete di code editor):

   ```powershell
   pip install fastapi uvicorn pymongo pydantic
   ```

3. **Locust** — yang ini diinstall di laptop karena dijalankan dari luar Docker:

   ```powershell
   pip install locust
   ```

### 4.1 Menyiapkan Struktur Folder/File

Struktur akhir project:

```
modul5/
├── app/
│   ├── Dockerfile
│   ├── main.py
│   └── requirements.txt
├── app2/
│   ├── Dockerfile
│   ├── main.py
│   └── requirements.txt
├── app3/
│   ├── Dockerfile
│   ├── main.py
│   └── requirements.txt
├── nginx/
│   ├── Dockerfile
│   └── nginx.conf
├── docker-compose.yml
└── locustfile.py
```

Bisa dibuat cepat lewat PowerShell (jalankan di dalam folder `modul5`):

```powershell
mkdir app, app2, app3, nginx

New-Item -ItemType File app\Dockerfile, app\main.py, app\requirements.txt
New-Item -ItemType File app2\Dockerfile, app2\main.py, app2\requirements.txt
New-Item -ItemType File app3\Dockerfile, app3\main.py, app3\requirements.txt
New-Item -ItemType File nginx\Dockerfile, nginx\nginx.conf
New-Item -ItemType File docker-compose.yml, locustfile.py
```

Cek hasilnya dengan `tree /f`.

### 4.2 Konfigurasi Aplikasi (FastAPI)

Isi `app/main.py` dengan kode berikut:

```python
from fastapi import FastAPI, Body, Request
from fastapi.encoders import jsonable_encoder
import pymongo
from pydantic import BaseModel
from bson.objectid import ObjectId
import socket
import time

SERVER_NAME = "A"  # ganti jadi "B" di app2 dan "C" di app3

MONGO_DETAILS = "mongodb://admin:admin@mongodb:27017/"
client = pymongo.MongoClient(MONGO_DETAILS)
db = client['tes']          # nama database
collection = db['tes']      # nama collection

class Item(BaseModel):
    name: str
    age: int
    rank: str

def myData(data):
    return {
        "id": str(data["_id"]),
        "name": data["name"],
        "age": data["age"],
        "rank": data["rank"],
    }

def myFullData(datas):
    return [myData(data) for data in datas]

def ResponseModel(data, message="Success"):
    return {"data": [data], "code": 200, "message": message}

def ErrorResponseModel(error, code, message):
    return {"error": error, "code": code, "message": message}

app = FastAPI()

@app.get('/')
async def home():
    return {"message": f"This is server {SERVER_NAME}", "hostname": socket.gethostname()}

@app.get('/fast')
async def hello_fast():
    time.sleep(0.5)
    return {"message": f"Hello world from server {SERVER_NAME}", "opt": "fast"}

@app.get('/slow')
async def hello_slow():
    time.sleep(1)
    return {"message": f"Hello world from server {SERVER_NAME}", "opt": "slow"}

@app.get("/all")
async def get_all_data():
    return ResponseModel(myFullData(collection.find()), f"All Good from {SERVER_NAME}")

@app.post("/create", response_model=Item)
async def create_data(request: Request, lister: Item = Body(..., embed=True)):
    lister = jsonable_encoder(lister)
    collection.insert_one(lister.copy())
    return lister

@app.get("/get/{id}")
async def get_data(id: str):
    if not ObjectId.is_valid(id):
        return ErrorResponseModel("Invalid id.", 400, "ID format is wrong.")
    data = collection.find_one({"_id": ObjectId(id)})
    if data:
        return ResponseModel(myData(data), SERVER_NAME)
    return ErrorResponseModel("An error occurred.", 404, "Data doesn't exist.")
```

**Penjelasan endpoint:**

| Endpoint      | Method | Fungsi                                                         |
| :------------ | :----- | :------------------------------------------------------------- |
| `/`           | GET    | Menampilkan nama server & hostname container                   |
| `/fast`       | GET    | Simulasi proses ringan (0,5 detik)                             |
| `/slow`       | GET    | Simulasi proses berat (1 detik)                                |
| `/all`        | GET    | Mengambil seluruh data dari MongoDB                            |
| `/create`     | POST   | Menambah data baru ke MongoDB                                  |
| `/get/{id}`   | GET    | Mengambil satu data berdasarkan `_id`                          |

**Catatan perbaikan dari versi awal modul:**

- **`SERVER_NAME`** — cukup ganti 1 baris untuk tiap server, tidak perlu mengganti huruf "A" di banyak tempat.
- **Nama fungsi unik** (`hello_fast` & `hello_slow`) — sebelumnya keduanya bernama `hello`. Tetap berjalan, tetapi membingungkan.
- **`/create` memakai `collection`** — sebelumnya memakai `client.tes.tes` yang di-hardcode, sehingga tidak ikut berubah jika nama database diganti. Ditambah `.copy()` karena `insert_one` menyisipkan field `_id` (tipe `ObjectId`) ke dalam dict, yang bisa membuat response error.
- **Validasi id di `/get/{id}`** — sebelumnya id yang formatnya salah langsung menyebabkan error 500.

> 📌 **Penting untuk dipahami:** `time.sleep()` di dalam fungsi `async def` akan **memblokir** event loop, sehingga tiap server hanya bisa memproses **satu request dalam satu waktu**. Pada aplikasi sungguhan ini adalah praktik yang buruk, tetapi untuk modul ini justru membantu: efek load balancing jadi terlihat sangat jelas saat load testing (3 server ≈ 3x lebih kuat dari 1 server).

### 4.3 Konfigurasi Dockerfile

Isi `app/Dockerfile`:

```docker
FROM python:3.9-slim
WORKDIR /app
COPY . /app
RUN pip install --no-cache-dir -r requirements.txt
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Docker akan membuat folder `/app`, menyalin seluruh file, lalu menginstall semua library di `requirements.txt`. Saat container dijalankan, `uvicorn` menjalankan objek `app` di dalam file `main.py` pada port `8000`.

### 4.4 Konfigurasi requirements.txt

Isi `app/requirements.txt`:

```
fastapi==0.78.0
uvicorn==0.18.2
pymongo
pydantic<2
```

- `pydantic<2` — FastAPI 0.78 hanya kompatibel dengan Pydantic versi 1, jadi versinya dikunci agar aman.
- `uuid` dihapus dari versi awal modul — `uuid` sudah merupakan library bawaan Python (paket `uuid` di pip adalah paket lama untuk Python 2), dan juga tidak dipakai di kode.

### 4.5 Salin ke app2 dan app3

Setelah ketiga file di `app/` selesai, salin ke `app2` dan `app3`:

```powershell
Copy-Item app\* app2\ -Force
Copy-Item app\* app3\ -Force
```

Lalu ubah `SERVER_NAME` di `app2\main.py` menjadi `"B"` dan di `app3\main.py` menjadi `"C"`. Cek hasilnya:

```powershell
Select-String -Path app*\main.py -Pattern 'SERVER_NAME ='
```

Output yang diharapkan:

```
app\main.py:9:SERVER_NAME = "A"
app2\main.py:9:SERVER_NAME = "B"
app3\main.py:9:SERVER_NAME = "C"
```

### 4.6 Konfigurasi NGINX

Isi `nginx/Dockerfile`:

```docker
FROM nginx
RUN rm /etc/nginx/conf.d/default.conf
COPY ./nginx.conf /etc/nginx/conf.d/default.conf
```

Artinya: gunakan image NGINX resmi, hapus konfigurasi bawaannya, lalu ganti dengan konfigurasi kita.

Isi `nginx/nginx.conf`:

```nginx
upstream app {
    server app:8000;
    server app2:8000;
    server app3:8000;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://app;

        # opsional: menampilkan alamat container tujuan di response header
        add_header X-Upstream $upstream_addr always;
    }
}
```

**Penjelasan:**

- **`upstream app { ... }`** — daftar server tujuan. `app`, `app2`, `app3` adalah **nama service di `docker-compose.yml`**. Docker otomatis menerjemahkan nama service menjadi IP container (DNS internal Docker).
- **Port `8000`** — port **di dalam container** tempat uvicorn berjalan, bukan port 8001/8002/8003. NGINX berkomunikasi lewat jaringan internal Docker, bukan lewat laptop.
- **`proxy_pass http://app`** — `app` di sini merujuk ke **nama blok `upstream`** di atas. Jika nama upstream diganti (misalnya `backend`), maka `proxy_pass` juga harus menjadi `http://backend`.
- **Algoritma** — karena tidak ada algoritma yang ditulis di dalam `upstream`, NGINX otomatis menggunakan **Round Robin**.

**Variasi algoritma lain** (cukup ubah blok `upstream`):

```nginx
# Weighted round robin: server A mendapat 3x lebih banyak request
upstream app {
    server app:8000 weight=3;
    server app2:8000;
    server app3:8000;
}

# Least connection: kirim ke server dengan koneksi aktif paling sedikit
upstream app {
    least_conn;
    server app:8000;
    server app2:8000;
    server app3:8000;
}

# IP hash: client yang sama selalu diarahkan ke server yang sama
upstream app {
    ip_hash;
    server app:8000;
    server app2:8000;
    server app3:8000;
}
```

### 4.7 Konfigurasi docker-compose

Isi `docker-compose.yml`:

```yaml
services:
  app:
    build: ./app
    ports:
      - "8001:8000"
    depends_on:
      - mongodb
  app2:
    build: ./app2
    ports:
      - "8002:8000"
    depends_on:
      - mongodb
  app3:
    build: ./app3
    ports:
      - "8003:8000"
    depends_on:
      - mongodb
  nginx:
    build: ./nginx
    ports:
      - "80:80"
    depends_on:
      - app
      - app2
      - app3
  mongodb:
    image: mongo
    ports:
      - "27017:27017"
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=admin
    volumes:
      - mongodb_data:/data/db
  mongo-express:
    image: mongo-express
    restart: unless-stopped
    ports:
      - "8082:8081"
    environment:
      - ME_CONFIG_MONGODB_ADMINUSERNAME=admin
      - ME_CONFIG_MONGODB_ADMINPASSWORD=admin
      - ME_CONFIG_MONGODB_SERVER=mongodb
      - ME_CONFIG_MONGODB_ENABLE_ADMIN=true
      - ME_CONFIG_BASICAUTH_USERNAME=admin
      - ME_CONFIG_BASICAUTH_PASSWORD=admin
    depends_on:
      - mongodb
volumes:
  mongodb_data:
    driver: local
```

**Penjelasan:**

- **`"8001:8000"`** — format `port_laptop:port_container`. Ketiga app sama-sama berjalan di port `8000` di dalam container masing-masing, tetapi di laptop harus memakai port berbeda (**8001**, **8002**, **8003**) agar tidak bentrok.
- **`depends_on`** — memastikan MongoDB dinyalakan lebih dulu sebelum aplikasi, dan ketiga aplikasi dinyalakan sebelum NGINX.
- **`restart: unless-stopped` pada mongo-express** — `depends_on` hanya menunggu container MongoDB **menyala**, bukan **siap menerima koneksi**. Tanpa opsi ini, mongo-express sering crash karena terlalu cepat mencoba terhubung.
- **Port `443` dihapus** dari versi awal modul karena kita tidak menggunakan HTTPS.
- **`volumes`** — data MongoDB disimpan di volume `mongodb_data`, sehingga tidak hilang saat container dimatikan.

---

## 5. Menjalankan & Menguji Load Balancer

### 5.1 Cek Port 80 (khusus Windows)

Port 80 sering sudah dipakai aplikasi lain (IIS, XAMPP/Apache, Laragon). Cek dulu:

```powershell
Get-NetTCPConnection -LocalPort 80 -State Listen -ErrorAction SilentlyContinue
```

- **Tidak ada output** → aman.
- **Ada output** → matikan aplikasi tersebut, atau ubah port nginx di `docker-compose.yml` menjadi `"8080:80"` lalu akses lewat `localhost:8080`.

### 5.2 Jalankan Docker Compose

```powershell
docker compose up --build
```

> `docker-compose` (pakai tanda hubung) adalah perintah versi lama, `docker compose` (pakai spasi) adalah versi baru. Keduanya umumnya sama-sama berjalan.

Build pertama membutuhkan beberapa menit karena harus mengunduh image Python, NGINX, dan MongoDB. Biarkan terminal ini terbuka untuk melihat log seluruh container.

> 📸 **[SCREENSHOT]** Terminal setelah `docker compose up --build` berhasil (seluruh container berjalan)
>
> ![docker compose up](img/docker-compose-up.png)

### 5.3 Uji Round Robin

Buka **terminal PowerShell baru**, lalu jalankan:

```powershell
1..6 | ForEach-Object { (Invoke-RestMethod http://localhost/).message }
```

Hasil:

```
This is server A
This is server B
This is server C
This is server A
This is server B
This is server C
```

Request dibagi **bergiliran** ke server A → B → C → A → ... Inilah **Round Robin**.

> 📸 **[SCREENSHOT]** Output perintah di atas di PowerShell
>
> ![round robin powershell](img/round-robin-powershell.png)

Bisa juga dengan membuka `http://localhost` di browser lalu refresh beberapa kali.

> 📸 **[SCREENSHOT]** Browser menampilkan Server A, B, dan C
>
> ![Server A](img/A.png)
>
> ![Server B](img/B.png)
>
> ![Server C](img/C.png)

> 💡 Di browser, urutannya kadang tidak rapi A → B → C. Penyebabnya, browser diam-diam ikut mengirim request `/favicon.ico` yang juga "memakan giliran" round robin. Karena itu pengujian lewat PowerShell lebih akurat.

Masing-masing server juga dapat diakses langsung **tanpa melewati load balancer** di `localhost:8001`, `localhost:8002`, dan `localhost:8003`.

---

## 6. Setup MongoDB & Uji Endpoint Database

### 6.1 Membuat Database, Collection, dan Dokumen

Buka `http://localhost:8082`, lalu login dengan username `admin` dan password `admin`.

> Jika halaman belum bisa dibuka, tunggu 10–20 detik lalu refresh (mongo-express sedang menunggu MongoDB siap).

> 📸 **[SCREENSHOT]** Halaman utama Mongo Express
>
> ![mongodb](img/mongodb.png)

Langkah-langkah:

1. Isi **Database Name** dengan `tes` → klik **Create Database**
2. Buka database `tes`
3. Isi **Collection Name** dengan `tes` → klik **Create Collection**
4. Buka collection `tes` → klik **New Document**
5. Isi dengan dokumen berikut, lalu **Save**:

```json
{
    "name": "Lorem ipsum",
    "age": 20,
    "rank": "admin"
}
```

> Field `_id` tidak perlu ditulis, MongoDB akan membuatnya otomatis.
>
> Nama database dan collection harus sesuai dengan yang ada di `main.py` (`client['tes']` dan `db['tes']`). Jika ingin memakai nama lain, ubah juga di `main.py` ketiga app lalu build ulang.

> 📸 **[SCREENSHOT]** Collection `tes` beserta dokumen yang sudah dibuat
>
> ![database-viewing](img/collection.png)

### 6.2 Uji `/all` — Ambil Semua Data

```powershell
1..3 | ForEach-Object { Invoke-RestMethod http://localhost/all | ConvertTo-Json -Depth 5 }
```

Perhatikan field `"message"`: berganti-ganti `All Good from A`, `B`, `C`, tetapi **datanya selalu sama**. Hal ini karena ketiga server membaca dari **satu MongoDB yang sama**.

Inilah syarat utama load balancing: server harus **stateless** (tidak menyimpan data sendiri), sehingga request yang masuk ke server mana pun akan menghasilkan data yang konsisten.

> 📸 **[SCREENSHOT]** Output `/all` dari ketiga server
>
> ![all](img/all.png)

### 6.3 Uji `/create` — Tambah Data

```powershell
$body = @{ lister = @{ name = "Budi"; age = 21; rank = "user" } } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri http://localhost/create -Body $body -ContentType "application/json"
```

> Data dibungkus dengan key `lister` karena di `main.py` terdapat `Body(..., embed=True)`. Dengan opsi ini FastAPI mengharapkan JSON berbentuk `{"lister": {...}}`.

Refresh Mongo Express, dokumen "Budi" akan muncul.

> 📸 **[SCREENSHOT]** Output `/create` dan dokumen baru di Mongo Express
>
> ![create](img/create.png)

### 6.4 Uji `/get/{id}` — Ambil Satu Data

```powershell
$id = (Invoke-RestMethod http://localhost/all).data[0][0].id
$id
Invoke-RestMethod http://localhost/get/$id | ConvertTo-Json -Depth 5
```

Coba juga dengan id yang tidak valid:

```powershell
Invoke-RestMethod http://localhost/get/ngawur
```

> 📸 **[SCREENSHOT]** Output `/get/{id}` (id valid dan id tidak valid)
>
> ![get by id](img/get-by-id.png)

### 6.5 Alternatif: Swagger UI

FastAPI otomatis menyediakan halaman dokumentasi interaktif di **`http://localhost/docs`**. Pilih endpoint → **Try it out** → **Execute**, tanpa perlu mengetik perintah di terminal.

> 📸 **[SCREENSHOT]** Halaman Swagger UI
>
> ![swagger](img/swagger.png)

---

## 7. Load Testing dengan Locust

Sejauh ini kita sudah melihat request dibagi ke 3 server. Tetapi **seberapa besar** pengaruhnya terhadap performa? Kita buktikan dengan angka menggunakan Locust.

### 7.1 Membuat `locustfile.py`

Buat `locustfile.py` di folder `modul5`:

```python
from locust import HttpUser, task, between

class WebUser(HttpUser):
    # setiap user "berpikir" 1-2 detik sebelum mengirim request berikutnya
    wait_time = between(1, 2)

    @task(3)
    def fast(self):
        self.client.get("/fast")

    @task(1)
    def slow(self):
        self.client.get("/slow")
```

**Penjelasan:**

- **`HttpUser`** — merepresentasikan satu user virtual yang terus mengirim request.
- **`wait_time = between(1, 2)`** — jeda acak 1–2 detik antar request, agar perilakunya mirip manusia.
- **`@task(3)` dan `@task(1)`** — bobot task. `/fast` dipanggil 3x lebih sering dibanding `/slow` (sekitar 75% : 25%).

### 7.2 Menjalankan Locust (Web UI)

Pastikan Docker masih berjalan, lalu:

```powershell
locust -f locustfile.py
```

> Jika muncul error `locust is not recognized`, gunakan `python -m locust -f locustfile.py`.

Buka **`http://localhost:8089`** di browser.

> 📸 **[SCREENSHOT]** Halaman awal Locust Web UI
>
> ![locust home](img/locust-home.png)

**Tab-tab penting di Locust:**

| Tab               | Isi                                                                              |
| :---------------- | :------------------------------------------------------------------------------- |
| **Statistics**    | Tabel statistik per endpoint (jumlah request, median, persentil, average, RPS)   |
| **Charts**        | Grafik real-time: RPS, Response Times, Number of Users                           |
| **Failures**      | Daftar request yang gagal                                                        |
| **Current ratio** | Perbandingan task yang dijalankan (`/fast` vs `/slow`)                           |
| **Download Data** | Unduh hasil dalam bentuk CSV atau **HTML report** (cocok untuk lampiran laporan) |

**Cara membaca kolom Statistics:**

| Kolom             | Arti                                                                              |
| :---------------- | :-------------------------------------------------------------------------------- |
| **Median**        | 50% request selesai lebih cepat dari angka ini                                    |
| **95%ile / 99%ile** | 95% / 99% request selesai lebih cepat dari angka ini (menggambarkan request "paling sial") |
| **Average**       | Rata-rata response time                                                           |
| **Current RPS**   | Request per second **sesaat** (bukan rata-rata seluruh tes)                       |

### 7.3 Eksperimen: 1 Server vs 3 Server

Kita jalankan dua tes dengan beban **yang sama persis**, hanya berbeda target:

| Tes   | Host                     | Artinya                                              |
| :---- | :----------------------- | :--------------------------------------------------- |
| **A** | `http://localhost:8001`  | Langsung ke Server A saja, **tanpa** load balancer   |
| **B** | `http://localhost`       | Melalui NGINX, dibagi ke **3 server** (round robin)  |

Parameter yang digunakan pada kedua tes:

| Parameter                          | Nilai  |
| :--------------------------------- | :----- |
| Number of users                    | `20`   |
| Ramp up (users/second)             | `5`    |
| Advanced options → Run time        | `60s`  |

**Langkah:**

1. Isi form dengan parameter di atas dan Host `http://localhost:8001` → **Start** (Tes A).
2. Setelah selesai, screenshot tab **Statistics** dan **Charts**.
3. Klik **Reset** (agar grafik bersih), lalu **New**.
4. Isi parameter yang sama, ganti Host menjadi `http://localhost` → **Start** (Tes B).
5. Screenshot lagi tab **Statistics** dan **Charts**.

> 💡 Jika tidak menekan **Reset** di antara dua tes, grafik akan menampilkan garis diagonal panjang yang menyambungkan titik akhir Tes A dengan titik awal Tes B. Garis tersebut **bukan data**.

> Alternatif tanpa UI (hasil langsung tampil di terminal):
>
> ```powershell
> locust -f locustfile.py --host http://localhost:8001 -u 20 -r 5 -t 60s --headless
> locust -f locustfile.py --host http://localhost      -u 20 -r 5 -t 60s --headless
> ```

#### Hasil Tes A — 1 Server

> 📸 **[SCREENSHOT]** Tab Statistics — Tes A (`localhost:8001`)
>
> ![Tes A statistics](img/locust-1server-statistics.png)

> 📸 **[SCREENSHOT]** Tab Charts — Tes A (`localhost:8001`)
>
> ![Tes A charts](img/locust-1server-charts.png)

#### Hasil Tes B — 3 Server (Round Robin)

> 📸 **[SCREENSHOT]** Tab Statistics — Tes B (`localhost`)
>
> ![Tes B statistics](img/locust-3server-statistics.png)

> 📸 **[SCREENSHOT]** Tab Charts — Tes B (`localhost`)
>
> ![Tes B charts](img/locust-3server-charts.png)

### 7.4 Prediksi Teoritis

Karena tiap server hanya bisa memproses **satu request dalam satu waktu** (efek `time.sleep()` di `async def`), kapasitas maksimum sistem bisa dihitung:

- Rata-rata waktu proses per request = (3 × 0,5 detik + 1 × 1 detik) / 4 = **0,625 detik**
- Kapasitas 1 server = 1 / 0,625 ≈ **1,6 request/detik**
- Kapasitas 3 server ≈ 3 × 1,6 = **4,8 request/detik**

Dengan 20 user, permintaan jauh melebihi kapasitas 1 server, sehingga request akan **mengantre** dan response time membengkak.

### 7.5 Hasil & Analisis

| Metrik              | Tes A (1 server) | Tes B (3 server) | Prediksi                 |
| :------------------ | :--------------- | :--------------- | :----------------------- |
| Total request (60s) | 90               | 253              | –                        |
| Median              | 12.000 ms        | 3.000 ms         | –                        |
| 95%ile              | 16.000 ms        | 6.100 ms         | –                        |
| Average             | **10.459 ms**    | **3.031 ms**     | ~10.000 vs ~2.000–3.000  |
| RPS                 | **1,5**          | **4,2**          | ~1,6 vs ~4,8             |
| Failures            | 0%               | 0%               | –                        |

**Kesimpulan:** dengan beban yang sama (20 user), load balancer membuat response time rata-rata **±3,5x lebih cepat** dan throughput **±2,8x lebih besar**.

**Membaca grafik:**

- **Tes A** — garis RPS "mentok" di sekitar 1,5 (batas kemampuan 1 server). Grafik Response Times naik dari ±2 detik hingga ±15 detik lalu bertahan tinggi, karena antrean di server terus menumpuk.
- **Tes B** — RPS naik hingga ±4,2 dan response time tetap rendah di sekitar 3 detik, karena antrean dibagi ke tiga server.

**Mengapa Tes B sedikit di bawah prediksi (4,2 vs 4,8)?**
Round robin membagi request berdasarkan **giliran**, bukan berdasarkan **beban**. Bisa saja satu server kebetulan mendapat beberapa `/slow` (1 detik) berturut-turut, sementara server lain sudah menganggur. Akibatnya beban tidak benar-benar merata, dan sebagian request "sial" harus menunggu lama — terlihat dari **95%ile yang masih cukup tinggi (6.100 ms)**. Hal ini akan diuji pada [bagian 8](#8-eksperimen-algoritma-least-connection).

**Validasi dengan Little's Law**

Hukum antrean *Little's Law* menyatakan: jumlah user ≈ throughput × (response time + waktu tunggu antar request).

- Tes A: 1,5 × (10,5 + 1,5) ≈ **18 user**
- Tes B: 4,2 × (3,0 + 1,5) ≈ **19 user**

Keduanya mendekati 20 user (selisih kecil disebabkan fase ramp-up di awal tes), sehingga data hasil pengujian **konsisten dan valid**.

---

## 8. Eksperimen Algoritma: Least Connection

**Hipotesis:** dengan *least connection*, request baru selalu dikirim ke server yang antreannya paling sedikit, sehingga beban lebih merata dan request "sial" (tail latency) berkurang dibanding round robin.

### 8.1 Ubah Konfigurasi NGINX

Tambahkan `least_conn;` di dalam blok `upstream` pada `nginx/nginx.conf`:

```nginx
upstream app {
    least_conn;
    server app:8000;
    server app2:8000;
    server app3:8000;
}
```

Build ulang **hanya** container NGINX (aplikasi lain tetap berjalan):

```powershell
docker compose up -d --build nginx
```

Pastikan konfigurasi sudah masuk:

```powershell
docker compose exec nginx cat /etc/nginx/conf.d/default.conf
```

> Jika request dikirim satu per satu (seperti tes PowerShell di bagian 5.3), semua server sama-sama punya 0 koneksi aktif, sehingga NGINX tetap membagi secara bergiliran. Perbedaan `least_conn` baru terlihat saat ada **banyak request bersamaan**.

### 8.2 Jalankan Tes C di Locust

Di Locust: **Reset** → **New** → isi parameter yang sama dengan Tes B (Host `http://localhost`, 20 user, ramp up 5, run time 60s) → **Start**.

> 📸 **[SCREENSHOT]** Tab Statistics — Tes C (least connection)
>
> ![Tes C statistics](img/locust-leastconn-statistics.png)

> 📸 **[SCREENSHOT]** Tab Charts — Tes C (least connection)
>
> ![Tes C charts](img/locust-leastconn-charts.png)

### 8.3 Hasil & Analisis

| Metrik              | Round Robin (Tes B) | Least Conn (Tes C) | Perubahan  |
| :------------------ | :------------------ | :----------------- | :--------- |
| Total request (60s) | 253                 | **275**            | +9%        |
| Average             | 3.031 ms            | **2.664 ms**       | −12%       |
| Median              | 3.000 ms            | **2.500 ms**       | −17%       |
| 95%ile              | 6.100 ms            | **4.500 ms**       | **−26%**   |
| 99%ile              | 9.100 ms            | **5.900 ms**       | **−35%**   |
| Max                 | 9.566 ms            | **7.179 ms**       | −25%       |

**Hipotesis terbukti.** Peningkatan terbesar justru terjadi pada **request paling lambat** (99%ile turun 35%), bukan pada rata-rata (hanya turun 12%).

- **Round robin** tidak memperhatikan kondisi server, sehingga kadang request dikirim ke server yang antreannya sedang panjang.
- **Least connection** selalu memilih server yang paling senggang, sehingga antrean ketiga server lebih seimbang.

> **Kesimpulan:** least connection tidak terlalu mengubah performa rata-rata, tetapi membuat performa **lebih konsisten dan adil** karena *tail latency* berkurang secara signifikan.

> ⚠️ **Catatan:**
> - Untuk membandingkan throughput, gunakan **total request dalam 60 detik**, bukan angka RPS di pojok kanan atas Locust (itu hanya nilai sesaat di akhir tes).
> - Locust memiliki unsur acak. Untuk hasil yang lebih valid, setiap tes idealnya dijalankan 2–3 kali lalu dirata-rata.

### 8.4 Kembalikan ke Round Robin

Hapus baris `least_conn;` dari `nginx/nginx.conf`, lalu:

```powershell
docker compose up -d --build nginx
```

---

## 9. Bonus: Uji Ketahanan (Failover)

Bagian ini menjawab tantangan pertama di pendahuluan: *bagaimana menjaga ketersediaan layanan ketika salah satu server gagal?*

Matikan Server B:

```powershell
docker compose stop app2
```

Kirim request lagi:

```powershell
1..6 | ForEach-Object { (Invoke-RestMethod http://localhost/).message }
```

Hasilnya hanya Server **A** dan **C**, **tanpa error**. Ketika NGINX gagal menghubungi Server B, request otomatis diteruskan ke server berikutnya. Request pertama yang mengenai B mungkin sedikit lebih lambat karena NGINX baru menyadari B mati.

> 📸 **[SCREENSHOT]** Output saat Server B dimatikan
>
> ![failover](img/failover.png)

Nyalakan kembali Server B:

```powershell
docker compose start app2
```

---

## 10. Troubleshooting

| Masalah                                                    | Solusi                                                                                                        |
| :--------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------ |
| `Bind for 0.0.0.0:80 failed: port is already allocated`    | Port 80 dipakai aplikasi lain. Matikan IIS/XAMPP/Laragon, atau ubah port nginx ke `"8080:80"`                  |
| Mongo Express tidak bisa dibuka / container restart terus  | Tunggu 10–20 detik, MongoDB belum siap. Pastikan ada `restart: unless-stopped`                                 |
| `/all` mengembalikan data kosong                           | Nama database/collection di Mongo Express tidak sama dengan di `main.py`                                       |
| `/create` error 422                                        | Body JSON belum dibungkus key `lister` (efek `embed=True`)                                                     |
| Perubahan kode tidak berpengaruh                           | Build ulang: `docker compose up --build` (atau `docker compose up -d --build <nama_service>`)                  |
| `locust is not recognized`                                 | Gunakan `python -m locust -f locustfile.py`                                                                    |
| Grafik Locust ada garis diagonal panjang                   | Tekan **Reset** di antara tes                                                                                  |

**Mematikan seluruh service:**

```powershell
docker compose down
```

Data MongoDB tetap aman karena tersimpan di volume `mongodb_data`. Untuk menghapus data juga, gunakan `docker compose down -v`.

---

## Done 🎉🎉

Pada modul ini kita telah:

1. Memahami konsep dan algoritma load balancing
2. Membangun 3 aplikasi FastAPI yang terhubung ke satu MongoDB
3. Menggunakan NGINX sebagai load balancer dengan algoritma **Round Robin**
4. Membuktikan dengan Locust bahwa load balancer meningkatkan throughput ±2,8x dan mempercepat response time ±3,5x
5. Membandingkan **Round Robin** dengan **Least Connection**, di mana least connection menurunkan tail latency (99%ile) hingga 35%
6. Menguji ketahanan sistem saat salah satu server mati (failover)
