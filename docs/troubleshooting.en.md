# Troubleshooting

<a href="troubleshooting.md">Bahasa Indonesia</a> · **[English](troubleshooting.en.md)**

## Start here

```sh
devkit doctor
```

This is the first command to run, and it is not the same question as
`service:status`. `service:status` answers *is it running*. `doctor` answers *does
it work* — each port is asked whether anything answers on it, each site whether
its URL returns a page, each certificate whether it still verifies, and each
failure carries the line to fix it.

```
  [1mPanel[0m
    ok    Token                      present, and removed when the panel stops
    ok    Reach it                   http://127.0.0.1:8898/__panel/
  [1mSites[0m
    ok    htdocs                     http://127.0.0.1:8080/ answers 200
```

For the device itself, before anything is installed:

```sh
./install.sh --device-info
```

It prints the device card, the caveats that apply to it, and the compatibility
table, and it changes nothing. When reporting a problem, its output is the first
thing worth including.

```sh
cat "$HOME/.devkit-answers"     # what the last run decided, and why
devkit env:info                 # what the installed stack is actually using
```

## A step refused something

If an option is greyed out, the reason is printed next to it. Those reasons come
from measurements, not from a table of folklore, so they are worth reading
literally:

| Message | What it means |
| --- | --- |
| `needs about 1200 MB of RAM; this device has 900 MB` | Real figures from `/proc/meminfo`. A database server that Android kills mid-task is worse than none. |
| `needs about 300 MB of free space to run; this device has 120 MB` | Floor for running the thing, separate from the download size. |
| `the 32-bit build of this toolchain is not part of Termux` | The package genuinely does not exist for `arm`. No version of this installer can offer it. |
| `Android 13 (API 21) is older than the Android 7 (API 24) minimum` | Below the floor. Termux itself does not support this. |

To choose a greyed-out option anyway, answer the question that precedes it with
`0` to abort, then re-run with a different answer. There is no override flag on
purpose: a deliberate override should look deliberate.

## "This plan does not fit on this device"

The real dependency tree, resolved by apt, needs more space than is free. Either
take something out of the plan or free up space and run it again. Nothing has
been installed when this appears.

To see the numbers yourself:

```sh
apt-get install --print-uris --no-install-recommends apache2 php mariadb
```

That prints the same plan without downloading anything.

## "Refusing to install over \<directory\>"

The runtime directory already contains files and is not a DevKit runtime, so it
is probably your own work. Nothing was moved, changed or deleted. Use a
different location:

```sh
DEVKIT_HOME="$HOME/stacks/dev" ./install.sh
```

An earlier DevKit runtime is recognised by its `bin/devkit` and *is* replaced,
with the old copy moved to `<directory>.backup.<timestamp>`. Only a directory
this installer put there is ever moved.

## A port is held but nothing answers on it

A listening socket belongs to the kernel until its last owner lets go of it, so a
process that inherited one keeps the port bound long after the process that owned
it is gone. From the outside the port looks occupied; from the inside, starting
anything on it fails.

**This used to be a permanent condition on this device, and it was a bug.** The
queue worker was started through a second, older implementation of "start this in
the background" that shelled out to `setsid ... &` and never closed the
descriptors it inherited. Started from the panel, it inherited the panel's own
listening socket — so after the panel stopped, port 8898 stayed bound by a worker
that served nothing, and `devkit service:start panel` could never succeed again.
The only thing on the device that could free it was `devkit service:stop queue`,
and only because that stopped the process holding it by accident.

Both faults are fixed. `Detach` now closes every inherited descriptor above the
standard three before it execs, and the queue worker is exec'd directly rather
than through a shell, so the pid recorded is the worker's own. Verified by
starting the queue from the panel, stopping the panel, and confirming the port
was free afterwards while the worker was still running.

So this message now means the app was killed without cleaning up, which Android
does on its own when it reclaims memory:

```sh
devkit doctor          # names the port, and says the listener does not answer
```

If it is really held, the release comes from the app, not from a command:

1. Android Settings → Apps → Termux → **Force stop**.
2. Reopen Termux.
3. `devkit service:start panel`

To find which process is holding it before resorting to that, stop the services
one at a time and watch the port:

```sh
devkit service:stop queue && php -r 'echo @stream_socket_server("tcp://127.0.0.1:8898") ? "free\n" : "held\n";'
```

If that frees it, the service you just stopped was the holder.

## The web server will not start

Termux ships Apache with a configuration that assumes it runs as root and owns
`/data`, and some packages — phpMyAdmin in particular — write directives from
Apache 2.2 that 2.4 refuses to load without a compatibility module. The
installer repairs both, and validates the configuration in place: it writes the
change, runs `httpd -t`, and puts the original back if the result is not
`Syntax OK`.

To check by hand:

```sh
httpd -t                      # must print: Syntax OK
tail -n 40 "$PREFIX/var/log/apache2/error_log" 2>/dev/null
```

If `httpd -t` passes and the server still does not answer, it is a port
conflict. Termux cannot bind below 1024, so a stock nginx listening on 8080
will collide with Apache on 8080. Only one web server can have a given port:

```sh
devkit service:status        # what is running, and the URLs each site answers on
```

## A site answers on 127.0.0.1 but its `.app` name is refused

This one was a real bug, and the symptom was measured rather than guessed.

`site:add` wrote the vhost, issued the certificate, reloaded the web server and
printed success — and the site was unreachable by name, because Squid's
configuration is only rewritten when the proxy starts. A name the device had just
registered was matching the rule that refuses names it has never heard of, and
every request came back `ERR_ACCESS_DENIED`.

```sh
devkit site:apply            # rewrites the vhosts *and* the proxy configuration
devkit proxy:doctor          # every address, with the status code it returns
```

`site:add`, `site:remove`, `site:enable` and `site:disable` all do this now, and
all of them print what the proxy did.

**HTTPS on a `.app` name has no route, and always did not.** A `CONNECT` cannot be
moved to another port, so `https://htdocs.app/` on 443 never reaches the device.
`proxy:doctor` says so rather than printing a link that cannot open. For HTTPS use
`localtest.me` or `sslip.io` with the port in the URL — neither needs a proxy.

## A `.app` hostname does not resolve at all

`localtest.me` needs nothing; `.app` needs the device's Wi-Fi proxy pointed at
`127.0.0.1:8899` and the proxy running. See [panel.en.md](panel.en.md).

## HTTPS shows a warning

Expected. The certificate comes from a local authority that Android does not
trust until you install it by hand, through
Settings → Security → Encryption & credentials → Install a certificate → CA
certificate. Termux cannot do that, and something that could would be a
dangerous thing to run. `devkit ca:info` prints the file to install.

`devkit doctor` reports this as two separate rows on purpose. "Termux: trusted"
and "Browser: unknown" are different states with one symptom, and reporting them
as one is how somebody spends an hour wondering why curl works and the browser
does not.

## A repository is reported unreachable

That is different from empty, and the difference matters. An unreachable
repository cannot be told apart from an empty one by looking at it, so the
installer says which one it found:

```sh
DEVKIT_NO_REPO_SCAN=1 ./install.sh     # skip probing, faster
```

Check one by hand:

```sh
curl -sSI "https://packages-cf.termux.dev/apt/termux-x11/dists/stable/Release" | head -1
```

## Something was installed but a version looks wrong

The plan's **Package sources** box explains this before anything happens, and
there are three distinct cases:

- `from main` — the repository offers exactly this version.
- `from termuxvoid (other version)` — a *different* version of this package is
  installed than the one the repository offers. `apt upgrade` will not change it.
- `not offered by any enabled repository` — the package is installed but no
  enabled repository has it. It came from somewhere that is no longer configured,
  or it was built by hand.

This is reported rather than hidden, because the third case is the one that
otherwise turns into a confusing afternoon.

## The installer stopped with "No more input to read"

The input ended before every question was answered, so it stopped rather than
guess. This happens when input is piped in:

```sh
printf '1\n1\n' | ./install.sh        # stops here
./install.sh < /dev/null               # same
```

Run it without piping and answer the questions, or use `--yes` for the
documented defaults. The alternative — treating end of input as an empty answer —
would have the menus redraw forever.

## SSH says "Login as home" or the connection is refused

The user name is the POSIX one — `u0_a247` on a device with no root — not the
name of the home directory. `basename($HOME)` is the literal string `home` here,
and `ssh home@…` is refused. Read the real one from:

```sh
devkit ssh:status
```

`ssh -p 8022 <user>@<address>` also has to be run **from your computer**: a phone
cannot reach itself over Wi-Fi, so the address in the connect line is the phone's
address on the network your computer is on. On the phone itself, use
`devkit job:run` or the terminal on the Projects page.

## SSH is listening but nobody can log in

That is the designed state, not a fault: with no key authorized, the port is
open and every login is refused. The panel says *"It is listening, but no key can
log in, so nobody is in yet."*

```sh
devkit ssh:keys                  # what is authorized
devkit ssh:authorize "$(cat ~/.ssh/laptop.pub)" "work laptop"
```

If a key is authorized and the login still fails, compare the fingerprints:

```sh
devkit ssh:status                # this device's host key
ssh-keygen -lf ~/.ssh/laptop.pub # the computer's key
```

## The terminal on the Projects page does not connect

It is an SSH session, so it needs `devkit ssh:start` first. `terminal:start`
says so rather than failing to connect:

```sh
devkit ssh:start
devkit terminal:start
devkit terminal:status       # the listener, the key, and its WebSocket URL
```

`devkit terminal:forget-key` revokes the panel's own key. The terminal then
refuses, which is correct: it is the same door a laptop comes through, and
revoking it is supposed to work.

If it connects and then closes straight away, look at the log:

```sh
tail -n 40 ~/IndoTech_Project/logs/terminal.log
```

## A job says it finished but nothing was created

Look at the output, not the status line:

```sh
devkit job:list
devkit job:log <id> 400
```

The status is the exit code; the reason is in the log. A `composer create-project`
that failed on a network hiccup exits non-zero with the error a few lines up.

## The panel says the runtime is older than it is

The panel is newer than the `devkit` it is talking to, or the payload was only
half copied. Re-run the installer, or copy `payload/` over the runtime:

```sh
cp -a "$PWD/payload/." ~/IndoTech_Project/
```

Then `devkit service:restart panel`.

## `code .` says the tunnel is not available

That is the accurate answer, not a failure to try. The binary `code-oss tunnel`
calls — `code-tunnel-oss` — is not packaged for Termux and is not published by
Microsoft as a standalone download; tunnels ship inside the desktop application.
Both `code-oss tunnel` and `code-oss serve-web` fail with `ENOENT` for that
reason.

`code .` therefore prints the Remote-SSH route, which reaches the same workspace
and needs nothing this device does not already run:

```sh
devkit code:address      # the address, and what is missing if anything is
devkit code:steps        # the full one-time setup
```

## Remote-SSH connects and then drops

A phone changes network. A config written on one Wi-Fi has the wrong address on
the next, and the connection is refused in a way that looks like a firewall.

```sh
devkit code:address      # re-read after changing networks
```

The config block printed by `code .` says to re-run this for exactly that reason.

If the address is right and it still fails, the key is the usual answer:

```sh
devkit ssh:keys                    # is the computer's key in the list?
devkit ssh:authorize "$(cat ~/.ssh/id_ed25519.pub)" "laptop"
```

The key must be made **on the computer**. A key generated on the phone and copied
across gives the phone a private key it does not need and the computer a public
one that is not authorized — the login is refused with no clue as to why.

## The editor tab says it cannot run

That is the truth, not a bug. The published release is glibc-only, and Android
has no glibc loader; the extension host re-executes itself into a process that
cannot find one. The route that works is `proot-distro`, which is not wired up
yet. `code .` gives you the Remote-SSH path instead, which does work. See
[panel.en.md](panel.en.md#the-browser-editor-does-not-run-here).
