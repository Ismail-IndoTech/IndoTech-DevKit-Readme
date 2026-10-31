# Panel, proxy, dan nama host `.app`

<a href="panel.md">**Bahasa Indonesia**</a> · <a href="panel.en.md">English</a>

## Apa itu panel

Aplikasi web lokal, disajikan oleh server bawaan PHP di
`http://127.0.0.1:8898/__panel/`, yang menjalankan perintah `devkit` yang sama
persis dengan yang akan Anda ketik di terminal. Panel punya port sendiri dengan
alasan yang disengaja: ia tidak pernah berada di belakang proxy, jadi proxy yang
mati tidak bisa menyeret ruang kendali bersamanya. Sepuluh tab:

| Tab | Fungsinya |
| --- | --- |
| **Services** | Apa yang jalan, web server mana yang mengendalikan, serta mulai/henti/restart |
| **Sites** | Site terdaftar, URL-nya, dan URL proxy yang bisa dibuka |
| **SSL** | CA lokal dan sertifikat yang diterbitkan dari CA itu |
| **SSH** | Masuk dari mesin lain, dan kelola kunci yang boleh masuk |
| **Projects** | Buat project dari template, build, jalankan artisan — **dengan terminal sungguhan di halaman yang sama** |
| **Editor** | Buka folder di VS Code — lihat [tunnelnya](#tunnel-ke-vscode-dev-tidak-bisa-dibuat-di-sini) |
| **GitHub** | Masuk, daftar repository, push project |
| **Device** | Sebenarnya apa ponsel ini, dan apa yang sedang dilacaknya sekarang |
| **Environment** | Versi dan path yang berhasil diresolusi stack ini |
| **Logs** | Keluaran terbaru dari service mana pun |

Ia berjalan di perangkat. Hanya terikat ke `127.0.0.1` dan tidak yang lain, jadi
tidak bisa dijangkau dari jaringan kecuali Anda mengubahnya sendiri.

## Memeriksa apakah ini benar-benar bekerja

```sh
devkit doctor
```

Ini perintah pertama yang harus dijalankan, dan pertanyaannya berbeda dari
`devkit service:status`. `service:status` menjawab *apakah dia jalan*.
`doctor` menjawab *apakah dia bekerja*, dan jarak antara kedua jawaban itulah
tempat kesalahan sesungguhnya berada.

Bedanya adalah setiap pemeriksaan menanyakan bendanya, bukan benderanya:

- setiap port ditanya apakah ada yang **menjawab** di sana, yang membedakan
  listener yang bekerja dari listener yang prosesnya sudah hilang;
- setiap site ditanya apakah **URL-nya mengembalikan halaman**, yang membedakan
  site terkonfigurasi dari site tersaji;
- konfigurasi proxy **di-parse oleh Squid itu sendiri**, dan web server tiap
  site ditanya apakah ia restart dengan bersih;
- setiap sertifikat ditanya **berapa lama lagi**, dan dua penyimpanan
  kepercayaan dilaporkan terpisah, karena "HTTPS tidak berfungsi" dan "CA Anda
  belum dipasang" terlihat sama dari browser.

Setiap baris membawa perbaikan kalau memang ada yang bisa saran:

```
  [1mServices[0m
    ok    Apache                     running
    warn  DevKit panel               stopped
                                    -> devkit service:start panel
  [1mPanel[0m
    FAIL  Port 8898                  tertahan listener yang tidak menjawab.
                                    -> Force-stop lalu buka ulang Termux, kemudian: devkit service:start panel
```

**Peringatan** adalah pilihan atau batas, bukan kesalahan — perangkat tanpa site
adalah perangkat yang bekerja, dan pemeriksaan yang gagal karena itu akan
diabaikan lalu jadi tidak berguna. Perintah ini keluar dengan status bukan nol
hanya kalau ada kegagalan, jadi bisa dipakai di CI.

## Service, dan web server mana yang mengendalikan

`state/server.conf` mencatat web server yang dijalankan DevKit. Installer yang
menulisnya; runtime memperlakukannya sebagai baca-saja.

Kalau berkas itu tidak ada, server **ditebak dari vhost di disk** — blok
`<VirtualHost>` hanya bisa dibaca Apache dan blok `server {` hanya bisa dibaca
nginx, dan itu bukti yang lebih kuat daripada binary mana yang kebetulan
terpasang. Adanya `nginx` yang terpasang tidak memberi tahu apa pun tentang
server mana yang ditulis untuk site-nya. Kapan pun panel menyebutkan mana
yang ia pakai dan kenapa, bukan menyajikan tebakan sebagai fakta:

```sh
devkit server:show            # apa yang dikonfigurasi, dan apakah itu dibaca atau ditebak
devkit server:set apache      # catat, supaya tidak lagi menjadi tebakan
devkit server:set nginx
```

`server:set` menolak berpindah kalau vhost di disk memakai format *yang lain*,
karena mengarahkan DevKit ke nginx sementara semua site adalah blok Apache akan
menjatuhkan semua site begitu Apache berhenti. `--force` menimpanya, dan
kemudian Anda menjalankan `devkit site:apply` untuk menulis ulang vhost-nya.

**Satu perangkat bisa punya `apache2` dan `nginx` sekaligus**, dan `pkg` tidak
punya alasan untuk membuang salah satunya. Jadi panel juga menampilkan server yang
lain kalau ada, diberi label sudah-terpasang-tapi-bukan-yang-dikonfigurasi,
beserta penjelasannya. Ia tidak mendapat tombol **Start**: menyalakannya hanya
akan menempatkan server kedua di port yang dibutuhkan site.

`service:start all` dan `service:stop all` berarti **semua** service, dan itu
daftar yang berbeda dari `Services::active()` dengan sengaja. `active()` adalah
stack *bawaan* — hal-hal yang diharapkan hidup pada instalasi DevKit, dan himpunan
yang boleh dipakai panel untuk menghitung stack yang rusak. SSH tidak ada di sana
karena ia satu-satunya service yang menjawab di jaringan.

`all` bukan bawaan, melainkan instruksi. Seseorang yang mengetiknya sudah
menyatakan "semuanya", dan menyisakan dua service dari "semuanya" adalah
persempitan cakupan yang sekali-kali mahal. Urutannya adalah urutan
ketergantungan: web server, queue, proxy, ssh, terminal, panel — karena terminal
adalah sesi SSH dan tidak bisa start sebelum sshd. Menghentikan memakai
kebalikannya, jadi sesuatu dimatikan sebelum milik yang dia butuhkan.

```sh
devkit service:start all
devkit service:stop all
```

Setelah `start all`, panel mengatakan bahwa SSH sekarang mendengarkan dan
mengapa itu penting, karena port yang terbuka di jaringan adalah hal yang
seharusnya tidak terjadi diam-diam.

Web server yang mengendalikan adalah kartu seperti yang lain, dengan Start, Stop,
dan Restart yang benar-benar berfungsi. Menghentikan proxy diperlakukan khusus —
itulah halaman yang sedang Anda lihat, jadi ia dikonfirmasi lebih dulu, dan
sesudahnya panel mengatakan dengan terus terang bahwa ia menutup dirinya sendiri
dan bagaimana cara mengembalikannya.

## Bagaimana panel diizinkan bicara dengan CLI

Browser tidak pernah menjalankan perintah. Ia meminta JSON dari
`http://127.0.0.1:8898/__api/<command>`, dan router panel mencocokkan
`<command>` terhadap daftar izinkan tetap berisi subcommand `devkit`. Perintah
yang tidak ada di daftar ditolak sebelum apa pun dijalankan. Permintaan yang
mengubah sesuatu juga harus membawa token per-sesi di `X-DevKit-Token`;
perintah baca-saja tidak.

Scaffold tidak pernah dirakit dari teks permintaan. Panel mengirim id template;
`Projects.php` mencari perintahnya di tabelnya sendiri (atau di
`state/templates.json` yang bisa Anda sunting) dan menjalankannya sebagai array
argumen. Tidak ada string shell di mana pun pada jalur itu, jadi nama project
tidak bisa menjadi perintah.

Operasi panjang — scaffold, `composer create-project`, build Flutter — diberi
batas waktu yang lebih longgar, bukan dipotong.

`tools/check-allowlist.php` membandingkan kedua daftar itu dan gagal kalau panel
meminta perintah yang tidak diimplementasikan CLI. Inilah pemeriksaan yang menangkap
tombol yang ditambahkan ke HTML tapi tidak pernah ke tabel.

## Mengapa ada proxy sama sekali

Termux tidak bisa mengikat port di bawah 1024. Satu fakta itu membentuk seluruh
tata letaknya:

| Service | Port | Kenapa bukan 80/443 |
| --- | --- | --- |
| Apache, biasa | 8080 | 80 butuh root |
| Apache, TLS | 8443 | 443 butuh root |
| Database | 3306 | sudah di atas 1024, jadi tidak masalah |
| Panel DevKit | 8898 | terpisah dari proxy, jadi proxy rusak tidak bisa menyembunyikannya |
| Terminal | 8896 | `php -S` tidak bisa menaikkan koneksi ke WebSocket |
| Proxy DevKit | 8899 | Squid, hanya untuk resolusi nama `.app` |
| Site project | 9000+ | satu per project terdaftar |

## Dua cara untuk memberi nama site

### `localtest.me` — tidak butuh apa pun

`localtest.me` dan `127.0.0.1.sslip.io` adalah catatan DNS publik yang resolve ke
`127.0.0.1`. Site terdaftar bernama `htdocs` menjawab di keduanya:

```
http://htdocs.localtest.me:8080/
https://htdocs.localtest.me:8443/
https://htdocs.127.0.0.1.sslip.io:8443/
```

Tidak ada yang perlu dikonfigurasi, tetap jalan offline setelah ter-resolve.
**Inilah jalur yang membawa HTTPS**, dan inilah yang layak disimpan di bookmark.

### `.app` — butuh proxy, dan hanya lewat HTTP

Site juga bisa dinamai `htdocs.app`, dan itulah yang sertifikatnya diterbitkan dan
yang paling enak dibaca. Agar itu resolve di perangkat, permintaan jaringan
perangkat itu sendiri harus dikirim ke proxy:

1. Buka jaringan Wi-Fi yang sedang dipakai.
2. Atur proxy ke **Manual**.
3. Host `127.0.0.1`, port `8899`.
4. Jalankan proxy: `devkit service:start proxy`.

Lalu `http://htdocs.app:8080/` sampai ke site. Proxy membaca hostname,
mencarinya di `state/sites.json`, dan meneruskannya ke port milik site itu.

**HTTPS pada nama `.app` tidak punya rute, dan itu struktural.** Sebuah `CONNECT`
tidak bisa dipindahkan ke port berbeda, jadi `https://htdocs.app/` di 443 tidak
pernah sampai ke perangkat ini. `devkit proxy:doctor` mengatakannya setiap kali
dijalankan, alih-alih mencetak tautan yang tidak bisa dibuka.

**A site terdaftar di dua tempat, dan keduanya harus ditulis.** Site adalah satu
entri di `state/sites.json` dan dua ekspresinya: sebuah vhost untuk web server,
dan sebuah aturan di proxy yang membiarkan namanya sampai ke sana. Menulis hanya
yang pertama itulah yang dulu terjadi, dan gejalanya terukur: `site:add`
mencetak keberhasilan, site menjawab di `127.0.0.1:9000`, dan setiap permintaan
lewat proxy kembali `ERR_ACCESS_DENIED` — karena nama yang baru saja didaftarkan
perangkat ini ikut cocok dengan aturan yang menolak nama yang belum pernah
pernah didaftarkan. Konfigurasi proxy hanya ditulis ulang saat proxy mulai, jadi apa pun
yang ditambahkan sesudahnya tidak terlihat olehnya. `applyAndReload` sekarang
juga mengonfigurasi ulang proxy, dan mencetak apa yang ia lakukan.

```sh
devkit proxy:doctor     # setiap alamat, dengan kode status yang dikembalikannya
devkit site:apply       # tulis ulang vhost *dan* konfigurasi proxy
```

**Ini berlaku per jaringan, dan hanya memengaruhi perangkat ini.** Diatur di
pengaturan Wi-Fi Android, bukan oleh installer ini, karena installer tidak punya
bisnis mengubah cara ponsel Anda mengarahkan seluruh lalu lintasnya. Ada biaya
sungguhan: selama proxy manual aktif, setiap permintaan yang dibuat perangkat
melewati Squid di ponsel. Matikan saat tidak dipakai.

## Project

`devkit project:root` menampilkan atau mengubah folder tempat project dibuat.
Bawaannya `~/projects` dan dibuat saat pertama kali dipakai.

```sh
devkit project:templates                    # apa yang bisa dibuat, apa yang kurang
devkit project:new blog --template=static   # buat dan daftarkan sitenya
devkit project:build app --template=flutter # flutter build web
devkit project:artisan blog migrate         # jalankan artisan di dalam project
devkit project:remove blog --delete         # hapus pendaftaran dan berkasnya
```

Project baru mendapat vhost yang menunjuk subdirektori yang tepat — `public` untuk
Laravel, `build/web` untuk Flutter, foldernya sendiri untuk site statis —
sertifikat untuk `<name>.app`, dan port sendiri. Namanya harus huruf kecil:
`MyApp` dan `myapp` akan menjadi satu hostname tapi dua direktori, karena
sistem berkas Android mempertahankan huruf besar, sehingga keduanya tidak bisa
dibedakan belakangan.

### Template

`laravel` dan `flutter` butuh alat yang tidak dikirim Termux, jadi keduanya
melaporkan apa yang perlu dipasang dan menolak, bukan membuat sesuatu setengah
jadi:

```sh
composer global require laravel/installer   # untuk laravel
pkg install flutter                          # untuk flutter
```

`static` selalu jalan, karena DevKit menulis berkasnya sendiri.

### Menambahkan template sendiri

`state/templates.json` dibaca saat runtime dan digabung dengan template bawaan.
Setiap entri dikunci dengan id-nya:

```json
{
  "my-template": {
    "label": "My template",
    "description": "What it makes",
    "command": ["npm", "create", "my-template@latest", "{name}"],
    "docroot": ".",
    "tool": "create-my-template",
    "tool_hint": "npm install -g create-my-template",
    "needs_build": false
  }
}
```

| Kunci | Arti |
| --- | --- |
| `command` | Wajib. Dijalankan sebagai array argv. `{name}` menjadi nama project. |
| `docroot` | Subdirektori yang ditunjuk vhost, relatif terhadap project. Bawaan `.` |
| `tool` | Binary yang harus ada agar template ini bisa dipakai |
| `tool_hint` | Baris yang dicetak ketika alat itu tidak ada |
| `needs_build` | Benar jika site tidak punya apa pun untuk disaji sampai build berjalan |
| `options` | Opsi scaffold tambahan yang ditawarkan |

`command` boleh juga berupa string, dan itu dipecah pada whitespace — tidak
pernah dikirim ke shell, jadi quoting dan `&&` tidak akan melakukan seperti di
terminal. Pakai array.

Entri yang tidak bisa dipakai dibuang dari daftar dengan alasannya disimpan di
`devkit project:templates`, bukan gagal tepat saat Anda memilihnya.

## Terminal, di halaman Projects

Perintah panjang diawasi, tidak ditunggu.

Membuat project Laravel adalah composer yang menyelesaikan pohon dependensi, git
yang meng-clone repository, build tool yang mengompilasi. Tidak ada yang bisa
dijadikan lebih cepat; semuanya hanya bisa diawasi.

### Ia adalah sesi SSH sungguhan, dan itulah seluruh desainnya

Tidak ada tab Terminal tersendiri, dan tidak ada shell kedua. Terminalnya
**berada di halaman Projects**, dan yang dibuka adalah:

```
xterm.js  ->  WebSocket  ->  ssh -p 8022 <user>@127.0.0.1  ->  login shell sungguhan
```

Shell yang dimulai dari permintaan web bukan hal yang sama, dan berpura-pura
itu menyebabkan masalah sungguhan. Ia punya lingkungan permintaan itu sendiri,
sehingga ia melewati autentikasi, daftar kunci, dan log SSH — lalu panel dan
laptop Anda jadi berbeda pendapat tentang apa yang sedang berjalan dan siapa
yang boleh masuk. Kunci milik panel diotorisasi lewat `Ssh::authorize` seperti
kunci biasa, dan komentarnya menyebutkan dari mana asalnya, sehingga enam bulan
kemudian pertanyaan "bisakah saya mencabut akses panel?" punya jawaban yang sama
dengan "bisakah saya mencabut akses laptop saya?".

Lewati `ssh` berarti panel datang lewat pintu yang sama dengan laptop.
**Menguji terminal berarti menguji SSH.**

```sh
devkit terminal:status       # listener, kunci, dan URL WebSocket-nya
devkit terminal:start        # listening di 127.0.0.1:8896
devkit terminal:stop
devkit terminal:restart
devkit terminal:forget-key   # cabut kunci panel; terminal lalu menolak
```

Ia memerlukan `devkit ssh:start` lebih dulu, dan mengatakannya alih-alih gagal
terhubung.

### Kenapa listener sendiri

Panel disajikan oleh `php -S`, dan `php -S` tidak bisa menaikkan koneksi ke
WebSocket. Jadi terminal adalah listener kecil milik sendiri di loopback `8896` —
bentuk yang sama dengan Squid yang terpisah dari panel, untuk alasan yang sama:
satu listener rusak tidak boleh bisa menjatuhkan yang lain.

Token dikirim lewat query string WebSocket, karena browser tidak bisa memasang
header khusus pada jabat tangan WebSocket. Itu aman di sini untuk satu alasan dan
bukan sebagai aturan umum: listener-nya terikat ke loopback, jadi tidak ada yang
di luar perangkat yang bisa menjangkauinya untuk mencuri token dari URL.

### Pipa, bukan PTY

`laravel new` bertanya dan `flutter create` mencetak progres. Keduanya berperilaku
seperti dari shell, karena ini memang shell. Koneksi ke `ssh` adalah pipa, bukan
pseudo-terminal, yang mengorbankan satu fitur nyata — tidak ada resize jendela,
tidak ada program layar penuh — dan menukar itu dengan tidak adanya pty, tidak
adanya protokol winsize, dan tidak adanya kelas hang yang menyertainya.

### Job masih ada, dan masih berguna

Beberapa perintah layak dijalankan tanpa terminal: pendek, tidak butuh input, dan
keluarannya layak disimpan untuk dibaca belakangan.

```sh
devkit job:run project:new myapp --template=laravel
devkit job:list
devkit job:status <id>
devkit job:log <id> [lines]
devkit job:stop <id>
```

Job *adalah* perintah itu sendiri, dijalankan terpisah dengan keluarannya di
sebuah berkas, dan kode kelu thereof dicatat oleh pembungkus satu baris. Tidak ada
yang memformat ulang keluaran kecuali melepas kode warna yang tidak bisa
dirender terminal. Job disimpan selama satu jam lalu dipangkas.

## GitHub

Masuk memakai OAuth **authorization-code grant dengan PKCE** yang biasa. Browser
di ponsel ini sudah masuk ke GitHub, jadi menyetujuinya cukup satu ketukan, dan
callback-nya mendarat di port panel loopback yang sudah mendengarkan — tidak ada
layar kedua dan tidak ada kode yang perlu disalin.

1. Panel memulai alur dan membuka halaman otorisasi GitHub.
2. Anda menyetujui di browser; GitHub mengarahkan ke
   `http://127.0.0.1:8898/__github/callback` dengan sebuah kode.
3. Panel menukar kode itu dengan token memakai verifier PKCE yang ia buat, dan
   menyimpannya di `state/github.json`, mode `0600`.

Token tidak pernah dikirim ke browser dan tidak pernah diterima dari browser.
Git memakainya lewat PHP credential helper, sehingga tidak berakhir di
`~/.git-credentials`, di URL remote, atau di riwayat shell.

Scope yang diminta adalah `repo` dan `read:org`. Bukan `workflow` — mendorong
berkas workflow dari ponsel harus jadi tindakan yang disengaja, bukan efek samping
dari masuk.

**Keluar tidak mencabut token di GitHub.** Mencabatnya perlu panggilan ke API
GitHub dengan client secret; keluar di sisi lokal menghapus salinan yang
tersimpan dan memberi tahu Anda untuk mencabutnya di
`https://github.com/settings/tokens` kalau Anda ingin benar-benar mati. Secret-nya
sendiri juga dihapus saat keluar — token-nya tetap sah di sisi GitHub sampai
dicabut di sana.

Karena PKCE, **client id** sudah cukup untuk memulai alur, tapi GitHub tetap
meminta **client secret** untuk menukar kode, jadi keduanya dibutuhkan. Secret itu
milik Anda untuk disimpan: ia ditulis ke `state/github.json` dengan mode `0600`,
dan apa yang bisa dilacaknya dinyatakan terus terang saat keluar — ia hanya bisa
mengotorisasi satu OAuth app ini, dan tidak yang lain.

Daftarkan aplikasinya di <https://github.com/settings/developers> sebagai
**OAuth App** (bukan GitHub App — alur web di sini adalah OAuth, bukan
instalasi):

- Homepage URL: `http://127.0.0.1:8898/__panel/`
- Authorization callback URL: `http://127.0.0.1:8898/__github/callback`
- Scopes: `repo`, `read:org`

Lalu berikan client id dan secret ke panel — di tab GitHub, atau:

```sh
devkit github:client-id <id> <secret>
```

Selama belum diisi, panel menampilkan kartu penyiapan alih-alih melempar error
setiap kali tab dibuka — tidak ada yang rusak, satu langkah penyiapan masih
tersisa.

Kalau sudah diisi tapi GitHub tetap menolak, panel menyebutkan penyebabnya yang
mana, karena jawaban GitHub sendiri (`Not Found`) tidak menyebutkan:

- **"client id tidak dikenali"** — id-nya salah, atau app yang memilikinya sudah
  dihapus.
- **"callback ditolak"** — URL callback yang terdaftar tidak cocok dengan
  `http://127.0.0.1:8898/__github/callback`.

## Tab SSH

Semua hal lain di halaman ini tentang service yang terikat ke `127.0.0.1`. Yang
ini tidak, dan itulah seluruh alasan ia ada: sebuah login, sebuah TTY, dan sebuah
shell, supaya `devkit` berperilaku sama dari keyboard laptop seperti dari
Termux.

```sh
devkit ssh:install                     # config, host key, authorized_keys
devkit ssh:authorize "<public key>" "laptop kerja"
devkit ssh:start
devkit ssh:status
devkit ssh:keys
devkit ssh:revoke SHA256:...
devkit ssh:stop
```

### Apa yang terikat, dan apa yang tidak

`0.0.0.0:8022`, karena port loopback tidak bisa dijangkau dari mesin lain dan
sebenarnya tidak berguna sama sekali. Setiap perangkat di jaringan bisa menjangkau
port itu, jadi konfigurasi ini adalah satu-satunya versi yang bisa dibenarkan
dari seberang ruangan:

| Setelan | Nilai | Kenapa |
| --- | --- | --- |
| `AuthenticationMethods` | `publickey` | Membuat kunci jadi lantai, bukan preferensi: tidak ada suntingan berikutnya yang bisa menurunkannya ke kata sandi. |
| `PasswordAuthentication` | `no` | Tidak ada kata sandi yang bisa ditebak, dipecahkan, atau dipakai ulang dari kebocoran di tempat lain. |
| `AllowUsers` | akun ini | Kunci yang bocor tidak bisa dipasangkan dengan pengguna lain di perangkat. Termux punya satu. |
| `AllowTcpForwarding` | `local` | Menunnel panel ke laptop (`ssh -L 8898:127.0.0.1:8898`) berguna; memakai sesi ini sebagai jump host ke mesin lain tidak. |
| `StrictModes` | `no` | Android menaruh `$HOME` di `/data/data/com.termux/files/home`, dan sshd membaca path itu sebagai masalah izin pada direktori yang tak bisa dimasuki siapa pun. |
| `SetEnv PATH` | diisi | sshd tidak memberi profil ke sesi, dan perangkat ini tidak punya `~/.bashrc`, jadi tanpanya `devkit` begitu saja tidak ditemukan. |
| `MaxAuthTries` | `3` | Port yang selebegas ini sebaiknya tidak sekaligus sabar. |

### Kunci yang ditambahkan DevKit, dan yang tidak

Kunci terotorisasi tinggal di `~/.ssh/authorized_keys` **di antara penanda**,
karena berkas itu mungkin sudah berisi kunci yang ditaruh pengguna sendiri.
DevKit hanya menambahkan dan mencabut apa yang ada di dalam bloknya sendiri;
kunci yang ditambahkan tangan tidak didaftarkan dan tidak bisa dihapus dari sini,
dan justru itu intiDNya — ini bukan editor umum untuk setup ssh orang.

Setiap kunci yang ditambahkan DevKit diberi komentar berisi tanggal dan
dari mana asalnya, karena komentar adalah satu-satunya yang dibaca orang saat
memutuskan, enam bulan kemudian, apakah sesuatu perlu dicabut.

Terukur di ponsel tempat ini ditulis: kunci terdaftar berhasil masuk, kunci
terdaftar ditolak, kata sandi ditolak, dan kunci yang sudah dicabut ditolak —
dengan `Permission denied (publickey)` di setiap penolakan.

### Ia tidak dinyalakan untuk Anda

`service:start all` tidak mencakup SSH, dan `Services::active()` tidak akan pernah
mencakupnya. Membuka port ke jaringan seharusnya sesuatu yang dilakukan orang
sekali, dengan sengaja.

## Memilih versi

Tab **Versions** menjawab "versi PHP, Apache, Node, Composer yang mana yang
terpasang, dan apa yang bisa saya pakai sebagai gantinya". Setiap angka di sana
dibaca dari perangkat dengan `apt-cache madison` dan `dpkg`; tidak ada satu pun
tabel yang ditulis tangan.

```sh
devkit versions:list                    # semua komponen, satu baris masing-masing
devkit versions:show php                # satu komponen, lengkap
devkit versions:use <komponen> <versi>   # pasang, lalu restart yang bergantung padanya
devkit versions:update <komponen>       # versi terbaru yang tersedia di sini
```

### Biasanya hanya ada satu versi, dan panel mengatakannya

Ini bentuk terukur dari sebuah ponsel, dan layak dinyatakan terang-terangan
daripada dipoles-poles:

| Komponen | Versi yang ditawarkan apt |
| --- | --- |
| php, php-fpm, composer, nodejs, flutter | 1 |
| apache2, nginx | 1 |
| mariadb, postgresql | 1 |

`apt-cache madison` melaporkan tepat satu baris untuk masing-masing, di keempat
repo yang dikonfigurasi. Tidak ada paket `php8.3`, tidak ada nama paket
ber-versi, dan tidak ada arsip build lama — repositori Termux hanya membawa
versi terkini.

**Flutter adalah kasus terberat:** Google tidak pernah menerbitkan SDK Flutter
untuk Linux arm64, jadi satu-satunya build yang ada untuk perangkat seperti ini
berasal dari repositori pihak ketiga. Tidak ada pilihan untuk ditawarkan, dan
panel mengatakannya, bukan menampilkan dropdown berisi satu entri.

Jadi halaman itu menampilkan `1 versi saja` dengan lencana redup, disertai
alasannya. Pemilih versi yang menawarkan menu berisi satu entri tanpa penjelasan
lebih buruk daripada tidak berguna — ia terlihat seperti fitur yang gagal dimuat.

### Update berarti apa yang dikatakannya

Pada komponen bersatu versi, `versions:update` memasang ulang versi yang sudah
ada, dan pesannya menyatakan persis itu:

```
PHP is already on the newest version this device can install (8.5.1), and this
device carries only that one. There is nothing to move to - a newer one would
have to come from a build for Linux arm64, which most of these do not have.
```

Update yang melapor sukses padahal tidak ada yang berubah adalah kebohongan
kecil yang membuat daftar versi tidak bisa dipercaya, jadi kata-katanya dipilih
sesuai peristiwanya.

### Mengubah versi mengubah lebih dari versi itu

Ini bagian yang Anda angkat sendiri, dan alasan urutan restart-nya tinggal di
`Components::dependents()`, bukan di penangan klik.

**Apache memuat modul PHP untuk interpreter yang terpasang.** Ganti PHP, dan web
server masih memegang modul lama: ia start, ia menjawab, dan setiap permintaan ke
site PHP mengembalikan 500. Tidak ada yang terlihat rusak, dan justru itu yang
mahal. Jadi mengganti PHP me-restart panel dan Apache; mengganti FPM me-restart
Apache.

FPM adalah fakta yang sama dari sisi lain — pool yang berjalan milik interpreter
yang memulainya, jadi dihentikan dan diganti, bukan dibiarkan menjawab dari
berkas yang sudah tidak ada.

Setiap baris juga membawa pemblokir yang berlaku padanya, sebelum apa pun
dilakukan:

- **php** — ekstensi dikompilasi terhadap versi interpreter itu sendiri. Site
  yang butuh ekstensi yang tidak lagi dibawa PHP versi baru gagal saat dimuat,
  bukan saat dipasang.
- **apache** — vhost di disk memakai format satu server. Mengganti server tidak
  menulis ulang vhost-nya; jalankan `devkit site:apply` sesudahnya, atau site
  berhenti menjawab.
- **database** — versi mayor baru tidak akan membuka direktori data versi lama.
  Ini menolak, bukan melakukannya diam-diam.
- **nodejs** — addon native dibangun terhadap ABI versi yang membangunnya, dan
  harus dibangun ulang.

### Dua paket, satu perintah

Registry menyebut paket yang akan dipakai untuk mengubah komponen, dan itu tidak
selalu paket yang menyediakannya. Terukur di sini: `dpkg -S` pada binary `node`
menjawab `nodejs-lts`, sementara komponennya `nodejs`. Menawarkan "update" di
sana akan memasang Node kedua di atas yang pertama — apt menyebut itu sukses, dan
Node yang tadinya jalan berhenti jalan.

Jadi `versions:use` bertanya dulu ke `dpkg` siapa pemilik perintahnya, dan
menolak dengan alasan, dua jalan keluarnya, dan flag untuk memaksa:

```
$ devkit versions:use nodejs 26.4.0-1

FAIL Refusing, and this is the kind of refusal worth reading. Node.js on this
device is provided by the package nodejs-lts, and nodejs provides a command with
the same name. …
    Remove the owning package first if that is what you mean:
      pkg uninstall nodejs-lts
    Or override this check, which is not recommended:
      devkit versions:use nodejs 26.4.0-1 --replace-owner
```

### Sebelum apa pun dipasang

Biayanya diukur dengan meminta apt menyelesaikan pohon dependensi sungguhan
lewat `--print-uris`, yang mencetak rencana tanpa mengunduh apa pun — jadi Flutter
dilaporkan sebagai 117 MB unduh dan 326 MB disk sebelum satu byte pun bergerak.
Menjumlahkan paket yang disebut dan mengabaikan dependensinya meremehkan stack
database dengan selisih yang jauh, dan itu justru kasus saat kehabisan ruang itu
penting.

---

## Tunnel ke vscode.dev tidak bisa dibuat di sini

Mengetik `code .` di dalam folder project berarti meminta folder itu dibuka di VS
Code. Cara yang benar adalah tunnel: workspace-nya ada di ponsel, bisa dijangkau
dari browser di mana pun, tanpa client yang perlu dipasang. `code .` memanggil
binari tunnel kalau ada.

**Di ponsel tunnel itu tidak bisa dibuat, dan alasannya layak dinyatakan
terang-terang** daripada dibiarkan menjadi pesan spawn:

- `code-oss` 1.131 memang dikemas untuk Termux, dan CLI-nya jalan. `--help`,
  `--diff`, dan `--merge` semuanya berfungsi.
- `code-oss tunnel` dan `code-oss serve-web` sama-sama-sama ada di help CLI itu
  sendiri. Keduanya gagal seketika dan dengan cara yang persis sama:
  `spawn .../bin/code-tunnel-oss ENOENT`.
- Binarinya tidak ada di paket itu, dan tidak ada di dalam bundel 104 MB: string
  `code-tunnel` muncul nol kali di dalamnya. Yang ada adalah *pustakanya* —
  `@microsoft/dev-tunnels-connections` — dan itulah sebabnya subcommand-nya
  ditawarkan sama sekali. Kode yang akan memanggil programnya ada; programnya
  tidak.
- Microsoft tidak menerbitkan binarinya secara terpisah. API yang melayani
  build VS Code tidak memuat aset tunnel sama sekali, karena tunnel ikut dalam
  aplikasi desktop.

Jadi keputusannya diambil dari ketiadaan yang terukur, bukan dari preferensi.
`Tunnel::tunnelBinary()` mencari binarinya dengan ketiga nama yang dipakainya
(`code-tunnel-oss`, `code-tunnel`, `code-tunnel-insiders`) dan memakainya kalau
salah satu ada — jadi kalau Termux suatu saat mengemasnya, `code .` langsung
memakai tunnel tanpa perubahan di sini.

### Yang dilakukan sebagai gantinya

Jalur yang bekerja, dan itu bukan hadiah simpatik. Perangkat ini sudah menjalankan
server SSH dengan autentikasi publickey-saja, dan komputer yang menjalankan VS
Code berbicara Remote-SSH kepadanya adalah sesi edit yang sama persis: file-nya
ada di ponsel, editor-nya jalan di komputer, terminalnya adalah shell sungguhan di
perangkat, dan lalu lintasnya tidak pernah menyentuh server siapa pun.

```sh
code .                      # dari dalam folder project
code ~/projects/myapp       # atau sebutkan namanya
code . --print              # apa yang akan terjadi, tanpa melakukannya
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

Kuncinya milik komputer, dan config itu mengatakannya: `IdentityFile` adalah kunci
komputer itu sendiri, tidak pernah path di dalam perangkat ini. Komputer membuat
pasangan kuncinya dan hanya mengotorisasi setengah publiknya di sini —

```sh
ssh-keygen -t ed25519                                        # di komputer
devkit ssh:authorize "$(cat ~/.ssh/id_ed25519.pub)" "laptop"  # di ponsel
```

Kunci yang dibuat di ponsel lalu disalin ke mana-mana membuat ponsel memegang
private key yang tidak dibutuhkannya, dan komputer memegang public key yang belum
diotorisasi — lalu login ditolak tanpa alasan yang terlihat.

`code:address` melaporkan alamat *saat ini*, karena ponsel berpindah jaringan dan
config yang ditulis di satu jaringan salah di berikutnya. Kalau ada yang kurang —
tidak ada alamat, SSH belum dipasang, belum jalan, tidak ada kunci terotorisasi —
ia menyebutkan yang mana, bukan mencetak config yang tidak akan terhubung.

## Editor di browser tidak bisa jalan di sini

`devkit code:install` mengunduh rilis `linux-arm64` yang dipublikasikan,
mengekstraknya, dan menjalankan `--version` dengan sukses di bawah `glibc-runner`.
Pengunduhan dan ekstraksi keduanya baik — 207 MB masuk, 627 MB setelah
diekstrak. Editornya tetap tidak bisa jalan, dan alasannya struktural:

- Rilisnya hanya glibc. `lib/node` bawaannya meminta
  `/lib/ld-linux-aarch64.so.1`, yang tidak ada di Android.
- `glibc-runner` memulai program seperti itu dengan menyebut loader-nya secara
  eksplisit (`ld.so <program>`). Itulah sebabnya `--version` berhasil.
- Setelah berjalan, editor menjalankan ulang binernya sendiri untuk extension
  host. Proses kedua itu tidak dimulai oleh `glibc-runner`, jadi ia tidak punya
  loader yang bisa dicari, dan langsung keluar dengan kode 127.
- Menulis ulang interpreter binernya supaya loader ditemukan otomatis pernah
  dicoba. Ia segfault (keluar 139) di setiap kombinasi `LD_LIBRARY_PATH` dan
  `LD_PRELOAD` yang dicoba.

Panel mengatakan hal ini, bukan mengklaim berhasil. `code:status` mencantumkannya
sebagai pemblokir, dan tab Editor meminta `code:open` — jalur yang berfungsi —
bukan perintah mulai yang tidak mungkin berhasil.

Jalur yang juga akan berfungsi adalah `proot-distro`: userland Debian dengan
`/lib/ld-linux-aarch64.so.1` yang sungguhan, tempat loader berada di mana biner
mengharapkannya dan proses anak mewarisinya. Itu belum dipasang, dan ia
memerlukan unduhan rootfs Debian.

## Tab Device

Semuanya di sini berasal dari pembacaan perangkat sendiri, dan pembacaan yang
tidak bisa diambil **dinyatakan tidak tersedia beserta alasannya** — tidak pernah
ditampilkan sebagai `0`, yang tidak bisa dibedakan dari pengukuran sungguhan.

Yang biasanya tidak tersedia di Termux bawaan: status baterai dan termal (tidak
ada `termux-battery-status`, tidak ada `power_supply` yang bisa dibaca), serta
`/proc/loadavg` dan `/proc/stat`, yang diblokir Android. Beban rata-ratajatuh ke
perintah `uptime`, dan uptime ke `/proc/uptime` atau `uptime -p`.

## Sertifikat

```sh
devkit ca:create        # buat certificate authority lokal, sekali
devkit cert:issue-all   # terbitkan sertifikat untuk setiap site terdaftar
devkit ca:info          # tampilkan detail CA
```

CA-nya lokal, self-signed, di `ssl/ca` dalam direktori runtime.

**Ada dua penyimpanan kepercayaan yang berbeda, dan panel melaporkannya
terpisah.**

| Penyimpanan | Siapa yang menulis | Apa yang dipengaruhinya | Apa yang dikatakan panel |
|---|---|---|---|
| Bundel CA Termux (`$PREFIX/etc/tls/cert.pem`) | DevKit, atas permintaan | `curl`, Composer, PHP, `npm` di dalam Termux | **Termux: dipercaya / tidak dipercaya** — dicek dengan membaca bundel dan lewat jabat tangan TLS sungguhan |
| Penyimpanan CA pengguna Android | UI Settings Android saja | browser | **Browser: tidak diketahui** — tidak bisa dibaca atau ditulis dari Termux |

`ca:install` (tombol **Trust in Termux**) hanya mengisi baris pertama. Ia tidak
dan tidak bisa memengaruhi baris kedua, dan itulah sebabnya browser yang sudah
diberi tahu "tepercaya" masih menampilkan peringatan di URL HTTPS. Tombolnya
dinamai menurut apa yang ia lakukan, bukan menurut apa yang orangUNYAIhopes.

Setengah untuk browser butuh tiga ketukan, sekali saja, di ponsel:

1. Unduh CA dari `http://127.0.0.1:8898/__panel/ca.crt`.
2. **Install certificate** → **VPN and apps**.
3. Kunci layar, lalu tutup dan buka ulang browser.

Panel mencetak sidik jari SHA-256 CA di samping langkah-langkah itu supaya Anda
bisa memastikan sertifikat yang dipasang Android memang yang ini. Setelah
melakukan itu, tekan **I installed it in Android**; panel lalu mencatatnya sebagai
jawaban yang diberikan kepadanya, bukan yang diverifikasinya, karena ia tidak punya
cara memeriksanya:

```sh
devkit ca:browser-trust yes   # catat; juga bisa diatur dari panel
devkit ca:browser-trust no    # tarik kembali
```

Selama itu belum dilakukan, browser menampilkan peringatan di URL HTTPS. Site-nya
tetap termuat, dan URL HTTP biasa maupun berdasarkan IP tidak terpengaruh.

## Apa yang tidak akan dilakukan panel

- **Tidak menjalankan perintah yang tidak disebut daftar izinkan.** Menambahkan
  perintah ke daftar izinkan adalah penyuntingan yang disengaja pada berkas yang
  bisa Anda baca.
- **Tidak menyerahkan token GitHub Anda ke halaman.** Tokennya masuk ke disk, dan
  halaman hanya mengetahui apakah ada.
- **Tidak menerima token dari halaman.** Token GitHub datang dari GitHub sebagai
  tukaran kode yang_verifier-nya sudah dibuat perangkat ini.
- **Tidak menampilkan angka yang tidak bisa diukurnya.** Lihat tab Device.
- **Tidak menyentuh pengaturan jaringan perangkat.** Proxy `.app` diatur oleh
  Anda, di pengaturan Wi-Fi Android.
- **Tidak menyalakan SSH untuk Anda.** Itu satu-satunya service di jaringan, dan
  ia tetap mati sampai Anda menyalakannya.
- **Bukan shell kedua.** Terminalnya adalah sesi SSH sungguhan, jadi tidak ada
  apa pun di dalamnya yang bisa berbeda pendapat dengan shell yang Anda buka dari
  laptop.
