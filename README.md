# 📊 Laporan Pengeluaran Mingguan dengan AI menggunakan n8n & OpenRouter

> Workflow otomasi yang menganalisis pengeluaran mingguan secara otomatis menggunakan AI, lalu mengirimkan ringkasan dan saran hemat langsung ke Telegram setiap hari Minggu pagi.

---

## 🗂️ Deskripsi Proyek

Workflow ini dirancang untuk menyelesaikan masalah umum: banyak orang sudah mencatat pengeluaran di spreadsheet, tapi jarang sempat menganalisisnya. Dengan workflow ini, proses analisis berjalan **sepenuhnya otomatis** — tidak perlu buka aplikasi, tidak perlu hitung manual.

Setiap Minggu pukul 07.00, sistem akan:
1. Mengambil data pengeluaran dari Google Sheets
2. Memfilter transaksi 7 hari terakhir
3. Meminta AI menganalisis kebiasaan belanja dan memberikan 1 saran penghematan
4. Mengirimkan hasilnya langsung ke Telegram

---

## 🛠️ Stack & Tools

| Tool | Versi / Model | Peran |
|------|---------------|-------|
| **n8n** | Cloud / Self-hosted | Platform otomasi workflow |
| **Google Sheets** | — | Sumber data log pengeluaran |
| **OpenRouter** | `xiaomi/mimo-v2-flash` | Gateway akses model AI (gratis) |
| **Telegram Bot** | API v1.2 | Saluran pengiriman output |

---

## 🔄 Alur Workflow

```
Setiap Hari Minggu (07.00)
        │
        ▼
Ambil Log Pengeluaran (Google Sheets)
        │
        ▼
Filter 7 Hari & Format Teks (JavaScript Code)
        │
        ▼
IF: hasExpenses?
   │                    │
  TRUE                FALSE
   │                    │
   ▼                    ▼
Analisis             Pesan False
(Basic LLM Chain)    "Tidak ditemukan log..."
   │                 (Telegram)
   ▼
OpenRouter Chat Model
(xiaomi/mimo-v2-flash)
   │
   ▼
Pesan True
(Telegram — hasil analisis AI)
```

---

## 🧩 Penjelasan Setiap Node

### 1. `Setiap Hari Minggu` — Schedule Trigger
Memicu workflow secara otomatis setiap hari Minggu pukul 07.00 pagi. Tidak memerlukan interaksi manual apapun.

**Konfigurasi:**
```
Interval : Weeks
Trigger At Hour : 7 (07:00 AM)
```

---

### 2. `Ambil Log Pengeluaran` — Google Sheets
Mengambil **seluruh baris data** dari spreadsheet `Data Dummy Catatan Keuangan` tanpa filter. Semua data diambil terlebih dahulu, filter dilakukan di node berikutnya.

**Konfigurasi:**
```
Document ID : 1rExeqHpVixKIereyJ9cF9ElslyI0YK-cG1SlUUSr8aI
Sheet Name  : Sheet1 (gid=0)
Operation   : Get Many Rows
```

> ⚠️ Pastikan Google Sheets sudah terhubung via OAuth2 sebelum menjalankan workflow.

---

### 3. `Filter 7 Hari & Format Teks` — Code (JavaScript)
Node paling krusial. Melakukan dua tugas sekaligus:

**a. Filter tanggal 7 hari terakhir**

Format tanggal di Google Sheets adalah `DD/MM/YYYY`. JavaScript native `new Date()` tidak bisa membacanya dengan benar, sehingga perlu di-parse secara manual:

```javascript
const [day, month, year] = rawTanggal.split("/");
const expenseDate = new Date(`${year}-${month}-${day}`);
```

**b. Format data menjadi teks untuk AI**

Setelah difilter, data diubah menjadi string yang mudah dipahami oleh model bahasa:

```
Data pengeluaran saya minggu ini:
- Makan Siang di Kantin: Rp15000
- Grab ke Kantor: Rp25000
- ...
```

**Output node:**
```json
{
  "promptData": "Data pengeluaran saya minggu ini:\n- ...",
  "hasExpenses": true
}
```

---

### 4. `If` — Conditional Logic
Memeriksa nilai boolean `hasExpenses`. Tujuannya untuk memastikan AI **tidak dipanggil sia-sia** ketika tidak ada data.

```
Kondisi : $json.hasExpenses === true
TRUE    → lanjut ke node Analisis
FALSE   → lanjut ke node Pesan False
```

> 💡 Ini adalah praktik penting dalam workflow AI: hanya panggil LLM saat data tersedia untuk menghindari output yang tidak berguna dan pemborosan API call.

---

### 5. `Analisis` — Basic LLM Chain
Mengirimkan data pengeluaran yang sudah diformat ke model AI. Node ini menggabungkan dua bagian:

**Bagian 1 — Data Dinamis** (dari node sebelumnya):
```
{{ $('Filter 7 Hari & Format Teks').item.json.promptData }}
```

**Bagian 2 — Instruksi Statis:**
```
---
Instruksi: Tulis ringkasan kebiasaan belanja saya minggu ini dan beri 1 saran penghematan.

ATURAN FORMATTING (SANGAT PENTING):
1. Tulis pesan layaknya chat biasa.
2. DILARANG KERAS membuat grafik, diagram, atau menggunakan sintaks Mermaid.
3. DILARANG menggunakan blok kode (simbol ```).
4. Gunakan list dengan tanda strip (-) biasa jika diperlukan.
```

> 💡 Aturan formatting penting agar output AI tetap rapi saat dikirim ke Telegram yang tidak mendukung markdown kompleks.

---

### 6. `OpenRouter Chat Model` — LLM Provider
Sub-node yang terhubung ke node Analisis sebagai language model provider.

```
Provider : OpenRouter
Model    : xiaomi/mimo-v2-flash
```

> Model ini dipilih karena tersedia secara **gratis** via OpenRouter dan cukup untuk kebutuhan analisis teks keuangan sederhana. Bisa diganti ke `google/gemini-flash-1.5` atau `meta-llama/llama-3-8b-instruct` untuk kualitas lebih stabil.

---

### 7. `Pesan True` — Telegram (Jalur Sukses)
Mengirimkan hasil analisis AI ke Telegram. Teks diambil langsung dari output node Analisis:

```
Text    : {{ $json.text }}
Chat ID : [Chat ID Telegram kamu]
```

---

### 8. `Pesan False` — Telegram (Jalur Fallback)
Mengirimkan pesan pemberitahuan sederhana jika tidak ada data pengeluaran dalam 7 hari terakhir:

```
Text    : Tidak ditemukan log pengeluaran selama 7 hari terakhir.
Chat ID : [Chat ID Telegram kamu]
```

> Node ini memastikan workflow tidak pernah "diam" — selalu ada konfirmasi bahwa otomasi berjalan sesuai jadwal.

---

## ⚙️ Setup & Cara Menggunakan

### Prasyarat
- Akun n8n (cloud atau self-hosted)
- Akun Google dengan akses Google Sheets
- Bot Telegram (buat via @BotFather)
- Akun OpenRouter (daftar di openrouter.ai, gratis)

### Langkah Instalasi

**1. Import workflow**
```
n8n → Workflows → Import → Upload file JSON ini
```

**2. Hubungkan credentials**

| Node | Credential yang Dibutuhkan |
|------|---------------------------|
| Ambil Log Pengeluaran | Google Sheets OAuth2 |
| Analisis / OpenRouter | OpenRouter API Key |
| Pesan True & False | Telegram Bot API |

**3. Siapkan Google Sheets**

Buat spreadsheet dengan kolom minimal:

| Tanggal | Deskripsi | Jumlah (IDR) |
|---------|-----------|--------------|
| 13/05/2026 | Makan Siang | 25000 |
| 14/05/2026 | Grab ke Kantor | 35000 |

> ⚠️ Format tanggal **wajib** `DD/MM/YYYY` agar filter JavaScript berjalan dengan benar.

**4. Update Document ID**

Ganti value `documentId` di node `Ambil Log Pengeluaran` dengan ID Google Sheets milikmu.

**5. Update Chat ID Telegram**

Ganti nilai `chatId` di node `Pesan True` dan `Pesan False` dengan Chat ID Telegram kamu. Bisa dicek via bot @userinfobot.

**6. Aktifkan workflow**

Klik toggle **Active** di pojok kanan atas n8n. Workflow akan berjalan otomatis setiap Minggu pukul 07.00.

---

## ✅ Kelebihan

- Berjalan **100% otomatis** tanpa intervensi manual
- Ada **fallback path** — tidak error meski data kosong
- Hemat API: AI hanya dipanggil saat data tersedia
- Output rapi dan langsung bisa dibaca di Telegram
- Model AI gratis, tanpa biaya tambahan

---

## ⚠️ Keterbatasan & Pengembangan Selanjutnya

| Keterbatasan (v1.0) | Solusi (v2.0) |
|---------------------|---------------|
| Input ke Sheets masih manual | Integrasi Google Form atau API dompet digital |
| Model AI gratis kurang konsisten | Upgrade ke Gemini 1.5 Flash atau Llama 3 |
| Tidak ada konteks historis | Kirim data 4 minggu terakhir ke AI |
| Analisis tidak tersimpan | Tambah node Google Sheets (Write) untuk arsip |

---

## 📁 Struktur File

```
📦 workflow
 ┣ 📄 Laporan_Pengeluaran_Mingguan_dengan_AI_menggunakan_n8n_&_OpenRouter.json
 ┗ 📄 README.md  ← kamu di sini
```

---

## 👤 Author

**Wahyu** · Pelatihan AI for Business — Nurul Fikri Academy  
UJK BNSP · Bidang: Implementasi AI untuk Keuangan · Topik 7
