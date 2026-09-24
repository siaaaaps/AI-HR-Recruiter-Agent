# AI-HR-Recruiter-Agen
Sistem otomatisasi rekrutmen berbasis n8n dan AI (Google Gemini) yang membaca CV dari Gmail, mengekstrak data kandidat, menyimpannya ke Google Sheets, mengecek ketersediaan Google Calendar, dan mengirimkan notifikasi ke Telegram.
<img width="1903" height="723" alt="image" src="https://github.com/user-attachments/assets/0359baf2-09b3-4211-af32-e59903bc5286" />

# Tools

## 1️⃣ Buat Bot Telegram 🤖✉️

1. Buka aplikasi Telegram, cari `@BotFather`, lalu kirim `/start`.
2. Kirim `/newbot`, berikan nama dan username berakhiran `bot`.
3. BotFather memberi API token: `123456:ABC-DEF…`. Simpan sebagai `BOT_TOKEN`.
4. Di n8n → Credentials → Telegram → tempel token.

## 2️⃣ API Key AI Gemini 🤖

1. Kunjungi Google AI Studio → Get API key → beri nama → Create Key.
2. Salin key `AIzaSy…` → n8n → Credentials → Google Gemini Chat Model.

## 3️⃣ Autentikasi Google (Gmail, Sheets, Calendar) 📄

<details>
<summary>🚀 Jalur A — Login Sekali Klik (n8n.cloud/pre-configured)</summary>

1. Buka *n8n → Credentials → Google Sheets / Gmail / Google Calendar*.
2. Tipe: **OAuth2 (Google pre-configured)**.
3. Klik **Connect OAuth2 Account** → login Google, pilih email, `Allow`.
4. Selesai — token otomatis tersimpan.

</details>

<details>
<summary>🔧 Jalur B — Custom Client ID & Secret (Self-host kustom)</summary>

1. Google Cloud Console → *APIs & Services → Credentials* → **Create OAuth Client ID**.
2. Redirect URI: `https://<domain-n8n>/rest/oauth2-credential/callback`.
3. Salin **Client ID** & **Client Secret**.
4. n8n → Credentials → **Google Node** → **Generic OAuth2** → isi ID & Secret → **Connect**.

</details>

# Step

## 1️⃣ Gmail Trigger 📧

> **Goal:** Memantau email masuk yang memiliki lampiran file PDF CV secara real-time.

| Parameter | Nilai |
|-----------|-------|
| **Node Name** | `Gmail Trigger` |
| **Credential** | `Gmail OAuth2` |
| **Event** | `Message Received` |
| **Search Filter** | `has:attachment filename:pdf` |

---

## 2️⃣ Extract from File 📄

> **Goal:** Mengunduh dan mengekstrak teks mentah dari dokumen PDF CV pelamar.

| Parameter | Nilai |
|-----------|-------|
| **Node Name** | `Extract from File` |
| **Operation** | `Extract From PDF` |
| **Input Binary Field** | `attachment_0` |

---

## 3️⃣ Google Gemini Chat Model 🤖

> **Goal:** Membaca teks CV mentah dan menganalisisnya menjadi struktur data JSON terstandardisasi.

| Parameter | Nilai |
|-----------|-------|
| **Chain Node** | `Basic LLM Chain1` |
| **Model Node** | `Google Gemini Chat Model` |

### 📝 User Message (Prompt)

Ambil informasi berikut dari teks CV pelamar:
1. nama_lengkap
2. email
3. nomor_hp
4. pendidikan_terakhir
5. keterampilan
6. ringkasan_pengalaman

Teks CV:
{{ $json.text }}

Kembalikan HANYA format JSON murni tanpa markdown:

{
  "nama_lengkap": "",
  "email": "",
  "nomor_hp": "",
  "pendidikan_terakhir": "",
  "keterampilan": "",
  "ringkasan_pengalaman": ""
}

---

## 4️⃣ Code in JavaScript 💻

> **Goal:** Membersihkan teks respons dari AI Gemini agar terbebas dari pemformatan Markdown dan menjamin validitas JSON.

```javascript
const input = $input.first().json;
let raw = input.output || input.text || input.response || input.content || "";

if (typeof raw !== "string") {
  raw = JSON.stringify(raw);
}

// Bersihkan tag markdown
raw = raw.replace(/```json/gi, "").replace(/```/g, "").trim();

const start = raw.indexOf("{");
const end = raw.lastIndexOf("}");

if (start === -1 || end === -1) {
  return [{
    json: {
      nama_lengkap: "Tidak ditemukan",
      email: "Tidak ditemukan",
      nomor_hp: "Tidak ditemukan",
      pendidikan_terakhir: "Tidak ditemukan",
      keterampilan: "Tidak ditemukan",
      ringkasan_pengalaman: "Tidak ditemukan"
    }
  }];
}

try {
  const data = JSON.parse(raw.substring(start, end + 1));

  return [{
    json: {
      nama_lengkap: data.nama_lengkap || "Tidak ditemukan",
      email: data.email || "Tidak ditemukan",
      nomor_hp: data.nomor_hp || "Tidak ditemukan",
      pendidikan_terakhir: data.pendidikan_terakhir || "Tidak ditemukan",
      keterampilan: data.keterampilan || "Tidak ditemukan",
      ringkasan_pengalaman: data.ringkasan_pengalaman || "Tidak ditemukan"
    }
  }];
} catch (error) {
  return [{
    json: {
      nama_lengkap: "Tidak ditemukan",
      email: "Tidak ditemukan",
      nomor_hp: "Tidak ditemukan",
      pendidikan_terakhir: "Tidak ditemukan",
      keterampilan: "Tidak ditemukan",
      ringkasan_pengalaman: "Tidak ditemukan",
      error: error.message
    }
  }];
}
```

## 5️⃣ Append row in sheet 📊

> **Goal:** Menyimpan data kandidat hasil ekstraksi ke baris baru spreadsheet.

| Column Header | Expression Value |
|---------------|------------------|
| **Nama Lengkap** | `{{ $('Code in JavaScript').item.json.nama_lengkap }}` |
| **Email** | `{{ $('Code in JavaScript').item.json.email }}` |
| **Nomor HP** | `{{ $('Code in JavaScript').item.json.nomor_hp }}` |
| **Pendidikan Terakhir** | `{{ $('Code in JavaScript').item.json.pendidikan_terakhir }}` |
| **Keterampilan** | `{{ $('Code in JavaScript').item.json.keterampilan }}` |
| **Ringkasan Pengalaman** | `{{ $('Code in JavaScript').item.json.ringkasan_pengalaman }}` |

---

## 6️⃣ Get availability in a calendar 📅

> **Goal:** Memeriksa ketersediaan jadwal kosong di Google Calendar untuk sesi review/interview kandidat.

| Parameter | Nilai |
|-----------|-------|
| **Operation** | `availability: calendar` |
| **Time Min** | `{{ $now }}` |
| **Time Max** | `{{ $now.plus(7, 'days') }}` |

---

---

## 7️⃣ Create Event & Google Meet (Google Calendar) 📅

> **Goal:** Membuat jadwal wawancara otomatis di Google Calendar, menghasilkan tautan Google Meet, dan mengundang email kandidat.

| Parameter | Nilai | Catatan |
|-----------|-------|---------|
| **Resource** | `Event` | |
| **Operation** | `Create` | Membuat event baru |
| **Calendar** | `From list` -> (Pilih akun Google Calendar HR) | |
| **Start Time** | `{{ $now.plus({ days: 1 }).set({ hour: 10, minute: 0 }).format("yyyy-MM-dd'T'HH:mm:ss") }}` | Jadwal besok jam 10:00 pagi |
| **End Time** | `{{ $now.plus({ days: 1 }).set({ hour: 11, minute: 0 }).format("yyyy-MM-dd'T'HH:mm:ss") }}` | Durasi 1 jam (selesai 11:00) |
| **Attendees** | `{{ $('Code in JavaScript').item.json.email }}` | Email kandidat otomatis |
| **Conference Data** | `Google Meet` | Membuat link meeting otomatis |
| **Description** | `Wawancara posisi kandidat {{ $('Code in JavaScript').item.json.nama_lengkap }}. Nomor HP: {{ $('Code in JavaScript').item.json.nomor_hp }}` | Catatan detail event |
| **Send Updates** | `All` | Mengirim notifikasi undangan dari Google |

---

## 8️⃣ Send Email Invitation (Gmail) ✉️

> **Goal:** Mengirimkan email undangan wawancara resmi secara otomatis ke alamat email pelamar.

| Parameter | Nilai | Catatan |
|-----------|-------|---------|
| **Node Name** | `Send a message` | |
| **Credential** | `Gmail account` | Akun OAuth2 Gmail HR |
| **Resource** | `Message` | |
| **Operation** | `Send` | |
| **To** | `{{ $('Code in JavaScript').item.json.email }}` | Email kandidat dinamis |
| **Subject** | `Undangan Wawancara Kerja - {{ $('Code in JavaScript').item.json.nama_lengkap }}` | Subjek email |

### 📝 Message / Body Template

Halo {{ $('Code in JavaScript').item.json.nama_lengkap }},

Selamat! Berdasarkan hasil peninjauan CV Kamu, kami mengundang Kamu untuk mengikuti tahap Wawancara Kerja.

Berikut detail jadwal wawancara Kamu:
📅 Tanggal: {{ $now.plus({ days: 1 }).setLocale('id').format('cccc, dd MMMM yyyy') }}
⏰ Waktu: 10:00 WIB
📍 Media: Google Meet (Link tautan sudah disertakan di undangan Google Calendar)

Mohon konfirmasi kehadiran Kamu dengan membalas email ini.

Terima kasih,  
Tim HR Recruiter

## 7️⃣ Send a text message (Telegram) ✉️

> **Goal:** Mengirim pesan notifikasi ringkasan kandidat baru secara otomatis ke chat Telegram HR.

### 📝 Text Template

📩 *ADA PELAMAR BARU!*

👤 *Nama:* {{ $('Code in JavaScript').item.json.nama_lengkap }}  
✉️ *Email:* {{ $('Code in JavaScript').item.json.email }}  
📞 *No. HP:* {{ $('Code in JavaScript').item.json.nomor_hp }}  
🎓 *Pendidikan:* {{ $('Code in JavaScript').item.json.pendidikan_terakhir }}  
🛠️ *Keterampilan:* {{ $('Code in JavaScript').item.json.keterampilan }}  
💼 *Ringkasan:* {{ $('Code in JavaScript').item.json.ringkasan_pengalaman }}  

✅ Data telah tersimpan di Google Sheets & ketersediaan Google Calendar berhasil diperiksa.
