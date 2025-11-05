---
layout: post
title: "Reversing the DJI Agras T40 ESC Protocol"
date: 2025-11-05
categories: reverse-engineering hardware
tags: dji agras-t40 esc protocol rs485 embedded
---

So if your looking to build some big quads for heavy lift applications motors and escs get pretty expencive. However, you can find exess parts for DJI or XAG for really cheep. Like $150 bucks for a motor and esc off aliexpress for a t40. These are 12S escs driving 48KV motors with big props. each put out about ~25kg of thrust at full power. However DJI documents litterly nothing. Not even wiring diagrams. So my goal is to figure out how the flight controller talks to the ESCs so I can control them independently. The ideas to to have a MCU that takes PWM, or a standard digitial esc protocol like dshot or multishot and outputs the dji protocol with the proper arming and trottle commands. 
Part 1 the drone


The Agras T40 uses 8 ESCs communicating over RS-485 at 115200 baud, 8N1. I captured the power-up sequence with an oscilloscope - about 14 seconds of data, 1,517 frames total.


## Step 1: Finding Frame Boundaries

The initial capture was just a stream of bytes:

```
AC 03 79 CA 55 1A 00 D0 A0 00 00 07 AC 03 AC 03 ...
```

**The pattern**: `0x55` kept appearing followed by structured data. This looked like a sync byte.

```
55 1A 00 D0 A0 00 00 07 AC 03 AC 03 AC 03 AC 03 AC 03 AC 03 AC 03 5B AA
55 24 00 21 A0 01 00 00 50 15 07 00 98 00 00 00 00 00 00 00 00 00 00 00 01 00 00 00 00 00 40 00 00 00 AA F1
```

Testing the hypothesis: if the byte after `0x55` is a length field...

- First frame: `0x1A` (26 bytes) → counted 26 bytes of data + 2 checksum bytes
- Second frame: `0x24` (36 bytes) → counted 36 bytes of data + 2 checksum bytes

**Frame structure discovered**:
```
[0x55] [Length] [Length bytes of data] [2-byte checksum]
```

## Step 2: Cracking the Checksum

Tried common algorithms: simple sum, XOR, CRC-8... none matched. Finally tested CRC-16 variants. This is what codex is good for. 

Custom CRC-16 with initial value `0x3692` and polynomial `0x8005`.

```python
def calculate_crc16(data):
    crc = 0x3692
    for byte in data:
        crc ^= byte
        for _ in range(8):
            if crc & 1:
                crc = (crc >> 1) ^ 0x8005
            else:
                crc >>= 1
    return crc & 0xFFFF
```

Verified on multiple frames - 100% match.

## Step 3: Decoding the Header

After the length byte, there's a consistent header structure:

```
55 1A 00 D0 A0 00 00 07 [payload...] 5B AA
   │  │  └─┬──┘ └─┬──┘ │            └─┬──┘
   │  │    │      │     │              └─ CRC-16 (little-endian)
   │  │    │      │     └─ Sequence number
   │  │    │      └─ Reserved field (2 bytes)
   │  │    └─ Command ID (2 bytes, little-endian)
   │  └─ Flags
   └─ Length
```

**Command IDs found**:
- `0xA0D0`: ESC → FC telemetry (~115 Hz)
- `0xA021`: FC → ESC commands (~12.5 Hz)
- `0x0303`: ESC initialization broadcast

**Direction detection**: The reserved field acts as a direction flag:
- `0x01 0x00` = FC sending
- `0x00 0x??` = ESC sending

## Step 4: The Self-Test Pattern

Now its time to look at the first few frames the drone sends on power up

```
Frame 1:  F4 01 AC 03 AC 03 AC 03 AC 03 AC 03 AC 03 AC 03
Frame 3:  AC 03 F4 01 AC 03 AC 03 AC 03 AC 03 AC 03 AC 03
Frame 6:  AC 03 AC 03 F4 01 AC 03 AC 03 AC 03 AC 03 AC 03
```

The value `0x01F4` (500 decimal) **walks through each channel** - a classic self-test pattern. While this happens, the reserved field has bit 6 set (`0x0040`), indicating POST mode.

At frame 18 (~160ms), the flag clears to `0x0000` and all channels settle to `0x03AC` (940 = 48V battery voltage).

## Step 5: Payload Decoding

For `0xA0D0` telemetry frames, the payload contains 8 channels of 16-bit little-endian values:

```
AC 03 AC 03 AC 03 AC 03 AC 03 AC 03 AC 03 AC 03
└─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘
 Ch0   Ch1   Ch2   Ch3   Ch4   Ch5   Ch6   Ch7
```

`0x03AC` = 940 decimal → 940 × 0.051 = 47.94V (matches 12S battery voltage)

For `0xA021` command frames, the payload includes:
- 4 throttle values (16-bit each)
- Arm flag at byte offset 15 (`0x00` = disarmed, `0x80` = armed)
- Various state and control fields

