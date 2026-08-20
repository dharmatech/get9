# get9

get9 is an experimental ports tree for Plan 9 and 9front. Each port may offer
more than one version and more than one installation recipe—for example, an
upstream binary archive, a source build, or a Plan 9-specific layout.

The repository is the initial interface. There is no package database,
dependency solver, or `get9` front-end yet.

## Get the ports tree

This copy-and-paste command clones get9 without changing the current shell's
working directory:

```rc
@{mkdir -p $home/src && cd $home/src && webfs && git/clone https://github.com/dharmatech/get9.git}
```

Recipes can then be run from the repository root:

```rc
cd $home/src/get9
```

## Installation model

By default, recipes use this user-local layout:

```text
$home/lib/get9/
    profile
    active/
        ca-certificates.rc
        go.rc
        let-go.rc
    cache/
    distfiles/
    pkg/
        ca-certificates/
        go/
        let-go/
    tmp/
```

Installations are versioned under `pkg`. Selected packages contribute small
activation files under `active`; the generated `profile` sources them. Get9
adds one idempotent hook at the beginning of `$home/lib/profile`, before Rio
can start:

```rc
# get9: profile hook
if(test -r $home/lib/get9/profile)
	. $home/lib/get9/profile
# get9: end profile hook
```

The first time it changes an existing user profile, Get9 preserves the original
as `$home/lib/profile.pre-get9`. Installing another package does not add another
block to the user profile.

Set `get9_root` before running a recipe to choose another package, cache, and
temporary-file root. Get9's small activation and profile files remain under
`$home/lib/get9`:

```rc
get9_root=$home/opt/get9
ports/go/1.27.0/binary/amd64/install.rc
```

## CA certificates 2026-08-13

The CA certificates port installs curl's dated Mozilla CA bundle for Go's
`crypto/x509` package:

```rc
ports/ca-certificates/2026-08-13/user/install.rc
```

The bundle is installed user-locally and selected through `SSL_CERT_FILE`; this
recipe does not overwrite the machine-wide `/sys/lib/tls/ca.pem`. The dated
download is pinned to its published SHA-256 checksum, and a cached copy is
verified every time it is used.

Temporarily activate or deactivate this bundle in one current `rc` process:

```rc
. ports/ca-certificates/2026-08-13/user/activate.rc
. ports/ca-certificates/2026-08-13/user/deactivate.rc
```

## Go 1.27.0

The Go recipe installs the official Go 1.27.0 Plan 9/amd64 binary archive:

```rc
ports/go/1.27.0/binary/amd64/install.rc
```

The archive is pinned to its official SHA-256 checksum. A cached archive is
verified every time it is used. The installer refuses to overwrite an existing
incomplete installation.

Temporarily activate or deactivate Go in one current `rc` namespace:

```rc
. ports/go/1.27.0/binary/amd64/activate.rc
go version
. ports/go/1.27.0/binary/amd64/deactivate.rc
```

Activation sets `GOROOT` and binds that version's `bin` directory before
`/bin`. It does not copy files into the system `/bin` directory.

## LetGo from the latest main branch

The LetGo source recipe clones the current `main` branch, resolves its exact
commit, and builds `lg` with Go:

```rc
ports/let-go/main/source/amd64/install.rc
```

Its build prerequisites are:

- Go 1.26 or newer; and
- a readable CA certificate bundle for Go module downloads.

The recipe checks both prerequisites and reports the recipe needed when one is
missing. It sources the managed Get9 profile inside its own installer namespace,
so Go and CA certificates installed earlier in the same shell can be used
without reconnecting first.

Each source build is installed under its full upstream commit ID:

```text
$home/lib/get9/pkg/let-go/<commit>/
    bin/lg
    source-commit
    source-url
```

Running the recipe later checks the then-current `main` commit. An existing
commit is reused; a new commit is built and installed beside older builds. The
newly resolved commit becomes the persistent selection.

Temporarily activate or deactivate the selected LetGo build in one current
namespace:

```rc
. ports/let-go/main/source/amd64/activate.rc
lg -e '(+ 1 1)'
. ports/let-go/main/source/amd64/deactivate.rc
```

## Persistent activation

Reconnect with Drawterm after installation. The new connection reads the user
profile before Rio starts, so selected packages are available to Rio and the
windows it creates:

```rc
go version
lg -e '(+ 1 1)'
```

For convenience, all selected packages can be activated in one current window:

```rc
. $home/lib/get9/profile
```

That does not update an already-running Rio's original namespace. Ordinary new
windows created by that Rio may retain the old view; reconnecting with Drawterm
is the reliable activation boundary.

## Current boundaries

- Recipes are ordinary, directly executable, inspectable `rc` scripts.
- Installations are user-local and versioned unless a recipe explicitly says
  otherwise.
- Dependencies are checked by recipes and reported with actionable commands;
  there is no automatic dependency solver yet.
- Supported recipes currently cover CA certificates 2026-08-13, the official
  Go 1.27.0 Plan 9/amd64 archive, and LetGo `main` source builds on amd64.
- There is no automated uninstall command yet. Removal requires deactivating a
  package, removing its persistent selection, and removing its versioned package
  directory. The shared profile hook may remain for other Get9 packages.
