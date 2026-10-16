# FT8 Plus Protocol

**Version**: 1.0  
**Date**: 2026-10-03  
**Author**: BI3BJU  

---

## 1. Physical Layer

- `Use standard FT8 physical layer encoding for bits77 and below`
- `bits77 = (b72 << 5) | (f2 << 3) | 6`
- `b72`: 72 bits, i.e., 9 bytes of user data.
- `b72` uses big-endian order: among the 9 bytes of user data, the first byte is in the most significant 8 bits, and the last byte is in the least significant 8 bits.
- `f2`: 2-bit frame type.
- `i3`: 3 bits, fixed to `6`.

---

## 2. Frame Type (f2)

| f2 | Type | Handling |
|---|---|---|
| `0` | Single-frame mode | 
| `1` | Continuous-frame mode | 
| `2` | Reserved ARQ | Discard |
| `3` | Reserved ARQ | Discard |

---

## 3. Data Format (b72)

### 3.1 General Rules

- Input is limited to **printable Unicode BMP characters**, excluding control characters and format characters.
- All data uses **UTF-8** encoding.
- The receiver decodes according to UTF-8; invalid sequences are replaced with `�` (U+FFFD).

---

### 3.2 Single-Frame Mode (f2=0)

**Trigger condition**: UTF-8 text length is **1~9 bytes**.

Payload format:
```text
b72 = [ UTF-8 text (1~9 bytes) ][ 0x20 padding to 9 bytes ]
```

Rules:
- Text maximum 9 bytes.
- Left-aligned, padded on the right with spaces `0x20`.
- If text length is exactly 9 bytes, no padding is added.
- No CRC8 is appended, and no EOT is appended.
- Receiver removes trailing `0x20` bytes to recover the original text.

---

### 3.3 Continuous-Frame Mode (f2=1)

**Trigger condition**: UTF-8 text length is **greater than 9 bytes**.

Payload format:
```text
b72 = [ UTF-8 text ][ 0x20 padding ][ CRC8 ][ EOT 0x04 ]
```

Rules:
- Total length must be a **multiple of 9**.
- Total length maximum `576` bytes, i.e., at most `64` frames.
- Text maximum `574` bytes.
- The last byte is fixed to `0x04`.
- Padding uses spaces `0x20`, located before CRC8.
- CRC8 is located before EOT and covers from the first payload byte to the byte before CRC8.
- CRC-8/ATM: polynomial `0x07`, initial value `0x00`, result XOR `0x00`.

**Padding calculation**:
```text
total_len   = 9 * ceil((text_len + 2) / 9)
padding_len = total_len - text_len - 2
payload     = text + 0x20 * padding_len + CRC8 + 0x04
```
- `+2` reserves one byte for CRC8 and one byte for EOT.
- The payload is then split into `total_len / 9` frames, each exactly 9 bytes.
- Receiver groups frames by frequency (±10 Hz tolerance), concatenates them, searches for `0x04` (EOT), verifies CRC8, then removes trailing `0x20` padding to recover the original text.

---

## 4. Compatibility

- When a standard FT8 endpoint receives a Plus frame with `i3=6`: silently discard it, do not display garbled text, and do not enter standard parsing.

---

## 5. Application Layer: Beacon

The beacon is an **application-level feature**, independent of the `i3=6` transparent transmission. It uses standard FT8 **free-text mode**:

- `i3 = 0`, `n3 = 0`
- Character set: base-42 (` 0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ+-./?`), maximum 13 characters.
- Beacon text format: `<CALL> PLUS`
- `<CALL>`: valid amateur callsign, 3~7 characters, structure `1~2 prefix + 1 digit + 0~4 letters`; prefix contains at least one letter.
- Transmission period: every **4 minutes** (16 × 15 s slots), at slot start.
- Receiver identifies a beacon with regex `^([A-Z0-9]{3,7}) PLUS$`.
- GUI behavior:
  - If the TX input field is **non-empty**, the beacon slot is skipped.
  - If the TX input field is **empty or shows placeholder**, it is auto-filled with `<CALL> PLUS`, transmitted, and cleared afterwards.
- Before starting beacon mode, the local callsign must be valid.