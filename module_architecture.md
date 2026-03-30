# Cannonball Module Architecture

This document provides a comprehensive view of all modules in the Cannonball firmware system, their dependencies, and how they interconnect.

## Overview

| Category | Count | Location |
|----------|-------|----------|
| Libraries | 24 | `cannonball/libraries/` |
| Subsystems | 6 | `cannonball/subsys/` |
| Drivers | 8 | `cannonball/drivers/` |
| ZBus Channels | 13 | `cannonball/zbus_channels/` + subsys |

---

## High-Level Architecture

```mermaid
flowchart TB
    subgraph APP["Application Layer"]
        derailleur_app["Derailleur App<br/>(FD/RD)"]
        power_meter["Power Meter App"]
    end

    subgraph DF["Derailleur Framework"]
        shift["shift"]
        pathfinder["pathfinder"]
        pathrunner["pathrunner"]
        babysitting["babysitting"]
        homing["homing"]
        motion_control["motion_control"]
        calibration["calibration"]
    end

    subgraph WIRELESS["Wireless Stack"]
        sramlink_app["sramlink_application"]
        sramlink_engine["sramlink_engine"]
        sramlink_driver["sramlink_driver"]
    end

    subgraph BLE["BLE Stack"]
        sram_ble["sram_ble"]
        ble_sramfamily["ble_sramfamily"]
        ble_srambond["ble_srambond"]
    end

    subgraph CONTROLS["Wireless Controls"]
        wireless_controls["wireless_controls"]
        sach["SACH"]
        otab["OTAB_CLASSIFIER"]
    end

    subgraph DASH_GRP["DASH"]
        dash["dash"]
        smack["smack"]
        network_router["network_router"]
    end

    subgraph CORE["Core Services"]
        screpto["screpto"]
        ses["ses"]
        sid["sid"]
        app_power["app_power"]
        astra["astra"]
        event_logger["event_logger"]
    end

    subgraph CRYPTO["Cryptography"]
        crypto["sram_crypto"]
        aes["AES"]
        dh["DH"]
        eax["EAX"]
        hmac["HMAC_SHA256"]
    end

    subgraph DRIVERS["Drivers"]
        motor["motor"]
        encoder["as5055a/ma780"]
        axs_button["axs_button"]
        adxl367["adxl367"]
    end

    subgraph UI["User Interface"]
        led_ui["single_rgb_led_ui"]
    end

    %% App dependencies
    derailleur_app --> DF
    derailleur_app --> WIRELESS
    derailleur_app --> BLE
    derailleur_app --> CONTROLS
    derailleur_app --> CORE

    %% Derailleur framework
    shift --> pathfinder
    pathfinder --> pathrunner
    pathrunner --> motion_control
    babysitting --> motion_control
    homing --> motion_control
    motion_control --> motor
    motion_control --> encoder

    %% Wireless stack
    sramlink_app --> sramlink_engine
    sramlink_engine --> sramlink_driver
    sramlink_engine --> CRYPTO

    %% Controls
    wireless_controls --> sach
    wireless_controls --> otab
    sach --> sramlink_app

    %% BLE
    sram_ble --> ble_srambond
    ble_srambond --> CRYPTO

    %% DASH
    dash --> smack
    dash --> network_router
    smack --> sramlink_app
    smack --> sram_ble

    %% Core services
    sid --> screpto
    sid --> ses
    screpto --> CRYPTO
    app_power -.-> axs_button

    %% Crypto
    crypto --> aes
    crypto --> dh
    crypto --> eax
    crypto --> hmac

    %% UI
    sramlink_app -.-> led_ui
    sramlink_app -.-> axs_button
```

---

## Module Groupings by Function

### A. Shifting & Motion Control
- `derailleur_framework` (shift, pathfinder, pathrunner, babysitting, homing, motion_control)
- `drivetrain_service`
- `motor` driver
- encoder drivers (`as5055a`, `ma780`)

### B. Wireless Communication
- `sramlink_engine` (4-layer stack)
- `sramlink_application`
- `sramlink_driver`
- `sramlink_messages`
- `wireless_controls` (SACH, OTAB)

### C. Bluetooth
- `sram_ble` (advertising, SRAMBond)
- `ble_sramfamily` (GATT service)
- `ble_srambond_event_chan`

### D. Security & Config
- `cryptography` (AES, DH, EAX, HMAC)
- `screpto` (secure commands)
- `pconf` (persistent config)

### E. Data & Logging
- `ses` (events)
- `sid` (identifiable data)
- `event_logger`
- `flash_event_logger`
- `rbt` (reliable blob transfer)

### F. System Services
- `app_power` (sleep/wake)
- `watchdog`
- `time_sync`
- `battery`
- `astra` (monitoring)

### G. User Interface
- `single_rgb_led_ui`
- `axs_button` driver
- ledui (via sramlink_application)

### H. Developer/Debug
- `devcomm`
- `shell`
- `partition_cmds`
- `versions`

---

## Kconfig Dependency Chains

### SRAMLINK_ENGINE Dependencies

```mermaid
flowchart LR
    subgraph "Enabling SRAMLINK_ENGINE selects..."
        SLE[SRAMLINK_ENGINE] --> SLD[SRAMLINK_DRIVER]
        SLE --> SLM[SRAMLINK_MESSAGES]
        SLE --> EAX[SRAM_CRYPTO_EAX]
        SLE --> DH[SRAM_CRYPTO_DH]
        SLE --> AES[SRAM_CRYPTO_AES]
        SLE --> NVM[SRAMLINK_NETWORK_NVM_STORAGE]
        SLE --> SMF[SMF]
        SLE --> ZBUS[ZBUS]
    end

    subgraph "NVM Storage selects..."
        NVM --> FLASH[FLASH]
        NVM --> FLASH_MAP[FLASH_MAP]
        NVM --> SETTINGS[SETTINGS]
        NVM --> ZMS[ZMS]
    end

    subgraph "SRAMLINK_DRIVER selects..."
        SLD --> EVENTS[EVENTS]
        SLD --> MPSL[MPSL]
    end

    subgraph "Crypto algorithms select..."
        EAX --> CRYPTO[SRAM_CRYPTO]
        DH --> CRYPTO
        AES --> CRYPTO
    end
```

### SCREPTO Dependencies

```mermaid
flowchart LR
    subgraph "Enabling SCREPTO selects..."
        SCREPTO --> CRC[CRC]
        SCREPTO --> SMACK[SMACK]
        SCREPTO --> HMAC[SRAM_CRYPTO_HMAC_SHA256]
    end

    subgraph "HMAC_SHA256 selects..."
        HMAC --> NRF_SEC[NRF_SECURITY]
        HMAC --> PSA_HMAC[PSA_WANT_ALG_HMAC]
        HMAC --> PSA_SHA[PSA_WANT_ALG_SHA_256]
        HMAC --> MBEDTLS_HEAP[MBEDTLS_ENABLE_HEAP]
    end
```

### WIRELESS_CONTROLS Dependencies

```mermaid
flowchart LR
    subgraph "Enabling WIRELESS_CONTROLS selects..."
        WC[WIRELESS_CONTROLS] --> SACH
        WC --> OTAB[OTAB_CLASSIFIER]
    end

    subgraph "SACH selects..."
        SACH --> SETTINGS[SETTINGS]
        SACH --> FLASH[FLASH]
        SACH --> ZMS[ZMS]
        SACH --> REACTIONS[REACTIONS_LEGACY]
    end

    subgraph "OTAB_CLASSIFIER selects..."
        OTAB --> SMF[SMF]
        OTAB --> EVENTS[EVENTS]
        OTAB --> ZBUS[ZBUS]
    end
```

### SRAM_BLE_SRAMBOND Dependencies

```mermaid
flowchart LR
    subgraph "Enabling SRAM_BLE_SRAMBOND selects..."
        BOND[SRAM_BLE_SRAMBOND] --> SETTINGS[SETTINGS]
        BOND --> FLASH[FLASH]
        BOND --> CRYPTO[SRAM_CRYPTO]
        BOND --> AES[SRAM_CRYPTO_AES]
        BOND --> DH[SRAM_CRYPTO_DH]
        BOND --> EAX[SRAM_CRYPTO_EAX]
    end
```

---

## Zbus Channel Pub/Sub Flow

```mermaid
flowchart TB
    subgraph Publishers
        axs_drv["axs_button driver"]
        sramlink_drv["sramlink driver"]
        link_layer["link_layer"]
        network["network"]
        transport["transport"]
        joiner["joiner"]
        coordinator["coordinator"]
        ble_bond["ble_srambond"]
        astra_lib["astra"]
        wc["wireless_controls"]
        otab_cls["otab_classifier"]
    end

    subgraph Channels
        axs_ch[/"axs_button_chan"/]
        sl_evt_ch[/"sramlink_event_chan"/]
        ll_evt_ch[/"link_layer_event_chan"/]
        net_evt_ch[/"network_layer_event_chan"/]
        eng_evt_ch[/"sramlink_engine_event_chan"/]
        pair_evt_ch[/"sramlink_pairing_event_chan"/]
        roster_ch[/"roster_event_chan"/]
        bond_ch[/"ble_srambond_event_chan"/]
        astra_ch[/"astra_device_state_chan"/]
        activity_ch[/"activity_event_chan"/]
        action_ch[/"action_request_event_chan"/]
        otab_ch[/"otab_press_event_chan"/]
    end

    subgraph Subscribers
        sl_ux["sramlink_app_ux"]
        ll_sub["link_layer"]
        net_sub["network"]
        trans_sub["transport"]
        app_rx["sramlink_app_rx"]
        adv["ble advertising"]
        sramfam["ble_sramfamily"]
        astra_sub["astra"]
        screpto_auth["screpto_auth_srambond"]
        app_power["app_power"]
        ctrl_sync["controls_sync"]
        default_act["sach_default_actions"]
        wc_sub["wireless_controls"]
        fd_dispatch["fd action_dispatcher"]
        rd_dispatch["rd action_dispatcher"]
    end

    %% AXS Button
    axs_drv --> axs_ch --> sl_ux

    %% SRAMLink driver events
    sramlink_drv --> sl_evt_ch --> ll_sub

    %% Link layer events
    link_layer --> ll_evt_ch --> net_sub

    %% Network layer events
    network --> net_evt_ch --> trans_sub
    net_evt_ch --> joiner

    %% Engine events (to application)
    transport --> eng_evt_ch --> app_rx

    %% Pairing events
    joiner --> pair_evt_ch
    coordinator --> pair_evt_ch
    pair_evt_ch --> adv
    pair_evt_ch --> sl_ux
    pair_evt_ch --> sramfam
    pair_evt_ch --> astra_sub

    %% Roster events
    coordinator --> roster_ch
    default_act --> roster_ch
    roster_ch --> ctrl_sync
    roster_ch --> default_act

    %% BLE bond events
    ble_bond --> bond_ch --> screpto_auth
    bond_ch --> sl_ux

    %% Astra events
    astra_lib --> astra_ch

    %% Activity events (any module can publish)
    activity_ch --> app_power

    %% Wireless controls
    otab_cls --> otab_ch --> wc_sub
    wc --> action_ch --> fd_dispatch
    action_ch --> rd_dispatch
```

---

## SRAMLink Protocol Stack

```mermaid
flowchart TB
    subgraph "Application Layer"
        sramlink_app["sramlink_application"]
    end

    subgraph "Transport Layer"
        transport["transport.c"]
        transport_desc["Message framing<br/>Reliability"]
    end

    subgraph "Network Layer"
        network["network.c"]
        netmgr["netmgr (routing)"]
        rcmgr["rcmgr (protocol handlers)"]
    end

    subgraph "Link Layer"
        link_layer["link_layer.c"]
        link_desc["Packet management<br/>Deduplication"]
    end

    subgraph "Pairing Layer"
        pairing["sramlink_pairing.c"]
        coordinator["coordinator"]
        joiner["joiner"]
    end

    subgraph "Driver Layer"
        sramlink_driver["sramlink_driver"]
        radio["radio (802.15.4)"]
        timeslot["MPSL timeslot"]
    end

    sramlink_app <--> transport
    transport <--> network
    network <--> link_layer
    link_layer <--> sramlink_driver
    pairing <--> network
    sramlink_driver <--> radio
    radio <--> timeslot
```

---

## DASH Routing Architecture

```mermaid
flowchart LR
    subgraph "External"
        phone["Phone/App"]
        controller["AXS Controller"]
    end

    subgraph "BLE Transport"
        sram_nus["SRAM_NUS"]
        ble["sram_ble"]
    end

    subgraph "SRAMLink Transport"
        sl_app["sramlink_application"]
    end

    subgraph "DASH Layer"
        network_router["network_router"]
        smack["SMACK<br/>(defrag/assembly)"]
    end

    subgraph "Services"
        screpto["SCREPTO<br/>(commands)"]
        ses["SES<br/>(events)"]
    end

    phone <--> ble
    ble <--> sram_nus
    sram_nus <--> smack

    controller <--> sl_app
    sl_app <--> smack

    smack <--> network_router
    network_router <--> screpto
    network_router <--> ses
```

---

## Complete Module Reference

### Libraries (24)

| # | Module | Kconfig | Description |
|---|--------|---------|-------------|
| 1 | app_power | `APP_POWER` | Sleep/reset with system-wide notifications |
| 2 | astra | `ASTRA` | Status monitoring system |
| 3 | battery | `BATTERY` | Battery voltage reading |
| 4 | ble_sramfamily | `BLE_SRAMFAMILY` | BLE GATT service for device info |
| 5 | cryptography | `SRAM_CRYPTO` | AES, DH, EAX, HMAC-SHA256 algorithms |
| 6 | derailleur_framework | Multiple | 3-layer shifting control system |
| 7 | devcomm | `DEVCOMM` | Developer CLI shell |
| 8 | drivetrain_service | `DRIVETRAIN_SERVICE` | SID interface for drivetrain config |
| 9 | event_logger | `EVENT_LOGGER` | Multi-backend event logging |
| 10 | flash_event_logger | `FLASH_EVENT_LOGGER` | Binary logging to external flash |
| 11 | karoo | `KAROOLIB` | Karoo development library |
| 12 | partition_cmds | `NVM_PARTITION_SCREPTO` | NVM partition SCREPTO commands |
| 13 | pconf | `PCONF` | Persistent config to UICR |
| 14 | reliable_blob_transfer | `RBT` | Reliable blob transfer |
| 15 | screpto | `SCREPTO` | Secure device command handling |
| 16 | ses | `SES` | SRAM Eventing System |
| 17 | shell | (none) | SRAM shell library |
| 18 | sid | `SID` | SRAM Identifiable Data |
| 19 | single_rgb_led_ui | `SINGLE_RGB_LED_UI` | RGB LED UI control |
| 20 | smack | (via DASH) | Defragmentation/assembly |
| 21 | sramlink_messages | `SRAMLINK_MESSAGES` | Protocol message definitions |
| 22 | time_sync | `TIME_SYNC` | Wall clock time tracking |
| 23 | versions | (none) | Version info commands |
| 24 | watchdog | `SYSTEM_WATCHDOG` | System-level watchdog |

### Subsystems (6)

| # | Subsystem | Kconfig | Description |
|---|-----------|---------|-------------|
| 1 | ble | `SRAM_BLE` | SRAM-specific BLE integration |
| 2 | dash | `DASH` | Directly Addressable System Hardware |
| 3 | shell | (backends) | Shell backend infrastructure |
| 4 | sramlink_application | `SRAMLINK_APPLICATION` | App-layer SRAMLink abstraction |
| 5 | sramlink_engine | `SRAMLINK_ENGINE` | 4-layer wireless protocol stack |
| 6 | wireless_controls | `WIRELESS_CONTROLS` | Button controls with multi-device tracking |

### Drivers (8)

| # | Driver | Kconfig | Description |
|---|--------|---------|-------------|
| 1 | axs_button | `AXS_BUTTON` | Button input with tap/long-press |
| 2 | motor | `SRAM_MOTOR` | Motor control via PWM |
| 3 | sramlink | `SRAMLINK_DRIVER` | 802.15.4 radio driver |
| 4 | as5055a | `AS5055A` | AMS rotary encoder |
| 5 | as5055a_fake | `AS5055A_FAKE` | Mock encoder for testing |
| 6 | ma780 | `MPS_MA780` | MPS rotary encoder |
| 7 | sram_adxl367 | `SRAM_ADXL367` | 3-axis accelerometer |
| 8 | ps09 | `PS09` | Strain gauge interface |

### Zbus Channels (13)

| # | Channel | Location | Purpose |
|---|---------|----------|---------|
| 1 | activity_event_chan | `zbus_channels/` | Sleep timer reset |
| 2 | sramlink_pairing_event_chan | `zbus_channels/` | Pairing state |
| 3 | sramlink_engine_event_chan | `zbus_channels/` | Protocol messages |
| 4 | sramlink_event_chan | `drivers/sramlink/` | Driver events |
| 5 | roster_event_chan | `zbus_channels/` | Peer roster |
| 6 | astra_device_state_chan | `zbus_channels/` | Component state |
| 7 | ble_srambond_event_chan | `zbus_channels/` | BLE security |
| 8 | axs_button_chan | `drivers/axs_button/` | Button events |
| 9 | action_request_event_chan | `subsys/wireless_controls/` | Action dispatch |
| 10 | otab_press_event_chan | `subsys/wireless_controls/` | Button classification |
| 11 | link_layer_event_chan | `subsys/sramlink_engine/` | Link layer events |
| 12 | network_layer_event_chan | `subsys/sramlink_engine/` | Network events |
| 13 | cadence_event_chan | `apps/power_meter/` | Cadence data |
