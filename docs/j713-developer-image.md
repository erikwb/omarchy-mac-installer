# The 13-inch M4 MacBook Air (J713) as a developer board

Status: the engine admits `j713ap` (T8132) on macOS 26.6.2 OS firmware and knows its
board Stage 1 and Stage 2; `scripts/supported-models.json` lists `apple,j713` under
`developer`. No public catalog carries it: public builds keep every M4 Mac refused until
official support and physical qualification exist. A developer build's sealed catalog
may add it in a `developer_models` group with its own payload and metadata
(`make-unsigned-catalog.py --developer`).

One J713 has run this way since 2026-10-22: the 8-core M4 Air, booted through the
Stage 1 below into Eryk Wieliczko's kernel with native display and GPU, Wi-Fi,
Bluetooth, keyboard and trackpad, and an accelerated Hyprland desktop.

## Why only 26.6.2

Eryk's J713 stack accepts only 26.6.2 (25G83) firmware: his m1n1 stage 2
(aurora-silicon/m1n1 #35), the DCP display backend and the G16 GPU driver
(aurora-silicon/linux #366, #367). A stub paired with 27.x resets inside iBoot. So
`CHIP_MIN_VER[0x8132]` is 26.6.2, j713ap is on the 26.6.2 IPSW entry only, and the stub
takes the macOS 26 size. The image's `supported_fw` must be `["26.6.2"]`.

## What the image must carry

The engine's board contract (`BOARD_STAGE1`, `BOARD_STAGE2` in
`Engine/overlay/src/omarchy_asahi.py`); the install is refused before any disk change if
either member is missing:

| Member | Content |
|---|---|
| `esp/aurora/stage1-j713.bin` | aurora-silicon/m1n1 branch `m4/pr1-basic-bringup` with #36, built `make RELEASE=1 CHAINLOADING=1`. Carries the `AURORA-S1-CFG01` block; the engine fills the ESP PARTUUID and `m1n1/boot.bin` in place with a 0 ms proxy window. The image is a 64 KiB multiple and stays one, which T8132 iBoot requires of a boot object. |
| `esp/m1n1/boot-j713.bin` | The same m1n1 built `RELEASE=1 CHAINLOADING=1 ESP_STAGE2=1`, with the J713 control device tree and a gzipped U-Boot appended (m1n1 `docs/j713.md`, "embedded" layout). It replaces `m1n1/boot.bin` on the J713's ESP and is dropped on every other Mac. |

Inside that Stage 2 payload:

- **Control device tree**: the kernel's `t8132-j713.dtb` plus a `bluetooth@0,1` node
  (`pci14e4,5f72`, `apple,garden`) with a `bluetooth0` alias, and `atcphy0`/`atcphy1`
  aliases for `/soc/phy@402a90000` and `/soc/phy@40aa90000`.
- **U-Boot**: aurora-silicon/u-boot branch `m4/pr1-basic-bringup` (#7), built with GCC
  (`aarch64-linux-gnu-`, `apple_m1_defconfig`). An LLD link ignores the `.bss ADDR(.rela.dyn)
  (OVERLAY)` placement and produces an image header stage 2 rejects; a clang + GNU ld
  build linked but stalled before its console on this Mac.

And on the Linux side of the image:

- **Kernel**: aurora-silicon/linux `m4/pr2-gpu-dcp` (#366 + #367) with the 8-core G16
  firmware build (aurora-silicon/linux #389) for the 8-core Air. Command line needs
  `apple_dart.j713_dcp=1 apple_dart.j713_display=1 modprobe.blacklist=appledrm,asahi`, and a
  unit that loads `appledrm` before `asahi` before the display manager.
- **Device tree marker**: when systemd-boot supplies the DTB through its `devicetree`
  line, `/chosen/u-boot,j713-dtb-source` must name the kernel commit, or the kernel's J713
  import silently falls back to a static tree with no framebuffer and no NVMe.
- **Mesa**: aurora-silicon/mesa `m4/pr-mesa-bringup` (#28) as the system Mesa
  (`-Dgallium-drivers=asahi,llvmpipe -Dvulkan-drivers=asahi`). Stock Mesa cannot
  initialise his kernel: it binds its printf buffer at GPU VA 1<<36, which the driver
  refuses.
- Firmware for the BCM4388 Wi-Fi and Bluetooth comes from the recoveryOS through the
  pinned engine's `asahi_firmware`, as on other Macs; the J713 needs no system-image
  firmware recipe.

## Known limits of the stack

From #367 and the bring-up notes: no audio (speaker protection for this kernel does not
exist yet; keep it off), suspend, hibernate and kexec blocked while native graphics owns
DMA, keyboard dead in systemd-boot's menu, display power policy unqualified.

## A developer catalog inputs fragment

```json
"developer_models": [
  {
    "payload_name": "omarchy-j713-os-package.zip",
    "metadata_name": "installer_data-j713.json",
    "device_identifiers": ["apple,j713"]
  }
]
```

The group's metadata is a normal full-OS template with `supported_fw: ["26.6.2"]`.
