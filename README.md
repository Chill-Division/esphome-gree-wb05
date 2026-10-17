# ESPHome for Gree GRJWB05-J10 WiFi modules

Local, cloud-free control of current Gree air conditioners (tested on the **Gree Alto** range) by
running [ESPHome](https://esphome.io) on the unit's own WiFi module instead of Gree's firmware.

> [!WARNING]
> **Experimental: not yet run on hardware.** The firmware builds, but the module-side steps
> (backup, flashing, the AC link) haven't been carried out yet. Follow [docs/ON-SITE.md](docs/ON-SITE.md),
> which stops for a backup review before anything is written. Flashing replaces Gree's firmware,
> so the Gree+ app stops working for that unit until the backup is restored.

Until this is proven, the cloud integration [Chill-Division/ha-gree-plus](https://github.com/Chill-Division/ha-gree-plus)
controls the same units through Gree's cloud.

## Hardware

Gree Alto indoor units (e.g. GWH24ATEXF-K6DNA1E/I) carry their WiFi on the receiver/display board
behind the front panel (part `300001061521`, PCB `GRJ568-B V1`). The module is soldered onto it:

| | |
| --- | --- |
| Module | Gree **GRJWB05-J10**, part `300145000093`, FCC ID `2ADAP-GRJWB05J10` |
| SoC | Realtek **RTL8720DF** (AmebaD), WiFi 2.4/5 GHz + BLE |
| Link to the indoor unit | LP UART on pads 4/5, expected 4800 baud 8E1 |
| Flashing / log | log UART on pads 8/9 (PA7/PA8); ROM download mode with PA7 low at reset |

Pad map, wiring and the full procedure: [docs/ON-SITE.md](docs/ON-SITE.md). How the stock module
talks to Gree's cloud and the hardware identification:
[ha-gree-plus/docs/PROTOCOL.md](https://github.com/Chill-Division/ha-gree-plus/blob/main/docs/PROTOCOL.md).

## How it works

* **ESPHome on the RTL8720D.** Released ESPHome doesn't support AmebaD yet. This uses the
  in-progress upstream work, pinned by commit:
  * ESPHome: [esphome/esphome#18718](https://github.com/esphome/esphome/pull/18718) (RTL8720D family and BW16 board);
  * LibreTiny: [libretiny-eu/libretiny#403](https://github.com/libretiny-eu/libretiny/pull/403) (AmebaD family),
    plus fixes from [ChillingSilence/libretiny@ambd-lp-uart-softserial-mdns](https://github.com/ChillingSilence/libretiny/tree/ambd-lp-uart-softserial-mdns)
    (offered upstream in [this comment](https://github.com/libretiny-eu/libretiny/pull/403#issuecomment-6227934618)):
    * bring up the LP UART (`SERIAL2`), which carries the AC link on this module. **Not yet
      verified on hardware**;
    * stop advertising SoftwareSerial, which broke linking;
    * build lwIP's mDNS responder;
  * ltchiptool: [libretiny-eu/ltchiptool#98](https://github.com/libretiny-eu/ltchiptool/pull/98) (AmebaD flashing and UF2 images).
* **The board.** The module follows the BW16 reference layout, so it uses the `bw16` board. The LP
  UART pins (PB1/PB2) are assumed until confirmed from a flash dump.
* **The AC protocol.** [`components/sinclair_ac`](components/sinclair_ac) is piotrva's
  [Gree/Sinclair component](https://github.com/piotrva/esphome_gree_ac) with one fix for Realtek
  builds (see its [NOTICE](components/sinclair_ac/NOTICE.md)). It provides power, Auto/Cool/Dry/Fan/Heat,
  set point, fan Auto/Quiet/Low…High/Turbo, vertical and horizontal swing and fixed positions,
  display, plasma, sleep, X-fan and Save. Whether this module's link uses exactly the same
  protocol is one of the things the on-site capture checks.

## Setting up a flashed module

The repo follows ESPHome's ready-made project pattern:

* [`gree-wb05.yaml`](gree-wb05.yaml) is the device package: hardware, AC link and entities, no
  credentials.
* [`gree-wb05-factory.yaml`](gree-wb05-factory.yaml) is what gets flashed the first time. It adds
  [`dashboard_import`](https://esphome.io/components/dashboard_import/) for adoption.

After the first flash:

1. The module has no WiFi details, so it opens a **`Gree AC <suffix>`** hotspot. Join it from a
   phone and enter your WiFi in the captive portal.
2. It joins your network and appears in **Home Assistant** under *Settings → Devices & services →
   Discovered*, and in the **ESPHome dashboard** as ready for adoption.
3. **Adopt** it in the ESPHome dashboard. That writes a short config which pulls in
   `gree-wb05.yaml` as a package, adds your WiFi secrets and an API encryption key, and installs it
   over WiFi. Then add it to Home Assistant with that key.

Each unit gets a unique name (`gree-ac-1a2b3c`, from its MAC) until you rename it on adoption.

If you'd rather skip the hotspot, add a `wifi:` block with your network to the factory config
before building the first image.

## Building

```bash
# ESPHome with RTL8720D support (esphome/esphome#18718)
python3 -m venv .venv
.venv/bin/pip install "git+https://github.com/Bl00d-B0b/esphome.git@057c9625e1e7671464cedc0fa113edc489bd95c6"

.venv/bin/esphome compile gree-wb05-factory.yaml
```

The first build downloads LibreTiny from the pinned source and the ARM toolchain. It produces
`image_ota.0x006000.bin` (application image). With a released `ltchiptool` the final UF2 step
reports `Unsupported family - realtek-ambd`; for the UF2 (used by ESPHome's WiFi OTA) and for
flashing, install the AmebaD branch into LibreTiny's tool environment:

```bash
~/.platformio/penv/.libretiny/bin/pip install \
  "git+https://github.com/Bl00d-B0b/ltchiptool.git@a3f13093fa8892ecbce51910d4ff1cae453a06c9"
```

## Status

- [x] ESPHome firmware with the climate component builds for the RTL8720D
- [x] Module, SoC, pad map and download-mode entry identified
- [ ] Boot log and full flash backup of a stock module
- [ ] Secure boot / signature check status known
- [ ] LP UART pins confirmed and the LP UART fix verified on hardware
- [ ] AC link protocol confirmed against a real unit
- [ ] WiFi hotspot / captive portal, mDNS discovery and adoption verified on this chip
- [ ] First flash, then OTA updates

## License

GPL-3.0, because it includes the GPL-3.0 `sinclair_ac` component. Not affiliated with Gree.
