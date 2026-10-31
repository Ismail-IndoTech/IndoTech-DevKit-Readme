> 📚 **Documentation:** [Full ID/EN index](DOCS.md) · [English index](DOCS.en.md)

<div align="center">

<img src="docs/logo.png" alt="IndoTech" width="128" height="128">

# IndoTech-DevKit


### Your whole dev stack. One pocket.

**Turns a fresh Termux install on an Android phone into a working development
stack** — web server, PHP, database, SSL, GitHub, project hosting, a browser
control panel, and SSH so you can drive it all from a real keyboard.

[![Version](https://img.shields.io/badge/version-1.0.0-FF2D20?style=flat-square)](VERSION)
[![License: MIT](https://img.shields.io/badge/license-MIT-0EA5E9?style=flat-square)](LICENSE)
[![Termux](https://img.shields.io/badge/termux-required-000000?style=flat-square&logo=android)](https://f-droid.org/packages/com.termux/)
[![Android](https://img.shields.io/badge/android-7.0%2B-3DDC84?style=flat-square&logo=android)](https://developer.android.com/)
[![Made in Indonesia](https://img.shields.io/badge/made%20in-Indonesia-FF2D20?style=flat-square)](https://it-app.id)

**No account. No login. No clone. One command.**

```sh
curl -fsSL https://raw.githubusercontent.com/Ismail-IndoTech/IndoTech-DevKit/main/install-remote.sh | bash
```

That is the whole installation. It measures the phone before it offers anything,
asks you a short survey, installs exactly what you picked, starts the stack, and
opens the settings tool in your browser.

<div align="center">

<a href="README.md">Bahasa Indonesia</a> · **[English](README.en.md)**

</div>

</div>

---

## The stack it puts on your phone

<div align="center">

| | | | | |
|:---:|:---:|:---:|:---:|:---:|
| <img src="https://cdn.simpleicons.org/laravel/FF2D20" width="44" height="44" alt="Laravel"><br>**Laravel**<br>12 · 13 | <img src="https://cdn.simpleicons.org/php/777BB4" width="44" height="44" alt="PHP"><br>**PHP**<br>8.5 | <img src="https://cdn.simpleicons.org/mariadb/003545" width="44" height="44" alt="MariaDB"><br>**MariaDB**<br>MySQL-compatible | <img src="https://cdn.simpleicons.org/postgresql/4169E1" width="44" height="44" alt="PostgreSQL"><br>**PostgreSQL**<br>optional | <img src="https://cdn.simpleicons.org/phpmyadmin/FFF000" width="44" height="44" alt="phpMyAdmin"><br>**phpMyAdmin**<br>wired up |
| <img src="https://cdn.simpleicons.org/apache/EB1B26?viewbox=auto" width="44" height="44" alt="Apache"><br>**Apache**<br>or Nginx | <img src="https://cdn.simpleicons.org/nginx/009639" width="44" height="44" alt="Nginx"><br>**Nginx**<br>your choice | <img src="https://cdn.simpleicons.org/flutter/02569B" width="44" height="44" alt="Flutter"><br>**Flutter**<br>web + Android | <img src="https://cdn.simpleicons.org/composer/FFFFFF" width="44" height="44" alt="Composer"><br>**Composer**<br>dependencies | <img src="https://cdn.simpleicons.org/git/F05032" width="44" height="44" alt="Git"><br>**Git**<br>+ GitHub CLI |
| <img src="https://cdn.simpleicons.org/nodejs/5FA04E" width="44" height="44" alt="Node.js"><br>**Node.js**<br>LTS or current | <img src="https://cdn.simpleicons.org/python/3776AB" width="44" height="44" alt="Python"><br>**Python**<br>CPython | <img src="https://cdn.simpleicons.org/rust/000000" width="44" height="44" alt="Rust"><br>**Rust**<br>optional | <img src="https://cdn.simpleicons.org/golang/00ADD8" width="44" height="44" alt="Go"><br>**Go**<br>optional | <img src="https://img.shields.io/badge/-ripgrep-CC4A31?style=flat-square"><br>**ripgrep**<br>`rg`, not grep |

</div>

<table align="center">
<tr>
<td width="33%"><b>Termux</b><br><span style="color:#888">The environment it all runs in</span></td>
<td width="33%"><b>OpenSSH</b><br><span style="color:#888">Log in from your computer</span></td>
<td width="33%"><b>Squid</b><br><span style="color:#888">Makes <code>.app</code> names resolve</span></td>
</tr>
</table>

---

## Install

### The short way

```sh
curl -fsSL https://raw.githubusercontent.com/Ismail-IndoTech/IndoTech-DevKit/main/install-remote.sh | bash
```

No GitHub account, no SSH key, no `git clone`. The script downloads a release
tarball, unpacks it in a temporary directory, and hands over to the installer.

### Want to read it first?

Always reasonable for something that installs software:

```sh
curl -fsSL https://raw.githubusercontent.com/Ismail-IndoTech/IndoTech-DevKit/main/install-remote.sh -o install-remote.sh
less install-remote.sh          # read it
bash install-remote.sh          # then run it
```

### Options

```sh
bash install-remote.sh --dry-run        # ask everything, print the plan, install nothing
bash install-remote.sh --device-info    # report what this phone can run, change nothing
bash install-remote.sh --yes            # non-interactive, safe defaults
bash install-remote.sh --answers=FILE   # replay or edit a previous run's answers
```

### From a clone

```sh
git clone https://github.com/Ismail-IndoTech/IndoTech-DevKit.git
cd IndoTech-DevKit && ./install.sh
```

---

## What it does before it asks anything

A phone is not a small server, and the packages that work on a laptop do not all
work here. So the installer measures first, and the measurements decide what may
be offered at all:

- **Reads the device.** Architecture, Android release, CPU count, total RAM, free
  storage, memory page size, whether `su` really grants uid 0, whether the
  bootloader is locked.
- **Offers only what exists here.** Every version in every menu comes from this
  device's own apt metadata, and the repository each version comes from is named.
- **Resolves the real dependency tree** with `apt --print-uris` and compares it
  against free space, so a plan that cannot fit is refused *before* the download
  rather than halfway through it.
- **Greys out instead of guessing.** A database server on a 900 MB phone is shown
  but not selectable, with the measurement that ruled it out.

<details>
<summary>Examples of what gets refused, and why</summary>

| Message | What it means |
| --- | --- |
| `needs about 1200 MB of RAM; this device has 900 MB` | Real figures from `/proc/meminfo`. A database Android kills mid-task is worse than none. |
| `needs about 300 MB of free space to run; this device has 120 MB` | A floor for running it, separate from the download size. |
| `the 32-bit build of this toolchain is not part of Termux` | The package genuinely does not exist for this architecture. No installer can offer it. |
| `Android 13 (API 21) is older than the Android 7 (API 24) minimum` | Below the floor. Termux itself does not support this. |

There is no override flag: a deliberate override should look deliberate.

</details>

---

## Your settings tool

<div align="center">

`http://127.0.0.1:8898/__panel/`

</div>

A local web application that drives the same `devkit` commands you would type in
a terminal. It binds `127.0.0.1` and nothing else.

| Tab | What it does |
| --- | --- |
| **Services** | What is running, start/stop/restart, and which server is in charge |
| **Sites** | Registered sites, their URLs, and the links that work right now |
| **SSL** | The local certificate authority and the certificates from it |
| **SSH** | Log in from a computer, manage the keys that may |
| **Projects** | Scaffold a project, build it, run it — **with a real terminal on the same page** |
| **Editor** | Open a folder in VS Code, and an honest note on why the tunnel cannot be made here |
| **GitHub** | Sign in, list repositories, push a project |
| **Device** | What this phone actually is, and what it is doing now |
| **Environment** | Versions and paths this stack resolved to |
| **Logs** | Recent output from any service |

### The terminal, on the Projects page

There is no separate Terminal tab. The terminal is **on the Projects page**,
because that is where long commands are run — and creating a Laravel project is
composer resolving a dependency tree, git cloning a repository, a build tool
compiling. None of that can be sped up, only watched.

So the panel runs the real command detached and follows its output as it
happens, instead of holding the page open for two minutes and then printing the
last line:

```
$ laravel new myapp

  Which starter kit do you want?  laravel/laravel
  Installing dependencies...
  Package operations: 68 installs, 0 updates, 0 removals
  ...
  Application key set successfully.
```

The panel's terminal is a **real SSH session to this device** — xterm.js over a
WebSocket to `ssh` on `127.0.0.1`, using a key the panel owns and that is
authorized through `Ssh::authorize` like any other. It is not a second shell: a
shell started from a web request has the request's environment and skips the
authentication, the key list and the SSH logs, and then the panel and your laptop
disagree about who may in. Going through `ssh` means the panel arrives by the
same door as a laptop, so **testing the terminal is testing SSH**, and revoking
the panel's key removes its access exactly as it would a laptop's.

```sh
devkit terminal:status           # the listener, the key, and its WebSocket URL
devkit terminal:start            # listening on 127.0.0.1:8896
devkit terminal:restart
```

### The SSH tab

A real login on the device, so `devkit` and every other command can be used from
a computer with a keyboard.

```sh
devkit ssh:install                    # write the config, host key, and key file
devkit ssh:authorize "$(cat laptop.pub)" "work laptop"
devkit ssh:start                      # listening on 0.0.0.0:8022
devkit ssh:connect                    # just the command to run on the computer
```

```
ssh -p 8022 u0_a247@192.168.2.76
```

Publickey is the **only** authentication method — there is no password to guess,
crack, or reuse from a breach somewhere else. The port is on your network, which
is the honest cost of that, and the panel says so every time it is on.

> **Wireless debugging note:** a phone cannot reach itself over Wi-Fi, so
> `ssh …@192.168.2.76` has to be run *from your computer*. On the phone itself,
> use `adb reverse` or the terminal on the Projects page.

---

## Your first site

```sh
devkit site:add --id=htdocs --name=htdocs --host=htdocs.app \
                --root=~/htdocs --http=8080 --https=8443
```

`htdocs.app` gets a certificate, a vhost, and a port of its own. It answers on:

- `http://htdocs.localtest.me:8080/` — resolves to 127.0.0.1, needs nothing
- `https://htdocs.127.0.0.1.sslip.io:8443/` — TLS, also needs nothing
- `http://htdocs.app:8080/` — the pretty one, via the proxy (see below)

> **HTTPS on a `.app` name has no route.** A `CONNECT` cannot be moved to another
> port, so `https://htdocs.app/` will never reach this device. For HTTPS, use a
> `localtest.me` or `sslip.io` name with the port in the URL — neither needs a
> proxy at all. `devkit proxy:doctor` records this limit every time it runs.

### Trusting the local certificate authority

| Store | Who writes it | What the panel says |
| --- | --- | --- |
| Termux's CA bundle | DevKit, on request | **Termux: trusted / not trusted** — checked by reading the bundle and by a real TLS handshake |
| Android's user CA store | Android's Settings, by hand | **Browser: unknown** — it cannot be written from Termux |

`ca:install` (*Trust in Termux* in the SSL tab) fills in the first row. The
browser half is three taps, once, on the phone:

1. Open `http://127.0.0.1:8898/__panel/ca.crt` and download it.
2. **Install certificate** → **VPN and apps**.
3. Confirm your screen lock, then close and reopen the browser.

### The `.app` hostnames and the proxy

`localtest.me` and `sslip.io` resolve to 127.0.0.1 for free. `.app` does not, so
the proxy exists to resolve it on the device:

1. Open the Wi-Fi network you are on.
2. Set the proxy to **Manual**, host `127.0.0.1`, port `8899`.

This only affects your phone, and only while it is set. The panel is never behind
the proxy, so it works on any network either way.

To check every link at once, with the status code each one returns:

```sh
devkit proxy:doctor
```

---

## Projects

```sh
devkit project:root                             # where projects live
devkit project:templates                        # what can be made, and what is missing
devkit project:new blog --template=static       # create it and register its site
devkit project:new app --template=laravel       # laravel new + composer install
devkit project:new app --template=flutter       # flutter create
devkit project:build app --template=flutter     # flutter build web
devkit project:artisan blog migrate             # run artisan inside a project
devkit project:remove blog --delete
```

Every new project is registered as a site at once, gets a certificate, and shows
up in the panel. Templates are the built-in set plus anything you add in
`state/templates.json` — each entry is an argv array, never a shell string.

**Android Run** is the panel's honest version of "run it on the phone": the card
only appears when the project really is Flutter, `flutter` and `adb` are really
there, and a device is really connected. What is missing is stated rather than
offered as a button that does nothing.

---

## `code .` — opening a folder in VS Code

Standing inside a project folder, type `code .`:

```sh
cd ~/projects/myapp
code .
```

It tries to make a **tunnel to vscode.dev** first — the workspace on the phone,
open in a browser anywhere, with no client to install. If the tunnel binary is
there, that is what it uses.

**On a phone the tunnel cannot be made, and the reason is worth stating plainly**
rather than leaving it to a `spawn ... ENOENT`:

- `code-oss` is packaged for Termux, and its CLI runs.
- `code-oss tunnel` and `code-oss serve-web` are both advertised in `--help`, and
  both fail in exactly the same way: `spawn .../bin/code-tunnel-oss ENOENT`.
- The binary is not in that package, and not inside the 104 MB bundle: the string
  `code-tunnel` appears zero times in it. What *is* there is the library
  (`@microsoft/dev-tunnels-connections`) — which is why the subcommand is offered
  at all. The code that would call the program is present; the program is not.
- Microsoft does not publish that binary separately. The API that serves VS Code
  builds carries no tunnel asset at all, because tunnels ship inside the desktop
  application.

What works is below, and it is not a consolation prize.

```sh
code .                      # from inside a project folder
code . --print              # what would happen, without doing it
devkit code:address         # this device's address, and what is missing
devkit code:steps           # the full one-time setup
```

`code .` prints the `~/.ssh/config` block to paste on the computer:

```
Host indotech-devkit
    HostName 192.168.2.76
    User u0_a247
    Port 8022
    IdentityFile ~/.ssh/id_ed25519
```

Then in VS Code: **Remote-SSH: Connect to Host…** → `indotech-devkit`.

The editing session is the same: files on the phone, editor on the computer, the
terminal a real shell on the phone, and the traffic touching nobody else's
server.

The key is made **on the computer**, and that is what the config says —
`IdentityFile` is the computer's own key, never a path inside this device:

```sh
ssh-keygen -t ed25519                                          # on the computer
devkit ssh:authorize "$(cat ~/.ssh/id_ed25519.pub)" "laptop"    # on the phone
```

---

## The `devkit` command

Everything the panel does, from a shell. `devkit help` lists all of it.

### The two commands worth remembering

```sh
devkit doctor                  # check this device end to end
IndoTech-DevKit                # start whatever is not running, then open the panel
```

`devkit doctor` answers a different question from `service:status`.
`service:status` answers *is it running*; `doctor` answers *does it work*, and
that is where the real problems live. Each port is asked whether anything answers
on it, each site whether its URL returns a page, each certificate whether it
still verifies.

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

A warning is a choice or a limit, not a fault — a device with no sites is a
device that is working, and a check that failed on that would be one people
stopped reading. The command exits non-zero only on a failure.

### Choosing a component version

```sh
devkit versions:list                    # every component, one line each
devkit versions:show php                # one component, and what a change would break
devkit versions:use <component> <ver>   # install it, then restart what depends on it
devkit versions:update <component>      # the newest version available on this device
```

Every number is read from this device with `apt-cache madison` and `dpkg`. On a
phone there is usually **one version per component** — the Termux repositories do
not carry older builds — so the panel writes "1 version only" with the reason
underneath, rather than hiding a dropdown that contains one entry.

Changing PHP restarts Apache, because Apache holds the module for the interpreter
that was installed; without that restart the server answers and returns 500 on
every PHP request. `versions:show` names the blocker for each component before
anything is installed, and the download and disk cost are measured first from
apt's own dependency tree.

### A short list

```sh
devkit service:status                  # what is running
devkit service:start all               # every one: web server, queue, proxy, ssh, terminal, panel
devkit service:stop all                # every one, dependents before the things they need
devkit site:list                       # registered sites
devkit ca:create && devkit ca:install  # local certificate authority
devkit project:new app --template=laravel
devkit ssh:start                       # log in from a computer
devkit env:info                        # versions and paths this stack resolved to
devkit device:info                     # what this phone is, live
```

Add `--json` to any of them for machine-readable output.

### The one command

After installing, this name works in the Termux shell:

```sh
IndoTech-DevKit
```

It starts whatever is not running, waits until the settings tool actually
answers, and opens it in a new browser tab. Running it twice is harmless.

---

## How it is put together

| Service | Port | Why |
| --- | --- | --- |
| Panel | `127.0.0.1:8898` | PHP's built-in server. Separate from the proxy, so a broken proxy cannot hide the control room. |
| Terminal | `127.0.0.1:8896` | A WebSocket listener of its own. `php -S` cannot upgrade a connection to a WebSocket. |
| Proxy | `127.0.0.1:8899` | Squid. Resolves `.app` names and passes everything else through. |
| Web server | `8080` / `8443` | Termux cannot bind below 1024 without root. |
| Database | `3306` | Already above 1024, so it is untouched. |
| SSH | `8022` | The one service that answers on the network, on purpose. |
| Project sites | `9000+` | One port per registered project. |

**The panel never runs a command the browser names.** Every action is matched
against a fixed allow-list of `devkit` subcommands, and anything that changes
something also needs the per-session token the page itself carries. A scaffold is
a template id looked up in a table, never a command line assembled from text.

Nothing is reported that was not measured. A reading Android does not allow is
shown as unavailable *with the reason*, never as a `0` that would be
indistinguishable from a real answer.

---

## When something is wrong

```sh
devkit doctor                # start here: it is the one that knows what is broken
devkit env:info              # what the installed stack is using
devkit service:status        # what is running, and the URLs
```

- **`service:start` says already running but it is not** — the port is what
  decides, and a stale pid file no longer overrides it. `devkit service:restart`
  is safe at any time.
- **A port says occupied but nothing answers on it** — the listener is still in
  the kernel while its process is gone. Android only releases it when the app is
  force-stopped. Reopen Termux, then run it again. This cannot be fixed from
  inside, and the message says so.
- **HTTPS shows a warning in the browser** — expected until you install the CA in
  Android's own store; the panel cannot do that for you.
- **A `.app` name does not resolve** — the Wi-Fi proxy is not set, or the proxy is
  stopped. `localtest.me` always works without either.
- **`code .` says the tunnel is not available** — that is the accurate answer,
  not a failure to try. What it returns instead is the Remote-SSH route, which
  works and needs nothing this device does not already run.

More in [docs/troubleshooting.en.md](docs/troubleshooting.en.md).

---

## Uninstall

```sh
devkit service:stop all
rm -rf ~/IndoTech_Project          # or wherever DEVKIT_HOME points
rm -f  $PREFIX/bin/devkit $PREFIX/bin/IndoTech-DevKit $PREFIX/bin/code
pkg uninstall apache2 php mariadb   # whatever the installer reported it installed
```

The answers file (`~/.devkit-answers`) lists exactly what was installed.

---

## Contributing

Contributions are welcome, including small ones. `devkit doctor` exists because
somebody measured instead of reading — and found three faults that no command
reported.

```sh
git clone https://github.com/Ismail-IndoTech/IndoTech-DevKit.git
cd IndoTech-DevKit
./install.sh --dry-run          # make sure nothing is damaged
```

Before opening a pull request:

```sh
php -l on every .php file
bash -n on every .sh
node --check payload/devpanel/public/assets/panel.js
python3 tools/check-panel-ids.py
php tools/check-allowlist.php
devkit doctor
```

The short rule: **no claim that has not been measured.** If it concerns Termux,
run it in Termux. The detail is in
[CONTRIBUTING.en.md](CONTRIBUTING.en.md) — or
[dalam Bahasa Indonesia](CONTRIBUTING.md).

---

## Access & License

This repository is **my own work**, **Mochamad Ismail Soleh** (Ismail-IndoTech), and is specifically licensed under the **[MIT License](LICENSE)**.

To obtain access to the **private source repository** for this project, contact me as the **Repository Creator** via WhatsApp using the number registered to my DANA account: **08121600058** — [wa.me/628121600058](https://wa.me/628121600058). Access is granted by agreement.
## 💖 Contribution & Sponsorship

If this platform enhances your IT operations, DevOps workflows, or server management efficiency, consider supporting the project as an official contributor or **Special Sponsor**.  
*(Official sponsors will have their company logos prominently featured in this repository).*

Support continuous maintenance, research into hosting bypass vectors, and open-source module development by donating via:

<div align="center">

**DANA — 08121600058** (a.n. Mochamad Ismail Soleh)

<img src="docs/qris.jpg" alt="QRIS DANA IndoTech" width="280">

*Scan with any DANA, BCA, Mandiri, GoPay, OVO, or standard QRIS mobile banking application.*

</div>

All contributions directly fund server research infrastructure, testing licenses, and open-source tool development by IndoTech. If you or your organization would like to discuss partnership or corporate sponsorship, please reach out to us. Thank you for your support!

---

## Documentation

<div align="center">

<a href="README.md">Bahasa Indonesia</a> · **[English](README.en.md)**

</div>

- [docs/how-it-works.en.md](docs/how-it-works.en.md) — how the installer measures, and why · [Bahasa Indonesia](docs/how-it-works.md)
- [docs/panel.en.md](docs/panel.en.md) — every tab, every command, and what it refuses · [Bahasa Indonesia](docs/panel.md)
- [docs/troubleshooting.en.md](docs/troubleshooting.en.md) — when something looks wrong · [Bahasa Indonesia](docs/troubleshooting.md)
- [CONTRIBUTING.en.md](CONTRIBUTING.en.md) — how to contribute · [Bahasa Indonesia](CONTRIBUTING.md)

---

## License

MIT — see [LICENSE](LICENSE). Copyright © 2026 **Ismail.IndoTech**.

<div align="center">

Made with :heart: by [**Ismail.IndoTech**](https://it-app.id) · [it-app.id](https://it-app.id)

</div>