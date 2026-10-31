# Cara kerjanya

<a href="how-it-works.md">**Bahasa Indonesia**</a> · <a href="how-it-works.en.md">English</a>

Empat lapisan, masing-masing menjawab satu pertanyaan. Memisahkannya itulah yang
membuat installer bisa mengatakan hal-hal seperti "versi ini ada di repositori
ini, di perangkat ini" tanpa ada satupun yang ditulis mati di dalam kode.

```
lib/detect.sh   apa ini perangkat ini?          (fakta tentang mesin)
lib/device.sh   apa yang bisa dilakukan perangkat ini?  (penilaian atas fakta itu)
lib/catalog.sh  apa yang ada di repositori saat ini?
lib/core.sh     apa yang dijawab pengguna, dan ke mana hasilnya?
```

`install.sh` memiliki urutannya, rencananya, dan pestsaannya.

## 1. Deteksi perangkat — `lib/detect.sh` dan `lib/device.sh`

`detect.sh` mengukur. `device.sh` menilai.

`device.sh` tidak mengukur hal yang sama dua kali. Root, arsitektur, rilis
Android, dan ruang kosong sudah punya detektornya, dan itulah yang dipakai langkah
mode dan pemeriksaan awal; memakainya lagi berarti dua jawaban tidak mungkin
saling berbeda tentang fakta yang sama. Yang ditambahkan `device.sh` adalah
sinyal-sinyal yang tidak seorang pun mencari:

| Sinyal | Dibaca dari | Kenapa penting |
| --- | --- | --- |
| `DEVICE_API` | `ro.build.version.sdk` | Di bawah 24 tidak ada yang bisa dipasang sama sekali. |
| `DEVICE_RAM_MB` | `/proc/meminfo` | Menentukan apakah server database atau stack desktop realistis. |
| `DEVICE_PAGE_SIZE` | FPB dari alamat awal `/proc/self/maps` | Perangkat dengan halaman 16 KB merusak binary 4 KB prakompilasi. |
| `DEVICE_CORES` | `/proc/cpuinfo` | `nproc` hanya melaporkan inti yang boleh dipakai proses ini, dan Android mempersempit itu per aplikasi. |
| `DEVICE_FLAVOR` | `ro.product.build.flavor` | Build F-Droid ditandatangani dengan kunci berbeda, jadi `pkg upgrade` tidak mengelola paket pihak ketiga. |
| `DEVICE_BOOTLOADER` | `ro.boot.verifiedbootstate` | Menjelaskan kenapa segala sesuatu yang butuh sistem termodifikasi di luar jangkauan. |
| `DEVICE_SELINUX` | `getenforce` | Dilaporkan sebagai tidak diketahui saat Termux tidak bisa membacanya, tidak pernah ditebak. |

### Ukuran halaman tanpa `getconf`

Cara yang jelas untuk mendapatkan ukuran halaman memori adalah
`getconf PAGESIZE`, tapi `getconf` tidak ada di setiap instalasi Termux, dan
menarik Python untuk menggantikannya tidak masuk akal. Sebagai gantinya:
`/proc/self/maps` hanya pernah memulai sebuah mapping pada batas halaman, jadi
ukuran halaman adalah pangkat dua terbesar yang membagi setiap alamat di berkas
itu. FPB dari alamat-alamat itu persis itulah:

```sh
awk '
    function gcd(a, b,   t) { while (b) { t = a % b; a = b; b = t } return a }
    { split($1, range, "-"); v = strtonum("0x" range[1]); if (v > 0) g = (g == 0) ? v : gcd(g, v) }
    END { if (g > 0) print g }
' /proc/self/maps
```

4096 di perangkat biasa, 16384 di perangkat 16 KB. `getconf` hanya dipakai
sebagai cadangan kalau ini tidak menghasilkan apa pun — dan `getconf` memang
hilang di ponsel tempat ini ditulis, jadi cadangan itu kenyamanan, bukan rencana.

Ini lebih penting dari terdengar. Binary yang dikompilasi untuk halaman 4 KB bisa
gagal di perangkat 16 KB dengan cara yang terlihat seperti kerusakan, bukan
seperti ketidakcocokan ukuran halaman, dan kegagalannya datang setelah paket
terpasang dan berjalan.

### Penilaiannya

`device_component_blocker <component>` adalah satu-satunya tempat yang memutuskan
apakah sesuatu boleh dipakai. Ia mengembalikan kosong ketika jawabannya ya, dan
sebuah kalimat ketika jawabannya tidak. Setiap menu membaca kalimat itu dan
menonaktifkan opsinya, sehingga aturan yang sama tidak bisa ditegakkan di satu
menu dan dilupakan di menu lain.

Ambang sengaja dibuat konservatif. Server database yang dibunuh
low-memory killer Android di tengah pekerjaan Anda lebih buruk daripada tidak ada
server database sama sekali, jadi `mariadb` butuh 1200 MB meski ia akan berjalan
dengan ruang yang lebih sedikit.

## 2. Katalog paket — `lib/catalog.sh`

Setiap versi yang ditampilkan di setiap menu berasal dari metadata apt
perangkat ini sendiri.

`apt-cache policy <pkg>` memberi versi kandidat. `apt-cache madison <pkg>`
memberikan daftar versi lengkap **beserta repositori asal tiap versi**, dan
itulah asal informasi provenance di ringkasan rencana:

```
php | 8.5.1,8.5.0 | https://termux.niranjan.co/termux-main Packages
```

Kata pertama field 3 adalah URI repositori. Sisanya adalah
`suite component arch` diikuti nama berkas indeks.

`pkg_origin_note` membedakan tiga kasus yang mudah tertukar, dan yang dalam
praktiknya sangat penting:

- **Versi persisnya ada di sebuah repositori** — kasus biasa.
- **Versi yang *berbeda* ditawarkan** — Anda punya sesuatu yang terpasang yang
  tidak dimiliki repositori itu. `apt upgrade` tidak akan menyentuhnya, dan kalau
  Anda berada di repositori pihak ketiga inilah kasus yang menarik: `ripgrep`
  muncul sebagai "from termuxvoid (other version)" karena 0.14.1 terpasang
  sementara repositori menawarkan 0.13.0.
- **Terpasang tapi tidak ditawarkan di mana pun** — paketnya datang dari
  repositori yang sudah tidak diaktifkan, atau dibangun dengan tangan.

Kasus ketiga adalah yang paling sering disembunyikan installer. Di sini ia
dilaporkan terus terang.

## 3. Lapisan repositori — `lib/catalog.sh`

Termux tidak menaruh semuanya di satu tempat, dan tempat lain tidak selalu hidup.
Installer punya buku alamat repositori yang dikenal, dan untuk masing-masing yang
belum diaktifkan, memutuskan apakah layak ditawarkan.

### Menelusuri tanpa menebak URL

Pendekatan yang menggoda adalah menyusun URL indeks `Packages` dari URI
repositori lalu mengambilnya. Itu tidak berhasil: mirror berubah, beberapa
distribusi sudah tidak dilayani, kompresi berbeda-beda, dan URL yang 404 terlihat
persis seperti repositori tanpa paket. Kedua mode kegagalan berakhir sama:
menawarkan sesuatu yang tidak bisa berhasil kepada pengguna.

Sebagai gantinya installer menjalankan **apt itu sendiri**, di dalam sandbox,
terhadap konfigurasi sekali pakai:

```sh
apt-get update \
  -o Dir::Etc::sourcelist="$probe_dir/sources.list" \
  -o Dir::Etc::sourceparts=/dev/null \
  -o Dir::State::lists="$probe_dir/lists" \
  -o APT::Get::List-Cleanup=0
```

apt menyelesaikan jalur indeks, kompresi, dan arsitekturnya sendiri. Konfigurasi
apt milik perangkat tidak pernah dibaca atau ditulis, dan hasilnya di-cache per
sekali jalan.

Inilah sebabnya repositori mati dilaporkan sebagai **tidak terjangkau**, bukan
kosong. `tur` dan `unstable-packages` di buku alamat sama-sama mengembalikan HTTP
404 di perangkat ini, dan installer mengatakan persis itu, bukan berpura-pura
repositori itu layak ditambahkan.

### Menambahkan satu

`repo_enable` menulis entri `sources.list.d`, menjalankan `apt-get update` terhadap
konfigurasi sungguhan, lalu membatalkan cache katalog — karena daftar paket baru
saja berubah, dan setiap menu di hilir dibangun darinya.

## 4. Gerbang penyimpanan

Menjumlahkan ukuran paket yang Anda centang dan mengabaikan dependensinya
meremehkan stack PHP atau database dengan selisih yang jauh — dan justru itulah
kasus di mana kehabisan ruang itu penting.

Jadi installer meminta apt menyelesaikan pohon dependensi sungguhan dan mencetak
renaturannya, tanpa mengunduh apa pun:

```sh
apt-get install --print-uris --no-install-recommends <packages>
```

```
Need to get 28.9 MB of archives.
After this operation, 99.5 MB of additional disk space will be used.
```

Kedua angka itu dibaca dengan memindai nilai diikuti satuan byte, bukan dengan
posisi field, karena susunan kalimat apt bukan kontrak. Parse berdasarkan posisi
mengembalikan 0 saat susunannya berubah, dan 0 adalah satu-satunya angka yang
terlihat seperti jawaban yang sah.

Ada dua ambang, karena ada dua kegagalan yang berbeda:

- **Ruang kosong di bawah yang dibutuhkan rencana** — ditolak. Tidak mungkin
  berhasil.
- **Ruang kosong di bawah rencana ditambah setengahnya lagi** — diizinkan, dengan
  angkanya disebutkan. apt juga butuh ruang untuk daftar miliknya sendiri, untuk
  sumber yang sudah diekstrak dan dibangunnya, dan untuk log — dan orang yang
  memegang ponsel ini tahu ponselnya lebih baik daripada installer.

## 5. Jawaban

Jawaban disimpan ke `$HOME/.devkit-answers` sebagai baris `key=value`, sehingga
pemakaian ulang bisa diperiksa, disunting, atau diputar ulang.
`--answers=FILE` memindahkan berkasnya; `--yes` tidak pernah bertanya.

Berkas jawaban juga diperlakukan sebagai input yang tidak dipercaya:
`build_plan` memeriksa ulang setiap paket dalam rencana terhadap perangkat,
karena berkasnya bisa disunting tangan dan karena berkas yang ditulis di satu
ponsel bisa diputar ulang di ponsel lain. Paket yang tidak bisa dipakai di sini
disebutkan, beserta alasannya, bukan dibiarkan gagal di dalam apt.

## 6. Perilaku interaktif

Sebuah menu menggambar ulang sampai mendapat jawaban. Itu baik di terminal dan
fatal di pipa: `printf '1\n' | ./install.sh` kehabisan input, dan menu yang
menganggap "tidak ada lagi yang dibaca" sebagai "pengguna menekan enter" akan
menggambar ulang selamanya dengan kecepatan penuh — baterai panas dan tanpa
keluaran yang bisa dihentikan siapa pun.

Jadi `_read` membedakan keduanya, dan setiap pemanggil yang berputar memeriksa:

- **Akhir input** menghentikan installer dengan sebuah pesan. Ia tidak menebak.
- **Tidak ada terminal sama sekali** turun ke bawaan yang terdokumentasi, dan itu
  yang dipakai `--yes` dan CI.

Terkait: pada pilihan ganda, `0` membatalkan hanya sebagai token tersendiri.
Mencocokkan digit di mana pun dalam baris akan mengubah `10` — opsi kesepuluh —
menjadi pembatalan, dan menu alat pengembangan punya tujuh belas entri.
