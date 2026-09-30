# pi-kernel-optee

Custom Ubuntu Core kernel snap for Raspberry Pi with OP-TEE support.

This project provides a modified `pi-kernel` snap based on the Ubuntu
`linux-raspi` source tree. The kernel is built with the configuration required
to access OP-TEE from Linux and to support encrypted storage used by the
prototype.

The resulting snap is used as the kernel component of the Ubuntu Core fleet
image described in the main project.

## Why a custom kernel?

The standard `pi-kernel` snap used by Ubuntu Core for Raspberry Pi does not
provide all the kernel configuration and build-time components required by
this prototype. Therefore, a custom kernel snap, `pi-kernel-optee`, was
developed.

The custom kernel includes, among other changes:

- Linux support for the TEE subsystem and OP-TEE;
- `dm-crypt` support for encrypted storage;
- the kernel source version required for compatibility with the installed
  `snapd` version;
- OP-TEE client support in the initramfs, allowing the boot process to
  communicate with the secure world before the root filesystem is mounted.

The modifications are implemented on top of Ubuntu's `linux-raspi` source tree.

## Build environment

The kernel snap is built **natively on a Raspberry Pi 5 running Ubuntu Server
24.04 LTS**.

A native ARM64 build is used because attempts to build the snap on an AMD64
machine using ARM64 emulation repeatedly failed while unpacking the Raspberry
Pi firmware during the Snapcraft build process.

For this reason, the build should be performed on an ARM64 machine, preferably
a Raspberry Pi 5.

The build does not use emulation.

## Prerequisites

The build machine must have:

- ARM64 architecture;
- Ubuntu Server 24.04 LTS or a compatible Ubuntu environment;
- Snapcraft installed;
- sufficient storage space for the kernel build and intermediate files.

Check the architecture with:

```bash
uname -m
```

The expected result is:

```
aarch64
```

## Build procedure

### 1. Clone the repository

On the ARM64 build machine, clone the repository:

```bash
git clone https://github.com/BeaComp/pi-kernel-optee.git
```

Enter the project directory:

```bash
cd pi-kernel-optee
```

### 2. Update Snapcraft

The kernel snap is built using the Snapcraft 9.x channel:

```bash
sudo snap refresh snapcraft --classic --channel=9.x/stable
```

Check the installed version:

```bash
snapcraft --version
```

### 3. Build the kernel snap

Run Snapcraft in destructive mode:

```bash
sudo snapcraft pack --enable-experimental-plugins \
  --destructive-mode \
  --verbosity verbose
```

The build uses the experimental kernel plugin provided by Snapcraft, hence the
`--enable-experimental-plugins` option.

The `--destructive-mode` option causes Snapcraft to perform the build directly
on the host machine rather than creating an additional isolated build
environment. This is required for the native Raspberry Pi build used in this
project.

The build may take more than one hour on a Raspberry Pi 5.

### 4. Build output

After a successful build, Snapcraft produces an ARM64 kernel snap similar to:

```
pi-kernel-optee_6.8.0-1057-raspi_arm64.snap
```

This snap can then be used as the custom kernel component when generating the
Ubuntu Core fleet image.

## Snap structure

The `snapcraft.yaml` is organized into several parts, each responsible for
building or providing one component required by the final kernel snap.

The main parts are:

### kernel

This is the main kernel build.

It uses the Ubuntu `linux-raspi` source tree as the base and applies the
configuration changes required by this project, including:

```
CONFIG_TEE=y
CONFIG_OPTEE=y
CONFIG_DM_CRYPT=y
```

The part also fixes the kernel source revision to the version compatible with
the target Ubuntu Core environment.

### Firmware

The firmware-related part provides the Raspberry Pi firmware components
required by the resulting kernel snap.

These files are included in the final snap so that the generated image
contains the firmware expected by the Raspberry Pi boot process.

### OP-TEE client

The OP-TEE client components provide the userspace interface required to
communicate with OP-TEE.

In this project, the OP-TEE client library is also made available to the
initramfs so that the early boot environment can communicate with the secure
world before the normal Ubuntu Core system is started.

### Initramfs

The initramfs contains the components required during the early boot stage.

It includes the OP-TEE client library and the components necessary for the
prototype's encrypted-storage workflow. During boot, `snap-bootstrap`
communicates with OP-TEE to obtain the key required to unlock the LUKS2
encrypted storage before the normal system startup continues.

### Boot integration

The generated kernel snap is part of a boot chain in which the firmware starts
the Arm Trusted Firmware (BL31), which initializes the platform and enters the
secure world. OP-TEE runs as BL32.

The Linux kernel and initramfs are then started. During the early boot
process, `snap-bootstrap` communicates with OP-TEE through the OP-TEE client
interface. The key returned by the secure environment is used to unlock the
LUKS2 encrypted storage, allowing Ubuntu Core to continue booting.

The detailed boot-chain integration and its role in the prototype are
described in the dissertation.

## Result

The main artifact generated by this repository is:

```
pi-kernel-optee_6.8.0-1057-raspi_arm64.snap
```

This snap is consumed by the Ubuntu Core image build pipeline of the main
project.

## Repository contents

The repository contains the Snapcraft configuration and the modifications
required to build the custom kernel snap. The complete configuration is kept
in the repository to make the build reproducible.

The resulting `.snap` package is generated locally and is not required to be
stored in the repository.

## Related project

This repository is part of the SMARTLIGHT gateway prototype. The custom
kernel provides the OP-TEE support required by the encrypted-storage and
secure-boot integration implemented in the gateway image.
