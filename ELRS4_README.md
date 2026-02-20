# ExpressLRS SPI Receiver — OTA v4 Protocol Update

> **Note:** These changes are heavily AI-assisted. Manual review and real-hardware testing are strongly recommended.

## Overview

This document describes the update of Betaflight's ExpressLRS SPI receiver driver from **OTA protocol v3** to **OTA protocol v4**, matching the **ExpressLRS 4.0.0** release.

The v3 protocol is **not compatible** with ExpressLRS 4.0 transmitters. This update is required for Betaflight to work with any ELRS 4.0 TX module.

---

## Protocol Changes (v3 → v4)

### OTA Version

```c
// v3
#define ELRS_OTA_VERSION_ID 3

// v4
#define ELRS_OTA_VERSION_ID 4
```

### CRC14 Initializer

The CRC14 seed is now computed with the version shifted into the high byte, leaving the low byte available for nonce XOR on non-sync packets:

```
v3: (UID[4] << 8) | UID[5]
v4: ((UID[4] << 8) | UID[5]) ^ (OTA_VERSION_ID << 8)
```

For **non-sync packets**, the nonce is XORed into the CRC initializer low byte before validation. This replaces the v3 mechanism of adding the FHSS slot to `crcHigh`.

For **binding mode**, the CRC init uses `OTA_VERSION_ID` (=4) instead of 0.

### Sync Packet Structure

The sync packet was restructured significantly:

| Field            | v3                          | v4                                    |
|------------------|-----------------------------|---------------------------------------|
| `rfRate`         | `rateIndex` (4-bit)         | `rfRateEnum` (full 8-bit byte)        |
| `geminiMode`     | —                           | 1 bit (new)                           |
| `otaProtocol`    | —                           | 2 bits (new)                          |
| UID match        | UID[3] + UID[4] + UID[5]   | UID[4] + UID[5] only                  |
| `rateIndex` bits | 4 bits in switchEncMode byte| Removed (replaced by `rfRateEnum`)    |

### FHSS Sync Channel

```
v3: syncChannel = (freqCount / 2) + 1
v4: syncChannel = freqCount / 2
```

### Rate Enums

Rate enums changed from contiguous integers to **band-prefixed non-contiguous** values:

| Band    | v3 Range    | v4 Range     |
|---------|-------------|--------------|
| 900 MHz | 0–3         | 0–11         |
| 2.4 GHz | (shared)    | 20–33        |

The per-band lookup functions `airRateIndexToIndex900()` / `airRateIndexToIndex24()` are replaced by a single `enumRateToIndex()` that does a table scan.

---

## Air Rate Tables

### SX127x (900 MHz) — 4 → 6 entries

| Index | Rate Enum                   | Modulation | Packet Rate |
|-------|-----------------------------|------------|-------------|
| 0     | `RATE_LORA_900_200HZ`       | LoRa       | 200 Hz      |
| 1     | `RATE_LORA_900_100HZ_8CH`   | LoRa       | 100 Hz (8ch)|
| 2     | `RATE_LORA_900_100HZ`       | LoRa       | 100 Hz      |
| 3     | `RATE_LORA_900_50HZ`        | LoRa       | 50 Hz       |
| 4     | `RATE_LORA_900_25HZ`        | LoRa       | 25 Hz       |
| 5     | `RATE_LORA_900_50HZ_DVDA`   | LoRa DVDA  | 50 Hz (×4)  |

Binding rate index: **3** (was 2)

### SX1280 (2.4 GHz) — 6 → 10 entries

| Index | Rate Enum                   | Modulation | Packet Rate  |
|-------|-----------------------------|------------|--------------|
| 0     | `RATE_FLRC_2G4_1000HZ`      | FLRC       | 1000 Hz      |
| 1     | `RATE_FLRC_2G4_500HZ`       | FLRC       | 500 Hz       |
| 2     | `RATE_FLRC_2G4_500HZ_DVDA`  | FLRC DVDA  | 500 Hz (×2)  |
| 3     | `RATE_FLRC_2G4_250HZ_DVDA`  | FLRC DVDA  | 250 Hz (×4)  |
| 4     | `RATE_LORA_2G4_500HZ`       | LoRa       | 500 Hz       |
| 5     | `RATE_LORA_2G4_333HZ_8CH`   | LoRa       | 333 Hz (8ch) |
| 6     | `RATE_LORA_2G4_250HZ`       | LoRa       | 250 Hz       |
| 7     | `RATE_LORA_2G4_150HZ`       | LoRa       | 150 Hz       |
| 8     | `RATE_LORA_2G4_100HZ_8CH`   | LoRa       | 100 Hz (8ch) |
| 9     | `RATE_LORA_2G4_50HZ`        | LoRa       | 50 Hz        |

Binding rate index: **9** (was 5)

### New `elrsModSettings_t` Fields

```c
typedef struct elrsModSettings_s {
    // ... existing fields ...
    uint8_t payloadLen;   // OTA4_PACKET_SIZE or OTA8_PACKET_SIZE
    uint8_t numOfSends;   // number of times to send each packet (DVDA)
} elrsModSettings_t;
```

---

## Packet Structure Changes

### RC Data Packet

```c
// v3
uint8_t switches : 7, ch4 : 1;

// v4
uint8_t switches : 7, isArmed : 1;
```

### Uplink Data Packet (MSP)

```c
// v3 — msp_ul
struct { uint8_t packageIndex; uint8_t payload[...]; } msp_ul;

// v4 — data_ul
struct { uint8_t packageIndex : 7, stubbornAck : 1; uint8_t payload[...]; } data_ul;
```

### Downlink Telemetry / Link Stats

```c
// v3 — tlm_dl
struct {
    uint8_t type : ELRS_TELEMETRY_SHIFT, packageIndex : (8 - ELRS_TELEMETRY_SHIFT);
    // ...
    uint8_t lq : 7, mspConfirm : 1;
} tlm_dl;

// v4 — data_dl
struct {
    uint8_t packageIndex : 7, stubbornAck : 1;
    // ...
    uint8_t lq : 7, trueDiversityAvailable : 1;
} data_dl;
```

### Packet Types

```c
// v3
ELRS_RC_DATA_PACKET = 0x00
ELRS_MSP_DATA_PACKET = 0x01
ELRS_TLM_PACKET = 0x02   // removed
ELRS_SYNC_PACKET = 0x02

// v4
ELRS_RC_DATA_PACKET = 0x00
ELRS_DATA_PACKET = 0x01       // unified MSP/data
ELRS_SYNC_PACKET = 0x02
ELRS_LINKSTATS_PACKET = 0x00  // downlink only, same as RCDATA
```

### Switch Modes

```c
// v3
SM_WIDE = 0
SM_HYBRID = 1

// v4
SM_WIDE_OR_8CH = 0
SM_HYBRID_OR_16CH = 1
SM_12CH = 2            // new
```

### Radio Types

New LR1121 radio types added:

```c
RADIO_TYPE_SX127x_LORA        // existing
RADIO_TYPE_LR1121_LORA_900    // new
RADIO_TYPE_LR1121_LORA_2G4    // new
RADIO_TYPE_LR1121_GFSK_900    // new
RADIO_TYPE_LR1121_GFSK_2G4    // new
RADIO_TYPE_LR1121_LORA_DUAL   // new
RADIO_TYPE_SX128x_LORA        // existing
RADIO_TYPE_SX128x_FLRC        // existing
```

### Frequency Domains

New 433 MHz domains added:

```c
US433   // new
US433W  // new
```

---

## Bug Fixes

### 1. OSD RF Mode Display (`1a50970`)

**Problem:** In OTA v4, 2.4 GHz rate enums start at 20, so passing the raw `enumRate` to `rxSetRfMode()` displayed values like 23–24 instead of the table index (e.g. 9 for 500 Hz LoRa).

**Fix:** Pass the table index to `rxSetRfMode()` instead of the raw enum value.

### 2. FHSS Slot in CRC Validation (`80b2613`)

**Problem:** In v3, wide switch mode added the FHSS hop slot to `crcHigh` before CRC calculation. In v4, this was replaced by nonce XOR into the CRC initializer. Keeping the old logic caused all RC data packets to fail CRC in wide switch mode (LQ 0).

**Fix:** Remove the FHSS slot addition from CRC calculation.

### 3. FHSS Sync Channel & Binding CRC (`ffbcf9b`)

**Problem:** The sync channel formula was still using v3's `(freqCount / 2) + 1`. ELRS 4.0 changed it to `freqCount / 2`. This caused the receiver to listen on the wrong frequency (LQ=0, RSSI=-130). Additionally, binding mode used CRC init of 0 instead of `OTA_VERSION_ID`.

**Fix:** Update sync channel formula and binding CRC initializer.

### 4. Telemetry CRC Nonce Off-by-One (`65d4a4d`)

**Problem:** `nonceRX` is incremented in the TICK ISR (180° out of phase with TOCK). At TOCK time when telemetry is built, `nonceRX` is one behind the ELRS OtaNonce. The TX validates downlink telemetry CRC using its (already-incremented) nonce, so every telemetry packet failed CRC.

**Fix:** Use `nonceRX + 1` in the CRC calculation for telemetry packets.

### 5. Wide Switch Mode Aux Channel Cycling (`d7707da`)

**Problem:** In v4, the TX always uses 6-bit switch values with bit 6 reserved for `stubbornAck` in Wide switch mode. The old v3 logic conditionally used 7-bit values when `currTlmDenom >= 8`, misinterpreting the `stubbornAck` bit as switch data. This caused aux channels to oscillate randomly between endpoints.

**Fix:** Always mask switch value to 6 bits (`0x3F`) with `bins=63`, and unconditionally read `stubbornAck` from bit 6 in Wide mode.

---

## Testing Checklist

- [ ] Bind and connect with ELRS 4.0 TX on 2.4 GHz (SX1280)
- [ ] Bind and connect with ELRS 4.0 TX on 900 MHz (SX127x)
- [ ] Verify telemetry (LQ, RSSI, SNR) displayed on TX
- [ ] Verify aux channel stability across all switch modes (Wide/Hybrid/12CH)
- [ ] Verify OSD RF mode display shows correct human-readable value
- [ ] Test DVDA modes (250 Hz DVDA, 500 Hz DVDA, 50 Hz DVDA)
- [ ] Test 8CH modes (100 Hz 8CH, 333 Hz 8CH)
- [ ] Confirm model match still works
- [ ] Confirm binding procedure works on both bands

## References

- [ExpressLRS 4.0.0 Release](https://github.com/ExpressLRS/ExpressLRS)
- [ELRS OTA v4 protocol commit (a39f1c50)](https://github.com/ExpressLRS/ExpressLRS/commit/a39f1c50) — FHSS sync channel change
