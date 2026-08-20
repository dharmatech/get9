# get9

get9 is an experimental ports tree for Plan 9 and 9front. Each port may offer
more than one version and more than one installation recipe—for example, an
upstream binary archive, a source build, or a Plan 9-specific layout.

The repository is the initial interface. There is no package database,
dependency solver, or `get9` front-end yet.

## First port: Go 1.27.0

The first recipe installs the official Go 1.27.0 Plan 9/amd64 binary archive:

```rc
cd get9
ports/go/1.27.0/binary/amd64/install.rc
```

By default, it creates this user-local layout:

```text
$home/lib/get9/
    profile
    active/
        go.rc
    distfiles/
        go1.27.0.plan9-amd64.tar.gz
    pkg/
        go/
            1.27.0/
                bin/
                src/
                ...
    tmp/
```

The archive is pinned to its official SHA-256 checksum. A cached archive is
verified every time it is used. The installer refuses to overwrite an existing
incomplete installation.

Installation also selects Go 1.27.0 for future sessions. Get9 adds one
idempotent hook at the beginning of `$home/lib/profile`, before Rio can start:

```rc
# get9: profile hook
if(test -r $home/lib/get9/profile)
	. $home/lib/get9/profile
# get9: end profile hook
```

The first time it changes an existing user profile, Get9 preserves the original
as `$home/lib/profile.pre-get9`. Package selections remain in Get9-owned files;
installing another package does not add another block to the user profile.

Set `get9_root` before running the recipe to choose another installation root:

```rc
get9_root=$home/opt/get9
ports/go/1.27.0/binary/amd64/install.rc
```

## Start using Go

Reconnect with Drawterm after installation. The new connection reads the user
profile before Rio starts, so Go is available to Rio and the windows it creates:

```rc
go version
gofmt -h
```

For convenience, the managed profile can be dot-sourced in one current window:

```rc
. $home/lib/get9/profile
```

That does not update an already-running Rio's original namespace. Ordinary new
windows created by that Rio may therefore retain the old view; reconnecting
with Drawterm is the reliable activation boundary.

## Current-window activation and deactivation

The recipe also provides scripts for temporarily changing one current `rc`
namespace:

```rc
. ./ports/go/1.27.0/binary/amd64/activate.rc
go version
```

It sets `GOROOT` and binds that version's `bin` directory before `/bin`. The
binding is visible to the current namespace group and processes created from
it; it does not copy files into the system `/bin` directory.

Undo the get9-managed Go binding with:

```rc
. ./ports/go/1.27.0/binary/amd64/deactivate.rc
```

These two scripts do not change the persistent selection under
`$home/lib/get9/active`.

Later versions can use the same layout. Dot-sourcing another version's
activation script will remove the previous get9-managed Go binding before
binding the selected version.

## Current boundaries

- Recipes are ordinary, inspectable `rc` scripts.
- Installations are user-local and versioned unless a recipe explicitly says
  otherwise.
- Installation selects the installed Go version persistently; a new Drawterm
  connection is the supported activation boundary.
- This prototype supports only the official Go 1.27.0 Plan 9/amd64 archive.
- There is no automated uninstall command yet. Removing this version also
  requires removing its persistent `active/go.rc` selection; the shared profile
  hook may remain for other Get9 packages.
