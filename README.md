# Otak Chat APK

Aplikasi chat Android berbasis **Capacitor** yang menjadi wajah untuk server **Ollama** yang berjalan di Termux/Ubuntu pada HP.

## Struktur project

```text
otak-chat-apk/
├── www/
│   └── index.html          # UI chat dan koneksi streaming ke Ollama
├── capacitor.config.json   # konfigurasi aplikasi Capacitor
├── package.json            # dependensi dan skrip project
├── .github/
│   └── workflows/
│       └── build-apk.yml   # build APK otomatis di GitHub Actions
└── README.md
```

## Cara kerja

```text
HP (Termux + Ubuntu)                         GitHub Repo
┌──────────────────────────┐       ┌──────────────────────────────┐
│ Ollama + llama3.1:8b     │ HTTP  │ www/index.html                │
│ 127.0.0.1:11434          │◄─────►│ Capacitor + workflow build    │
└──────────────────────────┘       └──────────────────────────────┘
```

- **Otak** adalah Ollama dan model AI yang berjalan di HP.
- **Wajah/wadah** adalah `www/index.html` yang dibungkus menjadi APK dengan Capacitor.
- **GitHub Actions** otomatis membuat APK debug setiap ada push ke branch `main`.
- **Cloudflare Tunnel** bersifat opsional untuk mengakses Ollama dari luar jaringan lokal.

## Menjalankan Ollama di Termux

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

Cek model yang terpasang dengan `ollama list`. Hapus model dengan `ollama rm <nama_model>`.

## Menjalankan project secara lokal

```bash
npm install
npx cap add android
npx cap sync android
```

Untuk pengembangan UI, file utama berada di `www/index.html`. Atur alamat Ollama dan nama model langsung pada panel pengaturan aplikasi. Nilai tersebut disimpan di `localStorage` perangkat.

## Build APK otomatis

Workflow `.github/workflows/build-apk.yml` akan:

1. Checkout kode.
2. Menyiapkan Node.js dan Java 17.
3. Menginstal dependensi dengan `npm install`.
4. Menambahkan platform Android jika belum ada.
5. Menjalankan `npx cap sync android`.
6. Menjalankan `./gradlew assembleDebug`.
7. Mengunggah `app-debug.apk` sebagai artifact bernama `otak-chat-debug-apk`.

Build dapat dijalankan otomatis melalui push ke `main` atau manual dari menu **Actions → Build APK → Run workflow**.

## API chat

Aplikasi mengirim request streaming ke endpoint Ollama berikut:

```http
POST http://<alamat-ollama>:11434/api/chat
Content-Type: application/json
```

Contoh body:

```json
{
  "model": "llama3.1:8b",
  "messages": [
    {"role": "user", "content": "Halo"}
  ],
  "stream": true
}
```

Respons streaming JSON per baris dibaca dari `data.message.content`.

## Cloudflare Tunnel (opsional)

```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64 -o cloudflared
chmod +x cloudflared
./cloudflared tunnel --url http://127.0.0.1:11434
```

Masukkan alamat `https://xxxx.trycloudflare.com` yang dihasilkan ke kolom alamat Ollama di aplikasi.

> **Peringatan keamanan:** Ollama tidak menyediakan login/password bawaan. Siapa pun yang mengetahui alamat tunnel dapat mencoba memakai server tersebut. Jangan membagikan alamat tunnel dan matikan `cloudflared` saat tidak digunakan.

## Batasan saat ini

- Versi awal menyediakan chat Ollama, pengaturan host/model, penyimpanan konfigurasi lokal, dan tombol hapus chat.
- Connector GitHub, profil server multipel, serta push log chat masih merupakan fitur lanjutan yang direncanakan.
- Aplikasi tidak dapat menjalankan atau mematikan proses `cloudflared`/`git` secara langsung di luar sandbox Android.

## Checklist

- [x] Struktur project Capacitor dibuat.
- [x] UI chat pada `www/index.html` dibuat.
- [x] Konfigurasi `capacitor.config.json` dibuat.
- [x] Workflow build APK otomatis dibuat.
- [ ] Ollama server hidup dan model terdownload.
- [ ] GitHub Actions berhasil build APK.
- [ ] APK diinstall di HP.
- [ ] Host dan model diatur di aplikasi.
- [ ] Connector GitHub ditambahkan jika diperlukan.
