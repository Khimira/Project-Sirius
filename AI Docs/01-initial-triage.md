# 01 - Initial Firmware Triage

**Target:** Kodak Charmera (GPCV2247F)
**File:** `Sandbox/Original_Firmware.bin`
**Analysis date:** 2026-11-20
**Methodology:** Static triage workflow (header → entropy → strings → containers → assets)

---

## 1. File Metadata

| Property | Value |
|----------|-------|
| Size | `2,097,152` bytes (`0x200000` = 2 MB) |
| SHA-256 | `720ed0b65c91b149b0916498732b0ecf40227736ef5e3c126c487a7b5253436c` |
| MD5 | `d151365eb95c2dfcb8615f4a786b8321` |
| CRC32 | `0x279830A7` |
| Last non-`0xFF` byte | `0x1AEDFF` (effective data: ~1.72 MB) |
| Trailing empty region | `0x1AF000`–`0x1FFFFF` (331 KB of `0xFF`) |
| Backup | `Original_Firmware_Backup.bin` (2 MB, same `SPII` header) |

---

## 2. SPII Header (`0x0000`–`0x01FF`)

The firmware starts with Generalplus's proprietary boot container:

```
0000: 53 50 49 49  00 02 00 30  00 00 00 20  00 00 00 00   SPII...0... ....
0010: 00 00 1B 00  53 52 49 79  10 05 00 01  01 00 01 00   ....SRIy........
0020: 02 78 66 D0  05 00 00 00  88 00 00 D0  2B A1 00 00   .xf.........+...
0030: 8C 00 00 D0  95 24 00 00  28 00 00 D0  05 80 00 00   .....$..(.......
0040: AC 00 00 D0  04 01 00 00  00 00 00 00  00 00 00 00   ................
```

### Identified fields

| Offset | Bytes | Interpretation |
|--------|-------|----------------|
| `0x00` | `SPII` | Magic signature of the SPI container |
| `0x04` | `00 02 00 30` | Version/format (BE: `0x00020030`) |
| `0x08` | `00 00 00 20` | BE: `0x20` (32) — possibly alignment or offset |
| `0x10` | `00 00 1B 00` | BE: `0x1B00` (6912) — boot block size? |
| `0x14` | `SRIy` | Secondary vendor signature |
| `0x18` | `10 05 00 01` | Configuration flags |
| `0x1C` | `01 00 01 00` | Configuration flags |
| `0x20` | `02 78 66 D0` | Unknown field (BE `0x027866D0` / LE `0xD0667802`) |
| `0x24` | `05 00 00 00` | Segment counter (5) |

### RAM segment table (`0x28`–`0x47`)

`(destination_address, size)` pairs in **little-endian**:

| # | RAM address | Size | Bytes |
|---|-------------|------|-------|
| 1 | `0xD0000088` | `0x00A12B` | 41,259 |
| 2 | `0xD000008C` | `0x002495` | 9,365 |
| 3 | `0xD0000028` | `0x008005` | 32,773 |
| 4 | `0xD00000AC` | `0x000104` | 260 |
| 5 | `0x00000000` | `0x00000000` | — (terminator) |

> **Note:** the `0xD00000xx` addresses indicate that the SoC loads the firmware into internal RAM
> mapped at `0xD0000000`. The sizes add up to ~83 KB, consistent with the boot block
> (`0x240`–`0x1FFFF`) that is copied to RAM before the main execution.

---

## 3. Architecture and Boot Vectors

### Architecture: **ARM 32-bit little-endian**

Direct evidence in the code:

| Offset | Bytes | ARM instruction | Meaning |
|--------|-------|-----------------|---------|
| `0x200` | `18 F0 9F E5` | `LDR PC, [PC, #0x18]` | Reset vector |
| `0x204`–`0x210` | same | `LDR PC, [PC, #0x18]` | Exception vectors |
| `0x214` | `FE FF FF EA` | `B .` | Infinite loop (unused handler) |
| `0x240` | `FF 04 A0 E3` | `MOV r0, #0xFF` | Start of boot code |
| `0x24C` | `8C 01 9F E5` | `LDR r0, [PC, #0x18C]` | Loads peripheral table |

### Vector table (`0x200`–`0x23B`)

```
0x200: E59FF018  LDR PC, [PC, #0x18]  -> [0x220] = 0x0000003C
0x204: E59FF018  LDR PC, [PC, #0x18]  -> [0x224] = 0x00000168
0x208: E59FF018  LDR PC, [PC, #0x18]  -> [0x228] = 0x00000174
0x20C: E59FF018  LDR PC, [PC, #0x18]  -> [0x22C] = 0x00000180
0x210: E59FF018  LDR PC, [PC, #0x18]  -> [0x230] = 0x0000018C
0x214: EAFFFFFE  B .                  (infinite loop)
0x218: E59FF014  LDR PC, [PC, #0x14]  -> [0x234] = 0x00000198
0x21C: E59FF014  LDR PC, [PC, #0x14]  -> [0x238] = 0x000001C8
```

### Peripheral table (`0x3E0`–`0x41B`)

Generalplus hardware register addresses:

| Offset | Value | Likely peripheral |
|--------|-------|-------------------|
| `0x3E0` | `0xD0100038` | Clock/Power |
| `0x3E4` | `0xD0300000` | GPIO base |
| `0x3E8` | `0xD0300040` | GPIO + 0x40 |
| `0x3EC` | `0xD0300080` | GPIO + 0x80 |
| `0x3F0` | `0xD03000C0` | GPIO + 0xC0 |
| `0x3F4` | `0xD05001FC` | Timer |
| `0x3F8` | `0xD07023C0` | ISP/Camera |
| `0x3FC` | `0x000FF870` | Function address (boot) |
| `0x404` | `0x0000B6C0` | Function address |
| `0x408` | `0x0010AF30` | Function address |
| `0x40C` | `0x000A3820` | Function address |
| `0x410` | `0x00000203` | Configuration value |
| `0x414` | `0x00000103` | Configuration value |
| `0x418` | `0x00002003` | Configuration value |

> The initialization code at `0x240`–`0x3DF` configures these peripherals and jumps to
> `0x000FF870` (probably the main firmware entry point).

---

## 4. Entropy Map

Analysis in 4 KB blocks (512 blocks). Classification:
- **EMPTY:** >99% `0xFF`
- **CODE/DATA:** entropy > 5.5 bits/byte
- **MIXED/ASSETS:** entropy 3.0–5.5
- **SPARSE:** entropy < 3.0

### Main regions

| Range | Size | Class | Likely content |
|-------|------|-------|----------------|
| `0x000000`–`0x000FFF` | 4 KB | MIXED | SPII header + vectors |
| `0x001000`–`0x02BFFF` | 176 KB | CODE | Boot + early drivers |
| `0x02C000`–`0x02CFFF` | 4 KB | MIXED | Data table |
| `0x02D000`–`0x036FFF` | 40 KB | CODE | Filesystem / FAT driver |
| `0x037000`–`0x038FFF` | 8 KB | MIXED | Tables |
| `0x039000`–`0x088FFF` | 320 KB | CODE | Main application (UI, states) |
| `0x089000`–`0x08DFFF` | 20 KB | MIXED | Inline assets |
| `0x08E000`–`0x0B4FFF` | 160 KB | CODE | Video/audio codecs |
| `0x0B5000`–`0x0B5FFF` | 4 KB | MIXED | Tables |
| `0x0B6000`–`0x0F3FFF` | 248 KB | CODE | SSV6158 WiFi stack |
| `0x0F4000`–`0x0FCFFF` | 36 KB | MIXED | WiFi strings + tables |
| `0x0FD000`–`0x0FFFFF` | 12 KB | CODE | End of WiFi code |
| `0x100000`–`0x107FFF` | 32 KB | MIXED/CODE | Transition |
| `0x108000`–`0x10AFFF` | 12 KB | CODE | End of code |
| `0x10B000`–`0x11FFFF` | 84 KB | SPARSE | Sparse data / padding |
| `0x120000`–`0x122FFF` | 12 KB | MIXED | **GP+ resource table** |
| `0x123000`–`0x140FFF` | 120 KB | CODE/DATA | Compressed assets (GPZP) + WAV |
| `0x141000`–`0x168FFF` | 160 KB | MIXED/SPARSE | Assets + padding |
| `0x169000`–`0x16FFFF` | 28 KB | EMPTY | — |
| `0x170000`–`0x17FFFF` | 64 KB | SPARSE/EMPTY | Padding |
| `0x180000`–`0x188FFF` | 36 KB | CODE/DATA | **JPEG 960×720** |
| `0x189000`–`0x1A4FFF` | 112 KB | EMPTY | — |
| `0x1A5000`–`0x1A5FFF` | 4 KB | MIXED | JPEG header |
| `0x1A6000`–`0x1AEFFF` | 36 KB | CODE/DATA | **JPEG 960×720** |
| `0x1AF000`–`0x1FFFFF` | 332 KB | EMPTY | End of firmware |

### Observations
- The firmware is **neither compressed nor encrypted** as a whole — the code is readable and can be analyzed directly.
- The high entropy at `0x039000`–`0x0B4FFF` corresponds to compiled ARM code + strings.
- The region `0x120000`–`0x168FFF` contains the **resource table and assets** (not code).
- The two JPEGs at `0x180000` and `0x1A5800` are isolated in 36 KB regions.

---

## 5. Relevant Strings

ASCII extraction (≥4 characters): **7,251 strings**. Main groups:

### 5.1 SDK Identification

| Offset | String |
|--------|--------|
| `0x0F8BF4` | `F:\gp15\GPCV1235F_WIFI_SingleCam_SDK_20191212\wifi\ssv6158\host\core\host_apis.c` |
| `0x0F8DB4` | `...\wifi\ssv6158\host\core\host_cmd_engine.c` |
| `0x0F9180` | `...\wifi\ssv6158\host\drv\ssv_drv.c` |
| `0x0F96D0` | `...\wifi\ssv6158\host\mlme\ap\ap_sta_info.c` |
| `0x0FAD64` | `...\wifi\ssv6158\host\drv\sdio\sdio_if_impl.c` |
| `0x0FE160` | `...\wifi\ssv6158\host\hal\ssv6020\ssv6020_pkt.c` |

> **Conclusion:** the firmware was built from the **Generalplus GPCV1235F SingleCam SDK
> 20191212**, with the **SSV6158** WiFi stack (SSV6XXX), on a Windows host (`F:\gp15\...`).

### 5.2 Generalplus libraries and parsers

| Offset | String |
|--------|--------|
| `0x07D0EC` | `GP$308 Generalplus AVI Parser 2014.04.15` |
| `0x07FD94` | `GP$221 Generalplus Multi-Media Parser 2016.09.07` |
| `0x08C540` | `GP$322 Generalplus AviPackerV3 20140916` |
| `0x097548` | `GP$ v1.5.0 AE AWB library by Comi 20200318` |
| `0x0B7764` | `GPLIB_VIDMUX_SVN_VERSION: %s` |
| `0x0B7784` | `2018/08/30 14:51:44` (GPLIB_VIDMUX build) |

### 5.3 Camera features

| Offset | String | Function |
|--------|--------|----------|
| `0x0093AC` | `SPI_%02d.JPG` | Photo filename |
| `0x0033C8` | `C:\SPIDCIM` | Photo directory |
| `0x01CB8C` | `state_video_record_entry` | Video recording |
| `0x01046C` | `video_decode_jpeg` | JPEG decoding |
| `0x010658` | `video_decode_jpeg_read_file` | JPEG file reading |
| `0x0B46D4` | `VideoHandler` | Video handler |
| `0x0B6CF8` | `vidMP4VideoPack` | MP4 packing |
| `0x07A8C4` | `GPSOCK_Firmware_Download_Decode` | Firmware download |
| `0x001B70` | `nvspiflash_retrieval` | SPI flash retrieval |

### 5.4 WiFi (SSV6158)

| Offset | String |
|--------|--------|
| `0x062D14` | `Wifilib version:%s` |
| `0x0AC965` | `[35mHOST VERSION: %s FW_VERSION: %s` |
| `0x0465F4` | `ssv6xxx_dev_init` |
| `0x0A22AC` | `ssv6xxx_wifi_init` |
| `0x0A57DC` | `ssv6xxx_set_key` |
| `0x0A9B34` | `ssv6xxx_drv_init` |
| `0x0BF630` | `interface = SPI` |

### 5.5 Assets and UI

| Offset | String |
|--------|--------|
| `0x006294` | `BATTERY00.GPZP` |
| `0x00ED68` | `F_%d_%03d.GPZP` (UI frames) |
| `0x014100` | `PV0%02d.GPZP` (preview) |
| `0x025254` | `AP_ICON.GPZP` |
| `0x025238` | `HOME_SMALL.GPZP` |
| `0x078204` | `PLAYER_BG.GPZP` |
| `0x0B7784` | `2018/08/30 14:51:44` |
| `0x00BCCC` | `2010-06-29` (old date, possibly a third-party lib) |

---

## 6. Embedded Filesystems

### 6.1 No traditional filesystem

A search for FAT12/16/32, EXT2/3/4, JFFS2, YAFFS, UBI, SquashFS, CramFS,
ROMFS, LittleFS and SPIFFS signatures returned **only false positives** (strings inside code,
not valid FS structures).

> **Conclusion:** the firmware **does not contain a mountable filesystem**. Resources are
> stored in a proprietary "GP+" container with individual assets.

### 6.2 "GP+" resource table (`0x120000`)

Proprietary Generalplus container:

```
120000: 47 50 2B 00  03 00 03 00  00 00 00 00  00 00 00 00   GP+.............
120010: 00 00 00 00  00 00 00 00  41 50 5F 49  43 4F 4E 47   ........AP_ICONG
120020: 50 5A 50 00  00 00 00 00  00 00 00 00  03 00 00 00   PZP.............
120030: 41 55 44 49  4F 5F 52 45  43 5F 42 47  4A 50 47 00   AUDIO_REC_BGJPG.
120040: 00 00 00 00  04 00 00 00  42 41 43 4B  47 52 4F 55   ........BACKGROU
120050: 4E 44 47 50  5A 50 00 00  00 00 00 00  0D 00 00 00   NDGPZP..........
```

**Observed structure:**
- Magic: `"GP+\0"` at `0x120000`
- Header: `03 00 03 00` (version/format)
- **24-byte** entries: name (16 bytes, ASCII) + 4 zero bytes + ID (4 bytes LE)
- Total: **31 resources** listed (ends at `0x12041C`)

**Mapped resources:**

| ID | Name | Type |
|----|------|------|
| `0x03` | `AP_ICON.GPZP` | App icon |
| `0x04` | `AUDIO_REC_BG.JPG` | Audio recording background |
| `0x0D` | `BACKGROUND.GPZP` | Wallpaper |
| `0x0E` | `BATTERY00.GPZP` | Battery icon |
| `0x0F` | `BEEP.WAV` | Beep sound |
| `0x17` | `BROWSE_BK.GPZP` | Browse background |
| `0x19` | `CLICK.WAV` | Click sound |
| `0x1D` | `DAY_SELECTBAR.GPZP` | Day selection bar |
| `0x1E` | `DET_BAR.GPZP` | Detail bar |
| `0x1F` | `DET_SUB.GPZP` | Sub-detail |
| `0x20`–`0x43` | `F_0_000`–`F_3_002.GPZP` | UI frames (12 files) |
| `0x010A` | `GPRS.PAK` | GPRS package |
| `0x01FB` | `HOME_SMALL.GPZP` | Home icon |
| `0x01FD` | `ICON_PLAY.GPZP` | Play icon |
| `0x01FE` | `IMG.WAV` | Image sound |
| `0x0209` | `INSERTSD.GPZP` | SD card icon |
| `0x020A` | `PC_CAM.GPZ` | Webcam mode |
| `0x020B` | `POWER_ON_LOGO.JPG` | Boot logo |
| `0x0212` | `POWEROFF_AUDIO.WAV` | Power-off sound |
| `0x0223` | `POWERON_AUDIO.WAV` | Power-on sound |
| `0x022F`–`0x0235` | `PV000`–`PV003.GPZP` | Previews (4 files) |
| `0x0238` | `SELECTBAR_LONG.GPZP` | Long selection bar |
| `0x0239` | `SELECTBAR_MIDD.GPZP` | Medium selection bar |
| `0x023A` | `SELECTBOX_SUB.GPZP` | Selection box |
| `0x023B` | `SELECTBOX_SUB2.GPZP` | Selection box 2 |
| `0x023C` | `TIME_SELECT.GPZP` | Time selection |
| `0x023D` | `U_DISK.GPZ` | USB mode |
| `0x023E` | `UNSEL.GPZ` | Unselected state |
| `0x0240` | `USB_CONNECT.GPZP` | USB icon |

### 6.3 GPZP format (proprietary compressed)

`.GPZP` assets are stored with the `"GPZP"` magic followed by high-entropy compressed
data. The exact compression format has **not yet been identified** — it does not
match zlib, gzip, LZ4, LZMA or ZSTD. It is probably a proprietary
Generalplus codec (possibly RLE + custom Huffman).

**GPZP asset example** (`0x120600`):
```
120600: 47 50 5A 50  E3 FC C6 39  0A 47 E1 28  1C 30 C8 80   GPZP...9.G.(.0..
...
1207F0: 00 FF FF FF  FF FF FF FF  FF FF FF FF  FF FF FF FF   ................
```

The footer `... 00 64 00 00 FF FF ...` suggests a size/checksum field before the
padding.

---

## 7. Embedded Assets

### 7.1 JPEG Images

| Offset | Size | Dimensions | Format | Likely resource |
|--------|------|------------|--------|-----------------|
| `0x0FFB78` | 45,479 B | 640×480 | JFIF/EXIF | Test photo / splash |
| `0x107448` | 14,551 B | 640×480 | EXIF thumb | Embedded thumbnail |
| `0x1079D8` | 13,127 B | 640×480 | EXIF thumb | Embedded thumbnail |
| `0x107C48` | 12,503 B | 640×480 | EXIF thumb | Embedded thumbnail |
| `0x120800` | 4,358 B | 160×80 | JFIF/EXIF | `AUDIO_REC_BG.JPG` |
| `0x161600` | 3,532 B | — | JFIF/EXIF (Ducky) | UI asset |
| `0x180000` | 38,478 B | 960×720 | JFIF (`GPEncoder`) | Large image |
| `0x1A5800` | 38,286 B | 960×720 | JFIF | Large image |

> The JPEGs at `0x180000` and `0x1A5800` are the largest images in the firmware (960×720) and
> contain the `GPEncoder` comment (Generalplus encoder). They are probably boot
> screens or high-resolution backgrounds.

### 7.2 WAV Audio (uncompressed PCM)

| Offset | Size | Format | Likely resource |
|--------|------|--------|-----------------|
| `0x121E00` | 4,076 B | 11025 Hz, 16-bit, mono | `BEEP.WAV` |
| `0x123200` | 1,926 B | 8000 Hz, 8-bit, mono | `CLICK.WAV` or `IMG.WAV` |
| `0x15FC00` | 5,524 B | 11025 Hz, 8-bit, mono | `POWERON_AUDIO.WAV` |

### 7.3 GPZP Assets

At least **3 GPZP blobs** identified in the resource region:
- `0x120600`–`0x1207EF` (512 B) — probably `AP_ICON.GPZP`
- `0x121A00`–`0x121BFF` (512 B) — probably `BACKGROUND.GPZP`
- `0x121C00`–`0x121DFF` (512 B) — probably `BATTERY00.GPZP`

> Most UI assets (`F_x_xxx` frames, bars, icons) are stored as compressed GPZP
> and are distributed across `0x120600`–`0x168FFF`.

---

## 8. Checksums and Integrity

### 8.1 No firmware checksum identified

Tests performed:
- 16-bit BE sum of `0x0000`–`0x0FFF`: `0x043F` (does not match any header field)
- 32-bit BE sum of `0x0000`–`0x0FFF`: `0xD11234B6` (does not match)
- 32-bit LE sum of `0x000000`–`0x1AEDFF`: `0x1C63F72F` (does not match)
- CRC32 of the whole file: `0x279830A7` (not stored in the firmware)

### 8.2 Checksum candidate fields

| Offset | Value | Status |
|--------|-------|--------|
| `0x20` | `0x027866D0` (BE) | **Unidentified** — may be a build ID, timestamp or checksum |
| `0x04` | `0x00020030` (BE) | Probably the format version |

> **Note:** field `0x20` is the only 32-bit value in the header that has not been explained.
> It may be a boot block checksum, a build ID or an encoded timestamp.
> Additional tests (CRC16, one's complement sum, Fletcher) will be needed before
> any patching.

### 8.3 Implications for patching

- **Do not modify the original firmware** without first validating field `0x20`.
- If the bootloader validates this field, a naive patch may brick the camera.
- It is recommended to test patches in `patched_test.bin` and compare against the backup.

---

## 9. Triage Conclusions

### 9.1 Identification

| Item | Value |
|------|-------|
| Manufacturer | Generalplus |
| SoC | GPCV2247F (ARM 32-bit LE) |
| Base SDK | GPCV1235F_WIFI_SingleCam_SDK_20191212 |
| WiFi stack | SSV6158 (SSV6XXX) |
| Flash | 2 MB SPI (PY25D16HB) |
| Boot container | `SPII` + `SRIy` |
| Resource container | `GP+` (table) + `GPZP` (assets) |
| Filesystem | **None** (proprietary container) |
| Encryption | **None** detected in the code |
| Compression | GPZP assets only (proprietary codec) |

### 9.2 Summary map

```
0x000000 - 0x0001FF  SPII header + RAM segment table
0x000200 - 0x00023B  ARM vectors (LDR PC)
0x000240 - 0x0003DF  Boot code (peripheral configuration)
0x0003E0 - 0x00041B  Peripheral table (GPIO, timer, ISP)
0x00041C - 0x001FFFF  Main firmware (drivers, RTOS, UI)
0x020000 - 0x0FFFFF  Application code + SSV6158 WiFi + codecs
0x100000 - 0x11FFFF  End of code + sparse data
0x120000 - 0x12041C  "GP+" resource table
0x120600 - 0x168FFF  Assets (compressed GPZP, JPEG, WAV)
0x169000 - 0x17FFFF  Padding (0xFF)
0x180000 - 0x18964E  JPEG 960×720 (GPEncoder)
0x189650 - 0x1A57FF  Padding (0xFF)
0x1A5800 - 0x1AED8E  JPEG 960×720
0x1AED8F - 0x1FFFFF  Padding (0xFF)
```

### 9.3 Recommended next steps

1. **ARM analysis in Ghidra** — load the firmware as ARM32 LE, base `0x00000000`,
   and analyze the function at `0x000FF870` (probable entry point).
2. **Decode the GPZP format** — identify the compression algorithm to extract
   the UI assets (frames, icons, bars).
3. **Map the camera driver** — locate ISP routines, JPEG capture and sensor
   control.
4. **Investigate field `0x20`** — determine whether it is a checksum before any patching.
5. **Extract the 960×720 JPEGs** — view them to identify boot/UI screens.
6. **Analyze the SSV6158 WiFi stack** — identify commands and possible attack
   surface (if the goal is security).

---

## Appendices

### A. Tools and scripts used

| Script | Function |
|--------|----------|
| `extract_strings.py` | Extraction of 7,251 ASCII strings |
| `entropy_analysis.py` | Entropy map per 4 KB block |
| `find_filesystems.py` | Filesystem signature search |
| `analyze_assets.py` | Identification of JPEGs, WAVs and dimensions |
| `parse_resource_table.py` | GP+ table mapping |
| `gpzp_analysis.py` | GPZP header analysis |
| `checksum_check.py` | Checksum tests |

### B. Generated files

| File | Content |
|------|---------|
| `strings_ascii.txt` | 7,251 strings with offsets |
| `entropy_report.txt` | Entropy per 4 KB block |
| `docs/01-triagem.md` | This report |
| `MEMORY_MAP.md` | Memory map (updated) |

---

*Report generated following the static triage methodology of the REA repository
(header → entropy → strings → containers → assets → checksums).*
