<div align="center">

# Berkontribusi ke IndoTech-DevKit

**[Bahasa Indonesia](CONTRIBUTING.md)** · <a href="CONTRIBUTING.en.md">English</a>

Terima kasih telah mau membantu. Panduan ini pendek dan sengaja begitu:
proyek ini tidak butuh Contributor yang banyak, ia butuh orang yang mengukur
dengan jujur.

</div>

---

## Aturan yang tidak bisa ditawar

### 1. Tidak ada klaim yang belum diukur

Ini satu-satunya aturan yang benar-benar wajib, dan sisanya mengikuti darinya.

Kalau menyangkut Termux, jalankan di Termux. Kalau menyangkut Android, uji di
ponsel. Kalau kamu tidak bisa mengukurnya, tuliskan batasnya secara eksplisit di
komentar dan di pesan commit — **bukan** perkiraan yang terdengar yakin.

Ada contoh nyata di riwayat proyek ini:

- Klaim "code-oss bisa jalan di Termux" ternyata benar untuk CLI-nya, tapi
  `tunnel` dan `serve-web` keduanya gagal karena binarinya tidak ada. Yang ditulis
  di README adalah hasil pengukurannya, termasuk bahwa string `code-tunnel`
  muncul **nol kali** di dalam bundel 104 MB.
- Klaim "panel sudah meneruskan allowlist dengan benar" ternyata belum-founded:
  tombol editor meminta `code:start`, yang menjalankan code-server, yang tidak
  bisa jalan di ponsel. Sekarang ia meminta `code:open`.

Pesan commit yang bagus menjelaskan **apa yang diukur dan apa yang berubah**,
bukan hanya file apa yang disentuh.

> **Hindari kata-kata ini di komentar dan dokumentasi**, kecuali kalau memang itu
> yang terjadi: "tampaknya", "seharusnya", "mungkin", "kayaknya", "biasanya",
> "pasti jalan".
>
> Kalau kamu menulis "tampaknya jalan" lalu mengukurnya dan memang jalan,
> tulis "jalan". Kalau tidak sempat mengukur, tulis "belum diukur" dan katakan
> bagaimana mengukurnya. Keduanya jujur; yang ketiga tidak.

### 2. Jangan menambahkan dependensi tanpa alasan yang tertulis

Semua yang DevKitpasang berasal dari repositori Termux sendiri. Kalau kamu
menambahkan dependensi:

- sebutkan paket Termux-nya, bukan nama yang nicer di halaman web,
- jelaskan kenapa paket bawaan tidak cukup,
- pastikan ukuran dan dependensinya masih muat di ponsel yang kecil.

xterm.js adalah satu-satunya pihak ketiga, dan itu pun disengaja: ia di-*vendor*
ke dalam repo, bukan dimuat dari CDN, supaya panel tetap jalan tanpa jaringan dan
halaman yang memegang shell tidak menjalankan skrip dari server orang lain.

### 3. Jangan pernah menutup socket di anak proses PHP

Ini bukan preferensi gaya. Menghapus socket di anak proses PHP adalah
cara yang membuat anak proses itu mati tanpa jejak.

PHP tidak bisa melakukannya:

- `fclose($fd)` dengan integer → `TypeError`.
- `get_resources()` mengembalikan stream internal PHP juga, dan menutupnya adalah
  fatal engine yang tidak bisa ditangkap.

Tiga cara berbeda sudah dicoba dan semuanya membuat anak proses mati diam-diam:
pid kembali, log kosong, proses tidak ada. Karena itu bagian ini dihapus dan
kegagalannya dicatat di komentar, supaya tidak ada yang mencobanya lagi.

Konsekuensinya, `Detach` dipakai hanya dari CLI `devkit`, bukan dari
permintaan web. Socket yang diwariskan bertahan selama satu perintah, bukan
selama usia service.

### 4. Pesan commit

Satu perubahan, satu paragraf atau lebih, ditulis dalam bahasa Inggris (kecuali README dan
dokumentasi, yang dwibahasa dengan Bahasa Indonesia sebagai bawaan).

Format yang dipakai proyek ini: **apa yang terjadi, kenapa begitu, dan apa yang
diukur untuk menyertainya.**

> Add `devkit doctor`, and fix the three faults it turned up
>
> Verifying the stack end to end rather than reading it found three things that
> no existing command reported, because none of them was a service being down.
>
> A site added after the proxy started was never reachable by name. `site:add`
> wrote the vhost, issued the certificate, reloaded the web server and printed
> success, and the proxy's configuration — which is only rewritten when the
> proxy starts — still had no rule for the new name. …

Yang penting: **kenapa**, dan bukan hanya apa. "Perbaiki bug" tidak berguna
untuk siapa pun.

Jangan menambahkan trailer `Co-Authored-By` untuk alat apa pun. Proyek ini
dirilis oleh satu orang, dan itu yang ingin terlihat di histornya.

---

## Menjalankan secara lokal

```sh
git clone https://github.com/Ismail-IndoTech/IndoTech-DevKit.git
cd IndoTech-DevKit
./install.sh --dry-run     # tanyakan semuanya, cetak rencananya, jangan pasang apa pun
```

Untuk mencoba perubahan tanpa merusak stack yang sedang dipakai:

```sh
DEVKIT_HOME=~/devkit-ujicoba ./install.sh --yes
```

`DEVKIT_HOME` membuat semua state, log, dan sertifikat pindah ke sana. Itu
separation yang membuat aman bereksperimen di ponsel yang juga dipakai bekerja.

---

## Sebelum mengirim pull request

Jalankan semuanya. Semuanya cepat, dan semuanya pernah menangkap sesuatu.

```sh
# Sintaks
for f in $(find payload/devpanel tools -name '*.php') payload/bin/devkit; do php -l "$f"; done
for f in install.sh install-remote.sh lib/*.sh lib/steps/*.sh payload/bin/*; do bash -n "$f"; done
node --check payload/devpanel/public/assets/panel.js

# Konsistensi
python3 tools/check-panel-ids.py      # id DOM di panel: 0 hilang, 0 dobel
php tools/check-allowlist.php         # daftar izinkan panel dan CLI harus cocok
python3 tools/check-prose.py          # teks dokumentasi yang rusak saat ditulis

# Yang paling penting: apakah ini benar-benar bekerja?
devkit doctor
```

`devkit doctor` keluar dengan status bukan nol kalau ada kegagalan, jadi bisa
dipakai di CI. Peringatan tidak menggagalkan, dengan sengaja.

### Kalau menyangkut panel

Satu tab diklik satu per satu. Panel adalah tempat orang mencari tahu apa yang
sedang terjadi, jadi apa pun yang ditampilkan di sana harus jujur:

- Kalau sesuatu tidak tersedia, tuliskan **alasannya**, bukan "tidak tersedia".
- Kalau angka tidak bisa dibaca, tuliskan "tidak diketahui", jangan `0`.
- Kalau ada perintah yang tidak berhasil, tampilkan keluaran kesalahannya.

---

## Apa yang sedang dikerjakan

Buka [Issues](https://github.com/Ismail-IndoTech/IndoTech-DevKit/issues). Yang
biasanya berguna:

- **Laporan bug** yang menyebut model perangkat, versi Android, dan keluaran
  `devkit doctor`. Tiga baris ini sering lebih berharga daripada deskripsi panjang.
- **Pengukuran yang membuktikan atau menyangkal** suatu klaim. Kalau kamu menemukan
  bahwa sesuatu ternyata bisa jalan di ponsel, atau ternyata tidak bisa, itu
  kontribusi yang sangat berharga — lebih berharga daripada kode.
- **Dokumentasi** dalam Bahasa Indonesia. Kalau ada bagian yang membingungkan,
  bagian itu yang perlu diperbaiki.

Yang tidak membantu: permintaan fitur tanpa-alasannya, atau patch yang menambah
dependensi.

---

## Donasi

Proyek ini gratis, dan tidak akan pernah menjadi berbayar. Kalau kamu
mau membantu tetap begitu:

<div align="center">

**DANA — 08121600058** (a.n. Ismail)

<img src="docs/qris.jpg" alt="QRIS DANA" width="240">

</div>

Dana digunakan untuk biaya server, penerbitan, dan waktu untuk menguji di
perangkat nyata. Tidak ada yang dijanjikan balik.

---

## Lisensi

Kontribusi dilisensikan di bawah [LICENSE](LICENSE) yang sama — MIT, atas nama
**Ismail.IndoTech**. Dengan mengirim pull request, kamu setuju bahwa kontribusi
itu dirilis dengan lisensi itu.

---

<div align="center">

Made with :heart: by [**Ismail.IndoTech**](https://it-app.id) · [it-app.id](https://it-app.id)

</div>
