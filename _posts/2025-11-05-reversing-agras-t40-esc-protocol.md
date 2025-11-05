---
layout: post
title: "Reversing the DJI Agras T40 ESC Protocol"
date: 2025-11-05
categories: reverse-engineering hardware
tags: dji agras-t40 esc protocol rs485 embedded
---

So if you're looking to build some big quads for heavy lift applications, motors and ESCs get pretty expensive. However, you can find excess parts for DJI or XAG for really cheap. Like $150 bucks for a motor and ESC off AliExpress for a T40. These are 12S ESCs driving 48KV motors with big props. Each puts out about ~25kg of thrust at full power. However, DJI documents literally nothing. Not even wiring diagrams. So my goal is to figure out how the flight controller talks to the ESCs so I can control them independently. The idea is to have an MCU that takes PWM, or a standard digital ESC protocol like DShot or Multishot, and outputs the DJI protocol with the proper arming and throttle commands. 

## Part 1: The Drone

The Agras T40 is a heavy lift ag drone capable of lifting 50kg of payload for spraying or seeding. It has 8 motors in a coaxial octocopter config.

![DJI Motor Numbering](/assets/images/djimotornumbering.png)

There are many good videos on tearing down the drone: [YouTube Playlist](https://www.youtube.com/playlist?list=PLL_qYPl492bbeQikXkj2hTP7VNOUgR1ek)

## MITMing the ESC

Each arm of the T40 has a power and communication cable harness. The M1 & M2 arms carry power and ground for each motor, and M3 & M4 arms have power for the 2 motors and the rotary nozzle. To get connectivity to the ESC, I bought the [M1/M2 harness](https://www.aliexpress.us/item/3256805194788268.html?gatewayAdapt=glo2usa4itemAdapt) and tapped the 6 communication wires. A little probing found pins 2&3 and 5&6 had 120 ohm termination internal to the ESC and in the flight controller.

![Tapped signal wires](/assets/images/taps.png)

![Connectors going to ESC](/assets/images/to_esc.png)

![FC I/O board connections](/assets/images/tofc.png)

## Test Setup

To interface with the RS-485 bus, I built a simple SAMD21 breadboard setup with an RS-485 transceiver:

![SAMD21 RS-485 breadboard setup](/assets/images/samd21rs485.png)

Here's the setup connected to the drone on the ground for testing:

![Connectivity to drone setup](/assets/images/connecivitytodrone.png)

The Agras T40 uses 8 ESCs communicating over RS-485. I captured the power-up sequence with an oscilloscope.

## Finding Frame Boundaries

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

##  the Checksum

Tried common algorithms: simple sum, XOR, CRC-8d, CRC-16 variants. 

It's a custom CRC-16 with initial value `0x3692` and polynomial `0x8005`.


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

## Decoding the Header

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

## The Self-Test Pattern

Now it's time to look at the first few frames the drone sends on power up.

```
Frame 1:  F4 01 AC 03 AC 03 AC 03 AC 03 AC 03 AC 03 AC 03
Frame 3:  AC 03 F4 01 AC 03 AC 03 AC 03 AC 03 AC 03 AC 03
Frame 6:  AC 03 AC 03 F4 01 AC 03 AC 03 AC 03 AC 03 AC 03
```

The value `0x01F4` (500 decimal). While this happens, the reserved field has bit 6 set (`0x0040`), indicating POST mode.

At frame 18 (~160ms), the flag clears to `0x0000` and all channels settle to `0x03AC` (940 = 48V battery voltage).

## Payload Decoding

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


