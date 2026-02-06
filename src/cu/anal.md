# ChameleonUltra Firmware Analysis & BLE→RFID Bridge Design

## Step 1: Firmware Architecture Reconnaissance

### MCU / SoC and SDK

| Component | Details |
|-----------|---------|
| **MCU** | Nordic Semiconductor **nRF52840** (ARM Cortex-M4F @ 64MHz) |
| **SDK** | Nordic nRF5 SDK v17.x |
| **SoftDevice** | S140 v7.2.0 (BLE protocol stack) |
| **Build System** | GNU Make + ARM GCC toolchain |

### RTOS / Scheduling Model

**No traditional RTOS** - Uses cooperative bare-metal scheduling:

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXECUTION MODEL                              │
├─────────────────────────────────────────────────────────────────┤
│  SoftDevice S140 (BLE stack - highest priority, interrupt-driven)│
│  ────────────────────────────────────────────────────────────── │
│  Application main loop (cooperative polling)                     │
│  ────────────────────────────────────────────────────────────── │
│  App Timer (RTC-based soft timers)                              │
│  App Scheduler (deferred event processing)                      │
│  Power Management (sleep between events)                        │
└─────────────────────────────────────────────────────────────────┘
```

### Entry Point and Initialization Flow

**Main entry**: `firmware/application/src/app_main.c:main()` (line 849)

```
main()
  ├── hw_connect_init()         // GPIO/pin configuration
  ├── fds_util_init()           // Flash Data Storage
  ├── settings_load_config()    // NV settings
  ├── init_leds()               // LED subsystem
  ├── gpio_te_init()            // GPIO Task/Event
  ├── app_timers_init()         // Soft timer framework
  ├── power_management_init()   // Power mgmt
  ├── usb_cdc_init()            // USB serial
  ├── ble_slave_init()          // ◄── BLE INITIALIZATION
  ├── bsp_timer_init/start()    // BSP timers
  ├── button_init()             // Button handling
  ├── tag_emulation_init()      // ◄── RFID INITIALIZATION
  ├── rgb_marquee_init()        // LED animations
  └── Main Loop:
      while(1) {
          lesc_event_process();
          button_press_process();
          data_frame_process();
          NRF_LOG_PROCESS();
          app_usbd_event_queue_process();
          bsp_wdt_feed();
          sleep_system_run();    // Enter low-power when idle
      }
```

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CHAMELEON ULTRA FIRMWARE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐ │
│  │   USB CDC       │  │   BLE NUS       │  │     Command Protocol        │ │
│  │   (Serial)      │  │   (Peripheral)  │  │     (dataframe.c)           │ │
│  └────────┬────────┘  └────────┬────────┘  └──────────────┬──────────────┘ │
│           │                    │                          │                 │
│           └────────────────────┴──────────────────────────┘                 │
│                                │                                            │
│                    ┌───────────┴───────────┐                                │
│                    │     app_cmd.c         │                                │
│                    │  (Command Handlers)   │                                │
│                    └───────────┬───────────┘                                │
│                                │                                            │
│     ┌──────────────────────────┼──────────────────────────┐                 │
│     │                          │                          │                 │
│     ▼                          ▼                          ▼                 │
│  ┌──────────────┐    ┌──────────────────┐    ┌─────────────────────────┐   │
│  │ Tag Emulation│    │  Reader Mode     │    │    Settings/Config      │   │
│  │ (HF + LF)    │    │  (HF + LF)       │    │    (Flash storage)      │   │
│  └──────┬───────┘    └────────┬─────────┘    └─────────────────────────┘   │
│         │                     │                                             │
│  ┌──────┴──────────────┬──────┴─────────────────┐                          │
│  │                     │                        │                          │
│  ▼                     ▼                        ▼                          │
│ ┌────────────┐  ┌────────────┐  ┌───────────────────┐  ┌────────────────┐  │
│ │  HF Stack  │  │  LF Stack  │  │  RC522 (HF Reader)│  │  LF Reader     │  │
│ │ (NFCT HW)  │  │ (PWM/ADC)  │  │   (SPI)           │  │  (ADC/GPIO)    │  │
│ └────────────┘  └────────────┘  └───────────────────┘  └────────────────┘  │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      NORDIC SOFTDEVICE S140                          │  │
│  │                   (BLE Stack + Radio Scheduler)                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      nRF52840 HARDWARE                               │  │
│  │   NFCT │ PWM │ SAADC │ SPI │ GPIO │ LPCOMP │ RTC │ Radio             │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Subsystems

| Subsystem | File(s) | Responsibility |
|-----------|---------|----------------|
| **BLE Communication** | `ble_main.c` | BLE peripheral, NUS service, pairing |
| **USB Communication** | `usb_main.c` | CDC ACM serial emulation |
| **Command Protocol** | `dataframe.c`, `app_cmd.c` | Frame parsing, command dispatch |
| **Tag Emulation** | `tag_emulation.c` | Slot management, data loading |
| **HF Protocols** | `nfc_14a.c`, `nfc_mf1.c` | ISO14443A, MIFARE Classic |
| **LF Protocols** | `lf_tag_em.c`, `em410x.c` | EM410x, HID Prox |
| **Settings** | `settings.c`, `fds_util.c` | Flash persistence |

---

## Step 2: BLE Stack Analysis

### BLE Initialization Location

**File**: `firmware/application/src/ble_main.c`

```c
// ble_main.c:770-780
void ble_slave_init(void) {
    adc_configure();           // ADC for battery
    create_battery_timer();    // Battery update timer
    ble_stack_init();          // ◄── SoftDevice enable + BLE config
    gap_params_init();         // Device name, connection params
    gatt_init();               // GATT module
    services_init();           // NUS + Battery services
    advertising_init();        // Advertising setup
    conn_params_init();        // Connection parameter module
    peer_manager_init();       // Bonding/pairing
}
```

### BLE Role Configuration (CRITICAL FINDING)

**Current configuration in `sdk_config.h`**:

```c
// sdk_config.h:11441-11454
#define NRF_SDH_BLE_PERIPHERAL_LINK_COUNT 1  // ◄── Peripheral only
#define NRF_SDH_BLE_CENTRAL_LINK_COUNT    0  // ◄── NO CENTRAL/SCANNER
#define NRF_SDH_BLE_TOTAL_LINK_COUNT      1
```

**Conclusion**: **BLE Central (Scanner) role is NOT currently implemented.**

The device operates exclusively as a BLE **Peripheral** (advertising, accepting connections). No scanning or observer functionality exists.

### BLE Event Handler

```c
// ble_main.c:405-483
static void ble_evt_handler(ble_evt_t const *p_ble_evt, void *p_context) {
    switch (p_ble_evt->header.evt_id) {
        case BLE_GAP_EVT_CONNECTED:        // Connection established
        case BLE_GAP_EVT_DISCONNECTED:     // Connection terminated
        case BLE_GAP_EVT_PHY_UPDATE_REQUEST:
        case BLE_GAP_EVT_SEC_PARAMS_REQUEST:
        case BLE_GAP_EVT_PASSKEY_DISPLAY:
        // ... no scanning events handled
    }
}
```

**Missing Events for Scanning** (would need to be added):
- `BLE_GAP_EVT_ADV_REPORT` - Advertisement report
- `BLE_GAP_EVT_SCAN_REQ_REPORT` - Scan request received
- `BLE_GAP_EVT_TIMEOUT` with `BLE_GAP_TIMEOUT_SRC_SCAN`

### Is Passive/Active Scanning Implemented?

| Feature | Status |
|---------|--------|
| Passive Scanning | **NOT implemented** |
| Active Scanning | **NOT implemented** |
| BLE Observer | **NOT implemented** |
| ADV parsing | **NOT implemented** |

### Files That Must Be Modified for Scanner Support

| File | Required Changes |
|------|------------------|
| `sdk_config.h` | Set `NRF_SDH_BLE_CENTRAL_LINK_COUNT >= 1` |
| `ble_main.c` | Add scan init, start/stop functions, ADV report handler |
| `ble_main.h` | Export scan control functions |
| **NEW**: `ble_scan.c/.h` | (Recommended) Dedicated scanner module |

### Where RSSI, MAC, ADV Payload Are Accessible

When scanning is implemented, data arrives via `BLE_GAP_EVT_ADV_REPORT`:

```c
// From SoftDevice headers
typedef struct {
    ble_gap_addr_t peer_addr;           // MAC address
    ble_gap_addr_t direct_addr;         // Direct advertising address
    int8_t         rssi;                // RSSI in dBm
    uint8_t        type;                // ADV type (connectable, scannable, etc.)
    ble_data_t     data;                // ◄── Raw ADV payload (31 bytes max)
} ble_gap_evt_adv_report_t;
```

---

## Step 3: BLE Advertisement Parsing Strategy

### Hook Point for ADV Receive Path

**Recommended approach**: Create a new module `ble_scanner.c` that:
1. Configures and starts scanning
2. Receives `BLE_GAP_EVT_ADV_REPORT` events
3. Parses AD structures
4. Filters by criteria
5. Forwards matching data to RFID bridge

```
┌─────────────────────────────────────────────────────────────────┐
│                    BLE ADV RECEIVE PATH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   SoftDevice Radio                                              │
│        │                                                        │
│        ▼                                                        │
│   BLE_GAP_EVT_ADV_REPORT                                        │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────────────────────────────┐                          │
│   │  ble_scanner_on_adv_report()    │  ◄── NEW HOOK POINT      │
│   │  (ble_scanner.c)                │                          │
│   └───────────────┬─────────────────┘                          │
│                   │                                             │
│                   ▼                                             │
│   ┌─────────────────────────────────┐                          │
│   │  parse_adv_data()               │  AD structure parser     │
│   │  - Extract AD Types             │                          │
│   │  - Filter by UUID/MfgData       │                          │
│   └───────────────┬─────────────────┘                          │
│                   │                                             │
│                   ▼                                             │
│   ┌─────────────────────────────────┐                          │
│   │  ble_rfid_bridge_enqueue()      │  Queue for RFID TX       │
│   └─────────────────────────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Non-Invasive Parsing Layer Design

**Proposed data structures**:

```c
// ble_adv_parser.h

#include <stdint.h>
#include <stdbool.h>

// BLE AD Type definitions (Bluetooth SIG)
#define AD_TYPE_FLAGS                    0x01
#define AD_TYPE_INCOMPLETE_16BIT_UUID    0x02
#define AD_TYPE_COMPLETE_16BIT_UUID      0x03
#define AD_TYPE_INCOMPLETE_32BIT_UUID    0x04
#define AD_TYPE_COMPLETE_32BIT_UUID      0x05
#define AD_TYPE_INCOMPLETE_128BIT_UUID   0x06
#define AD_TYPE_COMPLETE_128BIT_UUID     0x07
#define AD_TYPE_SHORT_LOCAL_NAME         0x08
#define AD_TYPE_COMPLETE_LOCAL_NAME      0x09
#define AD_TYPE_TX_POWER_LEVEL           0x0A
#define AD_TYPE_SERVICE_DATA_16BIT       0x16
#define AD_TYPE_SERVICE_DATA_32BIT       0x20
#define AD_TYPE_SERVICE_DATA_128BIT      0x21
#define AD_TYPE_MANUFACTURER_DATA        0xFF

// Parsed AD element
typedef struct {
    uint8_t  type;
    uint8_t  length;
    uint8_t *data;
} ble_ad_element_t;

// Parsed ADV packet structure
typedef struct {
    uint8_t           mac[6];
    int8_t            rssi;
    uint8_t           adv_type;

    // Parsed elements (up to 8 AD structures typical)
    ble_ad_element_t  elements[8];
    uint8_t           element_count;

    // Quick-access pointers (NULL if not present)
    uint8_t          *manufacturer_data;
    uint8_t           manufacturer_data_len;
    uint16_t          manufacturer_id;

    uint8_t          *service_data;
    uint8_t           service_data_len;
    uint16_t          service_uuid;

    uint8_t          *local_name;
    uint8_t           local_name_len;
} ble_parsed_adv_t;

// Filter configuration
typedef struct {
    bool     filter_by_uuid;
    uint16_t target_uuid_16;

    bool     filter_by_manufacturer_id;
    uint16_t target_manufacturer_id;

    bool     filter_by_name_prefix;
    char     name_prefix[16];

    int8_t   min_rssi;  // Minimum RSSI threshold
} ble_adv_filter_t;
```

### TLV Parsing Implementation

```c
// ble_adv_parser.c

/**
 * Parse BLE Advertisement data (Type-Length-Value format)
 *
 * ADV data format:
 *   [Length1][Type1][Data1...][Length2][Type2][Data2...]...
 *
 * Where Length = sizeof(Type) + sizeof(Data) = 1 + DataLen
 */
bool ble_adv_parse(const uint8_t *adv_data, uint8_t adv_len,
                   ble_parsed_adv_t *parsed) {

    memset(parsed, 0, sizeof(ble_parsed_adv_t));

    uint8_t offset = 0;

    while (offset < adv_len && parsed->element_count < 8) {
        uint8_t field_len = adv_data[offset];

        if (field_len == 0) {
            break;  // End of meaningful data
        }

        if (offset + field_len >= adv_len) {
            break;  // Malformed data
        }

        uint8_t ad_type = adv_data[offset + 1];
        uint8_t data_len = field_len - 1;
        const uint8_t *data = &adv_data[offset + 2];

        // Store in elements array
        ble_ad_element_t *elem = &parsed->elements[parsed->element_count];
        elem->type = ad_type;
        elem->length = data_len;
        elem->data = (uint8_t *)data;
        parsed->element_count++;

        // Quick-access shortcuts
        switch (ad_type) {
            case AD_TYPE_MANUFACTURER_DATA:
                if (data_len >= 2) {
                    parsed->manufacturer_id = data[0] | (data[1] << 8);
                    parsed->manufacturer_data = (uint8_t *)&data[2];
                    parsed->manufacturer_data_len = data_len - 2;
                }
                break;

            case AD_TYPE_SERVICE_DATA_16BIT:
                if (data_len >= 2) {
                    parsed->service_uuid = data[0] | (data[1] << 8);
                    parsed->service_data = (uint8_t *)&data[2];
                    parsed->service_data_len = data_len - 2;
                }
                break;

            case AD_TYPE_COMPLETE_LOCAL_NAME:
            case AD_TYPE_SHORT_LOCAL_NAME:
                parsed->local_name = (uint8_t *)data;
                parsed->local_name_len = data_len;
                break;
        }

        offset += field_len + 1;
    }

    return (parsed->element_count > 0);
}

/**
 * Filter ADV packet against criteria
 */
bool ble_adv_filter_match(const ble_parsed_adv_t *parsed,
                          const ble_adv_filter_t *filter) {

    // RSSI threshold
    if (parsed->rssi < filter->min_rssi) {
        return false;
    }

    // Manufacturer ID filter
    if (filter->filter_by_manufacturer_id) {
        if (parsed->manufacturer_data == NULL ||
            parsed->manufacturer_id != filter->target_manufacturer_id) {
            return false;
        }
    }

    // Service UUID filter
    if (filter->filter_by_uuid) {
        if (parsed->service_uuid != filter->target_uuid_16) {
            return false;
        }
    }

    // Name prefix filter
    if (filter->filter_by_name_prefix) {
        if (parsed->local_name == NULL) {
            return false;
        }
        size_t prefix_len = strlen(filter->name_prefix);
        if (parsed->local_name_len < prefix_len ||
            memcmp(parsed->local_name, filter->name_prefix, prefix_len) != 0) {
            return false;
        }
    }

    return true;
}
```

### Mobile Phone Advertisement Filtering Examples

```c
// Example: Filter for Apple devices (iBeacon, AirDrop, etc.)
ble_adv_filter_t apple_filter = {
    .filter_by_manufacturer_id = true,
    .target_manufacturer_id = 0x004C,  // Apple Inc.
    .min_rssi = -80
};

// Example: Filter for specific custom service UUID
ble_adv_filter_t custom_filter = {
    .filter_by_uuid = true,
    .target_uuid_16 = 0xFFF0,  // Custom service
    .min_rssi = -70
};

// Example: Filter by device name prefix
ble_adv_filter_t name_filter = {
    .filter_by_name_prefix = true,
    .name_prefix = "MyPhone",
    .min_rssi = -90
};
```

---

## Step 4: RFID Transmission Path Analysis

### Supported RFID/NFC Protocols

| Frequency | Protocol | Implementation |
|-----------|----------|----------------|
| **HF (13.56 MHz)** | ISO14443-A | `nfc_14a.c` - Full state machine |
| | MIFARE Classic | `nfc_mf1.c` - Crypto1, key auth |
| | MIFARE Ultralight/NTAG | `nfc_mf0_ntag.c` |
| **LF (125 kHz)** | EM410x | `em410x.c` - Manchester encoding |
| | HID Prox | `hidprox.c` - FSK modulation |
| | Viking | `viking.c` - ASK protocol |

### Raw RFID Frame Generation

**HF Frame Generation** (`nfc_14a.c`):

```c
// nfc_14a.c - Core TX macros

// Transmit complete bytes with CRC
#define NFC_14A_TX_BYTE_CORE(data, bytes, appendCrc, delayMode) {
    NRF_NFCT->PACKETPTR = (uint32_t)(data);
    NRF_NFCT->TXD.AMOUNT = (bytes << NFCT_TXD_AMOUNT_TXDATABYTES_Pos);
    NRF_NFCT->TXD.FRAMECONFIG =
        NFCT_TXD_FRAMECONFIG_PARITY_Msk |      // Add parity bits
        (appendCrc ? NFCT_TXD_FRAMECONFIG_CRCMODETX_Msk : 0) |
        NFCT_TXD_FRAMECONFIG_SOF_Msk;          // Add Start-of-Frame
    NRF_NFCT->TASKS_STARTTX = 1;               // Trigger TX
}

// Key TX functions:
void nfc_tag_14a_tx_bytes(uint8_t *data, uint32_t bytes, bool appendCrc);
void nfc_tag_14a_tx_bits(uint8_t *data, uint32_t bits);
void nfc_tag_14a_tx_nbit(uint8_t data, uint32_t bits);
```

**LF Frame Generation** (`em410x.c`):

```c
// em410x.c - Manchester-encoded EM410x transmission

// EM410x data format:
// - 9-bit header (0x1FF)
// - 64 bits of data (10 rows x 5 bits, last bit is row parity)
// - Column parity + stop bit

// Modulation via PWM sequences
static void em410x_generate_pwm_sequence(uint8_t *id_bytes,
                                          nrf_pwm_values_common_t *seq);
```

**LF Modulation Hardware** (`lf_125khz_radio.c`):

```c
// PWM configuration for 125 kHz carrier
static nrfx_pwm_config_t pwm_config = {
    .output_pins = { LF_ANT_DRIVER, ... },
    .base_clock  = NRF_PWM_CLK_500kHz,
    .count_mode  = NRF_PWM_MODE_UP,
    .top_value   = 4,                    // 500kHz / 4 = 125 kHz
    .load_mode   = NRF_PWM_LOAD_INDIVIDUAL,
};
```

### Timing Constraints

| Protocol | Timing Requirement | Context |
|----------|-------------------|---------|
| ISO14443A | FDT (Frame Delay Time) ~86.4us | NFCT hardware handles automatically |
| MIFARE Auth | Crypto timing windows | Interrupt context |
| EM410x | 64 RF cycles per bit @ 125kHz | PWM + PPI driven |
| HID Prox | FSK modulation timing | PWM sequence playback |

### RFID Subsystem Data Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      RFID DATA FLOW                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                     tag_emulation.c                         │      │
│   │   - Slot management (8 slots, HF+LF per slot)               │      │
│   │   - Data loading from Flash to RAM                          │      │
│   │   - tag_base_handler_map[] dispatch table                   │      │
│   └───────────────────────┬─────────────────────────────────────┘      │
│                           │                                             │
│           ┌───────────────┴───────────────┐                            │
│           ▼                               ▼                            │
│   ┌───────────────┐               ┌───────────────┐                    │
│   │   HF Path     │               │   LF Path     │                    │
│   └───────┬───────┘               └───────┬───────┘                    │
│           │                               │                            │
│           ▼                               ▼                            │
│   ┌───────────────────┐           ┌───────────────────┐                │
│   │ nfc_14a.c         │           │ lf_tag_em.c       │                │
│   │ - ISO14443A FSM   │           │ - Field detection │                │
│   │ - REQA/WUPA/SEL   │           │ - Protocol select │                │
│   └─────────┬─────────┘           └─────────┬─────────┘                │
│             │                               │                          │
│             ▼                               ▼                          │
│   ┌───────────────────┐           ┌───────────────────┐                │
│   │ Protocol Impl:    │           │ Protocol Impl:    │                │
│   │ - nfc_mf1.c       │           │ - em410x.c        │                │
│   │ - nfc_mf0_ntag.c  │           │ - hidprox.c       │                │
│   └─────────┬─────────┘           └─────────┬─────────┘                │
│             │                               │                          │
│             ▼                               ▼                          │
│   ┌───────────────────┐           ┌───────────────────┐                │
│   │ NRF_NFCT          │           │ PWM + SAADC       │                │
│   │ (HW Peripheral)   │           │ (via PPI)         │                │
│   └───────────────────┘           └───────────────────┘                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Input Format Expected by RFID Subsystem

**For HF (MIFARE Classic)**:
```c
// From nfc_mf1.c - Block data structure
typedef struct {
    uint8_t data[16];  // 16 bytes per block
} nfc_mf1_block_t;

// Full 1K card = 64 blocks = 1024 bytes
// UID is in Block 0, bytes 0-3 (single-size) or 0-6 (double-size)
```

**For LF (EM410x)**:
```c
// 5-byte ID (40 bits)
uint8_t em410x_id[5];  // e.g., {0xDE, 0xAD, 0xBE, 0xEF, 0x12}

// Loaded via lf_tag_data_loadcb()
// Generates PWM sequences for Manchester modulation
```

**For LF (HID Prox)**:
```c
// HID Prox uses Wiegand data format
typedef struct {
    uint32_t facility_code;
    uint32_t card_number;
    uint8_t  format_idx;  // Index into wiegand_format table
} hid_prox_data_t;
```

---

## Step 5: BLE -> RFID Bridging Design

### Internal Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    BLE -> RFID BRIDGE PIPELINE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  STAGE 1: BLE Scanner                                              │    │
│  │  ┌──────────────────────────────────────────────────────────────┐  │    │
│  │  │  ble_scanner.c                                               │  │    │
│  │  │  - sd_ble_gap_scan_start() with scan params                  │  │    │
│  │  │  - BLE_GAP_EVT_ADV_REPORT handler                            │  │    │
│  │  │  - Configurable scan window/interval                         │  │    │
│  │  └──────────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────┬──────────────────────────────────────────┘    │
│                            │ Raw ADV packet                                 │
│                            ▼                                                │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  STAGE 2: ADV Parser & Filter                                      │    │
│  │  ┌──────────────────────────────────────────────────────────────┐  │    │
│  │  │  ble_adv_parser.c                                            │  │    │
│  │  │  - TLV parsing of AD structures                              │  │    │
│  │  │  - Extract: UUID, Mfg Data, Service Data, Name               │  │    │
│  │  │  - Apply filter criteria                                     │  │    │
│  │  └──────────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────┬──────────────────────────────────────────┘    │
│                            │ Parsed & filtered data                        │
│                            ▼                                                │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  STAGE 3: Data Transform                                           │    │
│  │  ┌──────────────────────────────────────────────────────────────┐  │    │
│  │  │  ble_rfid_transform.c                                        │  │    │
│  │  │  - Map BLE payload -> RFID format                            │  │    │
│  │  │  - Options:                                                   │  │    │
│  │  │    a) MAC -> EM410x ID (truncate/hash 6->5 bytes)            │  │    │
│  │  │    b) Mfg Data -> MIFARE blocks                              │  │    │
│  │  │    c) Service Data -> HID Wiegand                            │  │    │
│  │  │    d) Custom mapping logic                                    │  │    │
│  │  └──────────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────┬──────────────────────────────────────────┘    │
│                            │ RFID-ready data                               │
│                            ▼                                                │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  STAGE 4: Queue & Scheduler                                        │    │
│  │  ┌──────────────────────────────────────────────────────────────┐  │    │
│  │  │  ble_rfid_bridge.c                                           │  │    │
│  │  │  - Circular buffer for pending TX                            │  │    │
│  │  │  - Rate limiting (avoid overwhelming RF)                     │  │    │
│  │  │  - Priority handling                                         │  │    │
│  │  │  - Coordinate with SoftDevice timeslots                      │  │    │
│  │  └──────────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────┬──────────────────────────────────────────┘    │
│                            │ Scheduled TX                                  │
│                            ▼                                                │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  STAGE 5: RFID Transmission                                        │    │
│  │  ┌──────────────────────────────────────────────────────────────┐  │    │
│  │  │  Existing RFID Stack                                         │  │    │
│  │  │  - tag_emulation.c (update slot data dynamically)            │  │    │
│  │  │  - nfc_14a.c / lf_tag_em.c (field-triggered TX)              │  │    │
│  │  │                                                               │  │    │
│  │  │  OR Direct TX (for active transmission):                      │  │    │
│  │  │  - lf_125khz_radio.c (PWM sequence playback)                 │  │    │
│  │  │  - rc522.c (HF reader TX via SPI)                            │  │    │
│  │  └──────────────────────────────────────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Module Interface Design

```c
// ble_rfid_bridge.h

#ifndef BLE_RFID_BRIDGE_H
#define BLE_RFID_BRIDGE_H

#include <stdint.h>
#include <stdbool.h>
#include "tag_emulation.h"

// Bridge operating modes
typedef enum {
    BRIDGE_MODE_DISABLED,           // Bridge inactive
    BRIDGE_MODE_PASSIVE_UPDATE,     // Update tag data, wait for reader
    BRIDGE_MODE_ACTIVE_TX,          // Actively transmit on BLE trigger
} ble_rfid_bridge_mode_t;

// Target RFID protocol for conversion
typedef enum {
    RFID_TARGET_EM410X,             // LF 125kHz EM410x
    RFID_TARGET_HID_PROX,           // LF 125kHz HID Prox
    RFID_TARGET_MIFARE_UID,         // HF ISO14443A UID only
    RFID_TARGET_MIFARE_BLOCK,       // HF MIFARE Classic block data
    RFID_TARGET_NTAG,               // HF NTAG/Ultralight
} rfid_target_protocol_t;

// Mapping configuration
typedef struct {
    rfid_target_protocol_t target_protocol;

    // Source field in ADV packet
    enum {
        MAP_SOURCE_MAC,             // Use BLE MAC address
        MAP_SOURCE_MFG_DATA,        // Use Manufacturer Data
        MAP_SOURCE_SERVICE_DATA,    // Use Service Data
        MAP_SOURCE_CUSTOM,          // Custom extraction
    } source_field;

    // Byte offset within source field
    uint8_t source_offset;
    uint8_t source_length;

    // Transform options
    bool    reverse_bytes;          // Endianness swap
    bool    truncate_to_fit;        // Truncate if too long
    uint8_t xor_mask;               // Optional XOR transformation

} ble_to_rfid_mapping_t;

// Bridge configuration
typedef struct {
    ble_rfid_bridge_mode_t mode;
    ble_adv_filter_t       adv_filter;
    ble_to_rfid_mapping_t  mapping;
    uint8_t                target_slot;      // Tag emulation slot to update
    uint16_t               tx_delay_ms;      // Delay between TX events
    uint16_t               scan_interval_ms; // BLE scan interval
    uint16_t               scan_window_ms;   // BLE scan window
} ble_rfid_bridge_config_t;

// Initialize the bridge module
void ble_rfid_bridge_init(void);

// Configure bridge parameters
bool ble_rfid_bridge_configure(const ble_rfid_bridge_config_t *config);

// Start/stop bridge operation
void ble_rfid_bridge_start(void);
void ble_rfid_bridge_stop(void);

// Get bridge status
typedef struct {
    bool     is_running;
    uint32_t adv_received_count;
    uint32_t adv_filtered_count;
    uint32_t rfid_tx_count;
    uint8_t  last_mac[6];
    int8_t   last_rssi;
} ble_rfid_bridge_status_t;

void ble_rfid_bridge_get_status(ble_rfid_bridge_status_t *status);

// Process function (call from main loop)
void ble_rfid_bridge_process(void);

#endif // BLE_RFID_BRIDGE_H
```

### Data Transform Examples

```c
// ble_rfid_transform.c

/**
 * Transform BLE MAC (6 bytes) -> EM410x ID (5 bytes)
 * Strategy: XOR compression
 */
void transform_mac_to_em410x(const uint8_t *mac, uint8_t *em410x_id) {
    // XOR first byte with last byte for uniqueness
    em410x_id[0] = mac[0] ^ mac[5];
    em410x_id[1] = mac[1];
    em410x_id[2] = mac[2];
    em410x_id[3] = mac[3];
    em410x_id[4] = mac[4];
}

/**
 * Transform Manufacturer Data -> MIFARE UID
 * Assumes first 4-7 bytes of mfg data contain ID
 */
void transform_mfg_to_mifare_uid(const uint8_t *mfg_data, uint8_t mfg_len,
                                  uint8_t *uid, uint8_t *uid_len) {
    // MIFARE UID can be 4, 7, or 10 bytes
    if (mfg_len >= 7) {
        memcpy(uid, mfg_data, 7);
        *uid_len = 7;
    } else if (mfg_len >= 4) {
        memcpy(uid, mfg_data, 4);
        *uid_len = 4;
    } else {
        // Pad with zeros
        memset(uid, 0, 4);
        memcpy(uid, mfg_data, mfg_len);
        *uid_len = 4;
    }
}

/**
 * Transform Service Data -> HID Prox Wiegand
 * Expects: [facility_code_2bytes][card_number_2bytes]
 */
void transform_service_to_hid(const uint8_t *service_data, uint8_t len,
                               hid_prox_data_t *hid) {
    if (len >= 4) {
        hid->facility_code = (service_data[0] << 8) | service_data[1];
        hid->card_number = (service_data[2] << 8) | service_data[3];
        hid->format_idx = 0;  // H10301 standard 26-bit
    }
}
```

### SoftDevice Coordination (Critical)

Since SoftDevice controls the radio, BLE scanning and RFID operations need coordination:

```c
// Timeslot-based approach for simultaneous BLE + LF

// Option 1: Time-division multiplexing
// - Scan BLE for X ms
// - Stop scan, do RFID TX
// - Resume scan

void bridge_time_slice_handler(void *p_context) {
    static bridge_state_t state = STATE_BLE_SCAN;

    switch (state) {
        case STATE_BLE_SCAN:
            // Check if we have pending RFID data
            if (bridge_has_pending_rfid_tx()) {
                sd_ble_gap_scan_stop();
                state = STATE_RFID_TX;
                app_timer_start(m_slice_timer, RFID_TX_DURATION, NULL);
            }
            break;

        case STATE_RFID_TX:
            // TX complete, resume scanning
            start_ble_scan();
            state = STATE_BLE_SCAN;
            app_timer_start(m_slice_timer, BLE_SCAN_DURATION, NULL);
            break;
    }
}

// Option 2: Radio timeslot API (for advanced scenarios)
// Uses sd_radio_request() to get raw radio access
// See: utils/timeslot.c for existing implementation
```

### Integration Points with Existing Code

| Existing Module | Integration Point | Purpose |
|-----------------|-------------------|---------|
| `ble_main.c` | Add scanner init in `ble_slave_init()` | Initialize scanner module |
| `sdk_config.h` | `NRF_SDH_BLE_CENTRAL_LINK_COUNT=1` | Enable Central/Observer role |
| `tag_emulation.c` | `tag_emulation_load_by_buffer()` | Dynamic data update |
| `app_cmd.c` | Add new command handlers | Bridge control via USB/BLE |
| `app_main.c` | Add `ble_rfid_bridge_process()` to main loop | Periodic processing |
| `settings.c` | Store bridge config | Persist configuration |

### File Structure for New Modules

```
firmware/application/src/
├── ble_main.c              # Modify: add scanner integration
├── ble_main.h
├── ble_scanner.c           # NEW: BLE scanning implementation
├── ble_scanner.h
├── ble_adv_parser.c        # NEW: ADV TLV parsing
├── ble_adv_parser.h
├── ble_rfid_bridge.c       # NEW: Bridge pipeline
├── ble_rfid_bridge.h
├── ble_rfid_transform.c    # NEW: Data transformation
├── ble_rfid_transform.h
└── sdk_config.h            # Modify: enable Central links
```

---

## Summary & Key Findings

### Critical Implementation Requirements

1. **SDK Configuration Change Required**:
   ```c
   // sdk_config.h - MUST CHANGE
   #define NRF_SDH_BLE_CENTRAL_LINK_COUNT 1  // Was 0
   #define NRF_SDH_BLE_TOTAL_LINK_COUNT   2  // Was 1 (peripheral + observer)
   ```

2. **RAM Allocation Increase**: Enabling Central/Observer will require additional RAM for scanning buffers (~1-2KB)

3. **Radio Time Sharing**: SoftDevice schedules both BLE (scanning/advertising) and you'll need to coordinate RFID TX during scan idle periods

### Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| RAM overflow | High | Reduce scan buffer size, optimize existing allocations |
| BLE/RFID timing conflicts | Medium | Use time-division scheduling |
| SoftDevice API limitations | Low | Observer role is well-supported in S140 |
| Power consumption increase | Medium | Implement duty-cycling for scan windows |

### Recommended Implementation Order

1. **Phase 1**: Add BLE scanner module with basic ADV reception
2. **Phase 2**: Implement TLV parser and filtering
3. **Phase 3**: Add command interface for configuration
4. **Phase 4**: Integrate with tag emulation for passive mode
5. **Phase 5**: (Optional) Active TX mode with timeslot API

### Architecture Decision: Passive vs Active TX

| Mode | Description | Complexity | Use Case |
|------|-------------|------------|----------|
| **Passive Update** | BLE data updates tag emulation slot, waits for reader field | Low | Proxying phone -> reader |
| **Active TX** | Actively transmit RFID when BLE ADV received | High | Autonomous triggering |

**Recommendation**: Start with Passive Update mode - it's simpler and leverages the existing tag emulation infrastructure without radio scheduling complexity.
