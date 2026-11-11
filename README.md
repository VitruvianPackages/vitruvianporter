# vitruvianporter

Builds V\OS packages from a [vitruvianports](https://github.com/VitruvianOS/vitruvianports)
tree. Every build runs in a fresh container of the V\OS build root: the
Debian base V\OS ships on, plus `vos-dev` and `dh-vos` from the V\OS apt
repository (`https://repo.v-os.dev`). With `--native` the machine it
runs on is the build root instead, for a throwaway VM.

## Requirements

- Python 3.7 or later (standard library only)
- git
- Docker, usable by the current user (`docker run` without sudo), or
  `--native` on a machine that has the build dependencies (root to
  install them)
- Network access to Debian and the V\OS apt repository

## Install

The tool is a single script. Put it on your `PATH`:

```
install -m 755 vitruvianporter ~/.local/bin/
```

or run it in place as `./vitruvianporter/vitruvianporter`.

## Usage

```
vitruvianporter [-C TREE] [--suite SUITE] [--arch ARCH] COMMAND ...
```

Global options, given before the command:

| Option | Meaning |
|---|---|
| `-C TREE` | ports tree (default: `$VITRUVIANPORTS`, else the current directory) |
| `--suite SUITE` | V\OS suite, see below (default `testing-nightly`) |
| `--arch ARCH` | architecture, e.g. `amd64` or `arm64` (default `amd64`); the container runs as `linux/ARCH` |
| `--native` | use this machine as the build root, without Docker (see below) |

Commands:

```
vitruvianporter bootstrap [--extra-deb DEB]...
vitruvianporter list
vitruvianporter check [--all] [PORT...]
vitruvianporter build [--all] [--out DIR] [--work DIR] [--keep] [PORT...]
```

- `bootstrap` builds the build root image for the chosen suite and
  architecture. `--extra-deb` (repeatable) installs a local `.deb` into
  it, for trying an unpublished `vos-dev` or `dh-vos`.
- `list` prints every port in the tree in build order, with its kind
  (`git` or `debian`) and version.
- `check` lints the named ports, or all of them with `--all`. It exits
  non-zero if any port has an error. Warnings do not fail it.
- `build` builds the named ports, or all with `--all`. Packages go to
  `--out` (default `./out`), logs to `--out/logs/<port>.log`. `--work`
  is the scratch directory (default `~/.cache/vitruvianporter/work`);
  `--keep` leaves each port's work tree in place.

```
cd vitruvianports
vitruvianporter bootstrap
vitruvianporter check --all
vitruvianporter build takenotes xcalc
vitruvianporter --suite testing build --all --out /srv/incoming
```

## Suites and build root images

| Suite | Debian base |
|---|---|
| `trixie` | trixie |
| `trixie-nightly` | trixie |
| `testing` | testing |
| `testing-nightly` | testing |

The suite selects both the Debian base image and the V\OS apt suite
installed into the build root. `bootstrap` creates the Docker image
`vitruvianporter/buildroot:<suite>-<arch>`; `build` uses the image
matching `--suite` and `--arch`, so bootstrap each combination you
build for. The image contains Debian's build tools, `vos-dev` and
`dh-vos`, with `/usr/lib/vos/buildtools` first on `PATH`.

## Native mode

`--native` runs `bootstrap` and `build` directly on the current
machine, with no Docker. It is meant for a disposable VM that already
runs the suite's Debian base and architecture; it refuses to
run when either does not match `--suite` and `--arch`.

`bootstrap --native` installs the build tools, the V\OS repository and
its key, `vos-dev` and `dh-vos` on the machine, so it needs root.
`build --native` runs each port's build in place and needs root only to
install: as root it installs each port's build dependencies and makes
the ports built so far available to the next ones; as a normal user it
installs nothing and stops on a port whose build dependencies are
missing. Installed build dependencies stay between ports, so the
isolation between builds is the VM, not the tool: start from a fresh VM
for each run.

```
vitruvianporter --suite trixie --native bootstrap
vitruvianporter -C vitruvianports --suite trixie --native build --all --out /srv/incoming
```

## Port layout

A port is a directory under `ports/` that holds a `port.yaml`. The
directory name is the port name and must be unique in the tree.
Directories below `ports/` are free to group ports (`apps`, `toolkits`,
...).

`port.yaml` takes `key: value` lines (`#` starts a comment). It uses
one of two forms.

Upstream source from git (`git` and `commit` required):

| Key | Meaning |
|---|---|
| `git` | `https://` URL of the upstream repository |
| `commit` | full 40-character commit sha |
| `subdir` | optional; package only this directory of the repository |

```
ports/<name>/
  port.yaml
  debian/        control, changelog, rules, copyright, source/format, ...
```

The orig tarball is `git archive` of the pinned commit (of
`commit:subdir` if set), and the port's `debian/` is added to it. Source
name and upstream version come from `debian/control` and
`debian/changelog`. Upstream patches live in `debian/patches/`
(quilt).

A patched Debian package (`debian` and `version` required):

| Key | Meaning |
|---|---|
| `debian` | Debian source package name |
| `version` | exact Debian version to fetch with `apt-get source` |

```
ports/<name>/
  port.yaml
  debian/changelog    one entry, the V\OS revision of the package
  patches/            patch files and a `series` file listing them in order
  rules.diff          optional, applied with `patch -p1` to the source tree
```

Debian's own `debian/` is used, with the port's patches added to
`debian/patches/series` and the changelog entry placed on top.

## What a build does

1. Fetches the pinned upstream commit into a local cache and makes the
   orig tarball (git ports), or runs `apt-get source` (Debian ports).
2. In the container: builds a local apt repository from `.deb` files
   already in `--out`, so packages built earlier in the same run satisfy
   build dependencies; installs build dependencies with `mk-build-deps`;
   runs `dpkg-buildpackage -us -uc -b`.
3. Copies the `.deb` files to `--out` and the log to `--out/logs`.

Ports are built in dependency order: a port whose build dependencies
name binary packages of another selected port is built after it. A
failed port does not stop the others; the exit status reports it.

## check

`check` verifies, per port:

- `port.yaml` is well formed and has a consistent key set
- git ports: `https://` URL, full commit sha, relative `subdir`
- required files exist (`control`, `changelog`, `rules`, `copyright`,
  `source/format`; Debian ports need only `changelog`)
- no wrapper scripts in `debian/` (outside `patches/`)
- no maintainer scripts restoring attributes (`dh-vos` does that)
- `rules` does not override `dh_shlibdeps`, `dh_strip` or `dh_dwz`

Warning: `rules` not running `dh` with `--with vos`.

## License

LGPL-3.0-or-later. See `COPYING.LESSER` and `COPYING`.
