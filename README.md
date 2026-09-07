# STM32F103 4-Channel Relay Control Board (CAN-Bus Slave)

Firmware developed for the OTAGG project — a 4-channel relay board controlled over CAN-Bus using a **single CAN ID**. Designed to drive 4 separate vehicle loads (headlight, brake/stop light, right turn signal, left turn signal) from one CAN message.

## Hardware

| Component | Description |
|---|---|
| MCU | STM32F103C8T6 (Blue Pill) |
| CAN Transceiver | TJA1051 |
| Relay Driver | 4-channel 5V relay module (via logic level converter) |
| Power Supply | External |

Relays are driven with **active-low** logic: the MCU pin LOW means the relay is **active (on)**, HIGH means **inactive (off)**. All relays are set to inactive on startup.

### Pin Mapping

| Function | GPIO Pin |
|---|---|
| Headlight | PA0 |
| Stop/Brake Light | PA1 |
| Right Turn Signal | PA2 |
| Left Turn Signal | PA3 |
| Status LED | PC13 |

Since the MCU operates at 3.3V logic, a logic level converter is used both between the TJA1051 and the CAN bus and on the lines going to the relay module.

## CAN Communication Protocol

- **CAN ID:** `0x25` (Standard 11-bit ID)
- **Baud Rate:** The CAN configuration uses `Prescaler=16`, `BS1=2TQ`, `BS2=1TQ`; verify the actual resulting baud rate against your system clock.
- **Filter:** Only messages with ID `0x25` are accepted (ID-Mask filter, mask=`0x7FF`).

Instead of using separate IDs per output, control is done through a **single CAN ID**, with the target relay determined by the **byte position (index)** within the data payload:

| Byte Index | Function |
|---|---|
| `data[0]` | Headlight |
| `data[1]` | Stop/Brake Light |
| `data[2]` | Right Turn Signal |
| `data[3]` | Left Turn Signal |

Valid command values per byte:

| Value | Meaning |
|---|---|
| `0x00` | Turn off |
| `0x01` | Turn on |
| `0xFF` | Toggle (invert current state) |

> If the DLC (data length) is smaller than 4, the relays corresponding to the missing bytes are **left untouched** (fail-safe behavior).

### Example Messages

Headlight and stop light on, signals off:
```
ID: 0x25   DLC: 4   Data: 01 01 00 00
```

Toggle headlight, stop, and right signal only (DLC=3, left signal untouched):
```
ID: 0x25   DLC: 3   Data: FF FF FF
```

## Power Management

While waiting for CAN messages in the main loop, the MCU is put into **Sleep mode** via `HAL_PWR_EnterSLEEPMode` and wakes up on a CAN interrupt, reducing idle power consumption.

## Notes / Open Points

- `SystemClock_Config()` in this version is configured for **HSI + PLL (x4)**. If your board has an external 8MHz crystal (HSE) and your original `.ioc` file was HSE-based, update this function to match your actual clock tree — an incorrect clock source directly affects the CAN baud rate.
- Verify the actual CAN baud rate against your real system clock before deployment.

## Development Environment

- STM32CubeIDE / STM32 HAL libraries
- Target: STM32F103C8T6

## License

*(Add your preferred license here, e.g. MIT)*
