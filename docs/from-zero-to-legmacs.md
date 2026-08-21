# From Linux to Legmacs on 9front

This guide starts with a Linux machine, creates a 9front virtual machine, and
installs [Legmacs](https://github.com/nooga/legmacs) with Get9. The commands are
deliberately direct and assume some familiarity with a Linux terminal.

The host setup below targets a recent Ubuntu system with a Wayland desktop.
The Plan 9 and Get9 steps are the same on other Linux distributions, but their
host package names may differ.

## 1. Install uv

In a Linux terminal, install the packages needed to download uv:

```sh
sudo apt update
sudo apt install ca-certificates curl
```

Install [uv](https://docs.astral.sh/uv/getting-started/installation/):

```sh
curl -LsSf https://astral.sh/uv/install.sh | sh
```

The command uses uv's official installer. To inspect it before running
it, use `curl -LsSf https://astral.sh/uv/install.sh | less`.

Close this terminal and open a new one before continuing. In the new terminal,
verify that uv is available:

```sh
uv --version
```

## 2. Install P9QEMU

Install Git and QEMU:

```sh
sudo apt install git qemu-system-x86 qemu-utils
```

Install [P9QEMU](https://github.com/dharmatech/p9qemu):

```sh
uv tool install git+https://github.com/dharmatech/p9qemu.git
```

Confirm that P9QEMU is available:

```sh
command -v p9qemu
```

## 3. Install Drawterm

Install Git and the packages needed to build the Wayland version of Drawterm:

```sh
sudo apt install \
    build-essential git pkg-config \
    libdecor-0-dev libpipewire-0.3-dev libwayland-dev libxkbcommon-dev
```

Build and install the Wayland version of
[Drawterm](https://github.com/dharmatech/drawterm):

```sh
mkdir -p "$HOME/src"
git clone https://github.com/dharmatech/drawterm.git "$HOME/src/drawterm"
(
    cd "$HOME/src/drawterm"
    make CONF=linux clean
    make CONF=linux
    make CONF=linux install
)
```

Confirm that Drawterm is available:

```sh
command -v drawterm
```

## 4. Create the 9front baseline and working VM

Still on Linux, create an untouched baseline from P9QEMU's stable,
Drawterm-ready 9front image, then copy it to the working `dev` instance:

```sh
mkdir -p "$HOME/vm/legmacs"
cd "$HOME/vm/legmacs"
p9qemu image create \
    https://github.com/dharmatech/p9qemu/releases/download/ready-9front-11554-amd64-hjfs-gmt-drawterm-001/image.json \
    checkpoint-000
cp -a --sparse=always checkpoint-000 dev
```

The image is downloaded and verified, and both directories remain small
writable overlays backed by P9QEMU's immutable cached image. Treat
`checkpoint-000` as frozen: never start it or modify it. Start only `dev`.

To return to the original state or begin another experiment later, copy the
checkpoint to another new working directory and start that copy:

```sh
cd "$HOME/vm/legmacs"
cp -a --sparse=always checkpoint-000 experiment-001
```

Never copy an instance while it is running. Shut down its guest with `fshalt`
and wait for QEMU to exit first.

## 5. Start 9front

Start the VM on an unused loopback address:

```sh
cd "$HOME/vm/legmacs"
p9qemu start --instance dev --host-forward-address 127.0.0.40
```

Leave this terminal running. P9QEMU prints the complete QEMU command and the VM
boots without requiring input. If `127.0.0.40` is already in use, choose
another address in `127.0.0.0/8` and use it in both the P9QEMU and Drawterm
commands.

## 6. Connect with Drawterm

Open a second Linux terminal and run:

```sh
ip=127.0.0.40; PASS=p9qemu-demo drawterm -h $ip -a $ip -u glenda -c 'rio'
```

The ready image intentionally uses the public demonstration password
`p9qemu-demo`. P9QEMU exposes it only on the selected loopback address. Do not
expose this VM to another machine without changing the password first.

Do not add Drawterm's `-m` option. Get9's Legmacs launcher enables VT-Alt's own
Alt-as-Meta mode, which needs the original left-Alt events from Drawterm.

## 7. Install Get9 and Legmacs

The remaining commands run inside the Plan 9 terminal in Drawterm.

Clone Get9 without changing the current shell's working directory:

```rc
@{mkdir -p $home/src && cd $home/src && webfs && git/clone https://github.com/dharmatech/get9.git}
```

Enter the repository and run the five recipes in dependency order:

```rc
cd $home/src/get9
ports/go/1.27.0/binary/amd64/install.rc
ports/ca-certificates/2026-08-13/user/install.rc
ports/let-go/main/source/amd64/install.rc
ports/vt-alt/main/source/amd64/install.rc
ports/legmacs/main/source/amd64/install.rc
```

Go is installed from its official Plan 9 binary archive. LetGo, VT-Alt, and
Legmacs are built or installed from their current upstream `main` branches;
Get9 records the exact commit selected for each installation.

## 8. Reconnect and start Legmacs

Open another Drawterm connection from Linux by running the same command again:

```sh
ip=127.0.0.40; PASS=p9qemu-demo drawterm -h $ip -a $ip -u glenda -c 'rio'
```

The new connection reads the profile written by Get9 before Rio starts. In its
Plan 9 terminal, optionally check the installed runtime and then start Legmacs:

```rc
go version
lg -e '(+ 1 1)'
legmacs
```

To open a file directly:

```rc
legmacs $home/lib/profile
```

Legmacs uses left Alt for Meta commands through VT-Alt. Upstream Legmacs
currently implements `M-!` by invoking POSIX `sh`, so that one command is not
available on Plan 9 yet.

## 9. Shut down and return later

Shut down cleanly from a Plan 9 terminal:

```rc
fshalt
```

Drawterm disconnects and the P9QEMU process exits. To return later, repeat the
P9QEMU start command and then the Drawterm command. The installed Get9 packages
remain in the `dev` instance.

For details about the installation layout and individual recipes, see the
[Get9 README](../README.md). For image provenance and host alternatives, see
the [P9QEMU Drawterm image
page](https://github.com/dharmatech/p9qemu/tree/main/images/p9qemu-9front-11554-amd64-hjfs-gmt-drawterm-001).
