> 📚 **Dokumentasi:** [Indeks lengkap ID/EN](DOCS.md) · [English documentation index](DOCS.en.md)

<div align="center">

<img src="docs/logo.png" alt="IndoTech" width="128" height="128">

# IndoTech-DevKit


### Seluruh stack pengembangan Anda. Satu kantong.

**Mengubah Termux yang baru dipasang di ponsel Android menjadi stack
pengembangan yang benar-benar jalan** — web server, PHP, database, SSL, GitHub,
hosting project, panel kontrol berbasis browser, dan SSH agar semuanya bisa
dikendalikan dari keyboard yang sungguhan.

[![Versi](https://img.shields.io/badge/versi-1.0.0-FF2D20?style=flat-square)](VERSION)
[![Lisensi: MIT](https://img.shields.io/badge/lisensi-MIT-0EA5E9?style=flat-square)](LICENSE)
[![Termux](https://img.shields.io/badge/termux-wajib-000000?style=flat-square&logo=android)](https://f-droid.org/packages/com.termux/)
[![Android](https://img.shields.io/badge/android-7.0%2B-3DDC84?style=flat-square&logo=android)](https://developer.android.com/)
[![Buatan Indonesia](https://img.shields.io/badge/buatan%20di%20Indonesia-FF2D20?style=flat-square)](https://it-app.id)

**Tanpa akun. Tanpa login. Tanpa clone. Satu perintah.**

```sh
curl -fsSL https://raw.githubusercontent.com/Ismail-IndoTech/IndoTech-DevKit/main/install-remote.sh | bash
```

ORMOS itulah seluruh pemasangan. Skrip ini mengukur ponsel sebelum menawarkan
apa pun, melakukan survei singkat, memasang tepat apa yang Anda pilih,
menyalakan stack, lalu membuka panel pengaturan di browser.

<div align="center">

**[Bahasa Indonesia](README.md)** · <a href="README.en.md">English</a>

</div>

</div>

---

## Yang dipasang di ponsel Anda

<div align="center">

| | | | | |
|:---:|:---:|:---:|:---:|:---:|
| <img src="https://cdn.simpleicons.org/laravel/FF2D20" width="44" height="44" alt="Laravel"><br>**Laravel**<br>12 · 13 | <img src="https://cdn.simpleicons.org/php/777BB4" width="44" height="44" alt="PHP"><br>**PHP**<br>8.5 | <img src="https://cdn.simpleicons.org/mariadb/003545" width="44" height="44" alt="MariaDB"><br>**MariaDB**<br>kompatibel MySQL | <img src="https://cdn.simpleicons.org/postgresql/4169E1" width="44" height="44" alt="PostgreSQL"><br>**PostgreSQL**<br>opsional | <img src="https://cdn.simpleicons.org/phpmyadmin/FFF000" width="44" height="44" alt="phpMyAdmin"><br>**phpMyAdmin**<br>sudah terhubung |
| <img src="https://cdn.simpleicons.org/apache/EB1B26?viewbox=auto" width="44" height="44" alt="Apache"><br>**Apache**<br>atau Nginx | <img src="https://cdn.simpleicons.org/nginx/009639" width="44" height="44" alt="Nginx"><br>**Nginx**<br>pilihan Anda | <img src="https://cdn.simpleicons.org/flutter/02569B" width="44" height="44" alt="Flutter"><br>**Flutter**<br>web + Android | <img src="https://cdn.simpleicons.org/composer/FFFFFF" width="44" height="44" alt="Composer"><br>**Composer**<br>dependensi | <img src="https://cdn.simpleicons.org/git/F05032" width="44" height="44" alt="Git"><br>**Git**<br>+ GitHub CLI |
| <img src="https://cdn.simpleicons.org/nodejs/5FA04E" width="44" height="44" alt="Node.js"><br>**Node.js**<br>LTS atau current | <img src="https://cdn.simpleicons.org/python/3776AB" width="44" height="44" alt="Python"><br>**Python**<br>CPython | <img src="https://cdn.simpleicons.org/rust/000000" width="44" height="44" alt="Rust"><br>**Rust**<br>opsional | <img src="https://cdn.simpleicons.org/golang/00ADD8" width="44" height="44" alt="Go"><br>**Go**<br>opsional | <img src="https://img.shields.io/badge/-ripgrep-CC4A31?style=flat-square"><br>**ripgrep**<br>`rg`, bukan grep |

</div>

<table align="center">
<tr>
<td width="33%"><b>Termux</b><br><span style="color:#888">Lingkungan tempat semuanya berjalan</span></td>
<td width="33%"><b>OpenSSH</b><br><span style="color:#888">Masuk dari komputer Anda</span></td>
<td width="33%"><b>Squid</b><br><span style="color:#888">Membuat nama <code>.app</code> bisa diresolve</span></td>
</tr>
</table>

---

## Pasang

### Cara singkat

```sh
curl -fsSL https://raw.githubusercontent.com/Ismail-IndoTech/IndoTech-DevKit/main/install-remote.sh | bash
```

Tanpa akun GitHub, tanpa SSH key, tanpa `git clone`. Skrip ini mengunduh tarball
rilis, mengekstraknya di direktori sementara, lalu menyerahkan pekerjaan ke
installer.

### Ingin membacanya dulu?

Selalu wajar untuk sesuatu yang memasang perangkat lunak:

```sh
curl -fsSL https://raw.githubusercontent.com/Ismail-IndoTech/IndoTech-DevKit/main/install-remote.sh -o install-remote.sh
less install-remote.sh          # baca dulu
bash install-remote.sh          # baru jalankan
```

### Pilihan

```sh
bash install-remote.sh --dry-run        # tanya semuanya, cetak rencananya, tidak pasang apa pun
bash install-remote.sh --device-info    # laporkan apa yang bisa dijalankan ponsel ini, tidak ubah apa pun
bash install-remote.sh --yes            # non-interaktif, bawaan yang aman
bash install-remote.sh --answers=FILE   # ulangi atau sunting jawaban instalasi sebelumnya
```

### Dari clone

```sh
git clone https://github.com/Ismail-IndoTech/IndoTech-DevKit.git
cd IndoTech-DevKit && ./install.sh
```

---

## Apa yang diukur sebelum installer bertanya apa pun

Ponsel bukan server kecil, dan paket yang jalan di laptop tidak semuanya jalan
di sini. Jadi installer mengukur dulu, dan hasil pengukuran itulah yang menentukan
apa yang boleh ditawarkan:

- **Membaca perangkat.** Arsitektur, rilis Android, jumlah inti CPU, total RAM,
  ruang penyimpanan kosong, ukuran halaman memori, apakah `su` benar-benar
  memberi uid 0, apakah bootloader terkunci.
- **Hanya menawarkan yang benar-benar ada di sini.** Setiap versi di setiap menu
  berasal dari metadata apt perangkat ini sendiri, dan repositori asal tiap versi
  disebutkan.
- **Menyelesaikan pohon dependensi sungguhan** dengan `apt --print-uris` lalu
  membandingkannya dengan ruang kosong, sehingga rencana yang tidak muat
  ditolak *sebelum* mengunduh, bukan di tengah jalan.
- **Menonaktifkan, bukan menebak.** Server database di ponsel 900 MB ditampilkan
  tapi tidak bisa dipilih, lengkap dengan pengukuran yang menyingkirkan itu.

<details>
<summary>Contoh apa yang ditolak, dan alasannya</summary>

| Pesan | Apa artinya |
| --- | --- |
| `butuh sekitar 1200 MB RAM; perangkat ini punya 900 MB` | Angka sungguhan dari `/proc/meminfo`. Database yang dibunuh Android di tengah tugas lebih buruk daripada tidak ada. |
| `butuh sekitar 300 MB ruang kosong untuk berjalan; perangkat ini punya 120 MB` | Ambang bawah untuk menjalankannya, terpisah dari ukuran unduhan. |
| `build 32-bit dari toolchain ini bukan bagian dari Termux` | Paketnya memang tidak ada untuk arsitektur ini. Tidak ada installer yang bisa menawarkannya. |
| `Android 13 (API 21) lebih lama dari minimum Android 7 (API 24)` | Di bawah batas. Termux sendiri tidak mendukung ini. |

Tidak ada flag untuk memaksa: pilihan yang sengaja dibuat harus terlihat sengaja.

</details>

---

## Panel pengaturan Anda

<div align="center">

`http://127.0.0.1:8898/__panel/`

</div>

Aplikasi web lokal yang menjalankan perintah `devkit` yang sama persis dengan
yang akan Anda ketik di terminal. Ia hanya terikat ke `127.0.0.1`, tidak ke
yang lain.

| Tab | Fungsinya |
| --- | --- |
| **Services** | Apa yang jalan, mulai/henti/restart, dan server mana yang mengendalikan |
| **Sites** | Site terdaftar, URL-nya, dan tautan yang benar-benar jalan sekarang |
| **SSL** | Certificate authority lokal dan sertifikat dari CA itu |
| **SSH** | Masuk dari komputer, kelola kunci yang boleh masuk |
| **Projects** | Buat project, build, jalankan — **dengan terminal sungguhan di halaman yang sama** |
| **Editor** | Buka folder di VS Code, dan penjelasan jujur kenapa tunnel tidak bisa dibuat di ponsel |
| **Versions** | Versi tiap komponen, dan apa yang berubah kalau Anda menggantinya |
| **GitHub** | Masuk, daftar repository, push project |
| **Device** | Sebenarnya apa ponsel ini, dan apa yang sedang dilacaknya sekarang |
| **Environment** | Versi dan path yang berhasil diresolusi stack ini |
| **Logs** | Keluaran terbaru dari service mana pun |

### Terminal di halaman Projects

Tidak ada tab Terminal tersendiri. Terminalnya berada **di halaman Projects**,
karena di situlah perintah panjang dijalankan — dan membuat Laravel adalah
composer yang menyelesaikan pohon dependensi, git yang meng-clone repository,
build tool yang mengompilasi. Tidak ada yang bisa dipercepat, hanya bisa
diawasi.

Jadi panel menjalankan perintah sungguhan secara terpisah dan mengikuti
keluarannya apa adanya, bukan menahan halaman terbuka selama dua menit lalu
mencetak baris terakhir:

```
$ laravel new myapp

  Which starter kit do you want?  laravel/laravel
  Installing dependencies...
  Package operations: 68 installs, 0 updates, 0 removals
  ...
  Application key set successfully.
```

Terminal di panel adalah **sesi SSH sungguhan ke perangkat ini** — xterm.js di
atas WebSocket ke `ssh` di `127.0.0.1`, memakai kunci milik panel yang
diotorisasi lewat `Ssh::authorize` seperti kunci biasa. Bukan shell kedua: shell
yang dimulai dari permintaan web melewati autentikasi, daftar kunci, dan log
SSH, lalu panel dan laptop Anda jadi berbeda pendapat tentang siapa yang boleh
masuk. Lewati `ssh` berarti panel datang lewat pintu yang sama dengan laptop,
jadi **menguji terminal berarti menguji SSH**, dan mencabut kunci panel
menghapus aksesnya persis seperti mencabut kunci laptop.

```sh
devkit terminal:status           # listener, kunci, dan URL WebSocket-nya
devkit terminal:start            # listening di 127.0.0.1:8896
devkit terminal:restart
```

### Tab SSH

Login sungguhan di perangkat, jadi `devkit` dan perintah lain bisa dipakai dari
komputer yang punya keyboard.

```sh
devkit ssh:install                    # menulis config, host key, dan berkas kunci
devkit ssh:authorize "$(cat laptop.pub)" "laptop kerja"
devkit ssh:start                      # listening di 0.0.0.0:8022
devkit ssh:connect                    # hanya perintah yang harus dijalankan di komputer
```

```
ssh -p 8022 u0_a247@192.168.2.76
```

Publickey adalah **satu-satunya** metode autentikasi — tidak ada kata sandi yang
bisa ditebak, dipecahkan, atau dipakai ulang dari kebocoran di tempat lain.
Portnya ada di jaringan Anda, dan itu harga jujur dari hal tersebut, dan panel
mengatakannya setiap kali dinyalakan.

> **Catatan wireless debugging:** ponsel tidak bisa menjangkau dirinya sendiri
> lewat Wi-Fi, jadi `ssh …@192.168.2.76` harus dijalankan *dari komputer Anda*.
> Di ponsel sendiri, pakai `adb reverse` atau terminal di halaman Projects.

---

## Site pertama Anda

```sh
devkit site:add --id=htdocs --name=htdocs --host=htdocs.app \
                --root=~/htdocs --http=8080 --https=8443
```

`htdocs.app` mendapat sertifikat, vhost, dan port sendiri. Ia menjawab di:

- `http://htdocs.localtest.me:8080/` — resolve ke 127.0.0.1, tidak butuh apa pun
- `https://htdocs.127.0.0.1.sslip.io:8443/` — TLS, juga tidak butuh apa pun
- `http://htdocs.app:8080/` — yang cantik, lewat proxy (lihat di bawah)

> **HTTPS pada nama `.app` tidak punya rute.** Sebuah `CONNECT` tidak bisa
> dipindahkan ke port lain, jadi `https://htdocs.app/` tidak akan pernah sampai
> ke perangkat ini. Untuk HTTPS, pakai nama `localtest.me` atau `sslip.io`
> dengan port di dalam URL — keduanya tidak butuh proxy sama sekali.
> `devkit proxy:doctor` mencatat batas ini setiap kali dijalankan.

### Memercayai certificate authority lokal

| Penyimpanan | Siapa yang menulis | Apa yang ditampilkan panel |
| --- | --- | --- |
| Bundel CA Termux | DevKit, atas permintaan | **Termux: dipercaya / tidak dipercaya** — dicek dengan membaca bundel dan lewat jabat tangan TLS sungguhan |
| Penyimpanan CA pengguna Android | Settings Android, manual | **Browser: tidak diketahui** — tidak bisa ditulis dari Termux |

`ca:install` (*Trust in Termux* di tab SSL) mengisi baris pertama. Setengah
kedua butuh tiga ketukan, sekali saja, di ponsel:

1. Buka `http://127.0.0.1:8898/__panel/ca.crt` dan unduh.
2. **Install certificate** → **VPN and apps**.
3. Konfirmasi kunci layar Anda, lalu tutup dan buka ulang browser.

### Nama `.app` dan proxy

`localtest.me` dan `sslip.io` resolve ke 127.0.0.1 dengan gratis. `.app` tidak,
jadi proxy ada untuk meresolvenya di perangkat:

1. Buka jaringan Wi-Fi yang sedang dipakai.
2. Atur proxy ke **Manual**, host `127.0.0.1`, port `8899`.

Ini hanya memengaruhi ponsel Anda, dan hanya selama diatur. Panel tidak pernah
berada di belakang proxy, jadi tetap jalan di jaringan mana pun.

Untuk memeriksa semua tautan sekaligus, beserta kode jawabannya:

```sh
devkit proxy:doctor
```

---

## Project

```sh
devkit project:root                             # di mana project tinggal
devkit project:templates                        # apa yang bisa dibuat, apa yang kurang
devkit project:new blog --template=static       # buat dan daftarkan sitenya
devkit project:new app --template=laravel       # laravel new + composer install
devkit project:new app --template=flutter       # flutter create
devkit project:build app --template=flutter     # flutter build web
devkit project:artisan blog migrate             # jalankan artisan di dalam project
devkit project:remove blog --delete
```

Setiap project baru langsung didaftarkan sebagai site, mendapat sertifikat, dan
muncul di panel. Template adalah set bawaan ditambah apa pun yang Anda tambahkan
di `state/templates.json` — tiap entri adalah array argv, tidak pernah string
shell.

**Android Run** adalah versi panel yang jujur dari "jalankan di ponsel": kartunya
hanya muncul kalau project-nya benar-benar Flutter, `flutter` dan `adb`
benar-benar ada, dan perangkat benar-benar tersambung. Yang kurang disebutkan,
bukan ditawarkan sebagai tombol yang tidak melakukan apa pun.

---

## `code .` — membuka folder di VS Code

Berdiri di dalam folder project, lalu ketik `code .`:

```sh
cd ~/projects/myapp
code .
```

Ini mencoba membuat **tunnel ke vscode.dev** lebih dulu — workspace-nya ada di
ponsel, terbuka di browser mana pun, tanpa client yang perlu dipasang. Kalau
binari tunnel ada, itu yang dipakai.

**Di ponsel tunnel itu tidak bisa dibuat, dan alasannya layak dinyatakan
terang-terangan** daripada dibiarkan menjadi pesan `spawn ... ENOENT`:

- `code-oss` memang dikemas untuk Termux, dan CLI-nya jalan.
- `code-oss tunnel` dan `code-oss serve-web` sama-sama ada di daftar `--help`,
  dan keduanya gagal dengan cara yang persis sama:
  `spawn .../bin/code-tunnel-oss ENOENT`.
- Binarinya tidak ada di paket itu, dan tidak ada di dalam bundel 104 MB:
  string `code-tunnel` muncul nol kali di dalamnya. Yang ada adalah *pustakanya*
  (`@microsoft/dev-tunnels-connections`) — itulah sebabnya subcommand-nya
  ditawarkan. Kode yang akan memanggil programnya ada; programnya tidak.
- Microsoft tidak menerbitkan binarinya secara terpisah. API yang melayani build
  VS Code tidak memuat aset tunnel sama sekali, karena tunnel ikut dalam aplikasi
  desktop.

Yang benar-benar bekerja ada di bawah ini, dan itu bukan hadiah simpatik.

```sh
code .                      # dari dalam folder project
code . --print              # hanya menampilkan, tanpa melakukan
devkit code:address         # alamat perangkat ini, dan apa yang kurang
devkit code:steps           # panduan setup sekali jalan
```

`code .` mencetak blok `~/.ssh/config` untuk ditempel di komputer:

```
Host indotech-devkit
    HostName 192.168.2.76
    User u0_a247
    Port 8022
    IdentityFile ~/.ssh/id_ed25519
```

Lalu di VS Code: **Remote-SSH: Connect to Host…** → `indotech-devkit`.

Sesi editnya sama persis: file ada di ponsel, editor jalan di komputer, terminal
adalah shell sungguhan di ponsel, dan lalu lintas tidak pernah menyentuh server
siapa pun.

Kuncinya dibuat **di komputer**, dan itu yang ditulis di config — `IdentityFile`
adalah kunci komputer sendiri, tidak pernah path di dalam perangkat ini:

```sh
ssh-keygen -t ed25519                                          # di komputer
devkit ssh:authorize "$(cat ~/.ssh/id_ed25519.pub)" "laptop"    # di ponsel
```

---

## Perintah `devkit`

Semua yang dilakukan panel, dari shell. `devkit help` mendaftarnya.

### Perintah yang paling perlu diingat

```sh
devkit doctor                  # periksa perangkat ini dari ujung ke ujung
IndoTech-DevKit                # nyalakan semuanya, lalu buka panelnya
```

`devkit doctor` menjawab pertanyaan yang berbeda dari `service:status`.
"`service:status` menjawab *apakah dia jalan*"; `doctor` menjawab *apakah dia
bekerja* — dan di situlah masalah sesungguhnya hidup. Setiap port ditanya apakah
ada yang menjawab, setiap site ditanya apakah URL-nya mengembalikan halaman,
setiap sertifikat ditanya apakah masih bisa diverifikasi.

```
  [1mDevice[0m
    ok    PHP                        8.5.1
    ok    php-openssl                loaded
    ...
  [1mServices[0m
    ok    Apache                     running
    warn  DevKit panel               stopped
                                    -> devkit service:start panel
  [1mPanel[0m
    ok    Reach it                   http://127.0.0.1:8898/__panel/
  ...
  22 ok, 2 warning(s), 1 failure(s), 5 note(s)
```

Peringatan adalah pilihan atau batas, bukan kesalahan — perangkat tanpa site
adalah perangkat yang bekerja, dan pemeriksaan yang gagal karena itu akan
diabaikan lalu jadi tidak berguna. Perintah ini keluar dengan status bukan nol
hanya kalau ada `failure`.

### Memilih versi komponen

```sh
devkit versions:list                    # versi tiap komponen, satu baris
devkit versions:show php                # satu komponen, plus apa yang rusak kalau diganti
devkit versions:use <komponen> <versi>   # pasang, lalu restart yang bergantung padanya
devkit versions:update <komponen>       # versi terbaru yang tersedia di perangkat ini
```

Setiap angka dibaca dari perangkat ini dengan `apt-cache madison` dan `dpkg`. Di
ponsel biasanya **hanya ada satu versi** per komponen — repositori Termux tidak
membawa build lama — jadi panel menuliskannya sebagai "1 versi saja" beserta
alasannya, bukan menyembunyikan dropdown berisi satu entri.

Changing PHP restarts Apache, because Apache holds the module for the interpreter
that was installed; without that restart the server answers and returns 500 on
every PHP request. `versions:show` lists the blocker for each component before
anything is installed, and the download and disk cost are measured first with
apt's own dependency tree.

### Daftar singkat

```sh
devkit service:status                  # apa yang jalan
devkit service:start all               # semua: web server, queue, proxy, ssh, terminal, panel
devkit service:stop all                # semuanya, berhenti dari yang bergantung dulu
devkit site:list                       # site terdaftar
devkit ca:create && devkit ca:install  # certificate authority lokal
devkit project:new app --template=laravel
devkit ssh:start                       # masuk dari komputer
devkit env:info                        # versi dan path yang diresolusi stack ini
devkit device:info                     # apa sebenarnya ponsel ini
```

Tambahkan `--json` pada perintah mana pun untuk keluaran yang bisa dibaca mesin.

### Perintah yang cukup diingat satu

Setelah memasang, nama ini bekerja di shell Termux:

```sh
IndoTech-DevKit
```

Ia menyalakan apa pun yang belum jalan, menunggu sampai panel benar-benar
menjawab, lalu membukanya di tab browser baru. Menjalankannya dua kali tidak
berbahaya.

---

## Cara susunannya

| Service | Port | Kenapa |
| --- | --- | --- |
| Panel | `127.0.0.1:8898` | Server bawaan PHP. Terpisah dari proxy, jadi proxy yang rusak tidak bisa menyembunyikan ruang kendali. |
| Terminal | `127.0.0.1:8896` | Listener WebSocket sendiri. `php -S` tidak bisa menaikkan koneksi ke WebSocket. |
| Proxy | `127.0.0.1:8899` | Squid. Meresolve nama `.app` dan meneruskan semuanya ke internet. |
| Web server | `8080` / `8443` | Termux tidak bisa mengikat di bawah 1024 tanpa root. |
| Database | `3306` | Sudah di atas 1024, jadi tidak disentuh. |
| SSH | `8022` | Satu-satunya service yang menjawab di jaringan, dengan sengaja. |
| Site project | `9000+` | Satu port per project terdaftar. |

**Panel tidak pernah menjalankan perintah yang disebut browser.** Setiap aksi
dicocokkan terhadap daftar izinkan tetap berisi subcommand `devkit`, dan
sesuatu pun yang mengubah keadaan juga memerlukan token per-sesi yang dibawa
halaman itu sendiri. Scaffold adalah id template yang dicari di tabel, tidak
pernah baris perintah yang dirakit dari teks.

Tidak ada yang dilaporkan tanpa diukur lebih dulu. Pembacaan yang tidak diizinkan
Android ditampilkan sebagai tidak tersedia *beserta alasannya*, tidak pernah
sebagai `0` yang tidak bisa dibedakan dari jawaban sungguhan.

---

## Saat ada yang salah

```sh
devkit doctor                # mulai dari sini: ia yang tahu bagian mana yang rusak
devkit env:info              # stack terpasang memakai apa
devkit service:status        # apa yang jalan, dan URL-nya
```

- **`service:start` bilang sudah jalan padahal tidak** — yang menentukan adalah
  port, dan berkas pid basi tidak lagi menimpanya. `devkit service:restart`
  aman kapan saja.
- **Port bilang tertahan tapi tidak ada yang menjawab** — listener-nya masih ada
  di kernel sementara prosesnya sudah hilang. Android hanya melepaskannya saat
  aplikasi di-*force stop*. Buka ulang Termux, lalu jalankan lagi. Ini tidak bisa
  diperbaiki dari dalam, dan pesannya mengatakannya terus terang.
- **HTTPS menampilkan peringatan di browser** — itu wajar sampai CA dipasang di
  penyimpanan Android sendiri; panel tidak bisa melakukannya untuk Anda.
- **Nama `.app` tidak resolve** — proxy Wi-Fi belum diatur, atau proxy sedang
  mati. `localtest.me` selalu jalan tanpa keduanya.
- **`code .` bilang tunnel tidak tersedia** — itu jawaban yang akurat, bukan
  kegagalan mencoba. Yang dikembalikan adalah jalur Remote-SSH, yang bekerja dan
  tidak butuh apa pun yang belum ada di perangkat ini.

Lebih banyak di [docs/troubleshooting.md](docs/troubleshooting.md).

---

## Copot

```sh
devkit service:stop all
rm -rf ~/IndoTech_Project          # atau ke mana pun DEVKIT_HOME menunjuk
rm -f  $PREFIX/bin/devkit $PREFIX/bin/IndoTech-DevKit $PREFIX/bin/code
pkg uninstall apache2 php mariadb   # apa pun yang dilaporkan installer terpasang
```

Berkas jawaban (`~/.devkit-answers`) mencatat persis apa yang dipasang.

---

## Akses & Lisensi

Repositori ini adalah **karya saya sendiri**, **Mochamad Ismail Soleh** (Ismail-IndoTech), dan dilisensikan secara khusus di bawah **[MIT License](LICENSE)**.

Untuk memperoleh akses ke **repositori sumber (privat)** dari proyek ini, silakan hubungi saya sebagai **Creator Repo** melalui WhatsApp di nomor yang terdaftar pada akun DANA saya: **08121600058** — [wa.me/628121600058](https://wa.me/628121600058). Akses diberikan sesuai kesepakatan.
## Berkontribusi

Kontribusi sangat diterima, termasuk yang kecil. `devkit doctor` ada karena ada yang
mengukur alih-alih membaca — dan menemukan tiga kesalahan yang tidak
dilaporkan perintah mana pun.

```sh
git clone https://github.com/Ismail-IndoTech/IndoTech-DevKit.git
cd IndoTech-DevKit
./install.sh --dry-run          # pastikan tidak merusak apa pun
```

Sebelum mengirim pull request:

```sh
php -l pada tiap berkas .php
bash -n pada tiap .sh
node --check payload/devpanel/public/assets/panel.js
python3 tools/check-panel-ids.py
php tools/check-allowlist.php
devkit doctor
```

Aturan singkatnya: **tidak ada klaim yang belum diukur.** Jika menyangkut
Termux, jalankan di Termux. Rinciannya di
[CONTRIBUTING.md](CONTRIBUTING.md) — atau [in English](CONTRIBUTING.en.md).

---

## 💖 Kontribusi & Donasi

Jika aplikasi ini mempermudah operasional IT, DevOps, dan pengelolaan server di perusahaan atau proyek pribadi Anda, dukung pengembangannya dengan menjadi kontributor atau **Donatur Spesial**.  
*(Donatur dan Sponsor resmi akan kami cantumkan logo perusahaannya di repositori ini).*

Dukung pemeliharaan berkelanjutan, riset modul bypass panel baru, dan pembaruan sistem dengan berdonasi melalui:

<div align="center">

**DANA — 08121600058** (a.n. Mochamad Ismail Soleh)

<img src="docs/qris.jpg" alt="QRIS DANA IndoTech" width="280">

*Pindai dengan aplikasi DANA, BCA, Mandiri, GoPay, OVO, atau aplikasi pembayaran QRIS apa pun di Indonesia.*

</div>

Semua dana dialokasikan untuk biaya infrastruktur riset server, lisensi alat pengujian, dan pengembangan modul open-source IndoTech. Jika Anda atau institusi Anda ingin menjadi mitra resmi atau sponsor, silakan hubungi kami. Terima kasih atas dukungan Anda!

---

## Dokumentasi

<div align="center">

**[Bahasa Indonesia](README.md)** · <a href="README.en.md">English</a>

</div>

- [docs/how-it-works.md](docs/how-it-works.md) — cara installer mengukur, dan mengapa · [English](docs/how-it-works.en.md)
- [docs/panel.md](docs/panel.md) — setiap tab, setiap perintah, dan apa yang ditolaknya · [English](docs/panel.en.md)
- [docs/troubleshooting.md](docs/troubleshooting.md) — saat sesuatu tampak salah · [English](docs/troubleshooting.en.md)
- [CONTRIBUTING.md](CONTRIBUTING.md) — cara berkontribusi · [English](CONTRIBUTING.en.md)

---

## Lisensi

MIT — lihat [LICENSE](LICENSE). Hak cipta © 2026 **Ismail.IndoTech**.

<div align="center">

Dibuat dengan :heart: oleh [**Ismail.IndoTech**](https://it-app.id) · [it-app.id](https://it-app.id)

</div>