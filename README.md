# STM32F103 4-Channel Relay Control Board (CAN-Bus Slave)

STM32F103-based firmware that acts as a CAN-Bus slave device, controlling 4 relays (Headlight, Brake, Right Turn Signal, Left Turn Signal) based on commands received over CAN.

## Features

- **CAN-Bus communication**: Command reception over standard ID `0x25`
- **4 independent relay outputs**: Headlight, Brake, Right Signal, Left Signal
- **Active Low relay driving**: Relays are active when the pin is driven `LOW`
- **Byte-based control**: Each relay is controlled by a separate byte in the CAN data payload
- **On / Off / Toggle** command support
- **Low power consumption**: The main loop enters `SLEEP` mode, woken by the CAN interrupt (WFI)
- **Status LED**: Toggled on every received CAN message (activity/heartbeat indicator)

## Hardware

| Function            | Pin       |
|---------------------|-----------|
| Headlight Relay     | PA0       |
| Brake Relay         | PA1       |
| Right Signal Relay  | PA2       |
| Left Signal Relay   | PA3       |
| Status LED          | PC13      |
| CAN1                | Default HAL pins |

> Relay outputs are configured as **Active Low**: the relay is active (energized) when the pin is `LOW`, and inactive (de-energized) when the pin is `HIGH`.

## CAN Message Format

- **CAN ID**: `0x25` (Standard ID, exact match via 16-bit mask filter)

| Byte Index | Meaning        |
|-----------|-----------------|
| `data[0]` | Headlight       |
| `data[1]` | Brake           |
| `data[2]` | Right Signal    |
| `data[3]` | Left Signal     |

### Command Values

| Value  | Meaning |
|--------|---------|
| `0x00` | Turn Off |
| `0x01` | Turn On  |
| `0xFF` | Toggle   |

Any other value is also interpreted as `Toggle`.

**Example:** Turn the headlight on, turn the brake relay off, leave the signals unchanged (toggle):

```
ID: 0x25
Data: 01 00 FF FF
```

> If the DLC (data length) is less than 4, only the relays corresponding to the bytes actually received are updated; relays with no corresponding byte are left unchanged.

## How It Works

1. On startup, `main()` configures the system clock, GPIO, and CAN peripherals.
2. A CAN filter (16-bit ID+Mask) is set up to accept only messages with ID `0x25`.
3. All relays are initialized to the off (inactive) state.
4. The CPU enters SLEEP mode via `HAL_PWR_EnterSLEEPMode` to save power; a new CAN message wakes it up and triggers the `HAL_CAN_RxFifo0MsgPendingCallback` interrupt.
5. Inside the callback, the message is read, the status LED is toggled, and if the ID matches, `Relay_Control()` is called to update the corresponding relays.

## Code Structure

- `Relay_Set()` — Sets a relay pin active/inactive (implements the Active Low logic)
- `Relay_ApplyCmd()` — Applies a received command byte (`ON`/`OFF`/`TOGGLE`) to the corresponding pin
- `Relay_Control()` — Maps CAN payload bytes to the corresponding relay pins using the `relay_map` table
- `MX_CAN_Init()` / `MX_GPIO_Init()` — Peripheral initialization functions
- `HAL_CAN_RxFifo0MsgPendingCallback()` — CAN message reception interrupt (main control flow entry point)

## Development Environment

- **MCU**: STM32F103 (project structure generated with STM32CubeIDE / STM32CubeMX)
- **HAL Library**: STM32F1xx HAL Driver
