# Plan: Socket Transport Support for sam-desktop

Add `SocketSmackClient` (from `pybambam.smack.socket_smack`) as an alternative transport
alongside the existing BLE `SmackClient`, enabling sam-desktop to connect to `native_sim`
targets over TCP sockets.

---

## Phase 1: Make `Connection` transport-aware (`src/core/connection.py`)

### Step 1.1 — Add transport state and `connect_socket()` entry point

Add a `_transport` attribute (`"ble"` or `"socket"`) set in `__init__` (default `"ble"`).
Add a new public method:

```python
async def connect_socket(self, host: str = "localhost", tx_port: int = 7221, rx_port: int = 7222) -> None
```

This method will:
1. Swap `self._client` for a `SocketSmackClient` instance
2. Call `_register_socket_services()` (Step 1.2)
3. Run the same state-machine logic as `connect()` — set CONNECTING, create the
   `start_smack_client` task, wait for `ready_event`, set CONNECTED
4. Call `_check_discovered_services()` after ready

Note: `SocketSmackClient.start_smack_client` has a different signature than `SmackClient`:
```python
# BLE
await client.start_smack_client(device.address, ready_event=..., error_event=...)
# Socket
await client.start_smack_client(host, tx_port=..., rx_port=..., ready_event=...)
```
No `error_event` parameter exists on the socket version — omit it.

### Step 1.2 — Service registration for socket transport

`SocketSmackClient` has no `pending_nus` attribute. Add `_register_socket_services()` that
`await`s `register_sram_nus_connector(nus)` for each of the four NUS types (screpto_7726,
screpto_98b2, ses_7727, rbt_1111). Call this in `connect_socket()` before `start_smack_client`.

Existing `_register_pending_services()` (BLE path) is unchanged.

### Step 1.3 — Guard BLE-only methods

The following are called inside `connect()` and will crash or be meaningless for a socket
transport. Add `if self._transport == "ble":` guards at their call sites:

- `_patch_pybambam_event_loop()` — patches `DeviceConnection.connect_and_run`, irrelevant for socket
- `_install_packet_logging()` — wraps `notification_handler` / `write_gatt_char`, neither exists on `SocketSmackClient`
- `_install_disconnect_callback()` — calls `ble_client.set_disconnected_callback()`, Bleak-specific API

These methods themselves don't need to change — just don't call them on the socket path.

### Step 1.4 — Fix `_check_discovered_services` for socket

Remove or guard the BLE-specific block that introspects `connection_thread.client.services`
(currently lines 341–350). For the socket path, `available_sram_nus` is already populated
by `register_sram_nus_connector()` — skip the BLE service listing and go straight to the
NUS name checks. The rest of the method works identically for both transports.

---

## Phase 2: UI — socket connection surface (`src/apps/scanner_app/app.py`)

Add a collapsible "Connect via Socket" section below the existing scanner card. Contents:

- Host input field (default: `localhost`)
- TX Port input (default: `7221`)
- RX Port input (default: `7222`)
- Connect button → calls `asyncio.create_task(hub.connection.connect_socket(host, tx_port, rx_port))`

The existing connection status display and disconnect button handle any transport — no changes
needed there.

---

## Phase 3: Type annotations

`get_smack_client()` currently returns `SmackClient`. Update the return type annotation to
`SmackClient | SocketSmackClient`.

---

## Files touched

| File | Change |
|---|---|
| `src/core/connection.py` | Transport flag, `connect_socket()`, `_register_socket_services()`, guards in `_check_discovered_services` |
| `src/apps/scanner_app/app.py` | New socket connection UI section |

No changes to `connection_service.py`, `service_hub.py`, `src/core/models.py`, or any
protocol app (`screpto_app`, `ses_app`, `rbt_app`).

---

## Deferred / Out of Scope for MVP

| Item | Notes |
|---|---|
| Packet logging for socket | Would wrap `_read_loop` and `_send_smack_fragments`. Not required for MVP. |
| Socket disconnect detection | `_read_loop` raises `IncompleteReadError` on TCP drop. Could poll `_running` or hook the read loop to fire `on_disconnected`. |
| `SmackClientProtocol` | A `typing.Protocol` defining `register_sram_nus_connector`, `start_smack_client`, `disconnect` would make `Connection` properly typed without inheritance. |
| "Connect to Previous" for socket | Persist last-used host/port in `config_storage`. |
