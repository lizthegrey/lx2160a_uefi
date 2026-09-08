# LX2160A (HoneyComb LX2K / ClearFog CX) UEFI firmware

EDK2-based UEFI firmware with Secure Boot for the SolidRun LX2160A COM Express Type 7
module. This repository is the continuation of SolidRun's `lx2160a_uefi`, which SolidRun
archived on 2026-08-04 in favour of U-Boot. Development continues here, rebased onto
current upstream releases:

| Component | Version |
|-----------|---------|
| ATF (TF-A) | NXP `lf_v2.12` + SolidRun CEX7 patches |
| RCW | NXP `lf-6.6.52-2.2.0` + SolidRun CEX7 configs |
| EDK2 | `edk2-stable202608` |
| edk2-platforms | upstream master + SolidRun overlay |
| OP-TEE | 4.9.0, hosting StandaloneMm for authenticated UEFI variables |

Boots Ubuntu 24.04 / 26.04 with Secure Boot enabled via shim → GRUB → signed kernel.
Tested on a HoneyComb LX2K at 2200 MHz core / 750 MHz platform / 3200 MT/s DDR.

Full write-up of the modernization, patch inventory and upstreaming status:
https://gist.github.com/lizthegrey/9344dc71dc4ac11f9d6c79d7c143e535

## Build with host tools

Builds natively on aarch64 (for example on the HoneyComb itself). Verified with Ubuntu
26.04 (GCC 15) and 24.04 (GCC 13). Required packages: `build-essential`, `uuid-dev`,
`acpica-tools`, `device-tree-compiler`, `python3-pycryptodome`, `python3-pyelftools`.

```bash
git clone --recursive https://github.com/lizthegrey/lx2160a_uefi.git
cd lx2160a_uefi

# Stock speeds (2000 MHz core / 700 MHz platform / 2400 MT/s), no Secure Boot, RELEASE
bash ./runme.sh

# Overclock + Secure Boot (BUS_SPEED defaults to 750 for SOC_SPEED=2200; 700 also available)
SOC_SPEED=2200 DDR_SPEED=3200 SECURE_BOOT=true bash ./runme.sh

# Debug build
UEFI_RELEASE=DEBUG SECURE_BOOT=true bash ./runme.sh

# Clean build
CLEAN=1 SOC_SPEED=2200 DDR_SPEED=3200 SECURE_BOOT=true bash ./runme.sh
```

Images land in `images/` as `lx2160acex7_<soc>_<bus>_<ddr>_<serdes>_<boot>[_secure]_<githash>.img`.

## Flashing

The image contains its own offsets. Write it from block 0, with no `seek`:

```bash
dd if=images/lx2160acex7_*.img of=/dev/sdX bs=512 conv=notrunc
```

## Secure Boot key enrollment

A Secure Boot build first boots in Setup Mode. Enroll db and KEK (Microsoft UEFI CA 2011,
Windows Production PCA 2011, Microsoft KEK CA 2011) and then a self-generated PK using
`efitools`; `cert-to-efi-sig-list` needs PEM input, so convert Microsoft's DER `.crt` files
first. Setting PK exits Setup Mode.

## Build with docker or podman

```bash
docker build -t lx2160a_uefi docker/
docker run -v "$PWD":/work:Z --rm -i -t lx2160a_uefi build
docker run -e SOC_SPEED=2200 -e DDR_SPEED=3200 -e SECURE_BOOT=true -v "$PWD":/work:Z --rm -i -t lx2160a_uefi build
```

The container path has not been exercised recently; native builds are what is tested.
