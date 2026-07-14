# Syng Cell Alpha — UART Console Access & Dev-Mode Enable Guide

A practical walkthrough for getting a serial (UART) root console on a Syng Cell Alpha
and enabling developer mode so SSH works. This is for **hardware you own**. It involves
opening the unit and touching live low-voltage electronics — go slow, and back up first.

---

## 0. What you're doing and why it works

The Cell Alpha runs a Yocto Linux image on an **NXP i.MX8 (aarch64)** with **U-Boot** as the
bootloader. Root SSH is *not* on by default — a boot script (`syng-dev-enable.sh`) only turns it
on if the file **`/devdata/devenable`** exists. `/devdata` lives on a **separate flash partition
(`/dev/mtdblock1`)** that the firmware-restore tool never erases (the restore only rewrites the
main root filesystem via Mender).

So the entire goal of this procedure is simple: **get a root shell over the serial port and create
one empty file, `/devdata/devenable`.** Once it's there, it survives future firmware restores, and
the unit permanently accepts SSH (and opens a DBus control port).

The console runs at **115200 baud, 8N1**, 3.3 V logic.

---

## 1. Tools you'll need

- **USB-to-TTL serial adapter, 3.3 V logic** (FTDI FT232, CP2102, or CH340 based). Make sure it's
  set to **3.3 V**, not 5 V — 5 V can damage the SoC.
- **3 jumper wires** (female-to-female or with a small hook/probe on the board end).
- **A multimeter** (continuity + DC volts) to identify the pads.
- **Precision screwdrivers / spudger / plastic pry tools** to open the enclosure.
- **Terminal software** on your computer:
  - Linux/macOS: `picocom`, `minicom`, or `screen`
  - Windows: PuTTY or Tera Term
- Optional but recommended: a **magnifier or phone macro lens**, and **Kapton/electrical tape** to
  secure wires.

---

## 2. Safety and precautions

- **Back up first.** If you have any unit you can already reach, image `/devdata` and `/data` so you
  have the per-device provisioning data if something goes wrong.
- **3.3 V only.** Never connect the adapter's VCC/5 V pin to the board. You only need **TX, RX, GND.**
- **Unplug the speaker** before opening it and before probing with a multimeter for continuity.
- **ESD:** touch a grounded surface first; avoid working on carpet.
- **Common ground:** the adapter and the board must share GND, or you'll see garbage or nothing.
- Work on **one unit** end-to-end before doing the rest, so you learn the pad layout once.

---

## 3. Open the unit and find the UART pads

1. With the speaker unplugged, remove the base/foot and any visible screws (some are under rubber
   feet or fabric — a spudger helps). Work around the seam gently; there are usually plastic clips.
2. Locate the main logic board (the one with the i.MX8 SoC and eMMC). Look for a **row of 3–4
   unpopulated pads or a small header**, often silkscreened `TX`/`RX`/`GND` or `J-something`, near
   the SoC or a test-point cluster. Debug UART is frequently a 4-pad inline group: **VCC, TX, RX, GND.**

### Identifying the pads with a multimeter (unpowered)

- **GND:** set the meter to continuity; one probe on a known ground (e.g., a USB/metal shield or a
  large ground plane / screw boss), the other on each pad. The one that beeps is **GND**.

### Identifying TX vs RX (powered, careful)

- Reconnect power, set the meter to **DC volts**, black probe on GND.
- A pad sitting steady near **3.3 V** that briefly flickers during boot is the board's **TX**
  (it idles high and pulses as it prints the boot log).
- A pad near **0 V or floating** is usually the board's **RX**.
- The 4th pad sitting at a constant 3.3 V (never flickering) is **VCC — do not use it.**

---

## 4. Wire it up

Cross TX and RX:

| Adapter pin | Board pad |
|-------------|-----------|
| GND         | GND       |
| RX          | TX (board) |
| TX          | RX (board) |
| VCC / 3.3 V | **leave disconnected** |

Secure the wires with tape so they don't slip while you type.

---

## 5. Open the terminal

Plug the adapter into your computer and find its device name:

- Linux: `ls /dev/ttyUSB*` or `ls /dev/ttyACM*`
- macOS: `ls /dev/tty.usbserial-*` or `/dev/tty.usbmodem*`
- Windows: check Device Manager for the COM port

Connect at **115200 8N1**:

```bash
# picocom
picocom -b 115200 /dev/ttyUSB0

# or screen
screen /dev/ttyUSB0 115200

# or minicom
minicom -D /dev/ttyUSB0 -b 115200
```

(PuTTY: Connection type = Serial, Speed = 115200.)

Now **power on the speaker.** You should see U-Boot and kernel boot messages scroll past. If you see
garbage, the baud is wrong; if you see nothing, swap TX/RX or recheck GND.

> Note: The build uses `ttymxc0` for the Bluetooth radio, so the **console is typically on a
> different i.MX UART (e.g., `ttymxc1`)**. If one pad group is silent or Bluetooth-looking, try the
> other UART header — the console is the one printing the boot log.

---

## 6. Get a root shell

### Path A — U-Boot single-user (preferred, cleanest)

1. Watch the very start of boot. When U-Boot prints something like
   *"Hit any key to stop autoboot"*, **press a key** to drop to the `=>` U-Boot prompt.
2. Inspect the current boot arguments:
   ```
   => printenv bootargs
   ```
3. Boot the kernel into a root shell by appending an init override. The exact boot command varies;
   the common approach is to add `init=/bin/sh` (or `single`) to the kernel args, then run the normal
   boot macro. For example:
   ```
   => setenv bootargs ${bootargs} init=/bin/sh
   => run bootcmd
   ```
   (If `bootcmd` immediately resets the args, instead find the script that sets `bootargs` with
   `printenv`, copy it, append `init=/bin/sh`, `setenv bootargs ...`, then `boot`.)
4. The kernel boots and drops you straight to a `#` shell as root, no password.

### Path B — U-Boot is locked

If autoboot can't be interrupted or the console is password-protected, skip UART for the shell and
use **i.MX serial-download mode with NXP's `uuu` (mfgtools)** to write the `/devdata` partition
directly. That's a separate procedure (put the SoC in serial-download/USB-SDP mode, then script `uuu`
to mount/modify the devdata MTD). Mentioned here for completeness; Path A is easier when available.

---

## 7. Enable dev mode (the one file that matters)

You now have a root shell. The root filesystem may be mounted read-only and `/devdata` may not be
auto-mounted in single-user mode, so mount it yourself.

1. Remount root read-write (if needed) and check the flash partitions:
   ```sh
   mount -o remount,rw /
   cat /proc/mtd            # confirm which mtd is the devdata partition
   ```
2. Mount the devdata partition. It's `/dev/mtdblock1`; filesystem is usually jffs2 (try that first,
   then others if it fails):
   ```sh
   mkdir -p /mnt/devdata
   mount -t jffs2 /dev/mtdblock1 /mnt/devdata   # if that errors, try: -t ubifs, or plain mount
   ```
   If it's already mounted at `/devdata` in your boot mode, just use that path directly.
3. Create the flag file and flush it to flash:
   ```sh
   touch /mnt/devdata/devenable
   sync
   ```
   (Optional — also enables verbose developer debug logging on next boot:)
   ```sh
   touch /mnt/devdata/debugenable
   sync
   ```
4. Reboot into the normal firmware:
   ```sh
   reboot -f
   ```
   (or from the U-Boot prompt after a power cycle, just let it boot normally.)

On this normal boot, `syng-dev-enable.sh` sees `/devdata/devenable`, installs the permissive SSH
config, and brings the unit up with **root SSH enabled** (plus DBus control on TCP port 1010).

---

## 8. Verify

From another computer on the same network:

```bash
ssh root@<speaker-ip>
```

Use the root password for the build. If login succeeds, dev mode is on and **it will persist through
future firmware restores**, because the flag lives on the untouched `/devdata` partition.

Optional confirmation on the unit:

```sh
ls -la /devdata/           # devenable should be present
cat /etc/ssh/sshd_config   # should be the permissive dev version
```

---

## 9. Reassemble

Power down, remove the UART wires, and reassemble the enclosure. The change is stored in flash, so no
wiring needs to stay attached. Repeat sections 3–8 for each additional unit (the pad layout is
identical across units, so it's fast after the first).

---

## 10. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Nothing prints on boot | GND not connected, or TX/RX not actually the console UART — try the other UART header |
| Garbled/random characters | Wrong baud rate — confirm 115200 8N1 |
| Boot log prints but you can't type | Adapter TX not wired to board RX; check the cross-wiring |
| Can't interrupt autoboot | Hold a key from the instant power is applied; if truly locked, use Path B (`uuu`) |
| `mount` of mtdblock1 fails | Wrong fs type — try `-t jffs2`, then `-t ubifs`, then plain `mount`; check `cat /proc/mtd` |
| SSH still refused after reboot | `devenable` didn't persist (no `sync`, or wrong partition) — redo section 7 and confirm with `ls /devdata` |

---

## 11. Quick reference

- **Console:** 115200 8N1, 3.3 V, wires = GND + TX↔RX only
- **Root shell:** interrupt U-Boot → append `init=/bin/sh` to `bootargs` → boot
- **The magic file:** `touch /devdata/devenable` (on `/dev/mtdblock1`) → `sync` → reboot
- **Result:** permanent SSH + DBus:1010, survives firmware restores
