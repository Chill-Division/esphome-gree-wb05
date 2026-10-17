# On-site guide: back up, inspect and (later) flash a GRJWB05-J10

Work through the steps in order and **stop after step 4**. Nothing is written to the module
until its backup has been checked (see [Step 5](#step-5-flash-esphome)).

## What you need

* A 3.3 V USB-serial adapter. These notes use the M5Stack USB downloader (pins `3.3V`, `RXD`,
  `TXD`, `EN`, `G0`, `GND`).
* A 1 kΩ resistor, thin wire (30 AWG), a fine-tip soldering iron and a multimeter.
* The laptop with `ltchiptool` from [libretiny-eu/ltchiptool#98](https://github.com/libretiny-eu/ltchiptool/pull/98)
  (see the README's build section). On WSL, pass the adapter through with
  [usbipd-win](https://github.com/dorssel/usbipd-win): `usbipd list`, `usbipd bind --busid <id>`
  (admin, once), `usbipd attach --wsl --busid <id>`.

## Safety

* **Mains.** The display board sits behind the front panel, away from the terminal block, but
  isolate the unit before opening anything near the electrical box.
* **Isolation check before connecting a laptop to a powered unit.** The indoor unit's low-voltage
  side isn't guaranteed to be isolated from mains. With the unit powered, measure **AC volts
  between module pad 1 (GND) and the earth screw** on the terminal block. It must read close to
  0 V. If it reads more than a few volts, don't connect the laptop.
* Run the laptop **on battery** (charger unplugged) while connected, and don't touch metal.
* **Never connect the adapter's `3.3V` pin.** The module is powered by the display board.

## Module pads

Seen from the shield side with the antenna on the left (as the module sits on the display board):

```
Top row, left → right:     16    15    14    13    12    11    10     9       8
                          PB22  MISO   GND  3.3V  MOSI   SCL   SDA  LOG_RX  LOG_TX
Bottom row, left → right:  17    18     1     2     3     4       5      6      7
                           —   PA27   GND  3.3V   ADC  UART_TX UART_RX LP_SDA LP_SCL
```

* Pads **8/9** are the log UART (PA7/PA8) at 115200 baud. It's also the ROM download port.
* Pads **4/5** are the link to the indoor unit (LP UART), expected at 4800 baud, 8E1.
* Pad **1** is GND. Pad **2**, right next to it, is 3.3 V: don't bridge them.

## Wiring

| Step | Adapter | Module pad |
| --- | --- | --- |
| 1. Boot log | `GND` | 1 |
| | `RXD` (through 1 kΩ) | 8 |
| 2. AC link | `GND` | 1 |
| | `RXD` (through 1 kΩ) | 4, then 5 |
| 3. Backup | `GND` | 1 |
| | `RXD` (through 1 kΩ) | 8 |
| | `TXD` | 9 |
| | jumper wire, only while powering up | 8 to GND |

`EN`, `G0` and `3.3V` stay unconnected. If nothing at all appears in step 1, the adapter may
label its pins from the target's side; swap `RXD` for `TXD` (the resistor keeps that safe).

## Step 1: boot log (listen only)

1. Wire `GND` → pad 1 and `RXD` → pad 8.
2. Start a terminal at **115200 8N1** (`python3 -m serial.tools.miniterm /dev/ttyUSB0 115200` in
   WSL, or PuTTY on Windows).
3. Unplug the display board's cable for 5 seconds and plug it back in. The module reboots and
   reconnects to the Gree cloud by itself; the unit keeps running.
4. Save everything it prints. We're looking for the chip ID, the boot stages, and anything
   mentioning secure boot, signatures or encryption.

## Step 2: AC link (listen only, optional)

1. Move `RXD` to pad **4** and set the terminal to **4800 8E1**, in hex view:
   `python3 -m serial.tools.miniterm --parity E --encoding hexlify /dev/ttyUSB0 4800`.
2. Change something in the Gree+ app and save the bytes. Repeat with `RXD` on pad **5**.
3. Frames should start with `7E 7E` (the Gree WiFi-module protocol).

## Step 3: full backup

1. Wire `GND` → pad 1, `RXD` → pad 8, `TXD` → pad 9.
2. **Download mode:** hold a jumper from pad 8 to GND, unplug and replug the display cable,
   wait a second, then remove the jumper.
3. Read the whole flash, twice, and compare:

   ```bash
   ltchiptool flash read -d /dev/ttyUSB0 realtek-ambd backup-1.bin
   ltchiptool flash read -d /dev/ttyUSB0 realtek-ambd backup-2.bin
   sha256sum backup-1.bin backup-2.bin
   ```

   The two hashes must match. Also try `ltchiptool flash read -d /dev/ttyUSB0 -E realtek-ambd efuse.bin`
   for the eFuse contents.
4. Unplug and replug the display cable **without** the jumper. The module boots Gree's firmware
   again and the unit is back on the cloud.
5. Keep the backup safe: it contains the unit's WiFi credentials and Gree device key.

## Step 4: analyse before flashing

Stop here and send the backup (privately, not to this repo) for analysis. It tells us:

* the partition layout and boot chain, and whether Gree's bootloader checks image signatures
  (secure boot);
* which pins Gree's firmware uses for the LP UART;
* whether the flash layout matches the BW16 layout the build assumes.

## Step 5: flash ESPHome

To be written once the backup has been analysed. Expected shape: enter download mode as in step 3
and write the ESPHome image. After that first wired flash, every update goes over WiFi (ESPHome OTA).

## Restoring Gree's firmware

Enter download mode and write the backup back:

```bash
ltchiptool flash write -d /dev/ttyUSB0 -f realtek-ambd -s 0 backup-1.bin
```

The module boots Gree's firmware with its original WiFi settings and pairing, as before.
