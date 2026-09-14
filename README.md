# Catatan Project: Otak Chat

Ringkasan lengkap alur, arsitektur, dan fitur yang sudah direncanakan/dibuat.

## Konsep Dasar

```
┌──────────────────────────┐        ┌────────────────────────────────┐
│   HP (Termux + Ubuntu)   │        │         GitHub Repo              │
│                           │        │                                  │
│   Ollama server            │<------>│  www/index.html (kode app)      │
│   + model llama3.1:8b      │  API   │  .github/workflows/              │
│   = OTAK                   │        │    build-apk.yml (auto build)    │
│                           │        │                                  │
│   127.0.0.1:11434           │        │  APK di-build otomatis tiap push │
└──────────────────────────┘        └────────────────────────────────┘
            ▲
            │ opsional, akses dari luar
            │
     Cloudflare Tunnel (cloudflared)
     https://xxxx.trycloudflare.com
```

- **Otak** = Ollama + model AI, jalan di Termux/Ubuntu di HP.
- **Wajah/Wadah** = APK (dibungkus dari `index.html` pakai Capacitor), tampilan chat yang connect ke otak.
- **GitHub** = tempat kode disimpan; GitHub Actions otomatis build ulang APK setiap kode di-push.
- **Cloudflare Tunnel** = opsional, supaya otak di HP bisa diakses dari luar jaringan WiFi.

---

## 1. Setup Otak (Ollama di Termux)

```bash
pkg update && pkg upgrade -y
pkg install proot-distro -y
proot-distro install ubuntu
proot-distro login ubuntu

apt update && apt install curl -y
curl -fsSL https://ollama.com/install.sh | sh

OLLAMA_HOST=0.0.0.0 ollama serve &
ollama run llama3.1:8b
```

Cek model terinstall: `ollama list`  
Hapus model lama: `ollama rm <nama_model>`

## 2. Struktur Project APK (Capacitor)

```
otak-chat-apk/
├── www/
│   └── index.html          # kode tampilan chat (wajah app)
├── capacitor.config.json   # konfigurasi bungkus jadi APK
├── package.json
├── .github/
│   └── workflows/
│       └── build-apk.yml   # builder APK otomatis di GitHub Actions
└── README.md
```

`index.html` berisi UI chat yang mengirim request ke Ollama lewat endpoint `POST /api/chat` (format streaming JSON per baris).

## 3. Build APK Otomatis

Alur kerja `.github/workflows/build-apk.yml`:
1. Checkout kode
2. Setup Node + Java
3. `npx cap add android` (kalau folder android belum ada)
4. `npx cap sync android`
5. `./gradlew assembleDebug`
6. Upload hasil `.apk` sebagai artifact yang bisa didownload dari tab **Actions** di GitHub

Setiap kali `www/index.html` diedit dan di-push, APK baru otomatis dibuat — tanpa perlu Android Studio.

## 4. Upload / Update ke GitHub

```bash
git init
git add .
git commit -m "Setup awal"
git branch -M main
git remote add origin https://github.com/<username>/<nama-repo>.git
git push -u origin main
```

Update selanjutnya:
```bash
git add www/index.html
git commit -m "Perbaiki tampilan"
git push
```

## 5. Fitur Connectors di Dalam App

Panel **Connectors** (ikon 🔌) berisi dua bagian:

### a. GitHub
- **Simpan Koneksi**: masukkan Personal Access Token (fine-grained, izin "Contents: Read and write" khusus repo tersebut) + nama repo (`owner/nama-repo`).
- **Ambil Kode**: mengambil isi file (default `www/index.html`) dari repo lewat GitHub API (`GET /repos/{repo}/contents/{path}`), ditampilkan di editor teks dalam app.
- **Push Perbaikan**: mengirim isi editor kembali ke GitHub (`PUT /repos/{repo}/contents/{path}`) sehingga tersimpan sebagai commit baru → otomatis memicu GitHub Actions untuk build ulang APK.
- **Push Log Chat**: menyimpan riwayat percakapan sebagai file JSON baru di folder `logs/` pada repo, dengan nama file bertanda waktu.

### b. Server Profiles (untuk Cloudflare / alamat lain)
- Simpan beberapa alamat server dengan nama (misal "Lokal", "Cloudflare Tunnel").
- Tap salah satu profil untuk langsung menjadikannya alamat aktif (host) tanpa harus mengetik ulang.
- Berguna untuk gonta-ganti antara `http://127.0.0.1:11434` (lokal) dan `https://xxxx.trycloudflare.com` (saat pakai Cloudflare Tunnel).

## 6. Cloudflare Tunnel (opsional, akses dari luar HP)

```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64 -o cloudflared
chmod +x cloudflared
./cloudflared tunnel --url http://127.0.0.1:11434
```

Alamat `https://xxxx.trycloudflare.com` yang muncul dimasukkan sebagai Server Profile baru di app.

**Catatan keamanan:** Ollama tidak punya sistem login/password bawaan. Selama tunnel aktif, siapa pun yang tahu alamatnya bisa memakai otak di HP. Jangan sebar alamat tunnel ke publik; matikan `cloudflared` kalau tidak dipakai.

## 7. Batasan Teknis (penting untuk dipahami)

- App (APK/WebView) **tidak bisa** menjalankan atau mematikan proses `cloudflared`/`git` secara langsung dari dalam app — itu proses terpisah yang jalan di Termux/Ubuntu, sandbox Android tidak mengizinkan satu app menjalankan proses app lain tanpa root/plugin native khusus.
- Yang bisa dilakukan app lewat internet biasa (HTTPS/API) HANYA: memanggil GitHub API (baca/tulis file di repo) dan memanggil Ollama API lewat alamat manapun yang aktif (lokal atau lewat tunnel).
- Jadi fitur "connector" di app ini sifatnya: **mengatur konfigurasi & mendorong perubahan kode**, bukan mengendalikan proses sistem di HP.

## Checklist

- [ ] Ollama server hidup & model terdownload
- [ ] Project APK sudah di-push ke GitHub
- [ ] GitHub Actions berhasil build APK (centang hijau di tab Actions)
- [ ] APK sudah diinstall di HP
- [ ] Host & model diatur di app
- [ ] Token & repo GitHub diatur di Connectors (kalau mau edit kode dari app)
- [ ] Profil server (Lokal/Cloudflare) sudah disimpan sesuai kebutuhan
