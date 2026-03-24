# WiFi 802.11 Implementation in Zephyr RTOS

Zephyr uses an **offloaded Full-MAC** design -- the OS does not implement 802.11
MAC/MLME/PHY in software. Instead, all low-level protocol operations are handled
by WiFi chipset firmware or delegated to the `wpa_supplicant` module (hostap).
Zephyr provides a unified management abstraction layer on top.

## 1. Protocol Sub-Layers (802.11 Standard View)

```
+---------------------------------------------+
|  RSNA / Key Management                      |  802.11i/WPA3: 4-way handshake, GTK, PMK
+---------------------------------------------+
|  MLME (Management Layer)                    |  Scan, Auth, Assoc, Reassoc, Roam
|  - SME (Station Management Entity)          |  Orchestration of MLME primitives
+---------------------------------------------+
|  MAC (Medium Access Control)                |  Frame format, ACK, retransmission, A-MPDU
+---------------------------------------------+
|  PHY (Physical Layer)                       |  Radio, channels, modulation, TX power
+---------------------------------------------+
```

| Protocol Layer | Zephyr Handling | Location |
|---|---|---|
| **PHY** | Entirely in WiFi chip firmware; Zephyr only configures regulatory params (TX power, channel, coexistence) | `drivers/wifi/nrf_wifi/inc/coex.h`, `wifi_mgmt_ops->channel()` |
| **MAC** | Entirely in WiFi chip firmware; Zephyr configures aggregation params for some drivers | `drivers/wifi/nrf_wifi/src/fmac_main.c` (AMPDU config) |
| **MLME** | Chip firmware (most drivers) or `wpa_supplicant` module (nrf_wifi STA mode) | `modules/hostap/`, `drivers/wifi/nrf_wifi/src/wpa_supp_if.c` |
| **RSNA** | `wpa_supplicant` handles 4-way handshake, key derivation | `modules/hostap/src/supp_api.c` |

## 2. Implementation Sub-Layers

```
+--------------------------------------------------------------+
|  Application / Shell                                         |
|  subsys/net/l2/wifi/wifi_shell.c                             |
+--------------------------------------------------------------+
|  WiFi Management Layer (wifi_mgmt)                           |
|  subsys/net/l2/wifi/wifi_mgmt.c                              |
|  - Request validation, dispatch to driver/NM                 |
|  - Event raising (NET_EVENT_WIFI_*)                          |
+--------------------------------------------------------------+
|  WiFi Network Manager (wifi_nm)            [optional]        |
|  subsys/net/l2/wifi/wifi_nm.c                                |
|  - Allows wpa_supplicant to override driver mgmt ops         |
+--------------------------------------------------------------+
|  Driver Interface (wifi_mgmt_ops / net_wifi_mgmt_offload)    |
|  include/zephyr/net/wifi_mgmt.h                              |
|  - ~40 function pointers (scan, connect, disconnect, AP,     |
|    PS, TWT, reg_domain, mode, channel, DPP, P2P, roaming)   |
+--------------------------------------------------------------+
|  WiFi Hardware Driver                                         |
|  drivers/wifi/{nrf_wifi,esp32,esp_hosted,infineon,...}       |
|  - Wraps vendor SDK / chip commands                          |
|  - Converts chip events to Zephyr net_mgmt events            |
+--------------------------------------------------------------+
|  WiFi Chipset Firmware (RPU / external MCU)                  |
|  - Full 802.11 MAC/MLME/PHY implementation                   |
+--------------------------------------------------------------+
```

When `CONFIG_WIFI_NM_WPA_SUPPLICANT` is enabled, `wpa_supplicant` registers its
own `wifi_mgmt_ops` via `wifi_nm`, overriding the driver's native management
ops. The dispatch function `get_wifi_api()` in `wifi_mgmt.c` checks for a
network manager first, then falls back to the driver.

## 3. Critical Interfaces

### 3.1 Management Operations (`struct wifi_mgmt_ops`)

`include/zephyr/net/wifi_mgmt.h` -- the central driver contract. ~40 function
pointers:

| Category | Operations |
|---|---|
| **Connection** | `scan()`, `connect()`, `disconnect()` |
| **AP Mode** | `ap_enable()`, `ap_disable()`, `ap_sta_disconnect()`, `ap_config_params()` |
| **Status** | `iface_status()`, `get_stats()`, `get_conn_params()` |
| **Power** | `set_power_save()`, `get_power_save_config()`, `set_twt()`, `set_btwt()` |
| **Regulatory** | `reg_domain()`, `mode()`, `channel()` |
| **Security** | `wps_config()`, `dpp_dispatch()`, `pmksa_flush()`, `enterprise_creds()` |
| **Roaming** | `start_11r_roaming()`, `legacy_roam()`, `cfg_11k()`, `send_11k_neighbor_request()`, `btm_query()` |
| **P2P** | `p2p_oper()` |

### 3.2 Device Registration (`struct net_wifi_mgmt_offload`)

`include/zephyr/net/wifi_mgmt.h`

```c
struct net_wifi_mgmt_offload {
    struct ethernet_api wifi_iface;          /* or offloaded_if_api */
    const struct wifi_mgmt_ops *wifi_mgmt_api;
    const void *wifi_drv_ops;               /* optional: wpa_supplicant driver ops */
};
```

Drivers register via `NET_DEVICE_DT_INST_DEFINE()` or
`ETH_NET_DEVICE_DT_INST_DEFINE()`.

### 3.3 Network Management Event System

**Request path**: `net_mgmt(NET_REQUEST_WIFI_*, ...)` -> handler in
`wifi_mgmt.c` -> `wifi_mgmt_api->op()` -> driver/supplicant

**Event path**: Driver calls `wifi_mgmt_raise_*_event()` ->
`net_mgmt_event_notify_with_info()` -> application callback

### 3.4 WPA Supplicant Interface

`drivers/wifi/nrf_wifi/inc/wpa_supp_if.h` -- bridges `wpa_supplicant` to the
nRF70 RPU firmware:

- `nrf_wifi_wpa_supp_authenticate()`, `nrf_wifi_wpa_supp_associate()`
- `nrf_wifi_wpa_supp_scan2()`
- Event callbacks: `nrf_wifi_wpa_supp_event_proc_auth_resp()`,
  `nrf_wifi_wpa_supp_event_proc_assoc_resp()`
- Cipher suite translation (WEP/TKIP/CCMP/GCMP/BIP)

### 3.5 Supplicant Event Bridge

`modules/hostap/src/supp_events.c` -- translates `wpa_supplicant` internal
events into Zephyr `net_mgmt` events via
`supplicant_send_wifi_mgmt_event()`.

## 4. State Machine Mapping: Protocol to Implementation

### 4.1 STA Connection State Machine

```
802.11 MLME States          wifi_iface_state (wifi.h)      wpa_supp wpa_state
---------------------       --------------------------      ------------------
Initial                     DISCONNECTED (0)                WPA_DISCONNECTED
MLME-SCAN.request           SCANNING (3)                    WPA_SCANNING
MLME-AUTHENTICATE.request   AUTHENTICATING (4)              WPA_AUTHENTICATING
MLME-ASSOCIATE.request      ASSOCIATING (5)                 WPA_ASSOCIATING
MLME-ASSOCIATE.response     ASSOCIATED (6)                  WPA_ASSOCIATED
4-Way Handshake             4WAY_HANDSHAKE (7)              WPA_4WAY_HANDSHAKE
Group Key Handshake         GROUP_HANDSHAKE (8)             WPA_GROUP_HANDSHAKE
Ready for data              COMPLETED (9)                   WPA_COMPLETED
```

A `BUILD_ASSERT` in `wifi.h` enforces strict numeric ordering, enabling range
checks like `state >= WIFI_STATE_ASSOCIATED` (used e.g. in the power-save
handler to refuse PS changes while connected).

### 4.2 AP State Machine

```
802.11 AP States             wifi_ap_iface_state (wifi.h)
---------------------        ---------------------------
Not initialized              UNINITIALIZED (0)
Disabled                     DISABLED (1)
Country update               COUNTRY_UPDATE (2)
ACS (Auto Channel Select)    ACS (3)
HT Scan                      HT_SCAN (4)
DFS (Radar Detection)        DFS (5)
No-IR (No Init. Radiation)   NO_IR (6)
Enabled                      ENABLED (7)
```

### 4.3 Event Propagation Path

```
WiFi Chip Firmware
  | (hardware events/callbacks)
  v
Driver (drivers/wifi/*)
  | (wpa_supp_if.c callbacks for nrf_wifi)
  v
wpa_supplicant (modules/hostap/)
  | (supp_events.c: EVENT_* -> net_mgmt event)
  v
supplicant_send_wifi_mgmt_event()
  |
  v
wifi_mgmt_raise_connect_result_event() / wifi_mgmt_raise_disconnect_result_event()
  |
  v
net_mgmt_event_notify_with_info()
  |
  v
Application net_mgmt_event_callback
```

For simpler offloaded drivers (esp_at, eswifi, winc1500), the chip firmware
handles all MLME states internally. The driver simply translates chip events
directly to `wifi_mgmt_raise_*_event()` calls -- no intermediate supplicant
layer.

### 4.4 Roaming State Machine (with supplicant)

Defined in `wifi_mgmt.c: wifi_start_roaming()`, uses a cascading fallback:

```
Roam triggered
  -> 802.11r (FT) available?      -> start_11r_roaming()
  -> else: 11k neighbor reports?  -> send_11k_neighbor_request()
  -> else: BTM supported?         -> btm_query()
  -> else: legacy roam?           -> legacy_roam()
  -> else: return -ENOTSUP
```

### 4.5 TWT (Target Wake Time) Negotiation State Checks

The `wifi_set_twt()` handler performs multi-step validation before allowing TWT
setup:

1. Operation is teardown? (bypass further checks)
2. Interface state must be `WIFI_STATE_COMPLETED`
3. IP address assigned (if `CONFIG_WIFI_MGMT_TWT_CHECK_IP`)
4. Link mode must be >= WIFI_6 (HE capable)
5. TWT capability present
