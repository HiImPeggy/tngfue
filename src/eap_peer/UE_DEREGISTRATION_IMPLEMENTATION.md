# UE Deregistration Implementation for eap_vendor_test

## Overview

This document describes the UE deregistration implementation added to `eap_vendor_test.c` and `eap_vendor_test.h` based on 3GPP TS 24.501 and the reference implementation in `tngf_test.go`.

## Files Modified

### 1. `src/eap_peer/eap_vendor_test.h`

Added:
- `struct ue_deregister_context` - Deregistration context structure
- Deregistration type definitions (`DEREG_TYPE_NORMAL`, `DEREG_TYPE_NORMAL_SWITCH_OFF`)
- Function declaration: `eap_vendor_test_initiate_deregistration()`

### 2. `src/eap_peer/eap_vendor_test.c`

Added three main functions:

#### `eap_vendor_test_build_deregistration_request()`
- Builds a NAS Deregistration Request message per TS 24.501 Section 8.3.2
- Takes: `eap_vendor_test_data`, `ngksi`, `guti`, `guti_len`
- Returns: `struct wpabuf` containing the built NAS PDU
- Message structure:
  - EPD: 0x7e (5GS Mobility Management Message)
  - Security Header Type: 0x00 (Plain NAS)
  - Message Type: 69 (Deregistration Request UE-originating)
  - Deregistration Type + ngKSI
  - 5G-GUTI (LV-E format)

#### `eap_vendor_test_send_deregistration()`
- Sends the Deregistration Request over TCP connection
- Handles error cases and logging
- Updates internal counters

#### `eap_vendor_test_handle_ike_delete()`
- Receives and processes IKE INFORMATIONAL (DELETE) Request from TNGF
- Implements socket timeout for message reception
- Returns IKE INFORMATIONAL Response
- Per RFC 7296: Deleting IKE SA implicitly closes child SAs

#### `eap_vendor_test_process_deregistration_accept()`
- Receives and validates Deregistration Accept message (type 70)
- Verifies message structure:
  - EPD: 0x7e
  - Message Type: 70
- Indicates successful deregistration

#### `eap_vendor_test_initiate_deregistration()`
- Orchestrates the complete deregistration sequence
- Calls the above functions in proper order
- Provides comprehensive logging and error handling
- Main entry point for initiating UE deregistration

## Message Flow

```
UE                           TNGF/AMF
|                              |
| 1. Deregistration Req ------->| (msg 69)
|    (via TCP)                  |
|                              |
| 2. <---- IKE DELETE (IKE) ---|
|    (via UDP)                  |
|                              |
| 3. IKE RESP (empty payload)->| (via UDP)
|                              |
| 4. <-- Deregistration Accept | (msg 70)
|    (via TCP)                  |
|                              |
```

## Usage Example

### Basic Usage

To initiate UE deregistration from your EAP code:

```c
#include "eap_vendor_test.h"

/* In your EAP process handler or state machine */
struct eap_vendor_test_data *data = priv;

/* Trigger deregistration */
eap_vendor_test_initiate_deregistration(sm, data);
```

### Integration Points

1. **On user request** (e.g., from control interface):
```c
/* Handle "deregister" command */
if (strcmp(cmd, "deregister") == 0) {
    eap_vendor_test_initiate_deregistration(sm, priv);
}
```

2. **On application shutdown**:
```c
/* Before closing EAP session */
eap_vendor_test_initiate_deregistration(sm, priv);
```

3. **On network error recovery**:
```c
/* After detecting connection loss */
if (is_network_error) {
    eap_vendor_test_initiate_deregistration(sm, priv);
}
```

4. **Periodic deregistration** (with timer):
```c
/* In timer callback */
eap_vendor_test_initiate_deregistration(sm, priv);
```

## Implementation Details

### Prerequisites

The `eap_vendor_test_data` structure must have:
- `s` - UDP socket for IKE communication
- `s_tcp` - TCP socket for NAS communication
- `sin_tngf` - TNGF address information
- `k_nas_int`, `k_nas_enc` - NAS security keys (optional for this version)
- `ueid` - UE identity

### Deregistration Type

Currently implemented:
- **DEREG_TYPE_NORMAL (0x01)** - Normal deregistration without switch-off
- Available but not used: **DEREG_TYPE_NORMAL_SWITCH_OFF (0x09)** - Switch off variant

### Error Handling

The implementation includes comprehensive error checking for:
- Memory allocation failures
- Socket communication failures
- Message validation errors
- Timeout errors

All errors are logged with `wpa_printf()` for debugging.

### Logging Output

The implementation provides detailed logging at different levels:
- **MSG_INFO**: High-level flow (deregistration start/completion)
- **MSG_DEBUG**: Detailed message contents and operations
- **MSG_ERROR**: Error conditions with explanations

Example output:
```
====== UE Initiated Deregistration ======
Built Deregistration Request message (19 bytes)
Deregistration Request sent successfully (19 bytes)
--- Waiting for TNGF to send IKE INFORMATIONAL (DELETE) Request ---
Received IKE message from TNGF (XXX bytes)
--- Building and Sending IKE INFORMATIONAL Response ---
Successfully sent IKE INFORMATIONAL Response to TNGF
--- Waiting for Deregistration Accept from TNGF ---
Successfully received and validated Deregistration Accept
UE deregistration completed successfully
====== UE Deregistration Completed Successfully ======
```

## Key Features

✅ **Standards Compliance** - Follows TS 24.501 and TS 24.502 specifications  
✅ **Reference Implementation** - Based on tngf_test.go patterns  
✅ **Error Handling** - Comprehensive error checking and recovery  
✅ **Logging** - Detailed debug and info logging  
✅ **Modular Design** - Functions can be used independently  
✅ **Socket Timeout** - Configurable receive timeout (currently 5 seconds)  

## Future Enhancements

### TODO Items in Code

1. **Security Enhancement** - The current implementation uses plain NAS messages
   - Implement proper NAS encryption using k_nas_enc
   - Add MAC calculation using k_nas_int
   - Wrap with security header

2. **IKE DELETE Verification** - Enhance IKE message processing
   - Properly parse IKE header
   - Verify DELETE payload presence
   - Build complete IKE INFORMATIONAL response with proper encryption

3. **Resource Cleanup** - Add cleanup on deregistration completion
   - Close TCP connection
   - Close UDP socket
   - Clear security context
   - Close XFRM interface
   - Close GRE tunnel
   - Release child SAs

4. **Retry Logic** - Add automatic retry mechanism
   - Timeout handling
   - Retry counter
   - Exponential backoff

5. **UE-Terminated Deregistration** - Support for network-initiated deregistration
   - Handle messages 71/72 (UE-terminated)
   - Automatic SM procedure cleanup

## Testing

### Compile

```bash
cd wpa_supplicant
make
```

### Run with TNGF

1. Start TNGF service
2. Establish initial EAP/IKE connection
3. After successful registration, trigger deregistration:
   ```c
   eap_vendor_test_initiate_deregistration(sm, priv);
   ```

### Debug Output

Enable verbose logging:
```bash
wpa_supplicant -i wlan0 -d -D nl80211 -c wpa_supplicant.conf
```

## References

- **3GPP TS 24.501** - NAS protocol for 5GS
  - Section 8.3: Mobility Management procedures
  - Section 8.3.2: Deregistration procedure
  
- **3GPP TS 24.502** - Access Stratum related functions
  - Section 9.3.2: Non-Access Stratum related functions on access networks
  
- **RFC 7296** - Internet Key Exchange Protocol Version 2 (IKEv2)
  - Section 1.4.1: Deleting an IKE SA

- **tngf_test.go** - Reference test implementation

## Support

For issues or questions about the deregistration implementation:
1. Check the debug output with `MSG_DEBUG` level logging
2. Review the message flow diagram above
3. Compare with tngf_test.go implementation
4. Check the TS 24.501 specifications

## License

This implementation follows the same BSD license as the wpa_supplicant project.
