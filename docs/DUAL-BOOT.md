# rsDeck Dual-Boot

The dual-boot layout lets one flash carry three apps on the T-Deck Plus's 16MB
flash, with a boot-time chooser:

| Partition  | Slot  | Offset   | Size    | Contents |
|------------|-------|----------|---------|----------|
| launcher   | ota_0 | 0x10000  | 1MB     | Boot chooser UI |
| standalone | ota_1 | 0x110000 | 4MB     | Full Ratdeck messenger |
| rnode      | ota_2 | 0x510000 | 3MB     | RNode (KISS over USB CDC + BLE) |
| littlefs   | —     | 0x810000 | ~7.9MB  | User data (unchanged from single-app layout) |

The littlefs and coredump regions are byte-identical to the legacy
single-app layout (`partitions_16MB.csv`), so migrating to the dual image
preserves identity, messages, contacts, and settings.

## Boot flow

- The ESP32 ROM bootloader reads `otadata` and starts the selected slot.
- The launcher (ota_0) shows Standalone/RNode, remembers the last choice in
  NVS (`rslaunch/last`), auto-boots it after 7 seconds, and writes the choice
  via `esp_ota_set_boot_partition()`.
- Both Standalone and RNode re-arm the launcher at every boot
  (`rs_deck::returnToLauncherNextBoot()`), so any reset returns to the chooser.
  On the legacy single-app layout that call self-selects ota_0 and is a no-op.

## Launcher controls

Trackball up/down + click, or keys: `W`/`S` select, `Enter` boot,
`R` = Standalone now, `N` = RNode now. Any input cancels auto-boot.

## RNode mode

The vendored RNode firmware (`vendor/rnode_firmware`, `make firmware-tdeck`)
self-provisions EEPROM on first boot (PRODUCT_TDECK_V1 / MODEL_D9, 915 MHz
defaults) and enables BLE pairing on first run when no bonds exist. The T-Deck
RNode build is headless — the screen stays off; reset to return to the
launcher.

## Build

```bash
make prep-tdeck     # once: arduino-cli esp32 core + libs
make package        # dist/: rsdeck-full.zip, rsdeck-standalone.zip,
                    #        rsdeck-rnode.zip, rsdeck-*-app.bin
make flash port=/dev/cu.usbmodemXXXX   # flash full image
```

`rsdeck-full.bin` flashes at offset 0x0 and contains all three apps.
`rsdeck-standalone.zip` repackages the legacy single-app `ratdeck-merged.bin`
(old partition table, no launcher). The `*-app.bin` files are bare app images
for external launchers or manual OTA-slot flashing.
