<div align="center">

# Contributing to IndoTech-DevKit

<a href="CONTRIBUTING.md">Bahasa Indonesia</a> · **[English](CONTRIBUTING.en.md)**

Thank you for wanting to help. This guide is short, and deliberately so: this
project does not need many contributors, it needs people who measure honestly.

</div>

---

## The rule that cannot be traded away

### 1. No claim that has not been measured

This is the only rule that is genuinely mandatory, and the rest follow from it.

If it concerns Termux, run it in Termux. If it concerns Android, test it on a
phone. If you cannot measure it, write the boundary down explicitly — in the
comment and in the commit message — rather than as a confident-sounding estimate.

There are real examples in this project's history:

- The claim "code-oss runs on Termux" turned out to be true of its CLI, and false
  for `tunnel` and `serve-web`, which both fail because the binary is not there.
  What the README says is the measurement, including that the string
  `code-tunnel` appears **zero times** inside the 104 MB bundle.
- The claim "the panel's allow-list agrees with the CLI" turned out to be beside
  the point: the editor button asked for `code:start`, which starts code-server,
  which cannot run on a phone. It now asks for `code:open`.

A good commit message explains **what was measured and what changed**, not just
which files were touched.

> **Avoid these words in comments and documentation**, unless they are literally
> what is happening: "appears to", "should", "probably", "likely", "normally",
> "will work".
>
> If you write "appears to work" and then measure it and it does work, write
> "works". If you did not get to measure it, write "not measured" and say how to
> measure it. Both are honest. The third is not.

### 2. Do not add a dependency without a written reason

Everything DevKit installs comes from Termux's own repositories. If you are
adding one:

- name the Termux package, not the nicer name on a website,
- say why the built-in package is not enough,
- check that the size and the dependency tree still fit on a small phone.

xterm.js is the only third party, and it is deliberate: it is vendored into the
repository rather than loaded from a CDN, so the panel works with no network and a
page that holds a shell does not execute a script from somewhere else.

### 3. Never close a socket in a forked PHP child

This is not a matter of style. It is how you kill a child process without any
trace.

PHP cannot do it:

- `fclose($fd)` with an int is a `TypeError`.
- `get_resources()` includes PHP-internal streams too, and closing one of those is
  an uncatchable engine fatal.

Three different approaches were written and measured here, and all three killed
the child silently: the pid came back, the log was empty, no process existed. So
the sweep was removed and the failures recorded in a comment, so nobody tries
them again.

The consequence is that `Detach` is only called from the `devkit` CLI, never from
a web request. An inherited socket then lives for the length of one command
rather than for the life of a service.

### 4. Commit messages

One change, one paragraph or more, written in English. The README and the
documentation are the exception: they are bilingual, Indonesian by default.

The shape this project uses is: **what happened, why it is like that, and what was
measured to go with it.**

> Add `devkit doctor`, and fix the three faults it turned up
>
> Verifying the stack end to end rather than reading it found three things that
> no existing command reported, because none of them was a service being down.
>
> A site added after the proxy started was never reachable by name. `site:add`
> wrote the vhost, issued the certificate, reloaded the web server and printed
> success, and the proxy's configuration — which is only rewritten when the proxy
> starts — still had no rule for the new name. …

What matters is the **why**, not only the what. "Fix a bug" is useful to nobody.

Do not add a `Co-Authored-By` trailer for any tool. This project is released by
one person, and that is what its history should show.

---

## Running it locally

```sh
git clone https://github.com/Ismail-IndoTech/IndoTech-DevKit.git
cd IndoTech-DevKit
./install.sh --dry-run     # ask everything, print the plan, install nothing
```

To try a change without disturbing a stack you are already using:

```sh
DEVKIT_HOME=~/devkit-sandbox ./install.sh --yes
```

`DEVKIT_HOME` moves all state, logs and certificates aside. That separation is
what makes it safe to experiment on a phone you also work on.

---

## Before you open a pull request

Run all of it. It is quick, and every one of these has caught something.

```sh
# Syntax
for f in $(find payload/devpanel tools -name '*.php') payload/bin/devkit; do php -l "$f"; done
for f in install.sh install-remote.sh lib/*.sh lib/steps/*.sh payload/bin/*; do bash -n "$f"; done
node --check payload/devpanel/public/assets/panel.js

# Consistency
python3 tools/check-panel-ids.py      # DOM ids in the panel: 0 missing, 0 duplicated
php tools/check-allowlist.php         # the panel's allow-list and the CLI must agree
python3 tools/check-prose.py          # documentation text that got mangled while being written

# The one that matters most: does it actually work?
devkit doctor
```

`devkit doctor` exits non-zero when something has failed, so it can be used in
CI. Warnings do not fail it, on purpose.

### If it touches the panel

Click every tab once. The panel is where people go to find out what is happening,
so anything shown there has to be honest:

- If something is not available, give the **reason**, not "not available".
- If a number cannot be read, say "unknown". Do not print `0`.
- If a command failed, show its error output.

---

## What is being worked on

Open an [issue](https://github.com/Ismail-IndoTech/IndoTech-DevKit/issues). What
usually helps:

- **A bug report** naming the device model, the Android version, and the output
  of `devkit doctor`. Those three lines are often worth more than a long
  description.
- **A measurement that proves or disproves a claim.** If you find that something
  does work on a phone, or does not, that is a genuinely valuable contribution —
  more valuable than code.
- **Documentation in Indonesian.** If a part is confusing, that part is the one
  that needs fixing.

What does not: feature requests with no reason behind them, or patches that add a
dependency.

---

## Donating

The project is free and never going to stop being free. If you want to help keep
it that way:

<div align="center">

**DANA — 08121600058** (a.n. Ismail)

<img src="docs/qris.jpg" alt="QRIS DANA" width="240">

</div>

It goes towards server costs, publishing, and the time to test on real hardware.
Nothing is promised in return.

---

## Licence

Contributions are licensed under the same [LICENSE](LICENSE) — MIT, in the name of
**Ismail.IndoTech**. By opening a pull request you agree that your contribution
is released under it.

---

<div align="center">

Made with :heart: by [**Ismail.IndoTech**](https://it-app.id) · [it-app.id](https://it-app.id)

</div>
