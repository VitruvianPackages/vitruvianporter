# vitruvianporter

Builds V\OS packages from a [vitruvianports](https://github.com/VitruvianOS/vitruvianports)
tree. Every build runs in a fresh container of the V\OS build root: the
Debian base V\OS ships on, plus `vos-dev` and `dh-vos` from the V\OS apt
repository.

## Requirements

Python 3, git and Docker (the user must be able to run `docker`).

## Use

```
cd vitruvianports
vitruvianporter bootstrap                 # build root for testing-nightly/amd64
vitruvianporter list                      # ports, in build order
vitruvianporter check --all               # packaging rules
vitruvianporter build takenotes xcalc     # .debs and logs in ./out
vitruvianporter build --all --out /srv/incoming
```

`--suite` picks the V\OS suite (`testing-nightly`, `testing`,
`trixie-nightly`, `trixie`) and `--arch` the architecture. `-C <tree>` or
`$VITRUVIANPORTS` points at the ports tree.

`bootstrap --extra-deb <file>` installs a local package into the build
root, for testing changes to `vos-dev` or `dh-vos` before they are
published.

## What a build does

1. Fetches the pinned upstream commit, makes the orig tarball with
   `git archive`, and adds the port's `debian/`. A patched Debian package
   is fetched with `apt-get source` and gets the port's patches on top.
2. Installs the build dependencies, including packages built earlier in
   the same run, and runs `dpkg-buildpackage -b`.
3. Copies the `.deb` files to `--out` and the log to `--out/logs`.

Ports are built in dependency order. A failed port does not stop the
others; the exit status reports it.
