# Thread Direct — Implementation Design

**Baseline commit:** `fb274efe6` (upstream OpenThread `main`)  
**Spec reference:** Thread 2.0 Chapter 16 — Thread Direct  
**Status:** Implementation in progress — PR 0 (cleanup) and PR 1 (secure wake initiator/listener + guest key support) complete; PR 2+ sections track remaining work.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Terminology and Spec Quick-Reference](#2-terminology-and-spec-quick-reference)
3. [Config and Feature Flags](#3-config-and-feature-flags)
4. [Wire Format](#4-wire-format)
5. [Key Derivation](#5-key-derivation)
6. [Stack Changes: MAC Layer](#6-stack-changes-mac-layer)
7. [Stack Changes: Thread Direct Handler](#7-stack-changes-thread-direct-handler)
8. [Stack Changes: Post-Link Data Transfer](#8-stack-changes-post-link-data-transfer)
9. [Public API](#9-public-api)
10. [Platform Abstraction Layer](#10-platform-abstraction-layer)
11. [Spinel / Co-Processor Support](#11-spinel--co-processor-support)
12. [CLI Extensions](#12-cli-extensions)
13. [Open Spec Items (TBD / Unclear)](#13-open-spec-items-tbd--unclear)
14. [Pull-Request Plan](#14-pull-request-plan)

---

## 1. Overview

Thread Direct (TD) is a MAC-layer peer-to-peer link between two Thread devices that belong to the same Thread network. It allows a **Wake Initiator (WI)** to reach a deeply-sleeping **Wake Listener (WL)** without MLE-level interaction, using a compact three-frame exchange:

```
WI  ──[Wake Frame (0x54 / 0x01)]──►  WL   (broadcast or unicast, repeated at 7.5 ms)
WL  ──[TD Link Command (0x54 / 0x02)]──►  WI   (with SCA LTV + Challenge LTV)
WI  ──[Enh-ACK + Thread Header IE (0x2d)]──►  WL  (Challenge LTV echoed)
```

After this handshake, the two peers have exchanged SLW (Scheduled Listen Window) parameters and both can schedule data transmissions into each other's receive windows.

Key constraints that shape the design:
- The Enh-ACK must be generated within the IEEE 802.15.4 hardware ACK turnaround window (~192 µs on EFR32). The Challenge LTV bytes carried in it are not computed from scratch in that window; instead the WI **echoes** the received challenge bytes verbatim.
- WL listen scheduling (periodic ReceiveAt on Wake Channel 20) is already implemented in `sub_mac_wed.cpp`.
- Post-link frame scheduling is modelled directly on the existing `CslTxScheduler` pattern.

**Initial implementation scope:** Thread sleepy-to-sleepy device support, one-to-one wake, unicast-by-ExtAddress, no CoEx constraints (RAM Duration = 1). The wire format definitions and struct codecs in PR 1 are designed from the start to support the full eventual scope (group wake, WakeupId addressing, full CoEx / RAM bitmap, Advertisement Command), so later PRs can extend behavior without touching wire format infrastructure.

---

## 2. Terminology and Spec Quick-Reference

| Term | Meaning |
|------|---------|
| WI | Wake Initiator — the device that sends Wake Frames |
| WL | Wake Listener — the device listening on Wake Channel 20 |
| TD Link | The established peer-to-peer MAC link |
| SLW | Scheduled Listen Window — the periodic RX window each peer advertises |
| SCA LTV | Scheduled Channel Access LTV (Type=0x02 inside Thread Header IE) — carries SLW period, phase, and CoEx constraints |
| RAM | Radio Availability Mask — CoEx constraint bitmap encoded inside SCA LTV |
| Wake Channel | Channel 20 (spec §16.4.1) — dedicated channel for all Thread Direct signalling |
| Wake Interval | 7500 µs (spec §16.12) |
| Wake Duration | 1090 ms (spec §16.12) |
| Listen Interval | 1 000 000 µs (spec §16.12) |
| Listen Duration | 8000 µs (spec §16.12) |
| TD short address | Deferred/reserved in current chapter 16 baseline (SPEC-1365 direction); active scope uses extended-address path |
| Wake Key | HMAC-SHA256(NetworkKey, "Thread-Wake"), Key Index 129 (spec §16.5.9) |
| Thread Header IE | IEEE 802.15.4 Header IE, Element-ID = 0x2d (spec §16.5.7.2) |
| LTV | Length-Type-Value packing used inside Thread Header IE |

---

## 3. Config and Feature Flags

### 3.1 Existing flags (reused with new names)

The upstream tree at `fb274efe6` already contains the following flags in `src/core/config/wakeup.h`. These renames are now treated as applied baseline for Thread Direct terminology:

| Former name | Current Thread Direct name | Role |
|---|---|---|
| `OPENTHREAD_CONFIG_WAKEUP_COORDINATOR_ENABLE` | `OPENTHREAD_CONFIG_THREAD_DIRECT_WAKE_INITIATOR_ENABLE` | Enable WI role |
| `OPENTHREAD_CONFIG_WAKEUP_END_DEVICE_ENABLE` | `OPENTHREAD_CONFIG_THREAD_DIRECT_WAKE_LISTENER_ENABLE` | Enable WL role |
| `OPENTHREAD_CONFIG_WAKEUP_TX_INTERVAL` | `OPENTHREAD_CONFIG_THREAD_DIRECT_WAKE_INTERVAL_US` | Wake interval (default 7500 µs) |
| `OPENTHREAD_CONFIG_WAKEUP_MAX_DURATION` | `OPENTHREAD_CONFIG_THREAD_DIRECT_WAKE_DURATION_MS` | Max wake-phase duration (default 1090 ms) |
| `OPENTHREAD_CONFIG_WED_LISTEN_INTERVAL` | `OPENTHREAD_CONFIG_THREAD_DIRECT_LISTEN_INTERVAL_US` | WL listen interval (default 1 000 000 µs) |
| `OPENTHREAD_CONFIG_WED_LISTEN_DURATION` | `OPENTHREAD_CONFIG_THREAD_DIRECT_LISTEN_DURATION_US` | WL listen window (default 8000 µs) |

The P2P-era flags `OPENTHREAD_CONFIG_P2P_ENABLE` and `OPENTHREAD_CONFIG_P2P_MAX_PEERS` are replaced by:

| New flag | Default | Description |
|---|---|---|
| `OPENTHREAD_CONFIG_THREAD_DIRECT_WAKE_INITIATOR_ENABLE` | 0 | Enable WI role; guards `DirectHandler` WI path, `WakeupTxScheduler` |
| `OPENTHREAD_CONFIG_THREAD_DIRECT_WAKE_LISTENER_ENABLE` | 0 | Enable WL role; guards `sub_mac_wed` listen scheduling, `DirectHandler` WL path |
| `OPENTHREAD_CONFIG_THREAD_DIRECT_MAX_DIRECT_PEERS` | 1 | Maximum simultaneous TD peers (raise per product topology) |

### 3.2 New flags

```c
// src/core/config/thread_direct.h  (new file, replaces config/wakeup.h + config/p2p.h)

// Wake Channel (spec §16.4.1): Channel 20 is the Thread Direct Wake Channel.
#ifndef OPENTHREAD_CONFIG_THREAD_DIRECT_DEFAULT_WAKE_CHANNEL
#define OPENTHREAD_CONFIG_THREAD_DIRECT_DEFAULT_WAKE_CHANNEL 20
#endif

// SLW_MIN_DURATION: minimum advertised SLW duration in 160 µs slots (spec §16.8.3).
// 8 slots × 160 µs = 1.28 ms.
#ifndef OPENTHREAD_CONFIG_THREAD_DIRECT_SLW_MIN_DURATION_SLOTS
#define OPENTHREAD_CONFIG_THREAD_DIRECT_SLW_MIN_DURATION_SLOTS 8
#endif

// Enable multi-protocol CoEx (full RAM bitmap encoding in SCA LTV, spec §16.10.2).
// Disabled by default; initial implementation scope uses RAM Duration = 1 (no constraints).
// The wire format structs and serializers ALWAYS support the full RamBits path regardless
// of this flag; the flag gates only the DirectHandler / DirectTxScheduler CoEx scheduling logic.
#ifndef OPENTHREAD_CONFIG_THREAD_DIRECT_COEX_ENABLE
#define OPENTHREAD_CONFIG_THREAD_DIRECT_COEX_ENABLE 0
#endif

// Guest Wake Key support: allow out-of-band-provisioned raw 16-byte keys at indices 130–192.
// Enabled by default — not deferred for initial implementation.
#ifndef OPENTHREAD_CONFIG_THREAD_DIRECT_GUEST_WAKE_KEY_ENABLE
#define OPENTHREAD_CONFIG_THREAD_DIRECT_GUEST_WAKE_KEY_ENABLE 1
#endif

// Maximum number of guest wake keys that can be stored simultaneously.
#ifndef OPENTHREAD_CONFIG_THREAD_DIRECT_MAX_GUEST_WAKE_KEYS
#define OPENTHREAD_CONFIG_THREAD_DIRECT_MAX_GUEST_WAKE_KEYS 4
#endif
```

---

## 4. Wire Format

> **Scope note:** All struct definitions, constants, and serializers in this section cover the **full eventual feature set** (group wake, WakeupId addressing, full CoEx / RAM bitmap, Advertisement Command). The initial implementation behavior is constrained (one-to-one wake, RAM Duration = 1), but nothing in the wire layer needs to change to expand to the full scope later. `OPENTHREAD_CONFIG_THREAD_DIRECT_COEX_ENABLE` gates only the scheduler-level CoEx logic — not the codec.

### 4.1 IEEE 802.15.4 MAC Command 0x54

All Thread Direct frames share a single IEEE 802.15.4 MAC Command ID **`0x54`** (Thread MAC Command, pending formal IEEE 802.15.4 allocation; `0x54` is the provisional value per spec §16.14 — **see §13 item 1**). The first payload byte after `0x54` is the **Thread MAC Command ID**, which dispatches to one of three frame types:

| Thread MAC Cmd ID | Name | Direction |
|---|---|---|
| `0x00` | Advertisement Command | WI broadcast |
| `0x01` | Wake Command | WI → WL |
| `0x02` | TD Link Command | WL → WI |

Constants to add to `mac_frame.hpp`:
```cpp
static constexpr uint8_t kMacCmdDirect            = 0x54; // IEEE 802.15.4 MAC Command ID
static constexpr uint8_t kThreadMacCmdAdvertisement  = 0x00;
static constexpr uint8_t kThreadMacCmdWake           = 0x01;
static constexpr uint8_t kThreadMacCmdDirectLink         = 0x02;
```

### 4.2 Wake Command frame (Thread MAC Cmd 0x01)

```
[ MHR: FCF | Seq | DstPAN | DstAddr | SrcAddr | SrcPAN ]
[ Aux Security Header: KeyIdMode=1, KeyIndex=129 ]
[ Header IEs:
    Thread Header IE (0x2d):
        TargetId LTV (Type=0x01, 1–8 bytes) — OPTIONAL; only present when
        WakeupId addressing is used.  Omitted for unicast-by-ExtAddress wakes.
]
[ MAC Command Payload — see RFC bit diagram below ]
[ MFR: MIC-32 | FCS ]
```

**MAC Command Payload (5 bytes):**

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  MAC Cmd 0x54 |Thread Cmd 0x01|   Wake Type   | Rendezvous Tm |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| RI(4b)| RC(4b)|
+-+-+-+-+-+-+-+-+
```

| Field | Value | Description |
|-------|-------|-------------|
| MAC Cmd | `0x54` | `kMacCmdDirect`; signals Thread Direct frame |
| Thread Cmd | `0x01` | `kThreadMacCmdWake` |
| Wake Type | `0x00` | `kWakeFrameTypeDirectLink`; standard handshake |
| Rendezvous Time | uint8 | Offset in 10-symbol units; first Connection Window offset |
| RI | 4 bits | Retry Interval; upper nibble of byte 4 |
| RC | 4 bits | Retry Count; lower nibble of byte 4 |

**`WakeupTxScheduler::PrepareWakeupFrame()` changes:**
- Calls `TxFrame::GenerateThreadDirectWakeCommand()` (new) instead of the removed `GenerateWakeupFrame()`.
- Sets `KeyIdMode=1`, `KeyIndex=kDefaultWakeKeyIndex` (129), Wake Channel = 20.
- Unicast-by-ExtAddress: `DstAddr` = WL extended address; no TargetId LTV.
- WakeupId wake: `DstAddr` = 0xFFFF (broadcast); TargetId LTV written inside Thread Header IE 0x2d.

**Functions removed from `mac_frame.hpp/.cpp`:**
- `GenerateWakeupFrame()` — Multipurpose frame generator, no longer needed.
- `IsWakeupFrame()` — replaced by `IsTdWakeCommand()`.

**New `mac_frame.hpp/.cpp` functions:**
```cpp
// Returns true if this is a MAC Command 0x54 frame.
bool Frame::IsThreadDirectMacCommand() const;

// Returns the Thread MAC Command ID (byte 1 of MAC Command payload),
// or kErrorParse if not a 0x54 frame.
Error Frame::GetThreadMacCommandId(uint8_t &aId) const;

// Returns true if this is a Wake Command (Thread MAC Cmd 0x01).
bool Frame::IsTdWakeCommand() const;

// Builds a Wake Command frame in aFrame.
static void TxFrame::GenerateThreadDirectWakeCommand(TxFrame           &aFrame,
                                            uint8_t            aWakeFrameType,
                                            uint8_t            aRendezvousTime,
                                            uint8_t            aRi,
                                            uint8_t            aRc,
                                            const uint8_t     *aTargetId,    // NULL = no TargetId LTV
                                            uint8_t            aTargetIdLen); // length in bytes (1-8)

// Builds a TD Link Command frame in aFrame (see §4.3).
static void TxFrame::GenerateThreadDirectLinkCommand(TxFrame             &aFrame,
                                            uint16_t             aShortAddr,
                                            uint16_t             aSupervisionIntervalMs,
                                            uint8_t              aServicesBitmap,
                                            const ScaParams         *aSca,         // NULL = no SCA LTV
                                            const ChallengeLtv   &aChallenge);
```

### 4.3 TD Link Command frame (Thread MAC Cmd 0x02)

> **Latest chapter 16 alignment note:** short-address support is currently deferred/reserved in the active spec direction. For shipped and near-term scope, use the extended-address path and treat short-address handling as future spec follow-up.

```
[ MHR: FCF | Seq | DstPAN | DstAddr | SrcAddr | SrcPAN ]
[ Aux Security Header: same key type that secured the received Wake Frame ]
[ Header IEs:
    Thread Header IE (0x2d):
        SCA LTV (Type=0x02)      — RECOMMENDED; see §4.4
        Challenge LTV (Type=0x03) — MANDATORY; see §4.5
]
[ MAC Command Payload — see RFC bit diagram below ]
[ MFR: MIC-32 | FCS ]
```

**MAC Command Payload (all optional fields present):**

```
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  MAC Cmd 0x54 |Thread Cmd 0x02| Mask (8b) |SI |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Services (8b)|
+-+-+-+-+-+-+-+-+
```

| Field | Conditional | Description |
|-------|-------------|-------------|
| MAC Cmd | — | `0x54` (`kMacCmdDirect`) |
| Thread Cmd | — | `0x02` (`kThreadMacCmdDirectLink`) |
| Link Parameter Mask | — | 8-bit bitmask; bit 0 = supervision interval, bit 1 = services, bit 2 = reserved (future short address, must be 0) |
| Supervision Interval | bit 0 of mask | uint8; maximum idle time in units of 100 ms before link supervision must be sent |
| Services | bit 1 of mask | uint8 bitmap; bit 0 = SRP server present |

Short-address support is fully deferred: the Short Address field has been removed from the TD Link Command payload in the current spec baseline (SPEC-1365). The bit for Short Address in the Link Parameter Mask is reserved and must be zero. When the Link Parameter Mask is absent or all zero bits, the command consists only of the Command byte and exchanges SCA LTVs via Header IEs only.

**Enh-ACK reply from WI (Thread MAC Cmd 0x02 flow):**
```
[ MHR: FCF (AR=0, Frame Type=ACK) | Seq ]
[ Header IEs:
    Thread Header IE (0x2d):
        SCA LTV (Type=0x02)       — WI's own SCA schedule (when WI has a constrained
                                    receive schedule; spec permits WI to include
                                    SCA LTV in any frame including Enh-ACK)
        Challenge LTV (Type=0x03) — VERBATIM COPY of the Challenge LTV received
                                    in the TD Link Command.  The WI does NOT
                                    recompute the HMAC; it echoes the bytes.
]
[ MFR: MIC-32 | FCS ]
```

The WI pushes the echoed Challenge LTV bytes to the platform before the Enh-ACK turnaround deadline via `otPlatRadioConfigureThreadDirectEnhAckIe()` (§10.1). The platform injects the Thread Header IE into the hardware-generated Enh-ACK.

### 4.4 Thread Header IE (Element-ID = 0x2d)

A new standard IEEE 802.15.4 Header IE with Element-ID `0x2d` (spec §16.5.7.2). Its payload is a sequence of LTV-encoded elements.

> Note: This is distinct from the existing `ThreadIe` (Vendor IE with Thread OUI). The new IE uses a dedicated Element-ID in the non-IETF Header IE space.

Struct in `mac_header_ie.hpp`:
```cpp
struct ThreadHeaderIe
{
    static constexpr uint8_t kHeaderIeId = 0x2d;

    static constexpr uint8_t kTypeTargetId  = 0x01; ///< Target ID LTV (WakeupId filter)
    static constexpr uint8_t kTypeSca       = 0x02; ///< Scheduled Channel Access LTV
    static constexpr uint8_t kTypeChallenge = 0x03; ///< Thread Challenge LTV
};
```

#### LTV encoding

Each element uses the packed LTV format: `[L][T][V…]` where L is the byte-length of V, T is the type, and V is the value field.

**Target ID LTV (Type = 0x01)**

V = 1–8 bytes of the raw WakeupId value (low bytes of `uint64_t`, variable length). No fixed packed struct; serializer writes the minimum number of significant bytes.

**SCA LTV (Type = 0x02)**

The in-memory representation is `ScaParams` (not a packed wire struct). The serializer encodes/decodes the packed wire format from these fields:

```cpp
struct ScaParams
{
    static constexpr uint8_t kRamDurationNoChange      = 0;  // no change to prior RAM
    static constexpr uint8_t kRamDurationNoConstraints = 1;  // device has no CoEx constraints
    static constexpr uint8_t kRamDurationMax           = 31;
    static constexpr int16_t kRamOffsetUsMin           = -1024;
    static constexpr int16_t kRamOffsetUsMax           = 1023;

    uint16_t mSlwPeriodSlots;   // SLW Period in 160 us slots (0 = no SLW schedule)
    uint16_t mSlwPhaseSlots;    // SLW Phase in 160 us slots
    int16_t  mRamOffsetUs;      // RAM Offset in us, signed [-1024, 1023]
    uint8_t  mRamDuration;      // 0 = no change, 1 = no constraints, 2-31 = bitmap length
    uint8_t  mRamBits[4];       // valid bytes: ceil((mRamDuration+1)/8) when mRamDuration >= 2
};
```

SCA teardown = SCA LTV with L = 0 (empty payload). When `OPENTHREAD_CONFIG_THREAD_DIRECT_COEX_ENABLE = 0` (default): generators always emit `mRamDuration = 1`, `mRamBits` absent, SLW fields present.

> **Spec note (open item #11):** The spec introduced a new SCA LTV wire format (commit `6c6a22a`) that adds a "Slot Duration" field (2 bits, values: 0=625µs, 1=1.25ms, 2=625ms, 3=1.25s) and a "RAM Available" bit gating the RAM fields. This new format has not been implemented — the implementation uses the original `mRamHeader` encoding with implicit 160µs slot unit. Track this as open item #11 below.

**Challenge LTV (Type = 0x03)**

```cpp
struct ChallengeLtv
{
    static constexpr uint8_t kLength = 16; // truncated HMAC-SHA256 (first 16 bytes)
    uint8_t mChallenge[kLength];
};
```

**Helper methods on `Mac::Frame` / `Mac::TxFrame`:**
```cpp
// Reader helpers (return kErrorNone / kErrorNotFound / kErrorParse)
Error Frame::GetThreadHeaderIe(OffsetRange &aRange) const;
Error Frame::GetScaParams(ScaParams &aScaParams) const;
Error Frame::GetChallengeLtv(ChallengeLtv &aChallengeLtv) const;
Error Frame::GetTargetId(uint64_t &aTargetId) const;

// Writer helper
void TxFrame::AppendThreadHeaderIe(const ScaParams    *aSca,       // NULL = omit
                                    const ChallengeLtv *aChallenge, // NULL = omit
                                    const uint64_t     *aTargetId); // NULL = omit
```

### 4.5 Challenge LTV and HMAC computation

The WL generates a challenge and carries it in the TD Link Command Challenge LTV. The WI echoes it verbatim in the Enh-ACK.

**WL computes (spec §16.7.2–16.7.3):**
```
Challenge = HMAC-SHA256(Key, LinkFC ‖ WakeFC ‖ WakeID ‖ LinkSeq)[0:15]
```

where:
- `Key` = the key that secured the received Wake Frame (Wake Key or current MAC Key — see §5.2)
- `LinkFC` = the frame counter to be used in the TD Link Command (big-endian uint32)
- `WakeFC` = the frame counter from the received Wake Frame (big-endian uint32)
- `WakeID` = 8 bytes; zero-padded if the Wake Frame carried no WakeupId (**see §13 item 4**)
- `LinkSeq` = the TD Link sequence number (uint8)

**WI verifies:** computes the same HMAC with its own known values; compares to received Challenge LTV. Then immediately pushes the **received** Challenge LTV bytes to the platform for Enh-ACK injection (not the recomputed value).

**WL verifies echo:** confirms the Enh-ACK Challenge LTV bytes match the bytes WL originally sent.

```cpp
// In DirectHandler:
void DirectHandler::ComputeChallenge(ChallengeLtv        &aOut,
                                     uint32_t             aLinkFc,
                                     uint32_t             aWakeFc,
                                     uint64_t             aWakeId,
                                     uint8_t              aLinkSeq,
                                     bool                 aUseWakeKey);
```

---

## 5. Key Derivation

### 5.1 Default Wake Key

The default Wake Key is derived from the Thread Network Key and stored at Key Index 129:

```
defaultWakeKey = HMAC-SHA256(thrNetworkKey, "Thread-Wake")
```

This follows the same pattern as the existing `ComputeKeys()` / `ComputeTrelKey()` in `KeyManager`.

**`KeyManager` additions (`key_manager.hpp/.cpp`):**

```cpp
// Constants
static constexpr uint8_t kDefaultWakeKeyIndex = 129;
static const uint8_t     kWakeKeyString[];          // "Thread-Wake" (11 bytes)

// Methods
void           ComputeWakeKey(Mac::Key &aWakeKey) const;
const Mac::Key &GetDefaultWakeKey(void); // cached; invalidated when NetworkKey changes

// Internal state
Mac::Key  mDefaultWakeKey;
bool      mWakeKeyValid;   // false = needs recompute
```

Cache is invalidated inside `SetNetworkKey()`. `GetDefaultWakeKey()` returns the cache, recomputing lazily if `!mWakeKeyValid`.

### 5.2 Key Selection Rules

A WL accepts Wake Frames secured with **either** the Wake Key **or** the current MAC Key. This is required for the guest-access use case where the WL may not have the Network Key.

The key that secured the Wake Frame MUST be used for all subsequent frames in the same TD link establishment exchange (TD Link Command, Enh-ACK, and the Challenge HMAC computation).

`DirectHandler` records `mWakeKeyUsed : bool` and `mWakeKeyIndex : uint8_t` from the received Wake Frame's Aux Security Header, and passes this context to `GenerateThreadDirectLinkCommand()` and `ComputeChallenge()`.

### 5.3 Key Index Ranges

| Range | Key Index | Usage |
|---|---|---|
| Network-derived MAC keys | [0, 128] | Normal Thread MAC security |
| Default Wake Key | 129 | `HMAC-SHA256(NetworkKey, "Thread-Wake")` |
| Guest Wake Keys | [130, 192] | Individually provisioned; see §5.4 |
| Reserved | [193, 255] | Future use |

### 5.4 Guest Wake Key Provisioning

**Motivation.** The default Wake Key (Key Index 129) is derived from the Thread Network Key. Any device that holds the Network Key can therefore act as a Wake Initiator. The **guest wake key** use case exists for WI devices that do *not* hold the Thread Network Key — for example, a companion phone acting as a WI through a BLE→Thread bridge. A guest wake key is a raw 16-byte key provisioned out-of-band (BLE pairing, app-layer exchange, factory provisioning) and shared only between the WI and the specific WL(s) it is permitted to wake.

**This is not deferred.** Guest wake key storage and the public API to provision it are implemented in **PR 1** as part of initial wake security modeling, alongside the default Wake Key derivation.

#### Config flags

```c
// src/core/config/thread_direct.h

/** Enable guest wake key support. Default ON (feature is in scope for PR 1). */
#ifndef OPENTHREAD_CONFIG_THREAD_DIRECT_GUEST_WAKE_KEY_ENABLE
#define OPENTHREAD_CONFIG_THREAD_DIRECT_GUEST_WAKE_KEY_ENABLE 1
#endif

/** Maximum number of simultaneously configured guest wake keys per device. */
#ifndef OPENTHREAD_CONFIG_THREAD_DIRECT_MAX_GUEST_WAKE_KEYS
#define OPENTHREAD_CONFIG_THREAD_DIRECT_MAX_GUEST_WAKE_KEYS 4
#endif
```

#### KeyManager storage

`KeyManager` gains a flat array of guest wake key slots (guarded by `OPENTHREAD_CONFIG_THREAD_DIRECT_GUEST_WAKE_KEY_ENABLE`):

```cpp
// key_manager.hpp

#if OPENTHREAD_CONFIG_THREAD_DIRECT_GUEST_WAKE_KEY_ENABLE
struct GuestWakeKeyEntry
{
    Mac::Key mKey;       ///< 16-byte key material
    uint8_t  mKeyIndex;  ///< Key Index in [130, 192]; 0 = slot is empty
};

static constexpr uint8_t kGuestWakeKeyIndexMin = 130;
static constexpr uint8_t kGuestWakeKeyIndexMax = 192;

GuestWakeKeyEntry mGuestWakeKeys[OPENTHREAD_CONFIG_THREAD_DIRECT_MAX_GUEST_WAKE_KEYS];

Error SetGuestWakeKey(uint8_t aKeyIndex, const Mac::Key &aKey);
Error RemoveGuestWakeKey(uint8_t aKeyIndex);
const Mac::Key *GetGuestWakeKey(uint8_t aKeyIndex) const; // returns nullptr if not found
#endif
```

`SetGuestWakeKey()` validates that `aKeyIndex ∈ [130, 192]`, then overwrites any existing slot for that index or fills the first empty slot. Returns `OT_ERROR_NO_BUFS` if the table is full with no matching index.

`GetGuestWakeKey()` is called by `Mac::ProcessReceiveSecurity()` when a received Wake Frame carries a Key Index outside [0, 128] and not equal to 129 — it looks up the guest key table and returns the key material to the decryption engine.

#### Public API additions (see §9 of the OpenThread Implementation Confluence page)

```c
/**
 * Add or replace a guest wake key at the given key index.
 * Valid key indices: [130, 192].
 *
 * @param[in] aInstance   OpenThread instance.
 * @param[in] aKeyIndex   Key Index ∈ [130, 192].
 * @param[in] aKey        16-byte key material.
 *
 * @retval OT_ERROR_NONE          Key stored.
 * @retval OT_ERROR_INVALID_ARGS  aKeyIndex out of [130, 192] range.
 * @retval OT_ERROR_NO_BUFS       Guest key table full.
 * @retval OT_ERROR_DISABLED_FEATURE  Feature not compiled in.
 */
otError otThreadDirectSetGuestWakeKey(otInstance *aInstance, uint8_t aKeyIndex,
                             const otThreadDirectWakeKey *aKey);

/**
 * Remove a previously configured guest wake key.
 *
 * @retval OT_ERROR_NONE       Key removed.
 * @retval OT_ERROR_NOT_FOUND  No key at that index.
 */
otError otThreadDirectRemoveGuestWakeKey(otInstance *aInstance, uint8_t aKeyIndex);
```

The WI specifies which key to use for a given wake attempt via `otThreadDirectWakeup()`:

```c
/**
 * Initiate a wake burst targeting aExtAddress.
 * aKeyIndex = 0 or OT_MAC_FRAME_WAKE_KEY_INDEX (129) selects the default
 * (network-derived) Wake Key. aKeyIndex in [130, 192] selects a previously
 * provisioned guest key. aIntervalUs = 0 / aDurationMs = 0 use the
 * compile-time defaults from src/core/config/thread_direct.h.
 */
otError otThreadDirectWakeup(otInstance            *aInstance,
                             const otExtAddress    *aExtAddress,
                             otThreadDirectWakeType aWakeType,
                             uint16_t               aIntervalUs,
                             uint16_t               aDurationMs,
                             uint8_t                aKeyIndex);
```

Calling `otThreadDirectWakeup()` with `aKeyIndex = 129` uses the default Wake Key; `[130, 192]` uses a provisioned guest key.

#### Challenge HMAC with guest keys

The Challenge HMAC is computed as `HMAC-SHA256(WakeKey, ...)` where `WakeKey` is the key at `mWakeKeyIndex` (whichever key the Wake Frame was secured with). `ComputeChallenge()` in `DirectHandler` calls `KeyManager::GetGuestWakeKey(mWakeKeyIndex)` when `mWakeKeyIndex != 129`, falling back to `KeyManager::GetDefaultWakeKey()` for Key Index 129. This is transparent — the HMAC computation is identical; only the key material differs.

---

## 6. Stack Changes: MAC Layer

### 6.1 `sub_mac_wed.cpp` — Wake Listener

The existing WED listen scheduling logic (`HandleWedReceiveAt`, `HandleWedReceiveOrSleep`, `UpdateWakeupListening`) is retained and renamed (WED → WL throughout) with these changes:

**Wake frame detection (`ShouldHandleWakeupFrame()`):**
- Accept frames where `GetType() == kTypeMacCmd` AND the MAC Command ID byte == `0x54` AND Thread MAC Command ID (byte 1 of payload) == `0x01` (Wake Command).
- Wake Frame Type (byte 2) is read directly from the payload to distinguish TD link wake (`0x00`) from connectionless (`0x02`).
- WakeupId filtering (when Thread Header IE 0x2d / TargetId LTV is present): compare to the device's pre-configured WakeupId table.

**`WakeupInfo` struct gains fields** (in `mac_types.hpp`):
```cpp
struct WakeupInfo
{
    Mac::ExtAddress mExtAddress;       ///< WI extended address
    uint32_t        mAttachDelayUs;    ///< Rendezvous Time → µs offset to first Connection Window
    uint8_t         mRetryInterval;    ///< RI (4-bit wire field, stored as uint8_t)
    uint8_t         mRetryCount;       ///< RC (4-bit wire field, stored as uint8_t)
    uint8_t         mWakeFrameType;    ///< Wake Frame Type (0x00 / 0x01 / 0x02)
    bool            mIsGroupWakeup;    ///< true if DstAddr was broadcast with TargetId
    bool            mWakeKeyUsed;      ///< true = Wake Key; false = current MAC Key
    uint8_t         mWakeKeyIndex;     ///< Key Index from the Wake Frame Aux Sec Header
    uint32_t        mWakeFrameCounter; ///< WakeFC for Challenge HMAC input
};
```

`Mac::HandleTdWakeCommand()` extracts all WakeupInfo fields from the MAC Command payload bytes (after Thread MAC Command ID byte) and optionally from the Thread Header IE LTV.

### 6.2 `WakeupTxScheduler` → `WakeupTxScheduler` — Wake Initiator

File `wakeup_tx_scheduler.hpp/.cpp` is kept at its existing path (no file rename). Class is kept as `WakeupTxScheduler`.

Changes inside the renamed class:
- `PrepareWakeupFrame()` calls `TxFrame::GenerateThreadDirectWakeCommand()` (§4.2) instead of the removed `GenerateWakeupFrame()`.
- Sets Wake Channel = `OPENTHREAD_CONFIG_THREAD_DIRECT_DEFAULT_WAKE_CHANNEL` (20).
- `GetConnectionWindowUs()` computes the WI's total receive span from RI × RC × WAKE_INTERVAL:

```cpp
uint32_t GetConnectionWindowUs(void) const
{
    // Total span: RetryInterval × RetryCount × 7500 µs + minimum window (3 ms)
    static constexpr uint32_t kWakeIntervalUs      = 7500;
    static constexpr uint32_t kMinWindowDurationUs = 3000;
    return static_cast<uint32_t>(mRetryInterval) * mRetryCount * kWakeIntervalUs
           + kMinWindowDurationUs;
}
```

### 6.3 MAC command dispatch

`Mac::HandleMacCommand()` gains a new case:

```cpp
case Frame::kMacCmdDirect:  // 0x54 — Thread MAC Command dispatch
{
    uint8_t threadCmdId;
    IgnoreError(aFrame.GetThreadMacCommandId(threadCmdId));

    switch (threadCmdId)
    {
    case Frame::kThreadMacCmdWake:
#if OPENTHREAD_CONFIG_THREAD_DIRECT_WAKE_LISTENER_ENABLE
        HandleTdWakeCommand(aFrame);
#endif
        break;

    case Frame::kThreadMacCmdDirectLink:
#if OPENTHREAD_CONFIG_THREAD_DIRECT_WAKE_INITIATOR_ENABLE
        Get<DirectHandler>().HandleTdLinkCommand(aFrame);
#endif
        break;

    default:
        break;
    }
    didHandle = true;
    break;
}
```

---

## 7. Stack Changes: Thread Direct Handler

### 7.1 `DirectHandler` — new class

```
src/core/mac/direct_handler.hpp
src/core/mac/direct_handler.cpp
```

`DirectHandler` is an `InstanceLocator`; `Instance` owns it (guarded by `OPENTHREAD_CONFIG_THREAD_DIRECT_WAKE_INITIATOR_ENABLE || OPENTHREAD_CONFIG_THREAD_DIRECT_WAKE_LISTENER_ENABLE`), following the same pattern as `CslTxScheduler`.

### 7.2 Wake Initiator state machine

```
kIdle  ──[otThreadDirectWakeup()]──►  kWakingUp
kWakingUp  ──[WakeupTxScheduler ends]──►  kWaitingTdLinkCmd
kWaitingTdLinkCmd  ──[TD Link Command received]──►  process → kIdle (linked or failed)
```

**On TD Link Command reception** (`DirectHandler::HandleTdLinkCommand()`):

1. Validate MAC security (key lookup by the key type recorded in `WakeupInfo`).
2. Parse Thread Header IE 0x2d: extract `SupervisionInterval`, `ServicesBitmap`, `ChallengeLtv`, `ScaParams`.
3. Push the **received** `ChallengeLtv` bytes and the WI's own SCA LTV (if a local SLW schedule is configured) to the platform immediately via `otPlatRadioConfigureThreadDirectEnhAckIe()` so they can be injected into the Enh-ACK within the hardware ACK turnaround (~192 µs). The spec permits the WI to include its SCA LTV in any frame including the Enh-ACK. See §10.1.
4. Asynchronously (after Enh-ACK is on air): call `ComputeChallenge()` and compare to the received Challenge LTV bytes.
   - If mismatch: send TD teardown (SCA LTV with L=0) on the same channel. The Enh-ACK has already been transmitted; this is a spec-permitted outcome (§16.7.1).
   - If match: proceed.
5. Create `DirectPeer` entry in `DirectPeerTable`: record `SlwPeriodSlots`, `SlwPhaseSlots`, `SupervisionIntervalMs`, `ServicesBitmap`, `ExtAddress`.
6. Fire `OT_THREAD_DIRECT_EVENT_LINKED` callback.

### 7.3 Wake Listener state machine

```
kIdle  ──[WakeupInfo received from sub_mac_wed]──►  kAttachDelay
kAttachDelay  ──[timer expires]──►  kSendingTdLinkCmd
kSendingTdLinkCmd  ──[TX done callback]──►  kWaitingEnhAck
kWaitingEnhAck  ──[otPlatRadioTxDone(aAck)]──►  process → kIdle (linked or retry)
```

**On entering `kAttachDelay`:** timer set to `mWakeupInfo.mAttachDelayUs` (the Rendezvous Time). The WL continues listening for additional Wake Frames during this window.

**On sending TD Link Command:** `DirectHandler::GenerateThreadDirectLinkCommand()` builds the frame:
- Computes `ChallengeLtv` via `ComputeChallenge()`.
- Includes its own `ScaParams` as a SCA LTV if a local SLW schedule is configured.
- Uses the same key type that secured the received Wake Frame.
- Does NOT include a Short Address (SPEC-1365 deferral; the mask bit is reserved/zero).

**On `otPlatRadioTxDone(aAck)` with the Enh-ACK frame:**
- Parse the Enh-ACK for Thread Header IE 0x2d / ChallengeLtv.
- Verify: the echoed Challenge LTV bytes must exactly match what was sent. Also verify MIC and Frame Counter.
- On success: create `DirectPeer` entry, record WI SCA schedule (from Enh-ACK SCA LTV if present — WI includes its constrained receive schedule when it has one), fire `OT_THREAD_DIRECT_EVENT_LINKED`.
- On failure: discard silently per spec §16.7.1.
  - **Randomize the start time of the next WL listen window** (spec §16.7.4 — defense against Covert DoS; prevents an attacker from using a known-failed challenge to predict the WL's next wake time).
  - Retry if `RetryCount` > 0.

**Anti-DoS rules (spec §16.7.4):**
- The WL MUST NOT enforce any rate limit on incoming Wake Frames even if subsequent challenge handshakes fail. Enforcing a rate limit would allow an adversary to block legitimate wakes by triggering the limit.

### 7.4 Short address allocation (deferred)

Short address support is fully deferred (SPEC-1365). The Short Address field is absent from the TD Link Command payload; the mask bit is reserved and must be zero. All active scope uses extended addresses only. Do not add `AllocateTdShortAddress()` or a Short Address field to the TD Link Command in the current PR series.

### 7.5 `DirectPeer` and `DirectPeerTable` (renamed from `Peer`/`PeerTable`)

Files: `src/core/thread/direct_peer.hpp/.cpp`, `src/core/thread/direct_peer_table.hpp`.

The existing `Peer : CslNeighbor` class is reworked with TD-specific state (no original P2P semantics retained):

```cpp
class DirectPeer : public CslNeighbor
{
    // SCA schedule
    uint16_t mSlwPeriodSlots;     ///< SLW Period in 160 µs slots (0 = not configured)
    uint16_t mSlwPhaseSlots;      ///< SLW Phase in 160 µs slots

    // Post-link state
    uint16_t mSupervisionIntervalMs; ///< Supervision Interval from TD Link Command
    uint8_t  mServicesBitmap;     ///< Services: bit 0 = peer has SRP server

    // Wake security
    uint8_t  mWakeKeyIndex;       ///< Key Index used for this link's Wake / Link frames
    bool     mWakeKeyUsed : 1;    ///< true = Wake Key was used

    // CoEx
    bool     mCoexEnabled : 1;    ///< true = peer reported CoEx constraints (RAM Duration > 1)
    bool     mHasScaSchedule : 1; ///< SCA LTV has been received and applied

    // Replay protection
    uint32_t mLastWakeFrameCounter;   ///< Last accepted WakeFC (replay window tracking)
};
```

`DirectPeerTable` size is `OPENTHREAD_CONFIG_THREAD_DIRECT_MAX_DIRECT_PEERS`.

### 7.6 Teardown

SCA teardown = TD Link Command / any frame with Thread Header IE 0x2d where the SCA LTV has L=0 (empty payload), per spec §16.10.

`DirectHandler::SendTeardown()` constructs a MAC Command 0x54 frame with an empty SCA LTV in the Thread Header IE.

`otThreadDirectUnlink()` calls `SendTeardown()`, fires `OT_THREAD_DIRECT_EVENT_UNLINKED`, removes the `DirectPeer` entry.

---

## 8. Stack Changes: Post-Link Data Transfer

### 8.0 Post-link frame security

All frames on an established Thread Direct Link — including post-link SLW data frames — are secured with the **same key used to secure the Wake Frame** (spec §16.wake-frame-security: "When a Wake Key is used, the WL MUST establish the subsequent Thread Direct Link with all messages secured using the Wake Key"). For the network-wide default case this is Wake Key index 129; for guest keys it is the provisioned guest Wake Key (indices 130–192). The key index is established by the Wake Frame's Auxiliary Security Header and applies for the lifetime of the link.

> **Note:** If the Wake Frame were secured with a regular Thread MAC Key (key index 0–128, derived from the Network Key), the subsequent link would instead use the Current MAC Key. In the current implementation scope, this path is not exercised — the implementation uses Wake Keys exclusively.

### 8.1 `DirectTxScheduler` — new class

```
src/core/mac/direct_tx_scheduler.hpp
src/core/mac/direct_tx_scheduler.cpp
```

Directly analogous to `csl_tx_scheduler.cpp`. Schedules data frames to land within a peer's SLW window.

**For each pending frame addressed to a `DirectPeer`'s TD short address:**

1. Compute delay to next SLW window start:
   ```
   nextWindowStart = now + SlwPeriod - ((now - SlwPhaseEpoch) % SlwPeriod)
   ```
   This is the same epoch-relative phase computation used by `CslTxScheduler::GetNextCslTransmissionDelay()`.

2. Submit the frame via `Mac::RequestDirectFrameTransmission()` with the computed TX timestamp.

3. Set the IEEE 802.15.4 **Frame Pending** bit if further frames are queued for this peer.

4. The peer ACKs (with Frame Pending = 0 when its queue is empty) or issues a MAC Data Request to pull queued frames.

> **SLW phase epoch reference:** The SCA LTV carries period and phase in 160 µs slots but the spec does not currently define an absolute epoch for the phase (see **§13 item 3**). This implementation will track the same epoch reference that OpenThread uses for CSL phase — the start of the first known SLW window as established at link time, which is the receive time of the TD Link Command (WI side) or the Enh-ACK (WL side). This is identical to the CSL phase-tracking approach.

**Supervision frame (WI → WL direction, spec §16.10.5):**

`DirectHandler` owns a `mSupervisionTxTimer` per peer, fired at `supervision_interval / 2` after each delivery. On expiry with no pending data, sends a zero-payload MAC frame targeting the peer's next SLW window. Directly analogous to `ChildSupervisor` (`child_supervisor.cpp`).

On the WL side, `mSupervisionRxTimer` is reset on each received WI frame. On expiry: link loss detection → drop connection, return to WL listen mode, fire `OT_THREAD_DIRECT_EVENT_UNLINKED`.

> Note: The spec's current text specifies supervision only in the WI→WL direction. **See §13 item 5** for the open question on symmetric supervision.

### 8.2 Link loss and recovery (spec §16.10.6–16.10.7)

If either side exhausts `macCslMaxFrameRetries` (7) without an ACK:

- **WL:** drops connection, returns to WED listen mode, fires `OT_THREAD_DIRECT_EVENT_UNLINKED`.
- **WI:** fires `OT_THREAD_DIRECT_EVENT_UNLINKED` and may trigger a fresh wake cycle if the application requests reconnection.

---

## 9. Public API

### 9.1 New public header: `include/openthread/thread_direct.h`

Replaces the `provisional/p2p.h` and `provisional/link.h` headers (which are removed).

```c
// include/openthread/thread_direct.h

/**
 * @addtogroup api-thread-direct
 *
 * Thread Direct (Chapter 16) — MAC-layer peer-to-peer link between Thread devices.
 *
 * @{
 */

/* 16-byte guest wake key material (key indices 130-192). */
typedef struct otThreadDirectWakeKey
{
    uint8_t m8[16];
} otThreadDirectWakeKey;

/* Peer snapshot delivered with every event callback.
 * mWake* fields are valid on OT_THREAD_DIRECT_EVENT_WAKE_RECEIVED;
 * mSlw*/mTd* fields on LINKED/UNLINKED. */
typedef struct otThreadDirectPeerInfo
{
    otExtAddress mExtAddress;
    uint16_t     mTdShortAddress;        ///< reserved; OT_RADIO_INVALID_SHORT_ADDR today
    uint16_t     mSlwPeriodSlots;
    uint16_t     mSlwPhaseSlots;
    uint16_t     mSupervisionIntervalMs;
    uint8_t      mServicesBitmap;
    uint8_t      mWakeType;              ///< WAKE_RECEIVED-only
    uint32_t     mWakeRvTimeUs;          ///< WAKE_RECEIVED-only
    uint8_t      mWakeRetryCount;        ///< WAKE_RECEIVED-only
    uint8_t      mWakeRetryInterval;     ///< WAKE_RECEIVED-only
} otThreadDirectPeerInfo;

typedef enum otThreadDirectEvent
{
    OT_THREAD_DIRECT_EVENT_LINKED        = 0,  ///< Link established
    OT_THREAD_DIRECT_EVENT_LINK_FAILED   = 1,  ///< aPeerInfo may be NULL
    OT_THREAD_DIRECT_EVENT_UNLINKED      = 2,
    OT_THREAD_DIRECT_EVENT_WAKE_RECEIVED = 3,  ///< WL only
} otThreadDirectEvent;

typedef enum
{
    OT_THREAD_DIRECT_WAKE_TYPE_LINK           = 0,
    OT_THREAD_DIRECT_WAKE_TYPE_POWER_OUTAGE   = 1,
    OT_THREAD_DIRECT_WAKE_TYPE_CONNECTIONLESS = 2,
} otThreadDirectWakeType;

typedef struct otThreadDirectRamParams
{
    int16_t mOffsetUs;
    uint8_t mDuration;
    uint8_t mBits[4];
} otThreadDirectRamParams;

typedef struct otThreadDirectLocalSca
{
    uint16_t                mSlwPeriodSlots;
    otThreadDirectRamParams mRam;
} otThreadDirectLocalSca;

typedef void (*otThreadDirectEventCallback)(otThreadDirectEvent           aEvent,
                                            const otThreadDirectPeerInfo *aPeerInfo,
                                            void                         *aContext);

/* Register the event callback. */
void otThreadDirectSetEventCallback(otInstance                 *aInstance,
                                    otThreadDirectEventCallback aCallback,
                                    void                       *aContext);

/* (WI role) Start a wake burst. aIntervalUs / aDurationMs = 0 use defaults.
 * aKeyIndex 0 or 129 = default Wake Key; 130-192 = provisioned guest key. */
otError otThreadDirectWakeup(otInstance            *aInstance,
                             const otExtAddress    *aExtAddress,
                             otThreadDirectWakeType aWakeType,
                             uint16_t               aIntervalUs,
                             uint16_t               aDurationMs,
                             uint8_t                aKeyIndex);

bool otThreadDirectIsWakeBurstActive(otInstance *aInstance);

/* (Either role) Initiate link teardown. Currently returns OT_ERROR_NOT_IMPLEMENTED;
 * teardown happens via SLW inactivity or supervision timeout. */
otError otThreadDirectUnlink(otInstance *aInstance, const otExtAddress *aExtAddress);

/* (WL role) Enable / disable Wake Listener mode. */
otError otThreadDirectWakeListenerEnable(otInstance *aInstance, bool aEnable);
bool    otThreadDirectIsWakeListenerEnabled(otInstance *aInstance);

/* (Both roles) Local SLW period this device advertises in outgoing SCA LTVs.
 * Phase is stack-computed at frame-build time; not app-configurable. */
otError otThreadDirectSetSlwSchedule(otInstance *aInstance, uint16_t aSlwPeriodSlots);

/* Test/debug RAM override (honoured when OPENTHREAD_CONFIG_THREAD_DIRECT_COEX_ENABLE = 1). */
otError otThreadDirectSetRamOverride(otInstance *aInstance, const otThreadDirectRamParams *aParams);

/* Read back the local SLW + RAM state advertised in this device's SCA LTVs. */
otError otThreadDirectGetLocalSca(otInstance *aInstance, otThreadDirectLocalSca *aLocalSca);

/* SLW link inactivity timeout (seconds). 0 restores compile-time default. */
uint32_t otThreadDirectGetSlwTimeout(otInstance *aInstance);
otError  otThreadDirectSetSlwTimeout(otInstance *aInstance, uint32_t aTimeoutSeconds);

/* Peer info inspection. Currently returns OT_ERROR_NOT_IMPLEMENTED;
 * read peer state from aPeerInfo in the event callback instead. */
otError otThreadDirectGetPeerInfo(otInstance             *aInstance,
                                  const otExtAddress     *aExtAddress,
                                  otThreadDirectPeerInfo *aPeerInfo);

/* Guest wake key provisioning (key indices 130-192). */
otError otThreadDirectSetGuestWakeKey(otInstance                  *aInstance,
                                      uint8_t                      aKeyIndex,
                                      const otThreadDirectWakeKey *aKey);
otError otThreadDirectRemoveGuestWakeKey(otInstance *aInstance, uint8_t aKeyIndex);

/**
 * @}
 */
```

### 9.2 Existing API removed

- `include/openthread/provisional/p2p.h` — all `otP2p*` symbols removed.
- `include/openthread/provisional/link.h` — `otWakeupId`, `otWakeupType`, `otWakeupRequest` removed.

---

## 10. Platform Abstraction Layer

Thread Direct adds one new platform header — `include/openthread/platform/thread_direct.h` —
and three additions to `include/openthread/platform/radio.h`. No new `otRadioCaps` bit is
introduced; platforms that cannot support a function return `OT_ERROR_NOT_IMPLEMENTED` and the
stack treats that as the feature being absent.

### 10.1 `include/openthread/platform/thread_direct.h`

| Function | Direction | Purpose |
|---|---|---|
| `otPlatRadioConfigureThreadDirectEnhAckIe(otInstance *, const otExtAddress *aWiExtAddress, const uint8_t *aIeData, uint16_t aIeLength)` | Stack -> platform | Pre-arm the Thread Header IE (Element-ID 0x2d) that will be injected into the Enh-ACK generated for the next AR-bit frame from `aWiExtAddress`. Keyed by ExtAddress so concurrent peers do not collide. `aIeData = NULL` / `aIeLength = 0` removes the entry. |
| `otPlatRadioEnableThreadDirectSlw(otInstance *, uint16_t aSlwPeriod, const otExtAddress *aWiExtAddress)` | Stack -> platform | WL-side SLW receive schedule for the link with `aWiExtAddress`. `aSlwPeriod = 0` disables SLW for this peer. Units: 160 us slots. |
| `otPlatRadioUpdateThreadDirectSlwSampleTime(otInstance *, const otExtAddress *aWiExtAddress, uint32_t aSlwSampleTime)` | Stack -> platform | Advance the next-expected SLW arrival time on the WL after each received or missed frame. `aSlwSampleTime` is in local radio-clock us (see `otPlatRadioGetNow()`). |
| `otPlatRadioGetThreadDirectSlwAccuracy(otInstance *)` -> `uint8_t` | Platform -> stack | Worst-case clock accuracy in PPM. Used by the WI to compute SLW TX guard windows. |
| `otPlatRadioGetThreadDirectSlwUncertainty(otInstance *)` -> `uint8_t` | Platform -> stack | Fixed SLW arrival-time uncertainty in units of 10 us. Combined with PPM accuracy to size the WI's guard window. |
| `otPlatRadioGetThreadDirectRamParams(otInstance *, otThreadDirectRamParams *aParams)` | Platform -> stack | Read current CoEx Radio Availability Mask. Called when `OPENTHREAD_CONFIG_THREAD_DIRECT_COEX_ENABLE = 1`. Platforms without CoEx return `OT_ERROR_NOT_IMPLEMENTED` and the stack uses the override (or "no constraints") instead. |

The spec permits sending the Enh-ACK before HMAC verification completes
(*"In implementations where the challenge verification cannot be completed within the platform's
ACK turnaround time, the Enh-ACK MAY be generated prior to completion of the Challenge check"*).
This means the platform injects the pre-computed challenge echo immediately; the stack verifies
the HMAC asynchronously after the Enh-ACK has been sent. Implementations that cannot meet the
turnaround fall back to a separate data frame carrying the Thread Header IE at the cost of an
additional frame exchange.

### 10.2 Additions to `include/openthread/platform/radio.h`

```c
#define OT_MAC_FRAME_WAKE_KEY_INDEX           129
#define OT_MAC_FRAME_GUEST_WAKE_KEY_INDEX_MIN 130
#define OT_MAC_FRAME_GUEST_WAKE_KEY_INDEX_MAX 192

/* Register the network-derived (129) or guest (130-192) wake key with the
 * platform. Required on platforms with OT_RADIO_CAPS_TRANSMIT_SEC, since
 * hardware encryption runs from material registered via the platform. The
 * standard otPlatRadioSetMacKey() only carries prev/curr/next MAC keys
 * (indices 1-128); wake keys live outside that range. aWakeKey = NULL
 * deregisters. */
void otPlatRadioSetWakeKey(otInstance *aInstance, uint8_t aKeyIndex,
                           const otMacKeyMaterial *aWakeKey);

/* Mirror of SubMac::mWakeFrameCounter for hardware-encrypted TX, plus the
 * persistence/reload hooks. Same pattern as otPlatRadioGet/SetMacFrameCounter
 * for the standard MAC frame counter. */
void     otPlatRadioSetWakeFrameCounter(otInstance *aInstance, uint32_t aWakeFrameCounter);
uint32_t otPlatRadioGetWakeFrameCounter(otInstance *aInstance);
```

These three constants and three functions are declared unconditionally (not under
`OPENTHREAD_CONFIG_THREAD_DIRECT_*`) so that `radio.hpp` inline methods compile from every
translation unit that pulls in `radio.h` via `thread.h`.

### 10.3 Wake Frame reception filter

On platforms where the radio MAC filter must be explicitly configured during WL listen mode,
the platform must accept `kTypeMacCmd` (0x03) frames on Wake Channel 20 in addition to beacon
frames. No new API is required - existing `otPlatRadioReceiveAt()` already specifies channel
and duration; the filter configuration is platform-internal.

### 10.4 `otPlatRadioReceiveAt()` - slot ID

The current signature is:

```c
otError otPlatRadioReceiveAt(otInstance *aInstance, uint8_t aChannel,
                              uint32_t aStart, uint32_t aDuration, uint8_t aSlotId);
```

Platforms distinguish TD WL listen slots from CSL slots via `aSlotId`. Slot ID values:

```c
#define OT_RADIO_SLOT_ID_CSL 0
#define OT_RADIO_SLOT_ID_TD  1
```

The 5-argument form is the resolved API (already present in the SiLabs reference and the
current `silabs-thread` platform implementation - see open spec item 8 in section 13).

---

## 11. Spinel / Co-Processor Support

### 11.1 NCP (stack on co-processor)

All Thread Direct Spinel properties use the prefix `SPINEL_PROP_THREAD_DIRECT_*` (single
`THREAD_`, not `THREAD_THREAD_`). They are declared in `src/lib/spinel/spinel.h` and handled
in `src/ncp/ncp_base.cpp` / `ncp_base_mtd.cpp`.

| Property | Direction | Description |
|---|---|---|
| `SPINEL_PROP_THREAD_DIRECT_WAKE_CHANNEL` | Get/Set | Wake channel (default 20). Replaces the pre-spec `SPINEL_PROP_THREAD_WAKEUP_CHANNEL`. |
| `SPINEL_PROP_THREAD_DIRECT_WAKE_LISTEN_ENABLED` | Get/Set | Enable/disable WL periodic listen. |
| `SPINEL_PROP_THREAD_DIRECT_WAKE_LISTEN_PARAMS` | Get/Set | WL listen interval and duration (us). |
| `SPINEL_PROP_THREAD_DIRECT_SLW_SCHEDULE` | Get/Set | Local SLW period in 160 us slots (phase is stack-computed). |
| `SPINEL_PROP_THREAD_DIRECT_SLW_TIMEOUT` | Get/Set | SLW link inactivity timeout (seconds). |
| `SPINEL_PROP_THREAD_DIRECT_RAM_PARAMS` | Get/Set | CoEx RAM override (test/debug). |
| `SPINEL_PROP_THREAD_DIRECT_WAKE` | Set | Trigger WI wake burst (ExtAddress, wake type, interval, duration, key index). Async result via `LINK_EVENT`. |
| `SPINEL_PROP_THREAD_DIRECT_WAKE_BURST_ACTIVE` | Get | True while a WI wake burst is in progress. |
| `SPINEL_PROP_THREAD_DIRECT_UNLINK` | Set | Teardown by ExtAddress. Currently returns `NOT_IMPLEMENTED` (mirrors the stack API). |
| `SPINEL_PROP_THREAD_DIRECT_PEERS` | Get | Snapshot of active `DirectPeer` entries. Stack wiring lands with the TD Link PR. |
| `SPINEL_PROP_THREAD_DIRECT_GUEST_WAKE_KEY` | Set / Remove | Provision or remove a guest wake key by index (130-192). |
| `SPINEL_PROP_THREAD_DIRECT_LINK_EVENT` | Async notify | Mirror of `otThreadDirectEventCallback` (event code + peer info). |
| `SPINEL_PROP_THREAD_DIRECT_WAKE_FRAME_COUNTER` | Get/Set | Mirror of `SubMac::mWakeFrameCounter`, same pattern as the existing MAC frame counter property. |

Removed: `SPINEL_PROP_THREAD_WAKEUP_CHANNEL` (replaced) and the pre-spec `SPINEL_PROP_THREAD_P2P_*`
property family.

### 11.2 RCP (host runs stack, RCP is radio only)

All Thread Direct logic runs on the host. The only RCP-side Spinel work is exposing the new
`platform/thread_direct.h` and `platform/radio.h` calls (Enh-ACK IE configuration, SLW schedule,
wake key registration, wake frame counter) over Spinel - parallel to the existing
`SPINEL_PROP_RCP_ENH_ACK_PROBING` and MAC key / frame counter properties.

> **RCP support deferred.** Three architectural problems still need resolution before RCP is
> tractable: (1) per-WI challenge selection within the ~192 us Enh-ACK turnaround in
> one-to-many; (2) SLW window accuracy over Spinel given round-trip jitter; (3) multi-child
> schedule coordination when N WL peers each hold independent SLW period/phase. None of these
> block the initial WI/WL stack on the EFR32.

---

## 12. CLI Extensions

Implemented in `src/cli/cli_td.hpp/.cpp` under the top-level command `direct`. The CLI is
designed to be exercisable in `ot-cli` without additional tooling.

### 12.1 Top-level commands

```
direct help
direct channel [<num>]                   # get/set wake channel
direct wake <ext> [keyIndex [type [intervalUs [durationMs]]]]
direct wakelisten [enable|disable]
direct wakelisten params [<intervalUs> <durationUs>]
direct link <subcommand>
direct unlink                            # currently returns OT_ERROR_NOT_IMPLEMENTED
```

`direct wake` arguments:
- `keyIndex` - 0 or `OT_MAC_FRAME_WAKE_KEY_INDEX` (129) for the default Wake Key,
  `[130, 192]` for a provisioned guest key. Default: 0.
- `type` - 0 = link, 1 = power outage, 2 = connectionless. Default: 0.
- `intervalUs`, `durationMs` - 0 = use compile-time defaults from `config/thread_direct.h`.

### 12.2 `direct link` subcommands

```
direct link slw [<periodSlots>]          # local SLW period (160 us slots); phase is stack-computed
direct link ram [clear|set <hex> <offsetUs> <duration>]
direct link timeout [<seconds>]          # SLW link inactivity timeout
direct link state                        # local role flags, listen enabled, wake channel, SLW
direct link peers                        # currently returns OT_ERROR_NOT_IMPLEMENTED
direct link key <idx> <32-hex>           # provision guest wake key (idx in [130, 192])
direct link keyremove <idx>              # remove guest wake key
```

`direct link ram set` arguments mirror the SCA LTV RAM fields directly: 1-4 bytes of bitmask
(hex), 11-bit signed offset in us, and a RAM Duration code (0 = clear, 1 = no constraints,
2-31 = bitmap valid).

### 12.3 Diagnostic / test commands

The current CLI does not include the speculative `connect ext/id`, `wakeupid`, `dump rxie`,
`sched`, `stress`, or `reset` commands from earlier drafts. Those have been folded into the
TD Link PR backlog and will land alongside the corresponding stack functionality (notably
`direct link peers`, which already exists as a `NOT_IMPLEMENTED` stub).

## 13. Open Spec Items (TBD / Unclear)

Items marked **[BLOCKING]** must be resolved before the relevant PR can be finalized.

| # | Item | Notes |
|---|------|-------|
| 1 | **IEEE 802.15.4 MAC Command ID 0x54 formal allocation** | [BLOCKING for final upstream merge] Using `0x54` as the provisional value defined by the spec. Final upstream merge requires confirmed IEEE allocation. |
| 2 | **Advertisement Command LTV format** | "Compressed DNS" LTV (Type=0x01) format is marked TBD in the spec. Advertisement Command frame parsing is deferred until this is specified. |
| 3 | **SLW phase epoch reference** | Spec does not define an absolute epoch for the SLW phase. Implementation uses local clock at link establishment (same as CSL phase tracking). |
| 4 | **WakeID byte length in Challenge HMAC** | Spec defines WakeID as 1-8 bytes variable-length. This implementation always passes 8 bytes (zero-padded) to the HMAC. Confirm whether the HMAC input should use the exact wire length or always 8 bytes. |
| 5 | **Symmetric supervision direction** | Current spec text specifies supervision frames WI -> WL only. Spec has a note about symmetry. If both directions are required, `mSupervisionRxTimer` on the WI side and `mSupervisionTxTimer` on the WL side need to be added. |
| 6 | **TD Link Command: Clock Accuracy/Uncertainty field** | Field listed in spec table as "Link Parameter Mask optional" but mask bit assignment is TBD. Defer until mask bit is assigned. |
| 7 | **TD Link Command: Min Listen Duration field** | Same as item 6 - mask bit TBD. |
| 8 | **`otPlatRadioReceiveAt()` slot ID parameter** | RESOLVED. `aSlotId` (5-arg form) is the accepted signature and is already shipped in the current platform implementation. |
| 9 | **Group wake** | Group Wake (KeyIdMode2, shared WakeupId) is not included in the current PR series. Deferred to the post-PoC backlog. |
| 10 | **Full CoEx / RAM Bits** | Full multi-protocol CoEx (RAM Duration > 1, variable RAM Bits bitmap) is deferred. The current path uses RAM Duration = 1 (no constraints). `OPENTHREAD_CONFIG_THREAD_DIRECT_COEX_ENABLE` gates the full path. |
| 11 | **SCA LTV wire format stability** | The spec's SCA LTV evolved: 2-bit "Slot Duration" field, 1-bit "RAM Available" flag gating RAM fields, 16-bit SLW Period/Phase. The current implementation uses the earlier encoding (implicit 160 us slot, 12-bit Period/Phase, RAM Duration 0/1/2-31). Track when this wire format stabilizes; both forms are codec-compatible because all parser paths are always compiled in. |
| 12 | **TD short address allocation** | TD short address allocation/ownership is unresolved in the spec. The current implementation tracks `mTdShortAddress` in `DirectPeer` but always reports `OT_RADIO_INVALID_SHORT_ADDR`; all peer addressing uses ExtAddress. Comments and APIs say "TD short address" and avoid implying ownership. |

---

## 14. Pull-Request Plan

Each PR targets upstream `openthread/openthread`. The sequence follows a "clean base first,
then build" principle: PR 0 removed the pre-spec artifacts so every subsequent PR lands on a
clean surface.

Spinel/NCP and CLI work is **not deferred to a separate PR** - it ships in the same PR as the
feature it exposes. Each PR below lists its own Spinel/NCP/CLI scope.

**Overall target for PRs 0-3:** Velux PoC readiness - Thread sleepy-to-sleepy, one-to-one
wake, no CoEx. PRs 4+ extend toward mobile-platform use cases.

---

### PR 0 - Upstream Cleanup (COMPLETED)

**Purpose:** Remove pre-spec P2P / Gen1 / Gen2 artifacts; rename existing components to TD
terminology; leave the repo in a compilable, behavior-neutral state. No Thread Direct
behavior is added yet.

**Removes:**
- `src/core/thread/mle_p2p.cpp`
- `include/openthread/provisional/p2p.h`
- `include/openthread/provisional/link.h` (`otWakeupId`, `otWakeupType`, `otWakeupRequest`)
- `src/core/config/p2p.h`
- `mac_frame.hpp/.cpp`: `GenerateWakeupFrame()`, `IsWakeupFrame()`
- `mac_header_ie.hpp`: `CstIe`

**Renames / restructures:**
- `config/wakeup.h` -> `config/thread_direct.h` with all `WAKEUP_*`/`WED_*` flags renamed to `THREAD_DIRECT_*`
- `wakeup_tx_scheduler.*` - file path kept; class `WakeupTxScheduler` kept; payload format swapped
- `sub_mac_wed.cpp` -> WED -> WL terminology throughout (symbols and comments)
- `peer.hpp/.cpp` -> `direct_peer.hpp/.cpp`; `Peer` -> `DirectPeer` (P2P semantics removed)
- `peer_table.hpp` -> `direct_peer_table.hpp`; `PeerTable` -> `DirectPeerTable`

**Adds:**
- `include/openthread/thread_direct.h` - stub public API header

**Spinel / NCP:**
- `src/lib/spinel/spinel.h` - rename `SPINEL_PROP_THREAD_WAKEUP_CHANNEL` -> `SPINEL_PROP_THREAD_DIRECT_WAKE_CHANNEL`; remove pre-spec P2P property definitions
- `src/ncp/ncp_base.cpp` - update handler for renamed wake channel property; remove P2P NCP handlers

**CLI:**
- Remove old `p2p connect/disconnect/peers` and `wakeup listen/params/channel` commands
- Add `direct` command skeleton (`direct help`, `direct channel`, stubs for `wake`, `wakelisten`, `link`, `unlink`) - all return `OT_ERROR_NOT_IMPLEMENTED` until the wake / link PRs land

**Tests:** Existing wakeup-related tests pass unchanged (behavior is not altered - the
scheduler still fires, WL listen still works, just renamed).

---

### PR 1 - Wire Foundation + Secure Wake (COMPLETED)

**Purpose:** Bake in all wire format definitions, IE codec, and config constants for the
**full eventual scope** - group wake, WakeupId addressing, full CoEx/SCA with RAM bitmap,
all LTV types, all MAC Command types. Initial callers use a narrow path (one-to-one, RAM
Duration = 1); nothing in the wire layer needs to change later.

**Files touched:**
- `src/core/mac/mac_frame.hpp/.cpp`: `kMacCmdDirect = 0x54`, `kThreadMacCmdAdvertisement/Wake/DirectLink`, Wake Frame Type enum, `IsThreadDirectMacCommand()`, `GetThreadMacCommandId()`, `IsTdWakeCommand()`, `GenerateThreadDirectWakeCommand()`, `GenerateThreadDirectLinkCommand()`
- `src/core/mac/mac_header_ie.hpp`: `ThreadHeaderIe` (Element-ID 0x2d), `TargetIdLtv` (variable 1-8 bytes), `ScaParams` (full RamHeader + variable RamBits + SlwFields), `ChallengeLtv` (16 bytes), and helpers
- `src/core/config/thread_direct.h`: `DEFAULT_WAKE_CHANNEL = 20`, `SLW_MIN_DURATION_SLOTS = 8`, `MAX_DIRECT_PEERS = 1` (default - raise per product topology), `COEX_ENABLE = 0`, plus the new `SLW_TIMEOUT`, `SLW_MAX_TIMEOUT`, `LISTEN_RECEIVE_TIME_AFTER`, `WAKE_FRAME_TX_CCA_ENABLE`, `CONNECTION_RETRY_INTERVAL`, `CONNECTION_RETRY_COUNT` constants
- `src/core/thread/key_manager.hpp/.cpp`: `kDefaultWakeKeyIndex = 129`, `ComputeWakeKey()` (HMAC-SHA256 of Network Key), `GetDefaultWakeKey()`, guest key table (indices 130-192) with `SetGuestWakeKey()` / `RemoveGuestWakeKey()` / `FindGuestWakeKey()`
- `include/openthread/platform/radio.h`: `otPlatRadioConfigureThreadDirectEnhAckIe()`, `otPlatRadioSetWakeKey()`, `otPlatRadioSet/GetWakeFrameCounter()`, plus `OT_MAC_FRAME_*_KEY_INDEX` constants
- `include/openthread/platform/thread_direct.h`: SLW schedule, accuracy, uncertainty, and RAM-params platform getters

**Spinel / NCP:** `src/lib/spinel/spinel.h` - add the `SPINEL_PROP_THREAD_DIRECT_*` constant
definitions enumerated in section 11.1. Handlers land alongside the corresponding stack
functionality.

**CLI:** `direct help`, `direct channel`, `direct wake`, `direct wakelisten`, `direct link slw`,
`direct link ram`, `direct link timeout`, `direct link state`, `direct link key`,
`direct link keyremove`, and the `NOT_IMPLEMENTED` stubs for `direct link peers` and
`direct unlink`.

**Tests:** Unit tests for `GenerateThreadDirectWakeCommand()`, `GenerateThreadDirectLinkCommand()`,
SCA / Challenge / TargetId LTV round-trips, wake key derivation (HMAC test vectors), and wake
frame TX/RX security including the EFR32 hardware-encryption path.

---

### TD Link PR (in flight) - 3-way handshake + DirectHandler lifecycle

**Purpose:** Complete the MAC-layer 3-way handshake. WL responds to a Wake Frame with the TD
Link Command; WI replies via the Enh-ACK Thread Header IE with the echoed challenge plus its
own SCA LTV. TD link is established as a `DirectPeer` entry on both sides.

**Scope:** unicast-by-ExtAddress, one-to-one wake, RAM Duration = 1 (no CoEx). Wire format is
already general from PR 1; behavior is constrained.

**Stack files:**
- `src/core/mac/direct_handler.hpp/.cpp` - new; WI/WL state machines, `ComputeChallenge()`,
  `SendTeardown()`, calls `otPlatRadioConfigureThreadDirectEnhAckIe()` on TD Link Command
  receipt
- `src/core/mac/mac.cpp` - dispatch TD Link Command (Thread MAC Cmd 0x02) into the handler;
  narrow `Mac::IsThreadDirectLinkActive()` to "linked session only" (see thread-direct.mdc)
- `src/core/thread/direct_peer.hpp/.cpp` - finalize TD-specific fields (mWakeKeyIndex,
  mWakeFrameCounter, mTdShortAddress placeholder, supervision timers)
- `src/core/instance/instance.hpp` - `Get<DirectHandler>()`
- Wake frame counter guards widen to `WAKE_INITIATOR || WAKE_LISTENER` in `sub_mac.hpp/cpp`,
  `radio.hpp`, and `examples/platforms/simulation/radio.c` (WL now also TXes a frame secured
  with the wake key); WL `ProcessTransmitSecurity` / TxDone sync uses `SetWakeFrameCounter`
  the same way WI does
- `include/openthread/thread_direct.h` - wire `otThreadDirectUnlink`, `otThreadDirectGetPeerInfo`,
  `otThreadDirectSetSlwSchedule`, `otThreadDirectSetSlwTimeout`, `otThreadDirectSetRamOverride`,
  `otThreadDirectGetLocalSca` to the handler / direct_peer

**Spinel / NCP:** Implement handlers for `SPINEL_PROP_THREAD_DIRECT_SLW_SCHEDULE`,
`SLW_TIMEOUT`, `RAM_PARAMS`, `UNLINK`, `PEERS`, `WAKE_FRAME_COUNTER`, and the
`LINK_EVENT` async stream.

**CLI:** Wire `direct unlink` and `direct link peers` (currently `NOT_IMPLEMENTED`); add
diagnostic helpers for SLW timing inspection.

**Tests:**
- Full 3-way handshake on the simulation transport (WI + WL).
- Challenge HMAC verification pass and fail paths.
- Teardown via supervision timeout sends empty SCA LTV.
- DoS rule: WL accepts unlimited failed-challenge Wake Frames without rate limit.
- Randomized WL listen window start after failed challenge / handshake.

---

### PR 3 - Post-Link Data Transfer + Supervision

**Purpose:** SLW-aware data TX scheduler; WI supervision; WL supervision timer; link loss
detection and cleanup.

**Stack files:**
- `src/core/mac/direct_tx_scheduler.hpp/.cpp` - new; epoch-relative SLW phase computation
  (same shape as `CslTxScheduler::GetNextCslTransmissionDelay()`); initial RAM Duration = 1
- `src/core/mac/direct_handler.hpp/.cpp`: `mSupervisionTxTimer` (WI), `mSupervisionRxTimer`
  (WL); `SendSupervisionFrame()`; `HandleSupervisionTimeout()` -> `OT_THREAD_DIRECT_EVENT_UNLINKED`

**Spinel / NCP:** Flesh out the `PEERS` array encoding now that peers carry full post-link SLW
state.

**CLI:** Implement `direct link peers` (live data); add a per-peer `sched` helper.

**Tests:** SLW window accuracy, supervision firing intervals, supervision RX expiry, MAC
retransmission exhaustion -> link loss on both sides.

---

### PR 4 - Group Wake + WakeupId (deferred)

WakeupId addressing; broadcast Wake Frames with TargetId LTV; WL WakeupId filter table;
concurrent link handling.

> Deferred until the unicast path is fully validated in the field.

---

### PR 5 - Full CoEx / SCA (deferred)

Full RAM bitmap path (`OPENTHREAD_CONFIG_THREAD_DIRECT_COEX_ENABLE = 1`); CoEx-aware SLW
period constraints; `DirectTxScheduler` RAM-offset-aware window scheduling.

> Deferred: no CoEx-constrained platform is in the initial target set. Spec stability in the
> SCA LTV format is an additional benefit of waiting.

---

### PR 6 - RCP / Host Split (deferred)

`SPINEL_PROP_RCP_TD_ENH_ACK_IE` plus `radio_spinel.cpp` bindings for the new platform calls;
RCP firmware Enh-ACK IE injection handler; end-to-end test on RCP + host topology.

> Deferred. Three architectural problems must be solved first: (1) per-WI challenge selection
> within the ~192 us Enh-ACK turnaround in one-to-many; (2) SLW window accuracy over Spinel
> given round-trip jitter; (3) multi-child schedule coordination when N WL peers hold
> independent SLW state.
