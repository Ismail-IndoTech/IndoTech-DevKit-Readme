# The panel, the proxy, and `.app` hostnames

<a href="panel.md">Bahasa Indonesia</a> · **[English](panel.en.md)**

## What the panel is

A local web application, served by PHP's built-in dev server on
`http://127.0.0.1:8898/__panel/`, that drives the same `devkit` commands you would
type in a terminal. The panel has its own port on purpose: it is never behind
the proxy, so a proxy that is down cannot take the control room with it. Ten
tabs:

| Tab | What it does |
| --- | --- |
| **Services** | What is running, which web server is in charge, and start/stop/restart it |
| **Sites** | Registered sites, their URLs, and the proxy URLs to open |
| **SSL** | The local CA and the certificates issued from it |
| **SSH** | Log in from another machine, and manage the keys allowed to |
| **Projects** | Scaffold a project, build it, run artisan — **with a real terminal on the same page** |
| **Editor** | Open a folder in VS Code — see [the tunnel](#the-tunnel-to-vscode-dev-cannot-be-made-here) |
| **GitHub** | Sign in, list repositories, push a project |
| **Device** | What this phone actually is, and what it is doing right now |
| **Environment** | Versions and paths this stack resolved to |
| **Logs** | Recent output from any service |

It runs on the device. It binds `127.0.0.1` and nothing else, so it is not
reachable from the network unless you change that yourself.

## Checking that it works

```sh
devkit doctor
```

This is the first thing to run, and it is not the same question as
`devkit service:status`. `service:status` answers *is it running*. `doctor`
answers *does it work*, and the gap between those two answers is where the real
faults were.

The difference is that every check asks the thing rather than the flag:

- each port is asked whether anything **answers** on it, which separates a
  working listener from one whose process is gone;
- each site is asked whether its **URL returns a page**, which separates a
  configured site from a served one;
- the proxy's configuration is **parsed by Squid itself**, and each site's web
  server is asked whether it restarted cleanly;
- each certificate is asked **how long it has left**, and the two trust stores
  are reported separately, because "HTTPS does not work" and "your CA is not
  installed" look identical from a browser.

Each line carries a fix where there is one to give:

```
  [1mServices[0m
    ok    Apache                     running
    warn  DevKit panel               stopped
                                    -> devkit service:start panel
  [1mPanel[0m
    FAIL  Port 8898                  held by a listener that does not answer.
                                    -> Force-stop and reopen Termux, then: devkit service:start panel
```

A **warning** is a choice or a limit, not a fault — a device with no sites is a
device that is working, and a check that failed on that would be one people
stopped reading. The command exits non-zero only on a failure, so it can be used
in CI.

## Services, and which web server is in charge

`state/server.conf` records the web server DevKit drives. The installer writes it;
the runtime treats it as read-only.

When it is missing, the server is **inferred from the vhosts on disk** — a
`<VirtualHost>` block can only be read by Apache and a `server {` block only by
nginx, which is stronger evidence than which binaries happen to be installed.
Having `nginx` installed says nothing about which server the sites were written
for. Either way the panel says which it is doing and why, rather than presenting a
guess as a fact:

```sh
devkit server:show            # what is configured, and whether that was read or inferred
devkit server:set apache      # record it, so it stops being a guess
devkit server:set nginx
```

`server:set` refuses to switch when the vhosts on disk are in the *other* format,
because pointing DevKit at nginx while every site is an Apache block would take
every site down the moment Apache stopped. `--force` overrides, and you then run
`devkit site:apply` to rewrite the vhosts.

**A device can have both `apache2` and `nginx` installed**, and `pkg` has no reason
to remove one. So the panel also shows the other server when it is there, labelled
as installed-but-not-configured, with an explanation. It gets no **Start** button:
starting it would only put a second server on ports the sites need.

`service:start all` and `service:stop all` mean **every** service, and that is a
different list from `Services::active()` on purpose. `active()` is the *default*
stack — the things a DevKit installation is expected to have up, and the set the
panel is allowed to call broken. SSH is not in it, because it is the one service
that answers on the network.

`all` is not a default, it is an instruction. Somebody typing it has said
"everything", and leaving two services out of "everything" is the kind of quiet
scope reduction that costs an afternoon. The order is dependency order: web
server, queue, proxy, ssh, terminal, panel — the terminal is an SSH session and
cannot start before sshd. Stopping runs the other way, so a thing is shut down
before the things it needs.

```sh
devkit service:start all
devkit service:stop all
```

After `start all`, the panel says that SSH is now listening and why that matters,
because a port opened on the network is not something that should happen quietly.

The web server in charge is a card like any other, with working Start, Stop and
Restart. Stopping the proxy is treated specially — it is the page you are looking
at, so it is confirmed first, and afterwards the panel says plainly that it closed
itself and how to bring it back.

## How the panel is allowed to talk to the CLI

The browser never runs a command. It requests JSON from
`http://127.0.0.1:8898/__api/<command>`, and the panel's router matches
`<command>` against a fixed allow-list of `devkit` subcommands. A command that is
not in the list is refused before anything is executed. A request that mutates
something must also carry the per-session token in `X-DevKit-Token`; read-only
commands do not.

A scaffold is never assembled from request text. The panel sends a template id;
`Projects.php` looks the command up in its own table (or in your editable
`state/templates.json`) and runs it as an argument array. There is no shell string
anywhere on that path, so a project name cannot become a command.

Long operations — a scaffold, `composer create-project`, a Flutter build — are
given a longer timeout rather than being cut off.

`tools/check-allowlist.php` compares the two lists and fails if the panel asks
for a command the CLI does not implement. It is the check that catches a button
which was added to the HTML and never to the table.

## Why there is a proxy at all

Termux cannot bind ports below 1024. That single fact shapes the whole layout:

| Service | Port | Why not 80/443 |
| --- | --- | --- |
| Apache, plain | 8080 | 80 needs root |
| Apache, TLS | 8443 | 443 needs root |
| Database | 3306 | above 1024 already, so it is fine |
| DevKit panel | 8898 | separate from the proxy, so a broken proxy cannot hide it |
| Terminal | 8896 | `php -S` cannot upgrade a connection to a WebSocket |
| DevKit proxy | 8899 | Squid, only for `.app` hostname resolution |
| Project sites | 9000+ | one per registered project |

## Two ways to name a site

### `localtest.me` — needs nothing

`localtest.me` and `127.0.0.1.sslip.io` are public DNS records that resolve to
`127.0.0.1`. A registered site named `htdocs` answers on both:

```
http://htdocs.localtest.me:8080/
https://htdocs.localtest.me:8443/
https://htdocs.127.0.0.1.sslip.io:8443/
```

Nothing to configure, works offline once resolved. **This is the route that carries
HTTPS**, and it is the one to bookmark.

### `.app` — needs the proxy, and only over HTTP

A site can also be named `htdocs.app`, which is what the certificate is issued for
and what reads best. For that to resolve on the device, the device's own network
requests have to be sent to the proxy:

1. Open the Wi-Fi network you are on.
2. Set the proxy to **Manual**.
3. Host `127.0.0.1`, port `8899`.
4. Start the proxy: `devkit service:start proxy`.

Then `http://htdocs.app:8080/` reaches the site. The proxy reads the hostname,
looks it up in `state/sites.json`, and forwards to that site's own port.

**HTTPS on a `.app` name has no route, and this is structural.** A `CONNECT` cannot
be moved to a different port, so `https://htdocs.app/` on 443 never reaches this
device. `devkit proxy:doctor` says so every time it runs rather than printing a
link that cannot open.

**A site is registered in two places, and both have to be written.** A site is one
entry in `state/sites.json` and two expressions of it: a vhost for the web server,
and a rule in the proxy that lets the name reach it. Writing only the first is what
this used to do, and the symptom was measured: `site:add` printed success, the
site answered on `127.0.0.1:9000`, and every request through the proxy came back
`ERR_ACCESS_DENIED` — because a name the device had just registered was matching
the rule that refuses names it has never heard of. The proxy's configuration is
only rewritten when the proxy starts, so anything added afterwards was invisible
to it. `applyAndReload` now reconfigures the proxy as well, and prints what it did.

```sh
devkit proxy:doctor     # every address, with the status code it returns
devkit site:apply       # rewrite the vhosts *and* the proxy configuration
```

**This is per-network, and it only affects the device.** It is set in Android's
Wi-Fi settings, not by this installer, because the installer has no business
changing how your phone routes all its traffic. It has a real cost: while the
manual proxy is set, every request the device makes goes through Squid on the
phone. Turn it off when you are not using it.

## Projects

`devkit project:root` shows or changes the folder projects are created in. It
defaults to `~/projects` and is created on first use.

```sh
devkit project:templates                    # what can be made, and what is missing
devkit project:new blog --template=static   # create it and register its site
devkit project:build app --template=flutter # flutter build web
devkit project:artisan blog migrate         # run artisan inside a project
devkit project:remove blog --delete         # remove the registration, and the files
```

A new project gets a vhost pointing at the right subdirectory — `public` for
Laravel, `build/web` for Flutter, the folder itself for a static site — a
certificate for `<name>.app`, and a port of its own. The name must be lowercase:
`MyApp` and `myapp` would be one hostname but two directories, because Android's
filesystem preserves case, so they cannot be told apart later.

### Templates

`laravel` and `flutter` need a tool that Termux does not ship, so those two
report what to install and refuse rather than half-creating something:

```sh
composer global require laravel/installer   # for laravel
pkg install flutter                          # for flutter
```

`static` always works, because DevKit writes those files itself.

### Adding your own template

`state/templates.json` is read at runtime and merged with the built-in ones. Each
entry is keyed by its id:

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

| Key | Meaning |
| --- | --- |
| `command` | Required. Run as an argv array. `{name}` becomes the project name. |
| `docroot` | Subdirectory the vhost points at, relative to the project. Default `.` |
| `tool` | The binary that must exist for this template to be usable |
| `tool_hint` | The line to print when that tool is missing |
| `needs_build` | True if the site has nothing to serve until a build runs |
| `options` | Extra scaffold options to offer |

A `command` may also be a string, and it is split on whitespace — never sent to a
shell, so quoting and `&&` will not do what they do in a terminal. Use an array.

Entries that cannot be used are dropped from the list with the reason kept in
`devkit project:templates`, rather than failing at the moment you pick one.

## The terminal, on the Projects page

Long commands are watched, not waited for.

Creating a Laravel project is composer resolving a dependency tree, git cloning a
repository, a build tool compiling. None of that can be made faster; it can only
be watched.

### It is a real SSH session, and that is the whole design

There is no separate Terminal tab, and there is no second shell. The terminal is
**on the Projects page**, and what it opens is:

```
xterm.js  ->  WebSocket  ->  ssh -p 8022 <user>@127.0.0.1  ->  a real login shell
```

A shell started from a web request is not the same thing, and pretending it is
causes real problems. It has the request's environment, so it skips the
authentication, the key list and the SSH logs — and then the panel and a laptop
disagree about what is running and who may in. The key the panel uses is authorized
through `Ssh::authorize` like any other key, and its comment says where it came
from, so six months later the question "can I revoke the panel's access" has the
same answer as "can I revoke my laptop's".

Going through `ssh` means the panel arrives by the same door as a laptop.
**Testing the terminal is testing SSH.**

```sh
devkit terminal:status       # the listener, the key, and its WebSocket URL
devkit terminal:start        # listening on 127.0.0.1:8896
devkit terminal:stop
devkit terminal:restart
devkit terminal:forget-key   # revoke the panel's key; the terminal then refuses
```

It requires `devkit ssh:start` first, and says so rather than failing to connect.

### Why a listener of its own

The panel is served by `php -S`, and `php -S` cannot upgrade a connection to a
WebSocket. So the terminal is a small listener of its own on loopback `8896` — the
same shape as Squid being separate from the panel, for the same reason: one broken
listener should not be able to take the others down.

The token travels in the WebSocket query string, because a browser cannot set
custom headers on a WebSocket handshake. That is safe here for one reason and not
as a general rule: the listener is bound to loopback, so nothing off the device
can reach it to steal the token from the URL.

### A pipe, not a PTY

`laravel new` asks questions and `flutter create` prints a progress bar. Both
behave as they do from a shell, because this is a shell. The connection to `ssh`
is a pipe rather than a pseudo-terminal, which costs a real feature — no window
resize, no full-screen programs — and buys the absence of a pty and a winsize
protocol and the class of hangs that come with them.

### Jobs still exist, and are still useful

Some commands are worth running without a terminal: they are short, they need no
input, and their output is worth keeping after the fact.

```sh
devkit job:run project:new myapp --template=laravel
devkit job:list
devkit job:status <id>
devkit job:log <id> [lines]
devkit job:stop <id>
```

The job *is* the command, run detached with its output in a file, and the exit
code recorded by a one-line wrapper. Nothing reformats the output except stripping
the colour codes a terminal renders. Jobs are kept for an hour and pruned.

## GitHub

Sign-in uses the ordinary OAuth **authorization-code grant with PKCE**. The
browser on this phone is already signed in to GitHub, so approving is one tap,
and the callback lands on the loopback panel port that is already listening —
there is no second screen and no code to copy.

1. The panel starts the flow and opens GitHub's authorization page.
2. You approve in the browser; GitHub redirects to
   `http://127.0.0.1:8898/__github/callback` with a code.
3. The panel exchanges the code for a token using the PKCE verifier it minted,
   and stores it in `state/github.json`, mode `0600`.

The token is never sent to the browser and never accepted from it. Git uses it
through a PHP credential helper, so it does not end up in `~/.git-credentials`,
in the remote URL, or in a shell history entry.

Scopes requested are `repo` and `read:org`. Not `workflow` — pushing a workflow
file from a phone should be a deliberate act, not a side effect of signing in.

**Signing out does not revoke the token at GitHub.** Revoking it needs a call to
GitHub's API with the client secret; the local sign-out deletes the stored copy
and tells you to revoke it at `https://github.com/settings/tokens` if you want it
dead. The secret itself is also deleted on sign-out — the token remains valid
upstream until revoked there.

Because of PKCE, the **client id** is enough to start the flow, but GitHub still
requires the app's **client secret** to exchange the code, so both are needed.
The secret is yours to keep: it is written to `state/github.json` with mode
`0600`, and what it can do is stated plainly in sign-out — it can authorise this
one OAuth app, and nothing else.

Register the app at <https://github.com/settings/developers> as an **OAuth App**
(not a GitHub App — the web flow here is OAuth, not an installation):

- Homepage URL: `http://127.0.0.1:8898/__panel/`
- Authorization callback URL: `http://127.0.0.1:8898/__github/callback`
- Scopes: `repo`, `read:org`

Then give the panel the client id and secret — in the GitHub tab, or:

```sh
devkit github:client-id <id> <secret>
```

Until they are set, the panel shows the setup card rather than raising an error
every time the tab is opened — nothing is broken, one setup step is outstanding.

When they *are* set but GitHub still refuses, the panel says which of the two
causes it is, because GitHub's own reply (`Not Found`) does not:

- **"did not recognise the client id"** — the id is wrong, or the app it belonged
  to was deleted.
- **"the callback was rejected"** — the registered callback URL does not match
  `http://127.0.0.1:8898/__github/callback`.

## The SSH tab

Everything else on this page is about a service bound to `127.0.0.1`. This one is
not, and that is the whole reason it exists: a login, a TTY and a shell, so
`devkit` behaves the same from a laptop keyboard as it does in Termux.

```sh
devkit ssh:install                     # config, host key, authorized_keys
devkit ssh:authorize "<public key>" "work laptop"
devkit ssh:start
devkit ssh:status
devkit ssh:keys
devkit ssh:revoke SHA256:...
devkit ssh:stop
```

### What is bound, and what is not

`0.0.0.0:8022`, because a loopback port is not reachable from another machine
and would be no use at all. Every device on the network can reach that port, so
the configuration is the only version of this that is defensible from across a
room:

| Setting | Value | Why |
| --- | --- | --- |
| `AuthenticationMethods` | `publickey` | Makes the key a floor, not a preference: no later edit can downgrade it to a password. |
| `PasswordAuthentication` | `no` | There is no password to guess, crack, or reuse from a breach elsewhere. |
| `AllowUsers` | this account | A leaked key cannot be paired with another user on the device. Termux has one. |
| `AllowTcpForwarding` | `local` | Tunnelling the panel to a laptop (`ssh -L 8898:127.0.0.1:8898`) is useful; using the session as a jump host to another machine is not. |
| `StrictModes` | `no` | Android puts `$HOME` under `/data/data/com.termux/files/home`, and sshd reads that path as a permissions problem on a directory nobody else can enter. |
| `SetEnv PATH` | set | sshd gives a session no profile, and this device has no `~/.bashrc`, so without it `devkit` is simply not found. |
| `MaxAuthTries` | `3` | A port this open should not also be patient. |

### The keys DevKit adds, and the ones it does not

Authorized keys live in `~/.ssh/authorized_keys` **between markers**, because
that file may already hold keys the user put there. DevKit adds and revokes only
what is inside its own block; a key added by hand is neither listed nor
removable from here, and that is the point — this is not a general editor of
somebody's ssh setup.

Each key DevKit adds is commented with the date and where it came from, because
a comment is the only thing a person reads when deciding, six months later,
whether to revoke something.

Measured on the phone this was written on: a registered key logs in, an
unregistered key is refused, a password is refused, and a key that has been
revoked is refused — with `Permission denied (publickey)` in every refusal case.

### It is not started for you

`service:start all` does not include SSH, and `Services::active()` never will.
Opening a port to the network should be something a person does, once, on
purpose.

## Choosing a version

The **Versions** tab answers "which version of PHP, Apache, Node, Composer is
installed, and what could I have instead". Every number on it is read from the
device with `apt-cache madison` and `dpkg`; nothing is a table written by hand.

```sh
devkit versions:list                    # every component, one line each
devkit versions:show php                # one component in detail
devkit versions:use <component> <ver>   # install it, then restart what depends on it
devkit versions:update <component>      # the newest version available here
```

### There is usually one version, and the panel says so

This is the measured shape of a phone, and it is worth stating plainly rather
than dressing up:

| Component | Versions apt offers |
| --- | --- |
| php, php-fpm, composer, nodejs, flutter | 1 |
| apache2, nginx | 1 |
| mariadb, postgresql | 1 |

`apt-cache madison` reports a single line for every one of them, across all four
configured repositories. There is no `php8.3` package, no versioned package name,
and no archive of older builds — the Termux repositories carry the current
version and nothing else.

**Flutter is the hardest case of all**: Google has never published a Flutter SDK
for Linux on arm64, so the only build that exists for a device like this one comes
from a third-party repository. There is no choice to offer, and the panel says
that rather than showing a dropdown with one entry.

So the page shows `1 version only` in a muted badge, with the reason underneath.
A version picker that offered a menu of one and said nothing would be worse than
useless — it would look like a feature that had failed to load.

### Update means what it says

On a one-version component, `versions:update` reinstalls the version that is
already there, and the message says exactly that:

```
PHP is already on the newest version this device can install (8.5.1), and this
device carries only that one. There is nothing to move to - a newer one would
have to come from a build for Linux arm64, which most of these do not have.
```

An update that reports success while nothing changed is the kind of small lie
that makes a version list untrustworthy, so the words are chosen to match the
event.

### Changing a version changes more than that version

This is the part the request itself raised, and it is the reason the restart
order lives in `Components::dependents()` rather than in the click handler.

**Apache loads the PHP module for the interpreter that is installed.** Change
PHP, and the web server still holds the old module: it starts, it answers, and
every request to a PHP site returns 500. Nothing looks broken, which is what makes
it expensive. So changing PHP restarts the panel and Apache; changing FPM
restarts Apache.

FPM is the same fact from the other side — a running pool belongs to the
interpreter that started it, so it is stopped and replaced rather than left
answering from files that no longer exist.

A row also carries the blocker that applies to it, before anything is done:

- **php** — extensions are compiled against the interpreter's own version. A site
  needing an extension the new PHP no longer ships fails on load, not on install.
- **apache** — the vhosts on disk are in one server's format. Changing the server
  does not rewrite them; `devkit site:apply` afterwards, or the sites stop
  answering.
- **databases** — a newer major version will not open an older data directory.
  This refuses rather than doing that quietly.
- **nodejs** — native addons are built against the ABI of the version that built
  them, and have to be rebuilt.

### Two packages, one command

The registry names the package a component would be changed through, which is not
always the package that is providing it. Measured here: `dpkg -S` on the `node`
binary answers `nodejs-lts`, while the component is `nodejs`. Offering an
"update" there would have installed a second Node over the first — apt calls that
a success, and the Node that was working stops working.

So `versions:use` asks `dpkg` who owns the command first, and refuses with the
reason, the two ways out, and the override flag:

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

### Before anything is installed

The cost is measured by asking apt to resolve the real dependency tree with
`--print-uris`, which prints the plan without downloading anything — so a Flutter
is reported as 117 MB to download and 326 MB on disk before a byte moves. Adding
up the named packages and ignoring their dependencies understates a database
stack by a wide margin, which is exactly the case where running out of space
matters.

---

## The tunnel to vscode.dev cannot be made here

Typing `code .` in a project folder asks for that folder to be open in VS Code.
The right way to do that is a tunnel: the workspace on the phone, reachable from
a browser anywhere, with no client to install. `code .` calls the tunnel binary
when it is there.

**On a phone the tunnel cannot be made, and the reason is worth stating plainly**
rather than leaving it to a spawn error:

- `code-oss` 1.131 is packaged for Termux, and its CLI runs. `--help`, `--diff`
  and `--merge` all work.
- `code-oss tunnel` and `code-oss serve-web` are both advertised in that CLI's own
  help. Both fail immediately and identically, with
  `spawn .../bin/code-tunnel-oss ENOENT`.
- The binary they call is not in the package, and not in the 104 MB bundle: the
  string `code-tunnel` appears zero times inside it. The tunnel *library* is
  there — `@microsoft/dev-tunnels-connections` — which is why the subcommand is
  offered at all. The code that would call the program is present; the program is
  not.
- Microsoft does not publish that binary separately. The API that serves VS Code
  builds carries no tunnel asset, because tunnels ship inside the desktop
  application.

So the decision is made from a measured absence, not a preference.
`Tunnel::tunnelBinary()` looks for the binary under all three names it goes by
(`code-tunnel-oss`, `code-tunnel`, `code-tunnel-insiders`) and uses it if any is
present — so if Termux ever packages it, `code .` starts using the tunnel with no
change here.

### What it does instead

The path that works, and it is not a consolation prize. This device already runs
an SSH server with publickey-only authentication, and a computer running VS Code
talking Remote-SSH to it is the same editing session: the files are on the
phone, the editor runs on the computer, the terminal is a real shell on the
device, and the traffic never touches anyone else's server.

```sh
code .                      # from inside a project
code ~/projects/myapp       # or name it
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

The key belongs on the computer, and the config says so: `IdentityFile` is the
computer's own key, never a path inside this device. A computer makes its key pair
and authorizes only the public half here —

```sh
ssh-keygen -t ed25519                                        # on the computer
devkit ssh:authorize "$(cat ~/.ssh/id_ed25519.pub)" "laptop"  # on the phone
```

A key generated on the phone and copied across leaves the phone holding a private
key it does not need, and the computer holding a public one that is not authorized
— and the login is refused with no visible reason.

`code:address` reports the *current* address, because a phone moves between
networks and a config written on one is wrong on the next. When something is
missing — no address, SSH not installed, not running, no key authorized — it says
which, rather than printing a config that will not connect.

## The browser editor does not run here

`devkit code:install` downloads the published `linux-arm64` release, unpacks it,
and runs `--version` successfully under `glibc-runner`. The download and the
unpack are both fine — 207 MB in, 627 MB unpacked. The editor still cannot run,
and the reason is structural:

- The release is glibc-only. Its bundled `lib/node` asks for
  `/lib/ld-linux-aarch64.so.1`, which does not exist on Android.
- `glibc-runner` starts such a program by naming the loader explicitly
  (`ld.so <program>`). That is why `--version` works.
- Once running, the editor re-executes its own binary for the extension host.
  That second process is not started by `glibc-runner`, so it has no loader to
  find, and exits immediately with code 127.
- Rewriting the binary's interpreter so the loader is found automatically was
  tried. It segfaults (exit 139) in every combination of `LD_LIBRARY_PATH` and
  `LD_PRELOAD` that was tried.

The panel says this rather than claiming success. `code:status` lists it as a
blocker, and the Editor tab asks for `code:open` — the route that works — rather
than for a start command that cannot succeed.

The route that would also work is `proot-distro`: a Debian userland with a real
`/lib/ld-linux-aarch64.so.1`, where the loader is where the binary expects it and
child processes inherit it. That is not wired up, and it costs a Debian rootfs
download.

## The device tab

Everything here comes from the device's own readings, and a reading that could not
be taken is **stated as unavailable with the reason** — never shown as `0`, which
would be indistinguishable from a real measurement.

Commonly unavailable on a stock Termux install: battery and thermal state (no
`termux-battery-status`, no readable `power_supply`), and `/proc/loadavg` and
`/proc/stat`, which Android blocks. The load average falls back to the `uptime`
command, and uptime to `/proc/uptime` or `uptime -p`.

## Certificates

```sh
devkit ca:create        # create the local certificate authority, once
devkit cert:issue-all   # issue a certificate for every registered site
devkit ca:info          # show the CA details
```

The authority is local, self-signed, under `ssl/ca` in the runtime directory.

**There are two different trust stores, and the panel reports them separately.**

| Store | Who writes it | What it affects | What the panel says |
|---|---|---|---|
| Termux's CA bundle (`$PREFIX/etc/tls/cert.pem`) | DevKit, on request | `curl`, Composer, PHP, `npm` inside Termux | **Termux: trusted / not trusted** — checked by reading the bundle and by a real TLS handshake |
| Android's user CA store | Android's Settings UI only | the browser | **Browser: unknown** — it cannot be read or written from Termux |

`ca:install` (the **Trust in Termux** button) only fills in the first row. It does
not and cannot affect the second, which is why a browser that has been told
"trusted" still shows a warning on HTTPS URLs. The button is named for what it
does rather than for what people hope it does.

The browser half is three taps, once, on the phone:

1. Download the CA from `http://127.0.0.1:8898/__panel/ca.crt`.
2. **Install certificate** → **VPN and apps**.
3. Screen lock, then close and reopen the browser.

The panel prints the CA's SHA-256 fingerprint next to those steps so you can
confirm the certificate Android installs is this one. Once you have done it, press
**I installed it in Android**; the panel then records that as an answer it was
told rather than one it verified, because it has no way to check:

```sh
devkit ca:browser-trust yes   # record it; also settable from the panel
devkit ca:browser-trust no    # withdraw it
```

Until then the browser shows a warning on HTTPS URLs. The sites still load, and
the plain-HTTP and by-IP URLs are unaffected.

## What the panel will not do

- **It will not run a command the allow-list does not name.** Adding a command to
  the allow-list is a deliberate edit to a file you can read.
- **It will not hand your GitHub token to the page.** The token goes to disk, and
  the page only learns whether there is one.
- **It will not accept a token from the page.** The GitHub token comes from
  GitHub in exchange for a code this device minted a verifier for.
- **It will not show a number it could not measure.** See the device tab.
- **It will not touch the device's network settings.** The `.app` proxy is set by
  you, in Android's Wi-Fi settings.
- **It will not start SSH for you.** It is the one service on the network, and it
  stays off until you turn it on.
- **It will not be a second shell.** The terminal is a real SSH session, so there
  is nothing in it that can disagree with a shell you open from a laptop.
