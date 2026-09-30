# pi-kernel-optee

A custom Ubuntu Core **kernel snap** for the Raspberry Pi 5, built with OP-TEE
support. It exists because the officially published `pi-kernel` snap does not
include OP-TEE, and this project needs it for a disk-encryption proof of
concept anchored in the Arm TrustZone secure world.

## Why a separate snap

The stock `pi-kernel` is compiled from
[`linux-raspi`](https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux-raspi/+git/noble),
Canonical's kernel fork for the Raspberry Pi, without the TEE/OP-TEE drivers
enabled. Rather than patch or fork the published snap, `pi-kernel-optee`
rebuilds `linux-raspi` from source with those drivers compiled in, using
Snapcraft's [kernel plugin](https://snapcraft.io/docs/the-kernel-plugin).

## What it changes

| Change | Purpose |
|---|---|
| `CONFIG_TEE=y`, `CONFIG_OPTEE=y` (built in, not modules) | Expose `/dev/tee0` at boot, so user space can talk to the secure world |
| `CONFIG_DM_CRYPT=y` (built in) | Provides the device-mapper target used to encrypt the storage volumes |
| Source pinned to tag `Ubuntu-raspi-6.8.0-1057.61` | Matches the version of the `pi-kernel` it replaces on the device, which `snapd` otherwise requires |
| OP-TEE client library (`libteec`) injected into the initramfs | Lets `snap-bootstrap` reach the secure world during early boot, before the main system starts |

Building these three kconfig options in is not enough on its own — the exact
`snapcraft.yaml` configuration (cross toolchain, additional apt repository for
`arm64` packages, the `firmware`/`optee-client`/`initrd` parts that stage
`libteec` and trim the firmware) is documented in the dissertation's
Appendix on the gadget and kernel build.

The armstub (Arm Trusted Firmware + OP-TEE) and the device-tree overlay that
expose the TEE node to the kernel are **not** produced by this snap — they are
supplied by the gadget snap and injected into this snap's `dtbs` asset by a
post-build repacking step, run after `snapcraft pack` completes.

## Building

Cross-building this snap on an `amd64` host under `arm64` emulation
repeatedly failed while unpacking the firmware. The snap is instead built
**natively**, on a Raspberry Pi 5 running Ubuntu Server 24.04, with no
emulation involved:

```bash
# Build on an ARM64 machine (Raspberry Pi 5 with Ubuntu Server 24.04),
# natively, WITHOUT emulation:
sudo snap refresh snapcraft --classic --channel=9.x/stable
sudo snapcraft pack --enable-experimental-plugins --destructive-mode --verbosity verbose
```

- `--destructive-mode` builds directly on the host instead of spinning up a
  nested LXD container, which is unnecessary when the host already matches
  the target architecture.
- `--enable-experimental-plugins` is required because the `kernel` plugin
  used here is still experimental in this Snapcraft channel.
- The build is resource-heavy (RAM and CPU) and takes over an hour.

The result is `pi-kernel-optee_6.8.0-1057-raspi_arm64.snap`.

## Using the snap

This snap is not published to the Snap Store, so it cannot be referenced by
name in the model assertion the way the gadget is. The model still declares
`pi-kernel`, the officially published kernel, by its Store snap-id. At image
build time, the fleet-image pipeline sideloads this local snap in place of
that entry, passing it to `ubuntu-image` with an explicit `--snap` argument
and the `--allow-snapd-kernel-mismatch` flag, which is required because the
sideloaded snap's identity does not match the `pi-kernel` entry declared in
the model.

See the [`ubuntu-core-builder`](https://github.com/BeaComp/ubuntu-core-builder)
repository for the full image-build pipeline that consumes this snap.