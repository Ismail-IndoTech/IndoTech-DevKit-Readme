# How it works

<a href="how-it-works.md">Bahasa Indonesia</a> · **[English](how-it-works.en.md)**

Four layers, each answering one question. Keeping them apart is what lets the
installer say things like "this version exists in this repository, on this
device" without any of it being hardcoded.

```
lib/detect.sh   what is this device?           (facts about the machine)
lib/device.sh   what can this device do?       (judgements about those facts)
lib/catalog.sh  what exists in the repositories right now?
lib/core.sh     what did the user answer, and where does it go?
```

`install.sh` owns the order, the plan, and the run.

## 1. Device detection — `lib/detect.sh` and `lib/device.sh`

`detect.sh` measures. `device.sh` judges.

`device.sh` does not measure the same things twice. Root, architecture, Android
release and free space already have detectors, and those are the ones the mode
step and the preflight trust; reusing them means two answers can never disagree
about the same fact. What `device.sh` adds is the signals nobody was looking
for:

| Signal | Read from | Why it matters |
| --- | --- | --- |
| `DEVICE_API` | `ro.build.version.sdk` | Below 24 nothing can be installed at all. |
| `DEVICE_RAM_MB` | `/proc/meminfo` | Decides whether a database server or a desktop stack is realistic. |
| `DEVICE_PAGE_SIZE` | GCD of `/proc/self/maps` start addresses | 16 KB page devices break prebuilt 4 KB binaries. |
| `DEVICE_CORES` | `/proc/cpuinfo` | `nproc` only reports the cores this process may use, and Android narrows that per app. |
| `DEVICE_FLAVOR` | `ro.product.build.flavor` | An F-Droid build is signed with a different key, so `pkg upgrade` will not manage third-party packages. |
| `DEVICE_BOOTLOADER` | `ro.boot.verifiedbootstate` | Explains why anything needing a modified system is out of reach. |
| `DEVICE_SELINUX` | `getenforce` | Reported as unknown when Termux cannot read it, never guessed. |

### Page size without `getconf`

The obvious way to get the memory page size is `getconf PAGESIZE`, but `getconf`
is not present in every Termux install, and pulling in Python to substitute for
it is absurd. Instead: `/proc/self/maps` only ever starts a mapping on a page
boundary, so the page size is the largest power of two that divides every
address in the file. The GCD of those addresses is exactly that:

```sh
awk '
    function gcd(a, b,   t) { while (b) { t = a % b; a = b; b = t } return a }
    { split($1, range, "-"); v = strtonum("0x" range[1]); if (v > 0) g = (g == 0) ? v : gcd(g, v) }
    END { if (g > 0) print g }
' /proc/self/maps
```

4096 on a normal device, 16384 on a 16 KB one. `getconf` is used only as a
fallback if that produces nothing — and it is genuinely missing on the phone this
was written on, so the fallback is a convenience rather than a plan.

This matters more than it sounds. A binary compiled for 4 KB pages can fail on a
16 KB page device in ways that look like corruption rather than a page size
mismatch, and the failure arrives after the package is installed and running.

### The judgements

`device_component_blocker <component>` is the single place that decides whether
something may be used. It returns empty when the answer is yes, and a sentence
when the answer is no. Every menu reads the sentence and greys the option out, so
the same rule cannot be enforced in one menu and forgotten in another.

Floors are deliberately conservative. A database server that Android's
low-memory killer terminates halfway through your work is worse than no database
server, so `mariadb` needs 1200 MB even though it will start happily on less.

## 2. The package catalog — `lib/catalog.sh`

Every version shown in every menu comes from the device's own apt metadata.

`apt-cache policy <pkg>` gives the candidate version. `apt-cache madison <pkg>`
gives the full version list **and the repository each version came from**, which
is where the provenance in the plan summary comes from:

```
php | 8.5.1,8.5.0 | https://termux.niranjan.co/termux-main Packages
```

Field 3's first word is the repository URI. The rest is `suite component arch`
followed by the index filename.

`pkg_origin_note` distinguishes three cases that are easy to conflate, and that
matter a great deal in practice:

- **The exact version is in a repository** — the normal case.
- **A *different* version is offered** — you have something installed that the
  repository does not have. `apt upgrade` will not touch it, and if you are on a
  third-party repository this is the interesting case: `ripgrep` shows as
  "from termuxvoid (other version)" because 0.14.1 is installed and the
  repository offers 0.13.0.
- **Installed but offered nowhere** — the package came from a repository that is
  no longer enabled, or was built by hand.

The third case is the one most installers hide. It is reported plainly.

## 3. The repository layer — `lib/catalog.sh`

Termux does not put everything in one place, and the other places are not always
up. The installer has an address book of known repositories and, for each one
that is not enabled, decides whether it is worth offering.

### Probing without guessing URLs

The tempting approach is to construct the `Packages` index URL from the
repository URI and fetch it. It does not work: mirrors change, several
distributions are no longer served, compression varies, and a URL that 404s
looks exactly like a repository with no packages. Both failure modes lead to
offering the user something that cannot work.

Instead the installer runs **apt itself**, in a sandbox, against a throwaway
configuration:

```sh
apt-get update \
  -o Dir::Etc::sourcelist="$probe_dir/sources.list" \
  -o Dir::Etc::sourceparts=/dev/null \
  -o Dir::State::lists="$probe_dir/lists" \
  -o APT::Get::List-Cleanup=0
```

apt resolves the index path, the compression and the architecture by itself. The
device's own apt configuration is never read or written, and the results are
cached per run.

This is why a dead repository is reported as **unreachable** rather than empty.
`tur` and `unstable-packages` in the address book both return HTTP 404 on this
device, and the installer says exactly that instead of pretending they are
worth adding.

### Adding one

`repo_enable` writes a `sources.list.d` entry, runs `apt-get update` against the
real configuration, and then invalidates the catalog cache — because the package
list just changed, and every menu downstream is built from it.

## 4. The storage gate

Adding up the size of the packages you ticked and ignoring their dependencies
understates a PHP or database stack by a wide margin — which is precisely the
case where running out of space matters.

So the installer asks apt to resolve the real dependency tree and print the
plan, without downloading anything:

```sh
apt-get install --print-uris --no-install-recommends <packages>
```

```
Need to get 28.9 MB of archives.
After this operation, 99.5 MB of additional disk space will be used.
```

Both numbers are parsed by scanning for a value followed by a byte unit, rather
than by field position, because apt's wording is not a contract. A positional
parse returns 0 when the wording changes, and 0 is the one number that looks
like a valid answer.

There are two thresholds, because there are two different failures:

- **Free space below what the plan needs** — refused. It cannot work.
- **Free space below the plan plus half again** — allowed, with the number
  stated. apt also needs room for its own lists, for unpacked sources it builds,
  and for logs, and the person holding the phone knows their phone better than
  the installer does.

## 5. Answers

Answers are persisted to `$HOME/.devkit-answers` as `key=value` lines, so a
re-run can be inspected, edited, or replayed. `--answers=FILE` moves the file;
`--yes` never prompts.

The answer file is also treated as untrusted input: `build_plan` re-checks every
package in the plan against the device, because the file can be edited by hand
and because a file written on one phone can be replayed on another. A package
that cannot be used here is named, with the reason, rather than left to fail
inside apt.

## 6. Interactive behaviour

A menu redraws until it gets an answer. That is fine on a terminal and fatal on a
pipe: `printf '1\n' | ./install.sh` runs out of input, and a menu that treats
"nothing left to read" as "the user pressed enter" redraws forever at full
speed — a hot battery and no output anyone can interrupt.

So `_read` distinguishes the two, and every looping caller checks:

- **End of input** stops the installer with a message. It does not guess.
- **No terminal at all** degrades to documented defaults, which is what `--yes`
  and CI use.

Related: in a multi-select, `0` aborts only as a token of its own. Matching the
digit anywhere in the line would turn `10` — the tenth option — into an abort,
and the development tools menu has seventeen entries.
