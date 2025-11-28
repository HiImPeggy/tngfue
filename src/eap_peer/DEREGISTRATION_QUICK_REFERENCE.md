# UE Deregistration Quick Reference

## What Was Added

### Header File: `src/eap_peer/eap_vendor_test.h`

1. **Deregistration Type Constants:**
   - `DEREG_TYPE_NORMAL` (0x01) - Normal deregistration
   - `DEREG_TYPE_NORMAL_SWITCH_OFF` (0x09) - With switch-off

2. **Deregistration Context Structure:**
   ```c
   struct ue_deregister_context {
       int state;              /* 0=init, 1=deregistering, 2=deregistered */
       u8 *k_nas_int;         /* NAS integrity key */
       u8 *k_nas_enc;         /* NAS encryption key */
       u8 ngksi;              /* NG Key Set Identifier */
       u8 *guti;              /* 5G-GUTI */
       size_t guti_length;
       u32 uplink_count;
       u32 downlink_count;
       int cipher_alg;
       int integ_alg;
   };
   ```

3. **Public Function:**
   ```c
   void eap_vendor_test_initiate_deregistration(struct eap_sm *sm, void *priv);
   ```

### Source File: `src/eap_peer/eap_vendor_test.c`

5 new functions added:

#### 1. `eap_vendor_test_build_deregistration_request()` [STATIC]
**Purpose:** Build NAS Deregistration Request message (type 69)
**Parameters:**
- `data` - eap_vendor_test_data structure
- `ngksi` - NG Key Set Identifier (u8)
- `guti` - 5G-GUTI buffer (u8*)
- `guti_len` - GUTI length (size_t)

**Returns:** `struct wpabuf*` or NULL on error
**Message Format:**
```
EPD (0x7e) | Security Header (0x00) | Message Type (69) | 
Dereg Type + ngKSI | GUTI Length (2 octets) | GUTI Value
```

#### 2. `eap_vendor_test_send_deregistration()` [STATIC]
**Purpose:** Send Deregistration Request via TCP
**Parameters:**
- `sm` - EAP state machine
- `data` - eap_vendor_test_data
- `ngksi` - NG Key Set Identifier
- `guti` - 5G-GUTI buffer
- `guti_len` - GUTI length

**Returns:** 0 on success, -1 on failure
**Actions:**
- Builds deregistration request
- Validates TCP connection
- Sends over socket
- Logs operation

#### 3. `eap_vendor_test_handle_ike_delete()` [STATIC]
**Purpose:** Handle IKE INFORMATIONAL (DELETE) from TNGF
**Parameters:**
- `sm` - EAP state machine
- `data` - eap_vendor_test_data

**Returns:** 0 on success, -1 on failure
**Actions:**
- Sets socket timeout (5 seconds)
- Receives IKE message
- Validates message
- Sends IKE INFORMATIONAL Response
- Per RFC 7296: Empty response for IKE SA deletion

#### 4. `eap_vendor_test_process_deregistration_accept()` [STATIC]
**Purpose:** Process Deregistration Accept message (type 70)
**Parameters:**
- `sm` - EAP state machine
- `data` - eap_vendor_test_data

**Returns:** 0 on success, -1 on failure
**Actions:**
- Receives Deregistration Accept
- Validates EPD and message type
- Confirms successful deregistration
- Logs completion

#### 5. `eap_vendor_test_initiate_deregistration()` [PUBLIC]
**Purpose:** Main orchestrator - Initiate complete deregistration sequence
**Parameters:**
- `sm` - EAP state machine
- `priv` - Private data (eap_vendor_test_data)

**Returns:** void
**Sequence:**
1. Calls `eap_vendor_test_send_deregistration()`
2. Calls `eap_vendor_test_handle_ike_delete()`
3. Calls `eap_vendor_test_process_deregistration_accept()`
4. Logs completion

## How to Call It

### From Any Part of the Code

```c
#include "eap_vendor_test.h"

/* Assuming you have access to the EAP state machine and private data */
struct eap_sm *sm = ...;           /* EAP state machine */
void *priv = ...;                  /* eap_vendor_test_data structure */

/* Trigger deregistration */
eap_vendor_test_initiate_deregistration(sm, priv);
```

### Integration Points

1. **User Command (from wpa_cli):**
```c
/* In your control interface handler */
if (os_strcmp(cmd, "deregister") == 0) {
    eap_vendor_test_initiate_deregistration(sm, priv);
    return os_strdup("OK");
}
```

2. **Automatic on Shutdown:**
```c
/* Before disconnecting */
eap_vendor_test_initiate_deregistration(sm, priv);
eap_supplicant_disconnect(wpa_s);
```

3. **On Network Failure:**
```c
/* When connection lost */
if (connection_failed) {
    eap_vendor_test_initiate_deregistration(sm, priv);
    try_reconnect();
}
```

## Message Flow Overview

```
┌─ UE ─────────────────────── TNGF/AMF ─┐
│                                       │
├─ Build & Send Deregistration Request (msg 69)
│  ───────────────TCP──────────────────>
│
├─ Receive IKE DELETE (IKE INFORMATIONAL DELETE)
│  <─────────────UDP───────────────────
│
├─ Send IKE Response (empty payload)
│  ───────────────UDP──────────────────>
│
├─ Receive Deregistration Accept (msg 70)
│  <─────────────TCP───────────────────
│
└─ Deregistration Complete ─────────────┘
```

## Key Requirements

### Data Structure Requirements
The `eap_vendor_test_data` must have:
- ✅ `s` - UDP socket (initialized and connected to TNGF)
- ✅ `s_tcp` - TCP socket (initialized and connected to TNGF)
- ✅ `sin_tngf` - TNGF address structure
- ✅ `ueid` - UE identity (for GUTI derivation)
- ℹ️ `k_nas_int` - NAS integrity key (for future use)
- ℹ️ `k_nas_enc` - NAS encryption key (for future use)

### Message Requirements
- ✅ Deregistration Request: 19 bytes minimum
- ✅ IKE DELETE: Varies (minimum ~28 bytes)
- ✅ Deregistration Accept: 3+ bytes

## Error Handling

All functions include error checking for:
- ✅ Memory allocation failures
- ✅ Invalid socket
- ✅ Send/receive failures
- ✅ Socket timeout
- ✅ Invalid message format

Errors are logged with `wpa_printf(MSG_ERROR, ...)`

## Logging Output

### Level: MSG_INFO
- Deregistration start
- Request sent
- IKE DELETE received
- IKE Response sent
- Deregistration completed

### Level: MSG_DEBUG
- Message content (hex dump)
- Function entry/exit
- Intermediate states

### Level: MSG_ERROR
- Failed operations
- Invalid data
- Socket errors

## Standards References

| Standard | Section | Description |
|----------|---------|-------------|
| TS 24.501 | 8.3.2 | UE-originated deregistration |
| TS 24.501 | 8.3.3 | Deregistration Accept response |
| RFC 7296 | 1.4.1 | IKE SA deletion |
| TS 24.502 | 9.3.2 | Non-Access Stratum procedures |

## Next Steps (Future Enhancements)

1. **Add Message Security** - Implement encryption and MAC
2. **IKE Parsing** - Parse IKE DELETE properly
3. **Resource Cleanup** - Close sockets and interfaces
4. **Retry Logic** - Handle timeout and retry
5. **UT Deregistration** - Support network-initiated deregistration

## Files

✅ Modified: `src/eap_peer/eap_vendor_test.h`
✅ Modified: `src/eap_peer/eap_vendor_test.c`
✅ Created: `src/eap_peer/DEREGISTRATION_CHANGES_SUMMARY.md`
✅ Created: `src/eap_peer/UE_DEREGISTRATION_IMPLEMENTATION.md`
