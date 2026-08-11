# Heltec Mesh Pocket — Flash Layout & Dual-Boot Analysis

Generated from local builds on 2026-08-11.

## Build Summary

| Firmware | Environment | Status | Flash Used | RAM Used | UF2 |
|----------|-------------|--------|------------|----------|-----|
| **Meshtastic** | `heltec-mesh-pocket-5000` | ✅ Success | 669,248 B / 815,104 B (82.1%) | 101,392 B / 248,832 B (40.7%) | `firmware-heltec-mesh-pocket-5000-2.7.19.5230268.uf2` (1.3 MB) |
| **MeshCore Companion (BLE)** | `Mesh_pocket_companion_radio_ble` | ✅ Success | 378,792 B / 712,704 B (53.1%) | 148,632 B / 235,520 B (63.1%) | `firmware.uf2` (generated) |

Both targets use **nRF52840**, **S140 SoftDevice 6.1.1**, and the Adafruit UF2 bootloader (family `0xADA52840`). Flashing is done by copying a UF2 to the `HT-n5262` drive (double-press reset).

---

## nRF52840 Flash Memory Map (1 MiB total)

```
Address     Size      Region                          Meshtastic          MeshCore Companion
─────────────────────────────────────────────────────────────────────────────────────────────
0x000000    4 KiB     MBR                             Shared              Shared
0x001000  148 KiB     S140 SoftDevice 6.1.1           Shared              Shared
0x026000  808 KiB     Application slot (linker max)   FULL SLOT           TRUNCATED (696 KiB)
0x0D4000  100 KiB     Extra flash filesystem          (inside app slot)   CustomLFS ExtraFS
0x0ED000   28 KiB     InternalFS (BLE bonds, etc.)    InternalFS          InternalFS
0x0F4000   44 KiB     UF2 / Adafruit bootloader       Shared              Shared
0x0FF000    4 KiB     Bootloader settings (UICR)      Shared              Shared
0x100000    —         End of flash
```

### Linker scripts

| Project | Script | App FLASH origin | App FLASH length |
|---------|--------|------------------|------------------|
| Meshtastic | `nrf52840_s140_v6.ld` (Adafruit BSP default) | `0x26000` | `0xED000 - 0x26000` = **808 KiB** |
| MeshCore companion | `boards/nrf52840_s140_v6_extrafs.ld` | `0x26000` | `0xD4000 - 0x26000` = **696 KiB** |

### Filesystem regions

| Region | Address range | Size | Meshtastic | MeshCore Companion |
|--------|---------------|------|------------|-------------------|
| **InternalFS** | `0xED000` – `0xF3FFF` | 28 KiB (7 × 4 KiB pages) | NodeDB, config, channels | Identity, BLE bonding, small blobs |
| **ExtraFS** (CustomLFS) | `0xD4000` – `0xECFFF` | 100 KiB (`0x19000`) | *Not used* — app may extend here | Contacts, channels, message store |

MeshCore companion defines ExtraFS in `examples/companion_radio/main.cpp`:

```cpp
CustomLFS ExtraFS(0xD4000, 0x19000, 128);
```

Meshtastic InternalFS base address (Adafruit BSP):

```cpp
#define LFS_FLASH_ADDR  0xED000   // nRF52840
#define LFS_FLASH_TOTAL_SIZE  (7 * 4096)  // 28 KiB
```

---

## Actual Binary Footprint (from ELF)

| Metric | Meshtastic | MeshCore Companion |
|--------|------------|-------------------|
| `.text` (code) start | `0x26000` | `0x26000` |
| `.text` size | 666,176 B (651 KiB) | 377,636 B (369 KiB) |
| `.text` end | ~`0xC8A40` | ~`0x82324` |
| Headroom in app slot | ~146 KiB to `0xED000` | ~318 KiB to `0xD4000` |
| RAM `.bss` + `.data` | ~99 KiB | ~145 KiB |

---

## Dual-Boot Feasibility

### Can you run both at once?
**No.** There is no multi-boot selector on the Mesh Pocket. You flash **one** firmware image at a time via UF2. Each image overwrites the application region starting at `0x26000`.

### Can you switch between Meshtastic and MeshCore without re-flashing the bootloader?
**Yes.** Both images are standard Adafruit nRF52 UF2 packages targeting the same bootloader and SoftDevice. Switching firmware is a normal UF2 drag-and-drop.

### Will settings persist when switching?
**No — expect data loss in flash-backed storage.**

| Concern | Detail |
|---------|--------|
| **InternalFS (`0xED000`)** | Both stacks use this region for BLE bonding and small persistent data, but **on incompatible formats**. Flashing either firmware typically reformats or ignores the other's data. |
| **ExtraFS (`0xD4000–0xECFFF`)** | Only MeshCore companion uses this. Meshtastic's larger binary can extend code into this range as it grows (currently ends at `0xC8A40`, still below `0xD4000`). |
| **Meshtastic NodeDB / channels** | Stored in InternalFS; wiped when MeshCore takes over. |
| **MeshCore contacts / messages** | Stored in ExtraFS + InternalFS; wiped when Meshtastic takes over. |

### Layout compatibility summary

```
0x26000 ┬──────────────────────────────────────┐
        │  APPLICATION CODE (both firmwares)      │  ← mutually exclusive
        │  Meshtastic: up to 0xED000              │
        │  MeshCore:   up to 0xD4000 only         │
0xD4000 ├──────────────────────────────────────┤
        │  ExtraFS (MeshCore only)                │  ← Meshtastic may overlap if app grows
0xED000 ├──────────────────────────────────────┤
        │  InternalFS (both, incompatible data) │
0xF4000 ├──────────────────────────────────────┤
        │  Bootloader (do not overwrite)        │
0xFF000 └──────────────────────────────────────┘
```

**Verdict:** Dual-boot in the sense of **choosing firmware at boot time is not supported**. You can **alternate** between Meshtastic and MeshCore companion by re-flashing, but treat each switch as a **clean install** for stored mesh data, contacts, and BLE pairings.

---

## Build Commands

```bash
# Meshtastic (from repo root)
pio run -e heltec-mesh-pocket-5000

# MeshCore companion BLE (from MeshCore clone)
cd MeshCore
pio run -e Mesh_pocket_companion_radio_ble
python3 bin/uf2conv/uf2conv.py -f 0xADA52840 -c .pio/build/Mesh_pocket_companion_radio_ble/firmware.hex \
  -o .pio/build/Mesh_pocket_companion_radio_ble/firmware.uf2
```

## Map Files

| Firmware | Map file path |
|----------|---------------|
| Meshtastic | `.pio/build/output.map` |
| MeshCore | `MeshCore/.pio/build/Mesh_pocket_companion_radio_ble/output.map` |

Analyze with: `python3 bin/analyze_map.py --map <path> --top 20`
