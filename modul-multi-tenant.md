# Modul Praktikum - Implementasi Dasar Multi-Tenant (Shared Schema)

---

## Tujuan Praktikum

1. Memahami cara mengidentifikasi tenant dari sebuah request (permintaan) HTTP.
2. Membangun middleware untuk mengekstrak dan memvalidasi identitas tenant.
3. Menerapkan isolasi data (filter `tenant_id`) saat mengambil data dari database.

---

## Bahan Kajian

* Database
* * Cloud Computing
* Software as a Service (SaaS)

---

## Skenario
Kita akan membuat API sederhana yang mengembalikan "System Prompt" untuk chatbot. Ada dua tenant: **Klinik Sehat (Tenant A)** dan **Toko Sepatu (Tenant B)**. API tidak boleh salah mengembalikan prompt milik klinik ke toko sepatu.

---

## Alat & Bahan

* Node.js terinstal di komputer.
* Aplikasi Postman atau cURL untuk pengujian API.

---

## Langkah Praktikum

### **Langkah 1: Persiapan Proyek (Setup)**

Buka terminal/command prompt Anda, buat folder baru, dan inisialisasi proyek Node.js:

   ```bash
   mkdir multi-tenant
   cd multi-tenant
   npm init -y
   npm install express
   ```
Buat sebuah file bernama `server.js` di dalam folder tersebut. Ini akan menjadi file utama kita.

---

### **Langkah 2: Simulasi Database (Shared Schema)**

Dalam praktikum ini, kita akan menggunakan array sederhana sebagai pengganti database sungguhan agar Anda bisa langsung melihat logikanya.

Buka `server.js` dan tambahkan kode berikut:

   ```javascript
    const express = require('express');
    const app = express();
    app.use(express.json());

    // --- MOCK DATABASE (Satu Database, Skema Berbagi) ---

    // Tabel Tenants
    const tenants = [
        { id: 't-001', name: 'Klinik Sehat', apiKey: 'key-klinik-123' },
        { id: 't-002', name: 'Toko Sepatu Langkah', apiKey: 'key-sepatu-456' }
    ];

    // Tabel Chatbot Configs (Perhatikan kolom tenant_id yang wajib ada)
    const chatbotConfigs = [
        { 
            id: 1, 
            tenant_id: 't-001', 
            systemPrompt: 'Anda adalah asisten medis. Jawab dengan empati dan formal.' 
        },
        { 
            id: 2, 
            tenant_id: 't-002', 
            systemPrompt: 'Anda adalah admin toko sepatu. Jawab dengan santai dan gaul.' 
        }
    ];
   ```

---

### **Langkah 3: Membuat Middleware Identifikasi Tenant**

Middleware adalah satpam yang memeriksa setiap permintaan sebelum masuk ke logika inti aplikasi. Di sini, satpam akan memeriksa `x-api-key` yang dikirim dari client (frontend).

Tambahkan kode ini di bawah "Mock Database" pada `server.js`:

   ```javascript
    // --- MIDDLEWARE: Identifikasi dan Validasi Tenant ---
    const tenantMiddleware = (req, res, next) => {
        // 1. Ambil API key dari header request
        const apiKey = req.headers['x-api-key'];
    
        if (!apiKey) {
            return res.status(401).json({ error: 'API Key tidak ditemukan. Akses ditolak.' });
        }
    
        // 2. Cari tenant berdasarkan API key tersebut di database
        const currentTenant = tenants.find(t => t.apiKey === apiKey);
    
        if (!currentTenant) {
            return res.status(403).json({ error: 'API Key tidak valid atau Tenant tidak ditemukan.' });
        }
    
        // 3. Simpan data tenant ke dalam object `req` agar bisa dipakai di langkah selanjutnya
        req.tenant = currentTenant;
        
        // 4. Lanjut ke proses berikutnya
        next();
    };
    
    // Pasang middleware ke semua rute di bawah ini
    app.use(tenantMiddleware);
   ```

---

### **Langkah 4 – Endpoint API dengan Filter `tenant_id` (Isolasi Data)**

Sekarang kita buat endpoint (URL) untuk mengambil System Prompt. Di sinilah aturan emas Shared Schema diterapkan: Selalu sertakan filter `tenant_id`.

Tambahkan kode ini di bawah middleware:

   ```javascript
    // --- ENDPOINT: Mengambil System Prompt Chatbot ---
    app.get('/api/chatbot/config', (req, res) => {
        // Mengambil tenant_id dari middleware sebelumnya
        const activeTenantId = req.tenant.id;
    
        // ATURAN EMAS: Filter query berdasarkan tenant_id aktif
        // (Jika pakai SQL, ini setara dengan: SELECT * FROM configs WHERE tenant_id = activeTenantId)
        const config = chatbotConfigs.find(c => c.tenant_id === activeTenantId);
    
        if (!config) {
            return res.status(404).json({ error: 'Konfigurasi tidak ditemukan untuk tenant ini.' });
        }
    
        res.json({
            message: `Berhasil mengambil konfigurasi untuk ${req.tenant.name}`,
            systemPrompt: config.systemPrompt
        });
    });
    
    // Jalankan server
    const PORT = 3000;
    app.listen(PORT, () => {
        console.log(`Server Multi-Tenant berjalan di http://localhost:${PORT}`);
    });
   ```

---

### **Langkah 5 – Pengujian (Cross-Testing)**

Jalankan server Anda melalui terminal:

```bash
node server.js
```

Buka terminal baru atau aplikasi Postman, dan lakukan tiga pengujian berikut:

### Tes 1: Request dari Klinik Sehat (Tenant A)

```bash
curl -H "x-api-key: key-klinik-123" http://localhost:3000/api/chatbot/config
```

* **Ekspektasi Output:** Mendapatkan respons dengan System Prompt asisten medis.

### Tes 2: Request dari Toko Sepatu (Tenant B)

```bash
curl -H "x-api-key: key-sepatu-456" http://localhost:3000/api/chatbot/config
```

* **Ekspektasi Output:** Mendapatkan respons dengan System Prompt admin toko sepatu. Walaupun mereka mengakses endpoint URL yang persis sama, datanya terisolasi.

### Tes 3: Request Tanpa Akses / Salah Kunci (Simulasi Hacker)

```bash
curl -H "x-api-key: key-ngasal" http://localhost:3000/api/chatbot/config
```
* **Ekspektasi Output:** Mendapatkan Error 403 (Akses ditolak). Keamanan berlapis mencegah akses tanpa identitas tenant yang jelas.

---

## Tugas

1. Integrasikan multi-tenant ke aplikasi chatbot yang telah dibuat.
2. Gunakan database PostgreSQL untuk membuat data multi-tenant.

---

## Referensi

1. [Multitenancy dalam Cloud Computing: Konsep, Manfaat, dan Tantangannya](https://www.cloudeka.id/id/berita/cloud/multitenancy-dalam-cloud-computing-konsep-manfaat-dan-tantangannya/)
2. [Mendefinisikan ulang multi-tenancy](https://docs.aws.amazon.com/id_id/whitepapers/latest/saas-architecture-fundamentals/re-defining-multi-tenancy.html)

---
