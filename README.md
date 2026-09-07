# hifiberry-shairport

Debian packaging for the AirPlay player in HiFiBerryOS. The package builds
[Shairport Sync](https://github.com/mikebrady/shairport-sync) and
[NQPTP](https://github.com/mikebrady/nqptp) from pinned upstream sources and
wires them into the HiFiBerry player registry as an installable extension.

Nothing here is upstream source. The build clones the upstream repositories at
the exact commits named in `debian/rules`; this repository holds only the
packaging, the start and event scripts, the systemd units and the player
descriptors.

## Naming

Three separate naming schemes meet in this repository, and they are easy to
confuse with one another.

### Package version

`<upstream>.<hifiberry-patch>`, with an epoch of `1:` — upstream takes the
first three digits, our packaging revision is the fourth:

```
1:5.5.0.1   -> upstream shairport-sync 5.5, packaging revision 1
1:5.2.3.2   -> upstream shairport-sync 5.2.3, packaging revision 2
```

An upstream tag with only two components is padded to three: upstream `5.5` is
`5.5.0` in the package version, so its first packaging revision is `1:5.5.0.1`.
The fourth digit exists so our rebuilds never collide with an upstream patch
release, and the `1:` epoch must never be dropped.

**[`VERSIONING.md`](VERSIONING.md) is the authority** — it explains why the
fourth digit is there, why the epoch was introduced, and what to do when
upstream moves. Read it before choosing a version.

The version lives in exactly one place: the top entry of `debian/changelog`.
`build-deb.sh` parses it from there, so nothing else needs editing.

### Upstream binaries

Two builds of shairport-sync are compiled from the same source tree and
installed under different names, because they differ only in configure flags:

| Installed as | Build | Purpose |
|---|---|---|
| `/usr/bin/shairport-airplay2` | `--with-airplay-2` | AirPlay 2 receiver, needs `nqptp` |
| `/usr/bin/shairport` | no AirPlay 2 | classic AirPlay 1 receiver |
| `/usr/bin/nqptp` | NQPTP | PTP clock daemon, AirPlay 2 only |
| `/usr/bin/shairport-sync-metadata-reader` | metadata reader | decodes the metadata pipe |

The `airplay_version` setting in the WebUI descriptor selects which of the two
receivers `start-shairport` runs. Upstream calls both binaries
`shairport-sync`; that name is not installed by this package, though the
package does `Provides`/`Conflicts`/`Replaces` the distribution's
`shairport-sync` and `nqptp` packages.

### Player descriptors

Two descriptors with the same base name, different schemas, different readers:

| Path | Read by |
|---|---|
| `/etc/audiocontrol/players.d/shairport.json` | audiocontrol |
| `/usr/share/hifiberry/players.d/shairport.json` | the WebUI |

The WebUI descriptor is package data, not configuration, which is why it ships
from `/usr/share` rather than `/etc` — configurator lets `/etc` shadow
`/usr/share`, so a stale copy under `/etc` silently masks the packaged one.
See the `1:5.2.1.6` and `1:5.2.3.2` changelog entries for the full history.

## Changelog style

`debian/changelog` entries use the standard Debian layout, and the file is
consistent throughout — keep it that way:

```
hifiberry-shairport (1:5.5.0.1) stable; urgency=medium

  * Two spaces, an asterisk, one space. Continuation lines are indented by
    four spaces.
    - Sub-items are indented by four spaces and continue at six.

 -- Name <address>  Mon, 07 Sep 2026 21:12:24 +1000
```

Wrap at roughly 78 columns, zero-pad the day in the trailer date, and leave no
trailing whitespace.

## Building

Builds run under `sbuild`; the container route is the only supported one on
macOS. See [`packages/build.md`](../hifiberryos/packages/build.md) in the
hifiberry-os repository for the toolchain.

```sh
./build-deb.sh
```

The build clones upstream over the network, so `--enable-network` is set and
the build is not reproducible offline.

## Bumping upstream shairport-sync

1. Find the upstream tag and resolve it to a commit:
   `git rev-list -n 1 <tag>` in a clone of `mikebrady/shairport-sync`.
2. Set `SHAIRPORT_VERSION` and `SHAIRPORT_COMMIT` in `debian/rules`. Pin the
   commit, never the tag alone — tags can move.
3. Check whether NQPTP still matches: shairport-sync refuses to start against
   an NQPTP with a different `NQPTP_SHM_STRUCTURES_VERSION`. Compare
   `nqptp-shm-structures.h` in both trees and bump the `nqptp` checkout in
   `debian/rules` if they differ.
4. Check that every `./configure` flag in `debian/rules` still exists in the
   new upstream `configure.ac`, and that the shipped
   `shairport-sync.conf.default` still matches upstream's options.
5. Add a `debian/changelog` entry with the version from
   [`VERSIONING.md`](VERSIONING.md), summarising the upstream changes.

## Maintainers

See [`MAINTAINERS.md`](MAINTAINERS.md).
