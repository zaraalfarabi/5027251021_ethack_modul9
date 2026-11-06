# PRAKTIKUM ETHICAL HACKING MODUL 9

| Keterangan | Isi |
| :--- | :--- |
| Nama | Az Zahra Fiddien Al Farabi |
| NRP | 5027251021 |
| Kelas | B |

# Analisis Blue Team — Bab 9

## Titik Deteksi/Pencegahan di Rantai Serangan

| Tahap | Peluang deteksi/pencegahan | Kontrol relevan |
|---|---|---|
| Delivery (email/link) | Lampiran/link bermakro dicegat sebelum sampai inbox; deteksi pengirim dipalsukan | Email gateway strip/sandbox lampiran `.docm`, blok ekstensi macro-enabled, SPF/DKIM/DMARC (Bab 8), banner "External Sender" |
| Download dokumen | File jahat diblok/dibersihkan saat transit; eksekusi di sandbox dulu | Proxy/web filtering, CDR (Content Disarm & Reconstruction), sandbox/detonation |
| Dokumen dibuka (Protected View) | **Titik terkuat & termurah:** makro dari internet dimatikan total sebelum user bisa mengaktifkan | Group Policy/Intune "Block macros from running in Office files from the Internet"; Protected View (sandbox read-only) |
| User klik Enable Content | Mengurangi kemungkinan user menuruti rekayasa sosial | Awareness training, prosedur lapor dokumen mencurigakan |
| Makro jalan | Deteksi runtime & batasi dampak bila lolos | AMSI (scan VBA), EDR (Word spawn child process), ASR rules, user bukan admin lokal |

## Rekomendasi Kontrol
- **Teknis (kebijakan makro, email/proxy, EDR):**
  1. **Blokir makro dari internet** via GPO/Intune (prioritas #1, biaya nol) — pastikan Office versi baru yang **default-block** makro ber-MOTW aktif. *(Di lab ini kontrol inilah yang menghentikan makro: banner merah "macros blocked – source untrusted", message box tidak jalan.)*
  2. **ASR rules** Defender: "Block all Office applications from creating child processes" + "Block Win32 API calls from Office macros".
  3. **Signed macros + Trusted Locations** untuk makro internal yang sah; selain itu blok.
  4. **Email gateway**: sandbox/strip lampiran macro-enabled atau CDR.
  5. **EDR + AMSI** sebagai jaring pengaman runtime; user harian bukan admin lokal; rutin patch.
- **Non-teknis (awareness, hubungkan dengan Bab 8):**
  - Training berkala + simulasi phishing; ajarkan prompt "Enable Content" pada dokumen tak dikenal = red flag, dan dokumen yang "sengaja terlihat rusak lalu menyuruh Enable Content" adalah ciri serangan.
  - Tekankan sisi rekayasa sosialnya (urgency/otoritas/pretexting, Bab 8) — makro cuma muatan; yang membuat user menekan tombol adalah manipulasi psikologis.
  - Sediakan tombol/alur "Report Phishing" yang mudah.

## Kesimpulan
> Untuk UKM dengan budget terbatas, kontrol paling cost-effective adalah yang **gratis
> dan bekerja sebelum user sempat memutuskan**: aktifkan kebijakan blokir makro dari
> internet (plus pastikan Office terbaru dengan default-block menyala) dan ASR rules —
> nol biaya lisensi dan memutus jalur serangan makro paling umum. Lab ini membuktikannya:
> makro yang sama **diblok total saat ber-MOTW** namun **baru jalan setelah MOTW dihapus
> manual** — artinya satu kebijakan gratis sudah menetralkan mayoritas serangan makro,
> jauh sebelum perlu membeli EDR komersial.

# Observasi Peringatan Word — Bab 9

## Info Umum
- OS (Windows / macOS): Windows 11
- Versi Word: Microsoft Office Professional Plus 2019, Version 2408 (Build 17932.21056 Click-to-Run)
- Tanggal: 6 November 2026

## Setup
- File `Doc1.docm` di-hosting dari Kali (VM, host-only `192.168.56.101`) via
  `python3 -m http.server 8000`, diunduh dari Windows lewat browser Chromium →
  file dapat Mark-of-the-Web.

## Peringatan yang Muncul

| Tahap | Muncul? (Ya/Tidak) | Screenshot | Catatan |
|---|---|---|---|
| File ditandai "dari internet" oleh OS | Ya | <img width="1771" height="922" alt="image" src="https://github.com/user-attachments/assets/ebdb957e-9a0e-43c2-a7c3-7ed5112bb7c0" />| Properties → tab General ada checkbox **"Unblock"** + teks "came from another computer". Inilah MOTW. |
| Protected View (banner "Enable Editing") | Ya | <img width="1918" height="288" alt="image" src="https://github.com/user-attachments/assets/985fdf2e-7400-45a2-89b2-23cbf39822c4" />| Banner kuning muncul pertama, file read-only. Klik "Enable Editing" untuk lanjut. *(Kalau screenshot terlewat, buka ulang file yang masih ber-MOTW untuk menangkapnya.)* |
| Peringatan makro ("Enable Content") | Ya — dalam 2 bentuk | <img width="1918" height="459" alt="image" src="https://github.com/user-attachments/assets/314ea29e-a2a9-4049-a3d1-b7b2832a234a" />| **Saat MOTW aktif:** banner **merah "SECURITY RISK – Microsoft has blocked macros... the source of this file is untrusted"**, TANPA tombol Enable Content (makro diblok total). **Setelah MOTW dihapus (Unblock):** berganti jadi banner kuning klasik **"Enable Content"**. |
| Message box PoC muncul (makro jalan) | Tidak → lalu Ya | <img width="1918" height="1006" alt="image" src="https://github.com/user-attachments/assets/d6be90ac-919e-4b7f-aeee-5635273debff" />| **Saat MOTW aktif:** tidak muncul (makro diblok sebelum jalan). **Setelah Unblock + klik Enable Content:** message box "PoC Bab 9 – Client-Side Exploitation (Lab)" muncul → makro berjalan. |

## Cek Tanda "dari Internet"
- Windows: klik kanan `Doc1.docm` → Properties → tab General. Ada tombol/checkbox
  **"Unblock"** dengan teks "This file came from another computer and might be
  blocked to help protect this computer." → mengonfirmasi file ditandai Zone Internet (MOTW).

## Pertanyaan

**1. Ada beda perilaku kalau file diekstrak dari ZIP dibanding diunduh langsung?**
Ya. MOTW menempel pada file yang diunduh (termasuk ZIP-nya). Apakah tanda itu
**diturunkan** ke file hasil ekstrak tergantung archiver: Windows Explorer dan
archiver modern menurunkan MOTW ke file hasil ekstrak (peringatan tetap muncul),
tapi sebagian archiver lama dulu **tidak** menurunkannya — sehingga `.docm` hasil
ekstrak jadi "bersih" dan tidak kena Protected View/blokir makro. Teknik membungkus
dokumen dalam ZIP/ISO/IMG memang dipakai attacker untuk melewati MOTW. Pelajaran
defensif: jangan mengandalkan user "pasti dapat peringatan".

**2. Di titik mana user awam paling mungkin tetap klik lanjut? Kenapa?**
Di **"Enable Content"** (lapisan makro), bukan Protected View. Di Protected View
isi dokumen sudah terlihat, jadi klik "Enable Editing" terasa wajar. Lalu attacker
membuat isi dokumen tampak rusak/blur dengan pesan palsu "klik Enable Content untuk
melihat" — memanfaatkan rasa penasaran/urgency (rekayasa sosial Bab 8). Catatan dari
lab ini: ketika MOTW aktif di Office baru, tombol Enable Content **tidak diberikan**
sama sekali (blokir keras), sehingga keputusan user dihilangkan — justru inilah
kontrol yang paling efektif.
