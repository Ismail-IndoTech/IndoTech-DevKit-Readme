# Mengatasi masalah

<a href="troubleshooting.md">**Bahasa Indonesia**</a> · <a href="troubleshooting.en.md">English</a>

## Mulai dari sini

```sh
devkit doctor
```

Ini perintah pertama yang harus dijalankan, dan pertanyaannya berbeda dari
`service:status`. `service:status` menjawab *apakah dia jalan*. `doctor` menjawab
*apakah dia bekerja* — setiap port ditanya apakah ada yang menjawab, setiap site
apakah URL-nya mengembalikan halaman, setiap sertifikat apakah masih bisa
diverifikasi, dan setiap kegagalan membawa baris perbaikannya.

```
  [1mPanel[0m
    ok    Token                      ada, dan dihapus saat panel berhenti
    ok    Reach it                   http://127.0.0.1:8898/__panel/
  [1mSites[0m
    ok    htdocs                     http://127.0.0.1:8080/ menjawab 200
```

Untuk perangkatnya sendiri, sebelum apa pun terpasang:

```sh
./install.sh --device-info
```

Ini mencetak kartu perangkat, catatan yang berlaku untuknya, dan tabel
kompatibilitas, dan tidak mengubah apa pun. Saat melaporkan masalah, keluarannya
adalah hal pertama yang layak disertakan.

```sh
cat "$HOME/.devkit-answers"     # apa yang diputuskan pemrunsian terakhir, dan kenapa
devkit env:info                 # apa yang sebenarnya dipakai stack terpasang
```

## Sebuah langkah menolak sesuatu

Kalau sebuah opsi dinonaktifkan, alasannya dicetak di sebelahnya. Alasan-alasan
itu berasal dari pengukuran, bukan dari tabel mitos, jadi layak dibaca secara
harfiah:

| Pesan | Apa artinya |
| --- | --- |
| `butuh sekitar 1200 MB RAM; perangkat ini punya 900 MB` | Angka sungguhan dari `/proc/meminfo`. Server database yang dibunuh Android di tengah tugas lebih buruk daripada tidak ada. |
| `butuh sekitar 300 MB ruang kosong untuk berjalan; perangkat ini punya 120 MB` | Ambang bawah untuk menjalankannya, terpisah dari ukuran unduhan. |
| `build 32-bit dari toolchain ini bukan bagian dari Termux` | Paketnya memang tidak ada untuk `arm`. Tidak ada versi installer ini yang bisa menawarkannya. |
| `Android 13 (API 21) lebih lama dari minimum Android 7 (API 24)` | Di bawah batas. Termux sendiri tidak mendukung ini. |

Untuk tetap memilih opsi yang dinonaktifkan itu, jawab pertanyaan sebelum
dengannya dengan `0` untuk membatalkan, lalu jalankan lagi dengan jawaban lain.
Tidak ada flag pemaksa, dengan sengaja: pilihan yang sengaja dibuat harus terlihat
sengaja.

## "Rencana ini tidak muat di perangkat ini"

Pohon dependensi sungguhan, yang diselesaikan apt, butuh ruang lebih besar dari
yang tersedia. Ambil sesuatu dari rencananya atau bebaskan ruang lalu jalankan
lagi. Tidak ada yang terpasang ketika pesan ini muncul.

Untuk melihat angkanya sendiri:

```sh
apt-get install --print-uris --no-install-recommends apache2 php mariadb
```

Perintah itu mencetak rencana yang sama tanpa mengunduh apa pun.

## "Menolak memasang di atas \<direktori\>"

Direktori runtime sudah berisi berkas dan bukan runtime DevKit, jadi kemungkinan
itu pekerjaan Anda sendiri. Tidak ada yang dipindahkan, diubah, atau dihapus.
Gunakan lokasi lain:

```sh
DEVKIT_HOME="$HOME/stacks/dev" ./install.sh
```

Runtime DevKit sebelumnya dikenali dari `bin/devkit`-nya dan *akan* diganti,
dengan salinan lama dipindahkan ke `<direktori>.backup.<stempel waktu>`. Hanya
direktori yang ditaruh installer ini yang pernah dipindahkan.

## Sebuah port tertahan tapi tidak ada yang menjawab

Socket yang mendengarkan milik kernel sampai pemilik terakhirnya melepaskannya,
sehingga proses yang mewarisi satu tetap menahan port terikat jauh setelah proses
pemiliknya hilang. Dari luar, portnya terlihat terisi; dari dalam, menyalakan
apa pun di port itu gagal.

**Dulu ini keadaan permanen di perangkat ini, dan itu sebuah bug.** Queue worker
dijalankan lewat implementasi kedua dan lama dari "jalankan ini di latar
belakang" yang melakukan `shell` ke `setsid ... &` dan tidak pernah menutup
descriptor yang diwarisi. Saat dimulai dari panel, ia mewarisi socket listener
milik panel sendiri — jadi setelah panel berhenti, port 8898 tetap terikat oleh
sebuah worker yang tidak melayani apa pun, dan `devkit service:start panel` tidak
lalu berhasil lagi. Satu-satunya hal di perangkat ini yang bisa membebaskannya
adalah `devkit service:stop queue`, dan hanya karena itu kebetulan menghentikan
proses yang memegangnya.

Kedua kesalahan itu sudah diperbaiki. `Detach` sekarang menutup setiap descriptor
warisan di atas tiga yang standar sebelum menjalankan `exec`, dan queue worker
di-`exec` secara langsung tanpa perantara shell, sehingga pid yang tercatat adalah
pid worker itu sendiri. Terverifikasi dengan memulai queue dari panel,
menghentikan panel, dan memastikan port-nya bebas sesudahnya sementara worker
masih berjalan.

Jadi pesan ini sekarang berarti aplikasinya memang dibunuh tanpa membersihkan
diri, yang dilakukan Android sendiri saat mengambil kembali memori:

```sh
devkit doctor          # menyebut portnya, dan mengatakan listener tidak menjawab
```

Kalau benar-benar tertahan, pelepasannya datang dari aplikasinya, bukan dari
sebuah perintah:

1. Settings Android → Apps → Termux → **Force stop**.
2. Buka ulang Termux.
3. `devkit service:start panel`

Untuk mencari proses yang memegangnya sebelum resort ke itu, hentikan service satu
per satu sambil Ports diamati:

```sh
devkit service:stop queue && php -r 'echo @stream_socket_server("tcp://127.0.0.1:8898") ? "free\n" : "held\n";'
```

Kalau itu membebaskannya, service yang baru saja Anda hentikan adalah pemegangnya.

## Web server tidak mau start

Termux mengirim Apache dengan konfigurasi yang mengira ia berjalan sebagai root
dan memiliki `/data`, dan beberapa paket — phpMyAdmin khususnya — menulis
directive dari Apache 2.2 yang ditolak 2.4 tanpa modul kompatibilitas. Installer
memperbaiki keduanya, dan memvalidasi konfigurasi di tempatnya: ia menulis
perubahan, menjalankan `httpd -t`, dan mengembalikan yang asli kalau hasilnya
bukan `Syntax OK`.

Untuk memeriksanya sendiri:

```sh
httpd -t                      # harus mencetak: Syntax OK
tail -n 40 "$PREFIX/var/log/apache2/error_log" 2>/dev/null
```

Kalau `httpd -t` lolos dan server tetap tidak menjawab, itu konflik port. Termux
tidak bisa mengikat di bawah 1024, jadi nginx bawaan yang mendengarkan di 8080
akan bentrok dengan Apache di 8080. Hanya satu web server yang bisa memiliki
suatu port:

```sh
devkit service:status        # apa yang jalan, dan URL yang dijawab setiap site
```

## Site menjawab di 127.0.0.1 tapi nama `.app`-nya ditolak

Yang ini adalah bug sungguhan, dan gejalanya terukur, bukan ditebak.

`site:add` menulis vhost, menerbitkan sertifikat, me-restart web server, dan
mencetak keberhasilan — dan site-nya tidak bisa dijangkau dengan namanya,
karena konfigurasi Squid hanya ditulis ulang saat proxy mulai. Nama yang baru saja
didaftarkan perangkat ini ikut cocok dengan aturan yang menolak nama yang belum
pernah didengar, dan setiap permintaan kembali `ERR_ACCESS_DENIED`.

```sh
devkit site:apply            # menulis ulang vhost *dan* konfigurasi proxy
devkit proxy:doctor          # setiap alamat, dengan kode status yang dikembalikannya
```

`site:add`, `site:remove`, `site:enable`, dan `site:disable` semuanya sudah
melakukan ini, dan semuanya mencetak apa yang dilakukan proxy.

**HTTPS pada nama `.app` tidak punya rute, dan memang dari sananya tidak punya.**
Sebuah `CONNECT` tidak bisa dipindahkan ke port lain, jadi `https://htdocs.app/`
di 443 tidak pernah sampai ke perangkat. `proxy:doctor` mengatakannya, bukan
mencetak tautan yang tidak bisa dibuka. Untuk HTTPS pakai `localtest.me` atau
`sslip.io` dengan port di dalam URL — keduanya tidak butuh proxy.

## Nama host `.app` sama sekali tidak resolve

`localtest.me` tidak butuh apa pun; `.app` butuh proxy Wi-Fi perangkat diarahkan
ke `127.0.0.1:8899` dan proxy-nya jalan. Lihat [panel.md](panel.md).

## HTTPS menampilkan peringatan

Wajar. Sertifikatnya datang dari otoritas lokal yang belum dipercaya Android
sampai Anda memasangnya sendiri, lewat
Settings → Security → Encryption & credentials → Install a certificate → CA
certificate. Termux tidak bisa melakukannya, dan sesuatu yang bisa melakukannya
akan menjadi hal berbahaya untuk dijalankan. `devkit ca:info` mencetak berkas
yang perlu dipasang.

`devkit doctor` melaporkan ini sebagai dua baris terpisah, dengan sengaja.
"Termux: dipercaya" dan "Browser: tidak diketahui" adalah dua keadaan berbeda
dengan satu gejala, dan melaporkannya sebagai satu inilah penyebabnya
seseorang menghabiskan satu jam kenapa curl jalan tapi browser tidak.

## Sebuah repositori dilaporkan tidak terjangkau

Itu berbeda dari kosong, dan bedanya penting. Repositori yang tidak terjangkau
tidak bisa dibedakan dari yang kosong hanya dengan melihatnya, jadi installer
menyatakan mana yang ia temukan:

```sh
DEVKIT_NO_REPO_SCAN=1 ./install.sh     # lewati penelusuran, lebih cepat
```

Periksa satu per satu:

```sh
curl -sSI "https://packages-cf.termux.dev/apt/termux-x11/dists/stable/Release" | head -1
```

## Sesuatu terpasang tapi versinya tampak salah

Kotak **Package sources** di rencana menjelaskan ini sebelum apa pun terjadi, dan
ada tiga kasus yang berbeda:

- `from main` — repositori menawarkan tepat versi ini.
- `from termuxvoid (other version)` — versi *berbeda* dari paket ini yang
  terpasang daripada yang ditawarkan repositori. `apt upgrade` tidak akan
  mengubahnya.
- `not offered by any enabled repository` — paketnya terpasang tapi tidak ada
  repositori aktif yang memilikinya. Datang dari tempat yang sudah tidak
  dikonfigurasi, atau dibangun dengan tangan.

Ini dilaporkan, bukan disembunyikan, karena kasus ketiga itulah yang biasanya
berubah menjadi sore yang membingungkan.

## Installer berhenti dengan "No more input to read"

Input habis sebelum semua pertanyaan terjawab, jadi ia berhenti daripada menebak.
Ini terjadi kalau input dipipe:

```sh
printf '1\n1\n' | ./install.sh        # berhenti di sini
./install.sh < /dev/null               # sama saja
```

Jalankan tanpa pipe dan jawab pertanyaannya, atau pakai `--yes` untuk bawaan
yang terdokumentasi. Pilihan lain — memperlakukan akhir input sebagai jawaban
kosong — akan membuat menu menggambar ulang selamanya.

## SSH bilang "Login as home" atau koneksinya ditolak

Nama penggunanya adalah nama POSIX — `u0_a247` di perangkat tanpa root — bukan
nama direktori home-nya. `basename($HOME)` di sini adalah string harfiah `home`,
dan `ssh home@…` ditolak. Baca yang sebenarnya dari:

```sh
devkit ssh:status
```

`ssh -p 8022 <user>@<alamat>` juga harus dijalankan **dari komputer Anda**: ponsel
tidak bisa menjangkau dirinya sendiri lewat Wi-Fi, jadi alamat di baris koneksi
adalah alamat ponsel di jaringan tempat komputer Anda berada. Di ponsel sendiri,
pakai `devkit job:run` atau terminal di halaman Projects.

## SSH mendengarkan tapi tak ada yang bisa masuk

Itu keadaan yang dirancang, bukan kesalahan: tanpa ada kunci terotorisasi, portnya
terbuka dan setiap login ditolak. Panel mengatakan *"It is listening, but no key
can log in, so nobody is in yet."*

```sh
devkit ssh:keys                  # apa yang terotorisasi
devkit ssh:authorize "$(cat ~/.ssh/laptop.pub)" "laptop kerja"
```

Kalau kunci sudah terotorisasi dan login tetap gagal, bandingkan sidik jarinya:

```sh
devkit ssh:status                # host key perangkat ini
ssh-keygen -lf ~/.ssh/laptop.pub # kunci komputer
```

## Terminal di halaman Projects tidak konek

Itu sesi SSH, jadi ia memerlukan `devkit ssh:start` lebih dulu.
`terminal:start` mengatakannya alih-alih gagal terhubung:

```sh
devkit ssh:start
devkit terminal:start
devkit terminal:status       # listener, kunci, dan URL WebSocket-nya
```

`devkit terminal:forget-key` mencabut kunci milik panel sendiri. Setelah itu
terminal menolak, dan itu benar: itu pintu yang sama dengan yang dipakai laptop,
dan mencabutnya memang harus berhasil.

Kalau konek lalu langsung tertutup, lihat lognya:

```sh
tail -n 40 ~/IndoTech_Project/logs/terminal.log
```

## Sebuah job bilang selesai tapi tidak ada yang tercipta

Lihat keluarannya, bukan baris statusnya:

```sh
devkit job:list
devkit job:log <id> 400
```

Statusnya adalah kode keluar; alasannya ada di log. Sebuah `composer
create-project` yang gagal karena gangguan jaringan keluar dengan status bukan nol
beserta kesalahannya beberapa baris di atas.

## Panel bilang runtime-nya lebih tua darinya

Panel lebih baru daripada `devkit` yang ia ajak bicara, atau payload-nya hanya
tersalin setengah. Jalankan installer lagi, atau salin `payload/` ke atas runtime:

```sh
cp -a "$PWD/payload/." ~/IndoTech_Project/
```

Lalu `devkit service:restart panel`.

## `code .` bilang tunnel tidak tersedia

Itu jawaban yang akurat, bukan kegagalan mencoba. Binarinya yang dipanggil
`code-oss tunnel` — yaitu `code-tunnel-oss` — tidak dikemas untuk Termux dan tidak
diterbitkan Microsoft sebagai unduhan mandiri; tunnel ikut dalam aplikasi
desktop. `code-oss tunnel` dan `code-oss serve-web` keduanya gagal dengan
`ENOENT` karena itu.

Jadi `code .` mencetak jalur Remote-SSH, yang menjangkau workspace yang sama dan
tidak butuh apa pun yang belum ada di perangkat ini:

```sh
devkit code:address      # alamatnya, dan apa yang kurang kalau ada
devkit code:steps        # panduan setup sekali jalan
```

## Remote-SSH konek lalu putus

Ponsel berpindah jaringan. Config yang ditulis di satu Wi-Fi salah alamatnya di
berikutnya, dan koneksinya ditolak dengan cara yang terlihat seperti firewall.

```sh
devkit code:address      # baca ulang setelah berpindah jaringan
```

Blok config yang dicetak `code .` menyebutkan untuk menjalankan ulang ini
justru karena itu.

Kalau alamatnya sudah benar dan tetap gagal, kuncinya biasanya jawabannya:

```sh
devkit ssh:keys                    # apakah kunci komputer ada di daftar?
devkit ssh:authorize "$(cat ~/.ssh/id_ed25519.pub)" "laptop"
```

Kuncinya harus dibuat **di komputer**. Kunci yang dibuat di ponsel lalu disalin
ke mana-mana membuat ponsel memegang private key yang tidak dibutuhkannya dan
komputer memegang public key yang belum diotorisasi — lalu login ditolak tanpa
petunjuk kenapa.

## Tab editor bilang tidak bisa jalan

Itu kenyataan, bukan bug. Rilis yang dipublikasikan hanya glibc, dan Android
tidak punya glibc loader; extension host menjalankan ulang binernya sendiri
menjadi proses yang tidak menemukannya. Jalur yang bekerja adalah `proot-distro`,
yang belum dipasang. `code .` memberi Anda jalur Remote-SSH sebagai gantinya,
dan itu memang bekerja. Lihat
[panel.md](panel.md#editor-di-browser-tidak-bisa-jalan-di-sini).
