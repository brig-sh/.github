<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/brig-sh/brig/main/assets/brig-lockup-on-dark.svg">
    <img alt="brig" src="https://raw.githubusercontent.com/brig-sh/brig/main/assets/brig-lockup-on-light.svg" width="280">
  </picture>
</p>

<p align="center"><strong>Run a coding agent in a microVM on your own machine, with the credentials it needs and none of the ones it does not.</strong></p>

---

## Install

On macOS, with Homebrew, which brings the runtime with it:

```bash
brew tap brig-sh/brig
brew trust brig-sh/brig      # brew refuses untrusted third-party taps
brew install --cask brig

brig run claude ~/code/demo
```

Or without Homebrew. The script checks the archive against the published
`checksums.txt`, and it does not check the cosign signature on that file:

```bash
curl -fsSL https://raw.githubusercontent.com/brig-sh/brig/main/install.sh | sh
```

That run mounts `~/code/demo` read-write at `/work/demo` and starts the agent
there. The sandbox boots with no credential, so the agent asks you to log in
just as it does on a new machine.

## How it fits together

The agent runs in a guest with its own kernel. Two host directories reach
inside it: a guest home of its own, and the one project you name on the command
line. Nothing else is mounted, and brig reads no credential out of your
keychain, your SSH agent or your secret manager on the run path.

That mount is read-write, so an agent can change the project you handed it.
What it cannot see is the rest of your disk. On the default network it can
still reach the internet, so anything it can read it can also send. brig can
filter egress on one backend, and refuses a policy-bound boot on any backend
that cannot enforce it, rather than running unenforced.

<p align="center">
  <img alt="brig resolves a session on the host and boots it as a microVM: hull on macOS, urunc over containerd on Linux. Both give the guest the same contract of a guest home, a named project mount, credentials passed per exec, and a guest image whose signature brig checks." src="https://raw.githubusercontent.com/brig-sh/brig/main/assets/architecture.svg" width="900">
</p>

brig is not a container runtime and does not try to be one. It delegates boot,
exec and stop to the runtime underneath, and adds the four things neither has a
concept of: a guest home plus a named project mount, credentials resolved on
the host and forwarded per exec, a denylist for the provider keys that can
silently move you onto metered billing, and a signature check on the guest
image. Verification defaults to reporting an image it cannot verify and booting
anyway. `BRIG_VERIFY=require` refuses one instead.

## Supported platforms

| Host | Runtime | Status |
| --- | --- | --- |
| macOS 15 or newer, Apple silicon | [hull](https://github.com/brig-sh/hull), on one of three hypervisor backends | Supported |
| macOS 14, Apple silicon | hull, with `BRIG_HYPERVISOR=vz` | Works, not tested |
| macOS, Intel | | Not supported. There is no Intel build of hull |
| Linux, arm64 and x86-64 | `nerdctl` and containerd, with the urunc shim (`io.containerd.urunc.v2`) | Supported |

macOS 26 is what CI runs. Six of the eight agent profiles brig ships ask hull
for its `hvi` backend, which needs macOS 15, so `BRIG_HYPERVISOR=vz` is the way
past that floor on macOS 14.

The published guest images are arm64. Any Linux CLI in an OCI image runs as a
bring-your-own image, so the agent profiles are a convenience rather than a
requirement.

## The repositories

| | |
| --- | --- |
| [**brig**](https://github.com/brig-sh/brig) | the CLI and the session daemon |
| [**hull**](https://github.com/brig-sh/hull) | the microVM runtime on macOS, which boots an OCI image as a real VM |
| [**hvi-vmm**](https://github.com/brig-sh/hvi-vmm) | the microVMM behind hull's `hvi` backend: Hypervisor.framework on macOS, KVM on Linux |
| [**community-images**](https://github.com/brig-sh/community-images) | guest images, with open Dockerfiles |
| [**homebrew-brig**](https://github.com/brig-sh/homebrew-brig) | the Homebrew tap |
| [**brig-artwork**](https://github.com/brig-sh/brig-artwork) | the brand marks, and the generator that builds them |

brig and hull are Apache-2.0 and Go. hvi-vmm is Apache-2.0 and Rust, builds
from source, and publishes no crate and no release.

## What we sign

Released binaries carry a Developer ID signature and an Apple notarization
ticket. Release checksums and guest images are signed with keyless cosign, so a
signature is bound to the workflow that produced it rather than to a key
somebody holds.

Two gaps worth naming. `install.sh` verifies the checksum and not the cosign
signature on the checksum file, because that needs cosign installed. brig
checks the boot bundle's signature at its registry reference without binding
the kernel on disk to it.

Everything else, including where the boundary is weaker than it looks, is in
[brig's security documentation](https://github.com/brig-sh/brig/blob/main/docs/security.md).

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
