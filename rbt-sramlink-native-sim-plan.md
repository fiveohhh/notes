# RBT SRAMLink Transport — Native Sim Support Plan

## Background

The `sramlink_socket.c` native_sim transport is protocol-agnostic — it moves raw
SRAMLink engine bytes through two TCP sockets (TX port 7221, RX port 7222). Any
subsystem that routes through `sramlink_engine_transmit()` gets native_sim support
for free.

- **Screpto** (`SCREPTO_ROUTING`) — uses `network_router` → `transport_sramlink` →
  `sramlink_engine_transmit()` → socket. Already works.
- **SES** (`SES_ROUTING`) — same path via `network_router`. Already works.
- **RBT** — hardcoded `SMACK_TYPE_BLE`, bypasses SRAMLink entirely. Does not work.

## Root Cause

`reliable_blob_transfer.c` initializes its SMACK instance with `SMACK_TYPE_BLE`:

```c
smack_factory_init_t init = {
    .handler    = rx_handler,
    .smack_uuid = 0x1111,
    .type       = SMACK_TYPE_BLE   // blocks SRAMLink path
};
```

`smack_factory` already has a `SMACK_TYPE_SRAMLINK` branch that calls
`smack_transport_sramlink_create()` — the same transport screpto/SES use. The RBT
session protocol (windowing, ACK/NACK, retry thread) is fully transport-agnostic
and calls only `smack_send_data_direct(m_smack_ptr, ...)`. Nothing in that logic
needs to change.

## Proposed Change (~20 lines total)

### 1. `core/subsys/reliable_blob_transfer/Kconfig`

Add a transport choice modeled after `SCREPTO_ROUTING` / `SCREPTO_SIMPLE`:

```kconfig
choice RBT_TRANSPORT
    bool "RBT transport backend"
    default RBT_TRANSPORT_SRAMLINK if SRAMLINK_APPLICATION
    default RBT_TRANSPORT_BLE

config RBT_TRANSPORT_BLE
    bool "BLE (phone-facing)"

config RBT_TRANSPORT_SRAMLINK
    bool "SRAMLink (device-to-device / native_sim)"
    depends on SRAMLINK_APPLICATION

endchoice
```

### 2. `core/subsys/reliable_blob_transfer/reliable_blob_transfer.c`

Make `rbt_init()` conditional:

```c
smack_factory_init_t init = {
    .handler    = rx_handler,
    .smack_uuid = 0x1111,
#if defined(CONFIG_RBT_TRANSPORT_SRAMLINK)
    .type       = SMACK_TYPE_SRAMLINK,
#else
    .type       = SMACK_TYPE_BLE,
#endif
};
```

That's it for firmware. The Kconfig default means existing BLE-only boards keep
working without any config changes.

## External Tooling

The Python test scripts will need to speak the RBT session protocol over the
existing SRAMLink socket (ports 7221/7222). The protocol is well-defined in
`reliable_blob_transfer.h` and `reliable_blob_transfer.c`:

| Message              | Direction    | Format |
|----------------------|--------------|--------|
| `RBT_MSG_SESSION_START` (0) | firmware → tool | `[0x00][total_size: 4B BE][blob_type: 1B][blocks_per_window: 1B]` |
| `RBT_MSG_SESSION_START_RESPONSE` (1) | tool → firmware | `[0x01][ACK=0/NACK=1][nack_reason if NACK]` |
| `RBT_MSG_WINDOW_START` (2) | firmware → tool | `[0x02][window_num: 4B BE]` |
| `RBT_MSG_WINDOW_START_RESPONSE` (3) | tool → firmware | `[0x03][ACK=0/NACK=1][nack_reason if NACK]` |
| `BULK_BLE_DATA` (5) | firmware → tool | `[0x05][data...]` |
| `RBT_MSG_ABORT` (4) | either | `[0x04]` |

All messages are wrapped in the standard SRAMLink socket framing:
`[1B: payload length][payload]`

## Files to Change

- `core/subsys/reliable_blob_transfer/Kconfig`
- `core/subsys/reliable_blob_transfer/reliable_blob_transfer.c`
- Python test tooling (separate effort)

## What Does NOT Change

- `sramlink_socket.c` — no changes needed
- RBT session protocol logic — fully transport-agnostic already
- Existing BLE-based boards — Kconfig default preserves current behavior
- `smack_factory.c` / `transport_sramlink.c` — already supports this
