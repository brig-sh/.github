<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/brig-sh/brig/main/assets/brig-lockup-on-dark.svg">
    <img alt="brig" src="https://raw.githubusercontent.com/brig-sh/brig/main/assets/brig-lockup-on-light.svg" width="320">
  </picture>
</p>

<p align="center"><strong>Run a coding agent in a sandbox it cannot escape, with the credentials it needs and none of the ones it does not.</strong></p>

---

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/brig-sh/brig/main/install.sh | sh
```

Or with Homebrew, which also brings the macOS runtime:

```bash
brew tap brig-sh/brig
brew trust brig-sh/brig      # brew refuses untrusted third-party taps
brew install --cask brig

brig run claude
```

Put the projects the agent should work on in `~/brig/claude-code`. That
directory is the agent's entire world; the rest of your machine is invisible
to it.

## How it fits together

The agent runs in a guest with its own kernel. It sees one directory and the
credentials you chose to forward, and nothing else -- not your keychain, not
your SSH agent, not the rest of your disk. That inaccessibility is the
boundary, and it is also why credentials have to be forwarded in explicitly.

<p align="center">
  <img alt="brig sandbox architecture: brig run &lt;agent&gt; resolves the session on the host, hull over Hypervisor.framework on macOS and urunc over KVM on Linux boot it, and both give the guest the same contract" src="https://raw.githubusercontent.com/brig-sh/brig/main/assets/architecture.svg" width="900">
</p>

brig is not a container runtime and does not try to be one. It delegates boot,
exec and stop to the runtime underneath, and adds the four things neither has
a concept of: the workspace as guest home, credentials resolved on the host
and handed to the guest, a denylist for the provider keys that would silently
move you onto metered billing, and signature verification of the guest image
before it boots.

## Credentials

A sandbox with no credential still boots -- the agent asks you to log in
inside it, the way it would on a machine you had just set up. If you would
rather reuse the login already on your Mac, carry it in once:

```bash
brig secret import claude-code
```

That reads your host keychain **once, when you type it**, and copies the value
into brig's own store. Every run afterwards reads only that store, so a run
opens no other application's keychain item and raises no approval dialog.

Where the credential lands in the guest is the profile's decision. Claude Code
gets a file, at the path it already reads, written into a memory-only mount --
so the token never reaches your disk, by construction rather than by
inspection. Others travel as environment variables, for tools that offer no
file interface.

## Supported platforms

| | runtime | status |
| --- | --- | --- |
| macOS 26+, Apple Silicon | [hull](https://github.com/brig-sh/hull) over Virtualization.framework | supported |
| Linux, arm64 and amd64 | urunc over containerd (`io.containerd.urunc.v2`) | supported |
| macOS, Intel | -- | not supported |

Guest images are arm64 today. Any Linux CLI in an OCI image runs as a
bring-your-own image; the agent templates are convenience, not a requirement.

## The repositories

| | |
| --- | --- |
| [**brig**](https://github.com/brig-sh/brig) | the CLI and the session daemon |
| [**hull**](https://github.com/brig-sh/hull) | the microVM runtime on macOS, built on [urunc](https://github.com/urunc-dev/urunc) |
| [**community-images**](https://github.com/brig-sh/community-images) | guest images, with open Dockerfiles |
| [**homebrew-brig**](https://github.com/brig-sh/homebrew-brig) | the Homebrew tap |

Everything we publish is signed. Binaries carry a Developer ID signature and
an Apple notarization ticket; release checksums and guest images are signed
with keyless cosign, so a signature is bound to the workflow that produced it
rather than to a key somebody holds.

## Built on urunc

The Linux runtime, and the macOS one underneath hull, is
[urunc](https://github.com/urunc-dev/urunc), a [CNCF](https://www.cncf.io/)
Sandbox project. urunc does the hard part -- it runs unikernels and
lightweight VMs as OCI containers -- and hull carries that onto macOS, on top
of Virtualization.framework.

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
