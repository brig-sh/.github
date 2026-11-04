<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/brig-sh/brig/main/assets/brig-lockup-on-dark.svg">
    <img alt="brig" src="https://raw.githubusercontent.com/brig-sh/brig/main/assets/brig-lockup-on-light.svg" width="280">
  </picture>
</p>

<p align="center"><strong>Run a coding agent in a microVM on your own machine, with the credentials it needs and none of the ones it does not.</strong></p>

<p align="center"><a href="https://brig.sh">brig.sh</a> · <a href="https://brig.sh/docs/">documentation</a></p>

---

## Install

On macOS, with Homebrew, which brings hull and cosign with it:

```bash
brew tap brig-sh/brig
brew trust brig-sh/brig      # brew refuses untrusted third-party taps
brew install --cask brig

brig doctor
brig run claude ~/code/demo
```

Or with the install script, on macOS or Linux:

```bash
curl -fsSL https://brig.sh/install | sh
```

On Linux the script installs the runtime too: containerd, nerdctl, urunc, the
`hvi` monitor it boots on by default, and the guest kernel. Under `sudo` it
installs for the whole host. As a normal user it installs a rootless runtime
under `$HOME`, once root has prepared the host.
[Install](https://brig.sh/docs/install/) has the details.

`brig doctor` checks the host and prints a fix beside each problem it finds.
The run mounts `~/code/demo` read-write at `/work/demo` and starts the agent
there. brig prints `brig: image and boot assets verified` before the boot. The
sandbox has no credential yet, so the agent asks you to log in just as it does
on a new machine.

## Credentials

To reuse the login already on your host, carry it in once:

```bash
brig secret import claude-code
```

That reads your host login once, when you run it, and copies it into brig's
own store: brig's items in the login keychain on macOS, and Secret Service on
Linux. Every run afterwards takes the login from that store. A run decrypts no
keychain item brig did not write, and reads no SSH agent or secret manager.

The profile decides how a credential reaches the guest. Claude Code gets a file
at the path it already reads, on a memory-backed mount, so that file never
lands on host disk. Other credentials travel as environment variables, for
agents that read no credential file. A profile can also bind a variable from
the shell you run brig in, as `claude-code` does with `GH_TOKEN`.

A policy with a route keeps the login on the host instead. `brig-broker` runs
on the host and adds the credential to each request the route allows. The guest
gets the route URL and a session token, which `brig stop` revokes, and no
credential file. Routes run on hull's `hvi` backend and on nerdctl with urunc.
[Credential broker](https://brig.sh/docs/broker/) has the details.

## How it fits together

The agent runs in a guest with its own kernel. Two host directories reach
inside it: a guest home of its own, and the one project you name on the command
line. brig creates the guest home under `~/.brig/homes/`, and `brig rm` deletes
it. A shipped profile mounts nothing else.

That project mount is read-write, so an agent can change the project you handed
it. What it cannot see is the rest of your disk.

Every sandbox runs under one of three network postures. `shared` is the
default: the agent reaches the internet, so anything it can read it can also
send. Sandboxes on `shared` also reach each other on hull's `hvi` backend, and
should be assumed to on Linux. On `hvi` and on Linux, `isolated` gives a
sandbox a network of its own. `offline` gives it no route out. An egress policy
is enforced on `hvi` and on Linux, and brig refuses a policy-bound boot on any
backend that cannot enforce it.

A port opens on the host only when you publish one. It binds to `127.0.0.1`
unless you name another address:

```bash
brig network publish claude 3000   # the agent's dev server, on localhost:3000
```

On Linux a port is published when the sandbox is created, with
`brig run --publish`.

<p align="center">
  <img alt="brig resolves a session on the host and boots it as a microVM: hull on macOS, urunc over containerd on Linux. Both give the guest the same contract of a guest home, a named project mount, credentials passed per exec, and a guest image whose signature brig checks." src="https://raw.githubusercontent.com/brig-sh/brig/main/assets/architecture.svg" width="900">
</p>

brig is not a container runtime and does not try to be one. It delegates boot,
exec and stop to the runtime underneath, and adds six things on top:

- a guest home plus a named project mount;
- credentials resolved on the host and delivered per exec;
- a credential broker on the host, which adds a credential to the requests a
  route allows, so the guest never holds it;
- a denylist for the provider keys that can silently move you onto metered
  billing;
- a network posture per sandbox, and an egress policy bound to a profile or a
  session;
- a signature check on the guest image and on the kernel it boots.

Image verification defaults to reporting an image it cannot verify and booting
anyway. `BRIG_VERIFY=require` refuses one instead.

## Supported platforms

| Host | Runtime | Status |
| --- | --- | --- |
| macOS 15 or newer, Apple silicon | [hull](https://github.com/brig-sh/hull), on one of three hypervisor backends | Supported |
| macOS, Intel | | Not supported. There is no Intel build of hull |
| Linux, arm64 and x86-64 | `nerdctl` and containerd, with the urunc shim (`io.containerd.urunc.v2`) on the `hvi` monitor by default, installed by `install.sh` | Supported |

hull is tested on macOS 26. Seven of the nine agent profiles brig ships ask
hull for its `hvi` backend, which needs macOS 15.

The published guest images are multi-arch, for arm64 and x86-64. Any Linux CLI
in an OCI image runs as a bring-your-own image, so the agent profiles are a
convenience rather than a requirement.

## The repositories

| | |
| --- | --- |
| [**brig**](https://github.com/brig-sh/brig) | the CLI and the session daemon |
| [**hull**](https://github.com/brig-sh/hull) | the microVM runtime on macOS, which boots an OCI image as a real VM |
| [**hvi-vmm**](https://github.com/brig-sh/hvi-vmm) | the microVMM behind hull's `hvi` backend, and the default monitor on Linux: Hypervisor.framework on macOS, KVM on Linux |
| [**community-images**](https://github.com/brig-sh/community-images) | guest images, with open Dockerfiles |
| [**homebrew-brig**](https://github.com/brig-sh/homebrew-brig) | the Homebrew tap |
| [**brig-artwork**](https://github.com/brig-sh/brig-artwork) | the brand marks, and the generator that builds them |

brig and hull are Apache-2.0 and Go. hvi-vmm is Apache-2.0 and Rust. It
publishes signed releases of `hvi`, and hull ships the release pinned in its
`hvi.pin`, after it checks the signature and the hash.

## What we sign

The macOS binaries carry a Developer ID signature and are notarized by Apple.
Each tagged release signs its `checksums.txt` with keyless cosign, and the
guest images and the `hvi` releases are signed the same way. A keyless
signature is bound to the workflow that produced it rather than to a key
somebody holds.

The kernel and initrd a sandbox boots are checked too. brig verifies the boot
bundle's signature, fetches the bundle by the digest that verified, and hashes
both files against the bundle's manifest before each boot. On Linux brig
checks them against the runtime's signed release record instead. A file that
differs refuses the run. Only a kernel build of your own, named with
`BRIG_BOOT_ASSETS` and carrying no signed record, boots with a warning under
the default.

Two gaps worth naming. `install.sh` checks brig's and hull's archives against
`checksums.txt`, and does not check the cosign signature on that file. On
Linux it checks the signature on the runtime's checksums, with a pinned cosign
that it downloads when the host has none. It stops when that signature is
missing or does not verify.

The second gap is the time between the check and the boot. brig hashes the
kernel and initrd before the runtime starts, and the runtime opens them later,
by path. Anything that can write the asset directory in between can swap a
file. On macOS hull checks a staged copy against its own record. On Linux
nothing checks the files again.

Everything else, including where the boundary is weaker than it looks, is in
[brig's security documentation](https://brig.sh/docs/security/).

## What we count

brig keeps no telemetry store and runs no collector. On macOS, hull counts a
few events per brig command, such as a sandbox boot, and sends them with its
crash reports to NOFire AI. Through brig, a boot is not counted until you have
answered hull's consent prompt. On Linux nothing is sent. `brig telemetry off`,
or `DO_NOT_TRACK=1`, turns it off.
[hull's telemetry documentation](https://github.com/brig-sh/hull/blob/main/docs/telemetry.md)
lists every field an event carries.

## Built on urunc

The Linux runtime, and hull on macOS, are based on
[urunc](https://github.com/urunc-dev/urunc), a [CNCF](https://www.cncf.io/)
Sandbox project. urunc does the hard part, running unikernels and lightweight
VMs as OCI containers, and hull carries that onto macOS.

<p align="center">
  <a href="https://www.cncf.io/">
    <img alt="Cloud Native Computing Foundation" src="https://raw.githubusercontent.com/brig-sh/hull/main/assets/cncf-logo.svg" width="220">
  </a>
  &nbsp;&nbsp;&nbsp;
  <a href="https://github.com/urunc-dev/urunc">
    <img alt="urunc" src="https://raw.githubusercontent.com/brig-sh/hull/main/assets/urunc-logo.png" width="80">
  </a>
</p>

Neither brig nor hull is a CNCF project, and neither is endorsed by the CNCF.
The Linux Foundation has registered trademarks and uses trademarks. For a list
of trademarks of The Linux Foundation, please see our
[Trademark Usage page](https://www.linuxfoundation.org/trademark-usage).
urunc, CNCF and the CNCF logo are trademarks of The Linux Foundation.

---

<p align="center">
  <a href="https://nofire.ai">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/brig-sh/brig/main/assets/nofire-logo-on-dark.svg">
      <img alt="NOFire AI" src="https://raw.githubusercontent.com/brig-sh/brig/main/assets/nofire-logo.svg" width="150">
    </picture>
  </a>
</p>

<p align="center">Powered by <a href="https://nofire.ai">NOFire AI</a></p>
