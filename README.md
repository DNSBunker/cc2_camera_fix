# Prevent or recover the Elegoo CC2 stock camera failure

Some stock Elegoo Centauri Carbon 2 cameras can stop working after repeated
printer power cycles. This repository provides ways to protect or recover
an affected camera:

- a working camera can receive a small startup fix through USB without
  flashing a firmware image;
- if there is too little free space to install that fix, use a same-camera
  backup to build and restore a preventive image through USB;
- a camera that no longer starts can be recovered from its own flash backup
  with an external programmer.

All routes preserve the identity of **your** camera. There is no generic
firmware image in this repository, and you must never write another camera's
dump to your device.

> **AI-development note:** This fix was developed with the assistance of AI.
> It has produced the expected result in two tested cases, but if you have any
> doubts, independently audit the tools before using them on your hardware.

## First identify your camera

The known failure affects one 8 MiB flash-layout variant of the
`EF-S7-V1.0.30B` camera family. An earlier 16 MiB 30B variant uses materially
different hardware and firmware. A newer `EF-S7-V1.0.30D` revision also uses
different hardware and software and is not affected by this particular
problem.

Do this identification before building a USB cable, buying a programmer, or
running a repair operation.

If the camera feed still works, start with the read-only stream check. Install
[`cc2camera`](docs/INSTALLATION.md), find the printer's IP address or hostname,
and run:

```sh
cc2camera identify-camera PRINTER
```

For example:

```sh
cc2camera identify-camera 192.168.1.50
```

If the stream is unavailable, the signature is unknown, or you want authoritative
visual confirmation, inspect the hardware:

1. Power the printer off and unplug it from mains power.
2. Remove the camera module from the printer by undoing its single mounting
   screw, then unplug its four-wire cable.
3. Look at the large processor on the camera mainboard that's now in front of you.

If it is marked `TX5110`, the camera is likely the newer, unaffected family.
This recovery does not apply. If it is marked `Ingenic T23`, or the marking is
unclear, remove the two housing screws and check the complete PCB revision.

![TX5110 processor on a newer CC2 camera](docs/images/ef-s7-v1.0.30d-tx5110.jpg)

The four-digit PCB manufacturing code appears to use `WWYY` week/year order.
It is useful supporting evidence, but is not a safe cutoff on its own: component
or firmware changes may not align exactly with calendar weeks.

| Identification | Observed manufacturing codes | What is known |
|---|---|---|
| `EF-S7-V1.0.30B` / Ingenic T23 / 8 MiB `ZB25VQ64` family | `0226`, `0526` (two units), `1526`, `2526` | This is the supported layout on which the failure has been observed. Continue below. |
| `EF-S7-V1.0.30B` / Ingenic T23 / 16 MiB `P25Q128H` family | `4025` | This early layout is materially different. The known erase defect has not been established on it, and the current fix does not apply. |
| `EF-S7-V1.0.30D` / likely TX5110 | Not established | This revision is not affected by the known failure. This guide does not apply. |
| Any other revision, processor or flash layout | Not established | It has not been investigated. We do not know whether it is affected, and this guide does not apply. |

![EF-S7-V1.0.30D PCB marking](docs/images/ef-s7-v1.0.30d-revision.png)

Photographs of the `EF-S7-V1.0.30B` board are available in the
[OpenCentauri camera documentation](https://docs.opencentauri.cc/hardware/CC2/camera/).
The stream fingerprint is a convenient read-only family-identification aid.
The complete hardware markings and the tool's firmware and flash-layout checks
determine whether a 30B camera is supported. Never bypass a refusal because a
date code or stream signature appears to match.

## Choose the path that matches your camera

### The affected camera still works

Use the [USB prevention route](#working-camera-usb-prevention). It needs a
simple camera-to-USB cable but no SPI programmer.

### The affected camera no longer works

First check whether the fault is isolated to the camera:

1. Leave the stock camera disconnected.
2. While the printer is off, connect a known-working ordinary USB webcam to
   the printer's front USB port.
3. Power the printer on.

If the replacement webcam produces a feed, the stock camera is likely the
problem. Continue with the
[hardware recovery route](#failed-camera-hardware-recovery).

If the replacement webcam also fails, this test has not isolated the problem
to the stock camera. Stop this camera-recovery procedure.

> **Temporary replacement:** Once the stock camera is disconnected, most
> ordinary USB webcams can be used in the front USB port while you wait for a
> replacement or decide whether to attempt recovery. Connect the webcam before
> powering the printer on so it has the best chance of becoming `/dev/video0`,
> which is the camera device the printer expects. The likely drawback is an
> inconvenient camera angle unless you improvise or print a mount.

## Working camera: USB prevention

Start with the startup fix below. It writes small configuration files and
reapplies a correction each time the camera starts, allowing its filesystem to
reclaim deleted data. It requires no SPI programmer or firmware-image write.
The installer checks that enough already erased space is available before
writing. If space is insufficient, use the
[image restore fallback](#if-there-is-not-enough-space-build-and-restore-an-image).

The startup fix is [physically tested on one supported camera](docs/STARTUP-HOOKS-VALIDATION.md),
including repeated boots and a bounded write/delete pressure test. That camera
already had the earlier preventive image patch; hook-only operation on otherwise
unmodified stock startup was not independently tested.

### What you need

- the affected camera, still able to start;
- access to a printer with a 0.2 mm nozzle;
- four P50 pogo pins, such as P50-J1;
- the USB-A plug and cable from an unused USB data cable;
- [cc2camera installed](docs/INSTALLATION.md), either the Windows executable or Python package;
- the Android platform `adb` executable as described in the installation guide.

### Build the camera-to-USB cable

The included [printable pogo-pin adapter](docs/models/cc2-camera-p50-pogo-adapter.stl)
acts as a plug for the four-pin socket on the camera PCB. Print it with a
**0.2 mm nozzle** and fit four P50 pogo pins; P50-J1 is one suitable example.

![Printed pogo-pin camera adapter attached to a USB cable](docs/images/cc2-camera-p50-pogo-adapter.jpg)

The computer end can be salvaged from an old USB-A cable. Cut off the unwanted
device end, leaving the USB-A plug and enough cable to work with. Make sure it
is a data cable with all four conductors rather than a charge-only cable.

The camera connector carries ordinary USB 2.0:

| Camera pin | Signal | USB-A pin | Typical USB cable color |
|---:|---|---:|---|
| 1 | GND | 4 | Black |
| 2 | D+ | 3 | Green |
| 3 | D- | 2 | White |
| 4 | +5 V | 1 | Red |

Do not trust wire colors unverified. Before connecting the camera, use a
multimeter to confirm every conductor from the USB plug to its pogo pin and
confirm that +5 V is not shorted to ground or either data line.

### 1. Start ADB and make a backup

Follow the [installation guide](docs/INSTALLATION.md) for `cc2camera` and `adb`.
Connect only one camera, then run:

```sh
cc2camera start-adb
cc2camera backup backup.zip
```

Use the command prefix from your [platform's installation instructions](docs/INSTALLATION.md).
If ADB is not on PATH, add `--adb "path/to/adb"` to connected-camera commands.

`start-adb` temporarily starts the camera's existing root ADB service, which
lets the computer run maintenance commands. `backup` reads the complete flash
repeatedly and publishes `backup.zip` only after obtaining three consecutive
identical images. Keep that ZIP unchanged and in a separate safe location.

### 2. Install the startup fix

This step writes files to the camera's configuration partition. Keep power
connected until installation and file readback finish. Read the
[startup-hook guide](docs/STARTUP-HOOKS.md) for prerequisites and recovery
conditions before running:

```sh
cc2camera install-erase-fix --backup backup.zip
```

Confirm with `INSTALL-ERASE-FIX` when prompted. The command checks the camera,
backup, firmware and available space, then installs and reads back the files.
It applies the correction on the next boot; installation does not restart the
camera automatically.

If the command refuses installation **because there is insufficient space**,
continue with the image restore fallback below. For other refusals or any
write/readback failure, stop and resolve the reported problem before proceeding.
Do not bypass validation or retry a partial installation blindly.

### 3. Restart and verify

After successful installation, restart the camera manually. Without optional
persistent ADB, run `start-adb` again to reconnect for verification:

```sh
cc2camera start-adb
adb shell cat /tmp/cc2-hooks.log
```

The log should contain:

```text
cc2flash: erase fix active; master=0x1000; partition geometry=0x4000
cc2flash: 10-erase-fix.sh exit 0
```

Follow the [remaining verification checks](docs/STARTUP-HOOKS.md#verify-on-the-camera)
and confirm the camera feed works before returning it to service. A successful
installation message alone does not establish that the boot hook ran. Stop if
the hook fails or the camera no longer works normally.

### Optional: keep ADB available after restart

Persistent ADB is useful for future maintenance, but is not required for the
erase fix. To enable it, while ADB is online, run:

```sh
cc2camera install-adb-startup --backup backup.zip
```

Confirm with `ENABLE-ADB`. This writes a separate startup hook and performs its
own space checks and readback. After the next restart, ADB should return without
`start-adb`. The erase fix remains independently installed if optional ADB
installation is refused.

### If there is not enough space: build and restore an image

Use the unchanged backup from step 1. This route builds a camera-specific
preventive image and **writes that image to flash through USB**. Keep power
connected throughout restore and verification. Detailed prerequisites and stop
conditions are in the [USB-maintenance reference](usb-maintenance/REFERENCE.md).

```sh
cc2camera build-image backup.zip
cc2camera restore backup-cc2-recovery/cc2-camera-recovery.bin --backup backup.zip
```

The builder validates the backup and creates the preventive image. Do not pass
`--adb` to offline `build-image`. If needed, run `cc2camera start-adb` again before
`restore` and add `--adb "path/to/adb"` to that connected-camera command.

The optional `restore --dry-run` performs the same-camera and allowed-change
checks without opening USB. `restore` repeats those checks, validates the live
camera against the preserved backup, temporarily prepares the known stock SFC
driver, and requires the literal confirmation `RESTORE-CC2` before its first
write. Keep power connected until it returns to normal USB and completes the
required three-read verification. Preserve both the original ZIP and recovery
image, then confirm the camera feed works.

Stop if any command refuses the camera, backup, firmware, partition layout, or
generated image. Do not work around a validation failure. This fallback is for
insufficient space before hook installation; restoring an image from a backup
that already contains hooks has [additional limitations](docs/STARTUP-HOOKS.md#recovery-and-compatibility).

## Failed camera: hardware recovery

This route reads and rewrites the camera's eight-pin SPI flash with an external
programmer. You can use **Windows with NeoProgrammer** or **Linux with flashrom**.
Windows is not required. The Linux procedure below was tested on Fedora 44 with
a CH341 programmer and flashrom 1.6.0; the camera booted and its video feed worked
after recovery.

### What you need

- a camera matching the supported 8 MiB firmware and flash layout;
- a CH341A/CH341B programmer verified for the actual flash chip's voltage and
  SPI signal levels;
- an SOIC-8 test clip and cable;
- a multimeter;
- a separate drive for the original backups;
- either Windows with NeoProgrammer, or Linux with flashrom;
- [cc2camera installed](docs/INSTALLATION.md): the standalone executable on
  Windows, or the Python package on Linux.

ADB is not needed for external-programmer recovery or offline image building.
The CH341 worked with the flash still soldered onto the tested cameras. A Bus
Pirate was not reliable in-circuit on one tested setup; see the
[programmer reference](hardware-recovery/PROGRAMMER_REFERENCE.md).

### Tested Fedora setup

The following setup completed a recovery on 11 November 2026:

| Item | Observed value |
|---|---|
| Camera PCB | `EF-S7-V1.0.30B` |
| Processor | Ingenic T23 |
| Flash package marking | `25Q64DSIG` |
| Measured JEDEC ID | `5E 40 17` |
| Capacity reported through SFDP | 8 MiB / 8,388,608 bytes |
| Programmer USB ID | `1a86:5512` |
| Operating system | Fedora Linux 44 |
| Flash tool | flashrom 1.6.0, `ch341a_spi`, `SFDP-capable chip` |
| Recovery builder | cc2camera 0.9.0 |
| Result | Full verification and independent readback passed; camera video returned |

The marking had initially suggested a Zetta ZD25Q64B, but the chip actually
returned the Zbit-family ID `5E 40 17`. The `BA 32 17` value discussed in
[issue #28](https://github.com/OpenCentauri/cc2_camera_fix/issues/28) came from a
datasheet, not from this chip. Do not choose a Zetta profile from the package
marking alone.

The original configuration area was 97.854% occupied, with 751 CRC-valid JFFS2
nodes: two current and 749 obsolete. The builder accepted all three identical
original dumps and produced an image in its default `serial-only` mode. Only
`system_bashrc_patch` and `config` differed from the original. A further full read
matched the original immediately before programming. Both flashrom's full-chip
verification and a separate 8 MiB readback matched the recovery image. The owner
confirmed that the camera worked again after reinstalling it in the printer.

This is evidence for this camera, firmware and programmer combination. It does
not establish support for chips with a different ID, different flash layouts,
or every device that flashrom can detect through SFDP. The builder's firmware,
layout and identity checks remain required.

### 1. Verify voltage before connecting the camera

With the programmer connected to USB but no camera or clip attached, measure
CH341A pin 28 (`VCC`) relative to ground. It should be approximately **3.3 V**
for the tested 3 V flash. Also check the flash supply output and SPI logic-high
levels against the actual flash datasheet.

If a measured supply or logic level is near 5 V, stop. Board color, seller
description, a lit LED and jumper position do not prove that the programmer is
safe. Do not use a 1.8 V adapter for a chip confirmed to require a 3 V supply.

### 2. Connect and identify the flash

Keep the camera's normal USB/power cable disconnected. Connect the SOIC-8 clip
before plugging the programmer into USB. Align the clip's pin-1/red-stripe
conductor with the flash chip's pin-1 dot or notch, and use the programmer's
25-series SPI position. Never connect normal camera power and programmer power
at the same time.

Confirm the complete chip marking, its electrical requirements, measured JEDEC
ID and capacity. The tested `25Q64DSIG` chip returned `5E 40 17` and 8 MiB. If
identification is inconsistent or differs from this procedure, stop rather than
selecting a merely similar chip.

#### Windows: NeoProgrammer

Use **Detect IC** and confirm the exact chip profile and 8 MiB capacity.

#### Linux: install the tools and check detection

On Fedora, install flashrom from the distribution repositories:

```sh
sudo dnf install flashrom python3
```

Create a new working directory. Keep this directory open for all Linux examples
below. This installs the recovery-builder version used in the documented test:

```sh
mkdir cc2-camera-recovery
cd cc2-camera-recovery
python3 -m venv .venv
.venv/bin/python -m pip install "cc2camera @ https://github.com/OpenCentauri/cc2_camera_fix/archive/refs/tags/v0.9.0.zip"
.venv/bin/cc2camera --version
flashrom --version
```

The tested v0.9.0 source commit is
`28fae6d9a0ea5a703144ded4c4b241e52614cf01`. For other versions or distributions,
follow the [installation guide](docs/INSTALLATION.md). Do not install Python
packages into the system interpreter with `sudo pip`.

With only one programmer connected, check its USB presence, then probe the chip
without requesting an erase or write:

```sh
lsusb -d 1a86:5512
flashrom -p ch341a_spi -VV 2>&1 | tee chip-detection.log
```

For the documented setup, the `Generic unknown SPI chip (RDID)` probe reported
`id1 0x5e, id2 0x4017`, followed by:

```text
Found Unknown flash chip "SFDP-capable chip" (8192 kB, SPI)
```

flashrom 1.6.0 had no named profile for this part. It obtained the capacity and
standard flash-operation parameters from the chip's Serial Flash Discoverable
Parameters (SFDP) table. `-c "SFDP-capable chip"` selects that detected profile;
it does not force an unrelated chip definition.

Continue with the Linux commands below only when the expected ID, 8 MiB capacity
and SFDP detection agree. If no chip is found, access is denied, or detection
changes between attempts, resolve that problem first. Do not use `--force`,
disable verification, or change protection bits to get past a refusal. Close
other programmer applications before using flashrom.

### 3. Make three trustworthy backups

Do not move the clip between reads. Save each read as a separate file.

#### Windows: NeoProgrammer

1. Select **Read IC** and save the full buffer as `cc2-camera-1.bin`.
2. Read again and save as `cc2-camera-2.bin`.
3. Read a third time and save as `cc2-camera-3.bin`.

Use `certutil -hashfile FILE SHA256` for each file and record the hashes.
After copying the backups to the separate drive, repeat the checks there and
compare them with the recorded values.

#### Linux: flashrom

These commands only read the chip and save local files. Use the working
directory created above, with no existing files of these names:

```sh
flashrom -p ch341a_spi -c "SFDP-capable chip" -r cc2-camera-1.bin &&
flashrom -p ch341a_spi -c "SFDP-capable chip" -r cc2-camera-2.bin &&
flashrom -p ch341a_spi -c "SFDP-capable chip" -r cc2-camera-3.bin
```

Check sizes and compare every byte:

```sh
wc -c cc2-camera-1.bin cc2-camera-2.bin cc2-camera-3.bin
cmp cc2-camera-1.bin cc2-camera-2.bin && cmp cc2-camera-1.bin cc2-camera-3.bin
sha256sum cc2-camera-1.bin cc2-camera-2.bin cc2-camera-3.bin > original-SHA256SUMS.txt
```

Each file must be exactly **8,388,608 bytes**. Both `cmp` commands must exit
successfully without output, and all three hashes must match. All-`00`, all-`FF`,
unstable, wrong-size or inconsistent reads mean the connection is not
trustworthy. Do not erase anything.

Copy all three original files and their checksums to a separate drive before
continuing. On Linux, run this in the copied directory on that drive:

```sh
sha256sum -c original-SHA256SUMS.txt
```

All three entries must report `OK`. The recovery builder also compares the
three reads and validates the firmware contents; matching reads alone are not
proof of a supported image.

### 4. Build the recovery image

Use the three original files from the same camera. On Windows, follow the
[installation guide](docs/INSTALLATION.md) for the command prefix and run:

```bat
cc2camera.exe build-image cc2-camera-1.bin --confirm-read cc2-camera-2.bin --confirm-read cc2-camera-3.bin
```

In the Linux working directory created above:

```sh
.venv/bin/cc2camera build-image cc2-camera-1.bin --confirm-read cc2-camera-2.bin --confirm-read cc2-camera-3.bin
```

This is an offline operation. It checks read stability, firmware, flash layout,
identity, allowed changes and the generated image. It must finish without an
error and create:

```text
cc2-camera-1-cc2-recovery/cc2-camera-recovery.bin
```

If it refuses the dump or reports unfamiliar data, stop. Do not bypass a
validation failure or reduce the three-read requirement.

Check the generated files on Linux:

```sh
(cd cc2-camera-1-cc2-recovery && sha256sum -c SHA256SUMS.txt)
```

The full recovery image must be exactly 8,388,608 bytes. Do not use
`config-restored.bin`, `serial.cfg`, a region file or another camera's image as
the full-chip firmware.

The generated `FLASHING.txt` may contain a Bus Pirate/`COM11` command template.
Do not run that template unchanged with a CH341. Use the commands for your
actual programmer below.

### 5. Program and verify

**This step changes the camera's flash.** Proceed only after the original
backup has been verified on a separate drive and the recovery build has passed.
Keep the same stable power and clip connection throughout writing and
verification. Do not operate another programmer application at the same time.

#### Windows: NeoProgrammer

1. Detect the chip again and reconfirm its exact profile and 8 MiB capacity.
2. Open the generated full `cc2-camera-recovery.bin`.
3. Use the automatic write sequence with **Erase**, **Blank Check**,
   **Write/Program** and **Verify** enabled.
4. Require full-chip Verify to finish without any mismatch.
5. Without moving the clip, read the whole chip again and save the result as
   `cc2-camera-readback.bin` in the original working directory.
6. Compare the complete readback with the generated recovery image:

```bat
fc.exe /b cc2-camera-1-cc2-recovery\cc2-camera-recovery.bin cc2-camera-readback.bin
```

#### Linux: flashrom

First make a fresh full read and require it to match the preserved original.
If it differs, stop and investigate before writing:

```sh
flashrom -p ch341a_spi -c "SFDP-capable chip" -r cc2-camera-before-write.bin &&
cmp cc2-camera-1.bin cc2-camera-before-write.bin
```

After that comparison succeeds, write the generated full image:

```sh
flashrom -p ch341a_spi -c "SFDP-capable chip" -w cc2-camera-1-cc2-recovery/cc2-camera-recovery.bin
```

flashrom reads the current contents, erases/programs the required blocks and
performs full-chip verification by default. Do not add `-f`, `-n`, `-N` or
write-protection overrides. This command does not require a separate full-chip
erase. Wait for a successful exit and completed verification.

Then perform an independent full readback and byte comparison:

```sh
flashrom -p ch341a_spi -c "SFDP-capable chip" -r cc2-camera-readback.bin &&
cmp cc2-camera-1-cc2-recovery/cc2-camera-recovery.bin cc2-camera-readback.bin
sha256sum cc2-camera-1-cc2-recovery/cc2-camera-recovery.bin cc2-camera-readback.bin
```

`cmp` must exit successfully without output, and both hashes must match. A
successful write command alone is not a substitute for this readback.

### 6. Reconnect and test the camera

If erase, program, verification or readback fails, keep the setup connected and
diagnose the failure. Do not disconnect it to try booting the camera and do not
repeat a failed write blindly.

Only after complete verification succeeds:

1. Unplug the programmer from USB.
2. Remove the clip.
3. With the printer switched off and unplugged from mains power, reconnect and
   mount the camera normally.
4. Power on the printer and check the video feed.

Long-term power-cycle endurance remains untested for the documented Fedora unit.

## What the repair changes

Affected cameras repeatedly rewrite the same configuration files during every
boot. A defect in the flash driver prevents reliable cleanup of deleted data,
so old filesystem records accumulate until the writable configuration area can
no longer accept the next boot-time writes.

The startup fix corrects the driver's erase setting in memory each boot so
filesystem cleanup can work. It leaves the firmware image in place and stores
its startup files in the writable configuration area.

The image-based repair compacts that area, preserves the camera's own identity,
and changes startup behavior so default files are created only when missing.
It reduces unnecessary writes; it does not persistently repair the driver's
erase setting.

The builder accepts only the known 30B firmware family and allows changes only
inside the audited startup-script window and writable configuration partition.
Technical details, offsets, supported image fingerprints, config modes, and
limitations are in the
[hardware-recovery technical reference](hardware-recovery/TECHNICAL_DETAILS.md).

## Privacy and backups

Raw dumps, `backup.zip`, `serial.cfg`, and generated recovery images contain
unit-specific identifiers. Do not publish them unredacted. Preserve the
original backup separately from the computer used for recovery.

No full camera dump or vendor firmware image is included in this repository.

## Advanced and audit documentation

- [Installation and release downloads](docs/INSTALLATION.md)
- [Complete cc2camera command reference](docs/CLI.md)
- [Startup-fix installation and verification](docs/STARTUP-HOOKS.md)
- [Startup-fix hardware findings](docs/STARTUP-HOOKS-VALIDATION.md)
- [Programmer and alternative-hardware reference](hardware-recovery/PROGRAMMER_REFERENCE.md)
- [Independent recovery-tool verification](hardware-recovery/CC2_RECOVERY_VERIFICATION.md)
- [Hardware-recovery regression results](hardware-recovery/TEST_RESULTS.md)
- [USB-maintenance reference](usb-maintenance/REFERENCE.md)
- [USB project and hardware-validation status](usb-maintenance/PROJECT-STATUS.md)
- [Physical USB restore validation](usb-maintenance/PHYSICAL-VALIDATION.md)
- [Recovered USB protocol](usb-maintenance/PROTOCOL.md)
- [USB reverse-engineering evidence](usb-maintenance/EVIDENCE.md)
