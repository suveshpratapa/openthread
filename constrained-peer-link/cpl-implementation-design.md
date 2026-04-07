# Thread Constrained Peer Link — Implementation Design

**Baseline commit:** `0d4a43ab8` (from upstream)

**Spec reference:** Thread 2.0 spec section 16 — Constrained Peer Link

**Author:** Suvesh Pratapa

**Status:** Working Draft — seeking reviewer feedback

---

## 1. Scope

This document describes the implementation plan for the Thread Constrained Peer Link (CPL) feature, covering the OpenThread stack, platform abstraction layer (PAL), and Spinel commands for co-processor (RCP/NCP) architectures.

### 1.1  Existing Components and Configuration

The upstream codebase at `0d4a43ab8` provides the scaffolding below. Several config names pre-date Chapter 16 CPL terminology; this design retains them with their current names for ABI stability, adding new names only for novel CPL semantics (see Decision A in §1.2).

| Component | Reuse Status | CPL / Chapter 16 Mapping |
|---|---|---|
| [`OPENTHREAD_CONFIG_WAKEUP_COORDINATOR_ENABLE`](https://github.com/openthread/openthread/tree/main/src/core/config/wakeup.h) | Retain | WI (Wake Initiator) |
| [`OPENTHREAD_CONFIG_WAKEUP_END_DEVICE_ENABLE`](https://github.com/openthread/openthread/tree/main/src/core/config/wakeup.h) | Retain | WL (Wake Listener) |
| [`OPENTHREAD_CONFIG_WED_LISTEN_INTERVAL`](https://github.com/openthread/openthread/tree/main/src/core/config/wakeup.h) | Retain | `LISTEN_INTERVAL` (spec default: 1000 ms) |
| [`OPENTHREAD_CONFIG_WED_LISTEN_DURATION`](https://github.com/openthread/openthread/tree/main/src/core/config/wakeup.h) | Retain | `LISTEN_DURATION` (spec default: 8 ms) |
| [`OPENTHREAD_CONFIG_WAKEUP_COORDINATOR_CONNECTION_RETRY_INTERVAL`](https://github.com/openthread/openthread/tree/main/src/core/config/wakeup.h) | Retain | Retry Interval field (retry window count) |
| [`OPENTHREAD_CONFIG_WAKEUP_COORDINATOR_CONNECTION_RETRY_COUNT`](https://github.com/openthread/openthread/tree/main/src/core/config/wakeup.h) | Retain | Retry Count field |
| [`OPENTHREAD_CONFIG_P2P_ENABLE`](https://github.com/openthread/openthread/tree/main/src/core/config/p2p.h) | Retain | CPL feature enable |
| [`OPENTHREAD_CONFIG_P2P_MAX_PEERS`](https://github.com/openthread/openthread/tree/main/src/core/config/p2p.h) | Retain | CP peer table size |
| [`WakeupTxScheduler`](https://github.com/openthread/openthread/tree/main/src/core/mac/wakeup_tx_scheduler.hpp) | Reuse and extend | — |
| [`WakeupRequest`](https://github.com/openthread/openthread/tree/main/include/openthread/provisional/link.h) type | Reuse | — |
| [`WakeupId`](https://github.com/openthread/openthread/tree/main/include/openthread/provisional/link.h) (`uint64_t`) | Reuse | — |
| [`ConnectionIe` + `RendezvousTimeIe`](https://github.com/openthread/openthread/tree/main/src/core/mac/mac_header_ie.hpp) | Reuse wire format, can rename and extend | — |
| [`sub_mac_wed.cpp`](https://github.com/openthread/openthread/tree/main/src/core/mac/sub_mac_wed.cpp) — dual-path WED listen | Largely reuse | — |
| [`mle_p2p.cpp`](https://github.com/openthread/openthread/tree/main/src/core/thread/mle_p2p.cpp) — P2P MLE state machine | No changes required (see §3.6) | — |
| [`PeerTable` + `Peer : CslNeighbor`](https://github.com/openthread/openthread/tree/main/src/core/thread/peer_table.hpp) | Reuse and extend | — |
| [`HmacSha256`](https://github.com/openthread/openthread/tree/main/src/core/crypto/hmac_sha256.hpp) | Reuse | — |
| [`otP2pWakeupAndLink` / `otP2pUnlink` / `otP2pSetEventCallback`](https://github.com/openthread/openthread/tree/main/include/openthread/provisional/p2p.h) | Reuse API | — |
| [`otPlatRadioReceiveAt`](https://github.com/openthread/openthread/tree/main/include/openthread/platform/radio.h) | Reuse API | — |
| [`otLinkSetWakeUpListenEnabled`](https://github.com/openthread/openthread/tree/main/include/openthread/link.h) + params | Reuse API | — |
| [`otLinkGetWakeupChannel` / `otLinkSetWakeupChannel`](https://github.com/openthread/openthread/tree/main/include/openthread/link.h) | Reuse for CPL | — |

New configuration flags for CPL-specific behavior:

`config/wakeup.h`:

```c
// Default Wake Channel for CPL per spec section 16.4.1
#ifndef OPENTHREAD_CONFIG_CPL_DEFAULT_WAKE_CHANNEL
#define OPENTHREAD_CONFIG_CPL_DEFAULT_WAKE_CHANNEL 11
#endif

// SLW_MIN_DURATION: minimum CP-link SLW duration (spec §16.8.3), in units of 160µs slots
#ifndef OPENTHREAD_CONFIG_CPL_SLW_MIN_DURATION_SLOTS
#define OPENTHREAD_CONFIG_CPL_SLW_MIN_DURATION_SLOTS 8  // 8 × 160µs ≈ 1.25ms
#endif
```

Optional: `config/p2p.h`:

```c
// Group-wake TX/RX configuration support
#ifndef OPENTHREAD_CONFIG_P2P_GROUP_WAKEUP_ENABLE
#define OPENTHREAD_CONFIG_P2P_GROUP_WAKEUP_ENABLE 0  // opt-in; disabled by default
#endif
```

### 1.2  Open Design Questions for Feedback

**Decision A — Legacy WED flags**

The existing flags `OPENTHREAD_CONFIG_WAKEUP_COORDINATOR_ENABLE` and `OPENTHREAD_CONFIG_WAKEUP_END_DEVICE_ENABLE` in `config/wakeup.h` were introduced before the CPL spec reached its final form. Two options:

1. **Reuse as-is** (this design's choice): The existing flags guard the CPL code paths. No config renaming is needed because the names (`WAKEUP_COORDINATOR` / `WAKEUP_END_DEVICE`) align reasonably with the spec roles (WI / WL). This avoids a downstream flag migration.
   
   or
2. **Introduce new CPL flags** (`OPENTHREAD_CONFIG_CPL_WAKE_INITIATOR_ENABLE`, `OPENTHREAD_CONFIG_CPL_WAKE_LISTENER_ENABLE`) and deprecate the old names: gives a clean naming boundary but requires all existing users to update their build configuration.

**Decision B — Legacy P2P flags**

The upstream tree contains a working MLE-based 3-way handshake in `mle_p2p.cpp` that was written against an earlier spec draft. The final spec mandates a MAC-layer-only handshake (§16.7). Two options:

1. **Retain `mle_p2p.cpp` undisturbed** (this design's choice): The new `CplHandler` class (§3.3, `src/core/mac/cpl_handler.[ch]pp`) implements the spec-compliant MAC handshake. `mle_p2p.cpp` is left untouched and can be either retained as a legacy path behind a compile guard or removed separately. The two paths are fully independent.
   
   or
2. **Remove `mle_p2p.cpp` now**: Reduces code surface but risks disruption for any PoC deployment relying on it.

**Decision C — Legacy Multipurpose wake frame**

The current `WakeupTxScheduler` sends Multipurpose frames (non-spec). The spec mandates MAC Command 0x54. Options:

1. **Compile guard**  (this design's default):(`OPENTHREAD_CONFIG_CPL_USE_LEGACY_WAKEUP_FRAME`): Retains the Multipurpose path for interoperability with pre-spec devices. A runtime boolean flag can then choose which wakeup format to use at runtime, supporting legacy multipurpose frames.
   
   or
2. **Hard cutover**: `IsWakeupFrame()` and `GenerateWakeupFrame()` are updated to the MAC Command 0x54 format unconditionally.

**Decision D — Enh-ACK IE injection on WI** *(open question)*:

The 3-way handshake requires the WI to include the WL's Challenge IE bytes verbatim in its Enh-ACK — the WI *echoes* the received challenge (spec §16.7.3). This creates a hard real-time constraint: the IEEE 802.15.4 Enh-ACK must be generated within the hardware turnaround window after the CP Link Command arrives. SiLabs EFR32 platforms can meet ~192 µs in practice for Enh-ACK turnaround (though in previous interops we had to adjust to higher values with some other devices, I'm not sure we mandate this because the 15.4 spec is unclear about mandating this value for EnhAcks). The Challenge IE bytes to echo in Enh-ACK are not known until the triggering CP Link Command arrives.
We need to:
- Complete HMAC-SHA256 verification
- In an RCP architecture, also account for round trip to the host

 That being said, the spec explicitly permits sending the Enh-ACK before verification completes: *"it’s possible the challenge check couldn’t be done in the platform layer, then the Enh-ACK may be generated first"* (§16.7.1).

I'm looking for advice on the implementation options, especially for RCP.

---

## 2. Wire Format Changes

### 2.1  Wakeup Frame Type: Multipurpose → MAC Command 0x54

The current upstream code uses **Multipurpose frames** with KeyIdMode2 for the wake frame. The final spec mandates **IEEE 802.15.4 MAC Command frames with command ID `0x54`** (Thread MAC Command, pending formal IEEE allocation; we use `0x54` as the provisional value per spec section 16.14).

#### Thread MAC Command type hierarchy

All CPL frames share a single IEEE 802.15.4 MAC Command ID (`0x54`). The first payload byte after `0x54` is the **Thread MAC Command ID** (§16.5.7.4), which determines the frame's purpose and the layout of the bytes that follow:

- **`0x00` — Advertisement Command** (§16.9.3.1)

  Remaining payload bytes are zero or more Advertisement LTVs:

  | Advert LTV Type | Meaning |
  |---|---|
  | `0x01` | Compressed DNS *(detailed format TBD in spec)* |

- **`0x01` — Wake Command** (§16.5.7.5) — *WI → WL*

  The next payload byte is the **Wake Frame Type** (§16.5.2), followed by Rendezvous Time, Retry Interval (RI), Retry Count (RC), and an optional Channel byte:

  | Wake Frame Type | Meaning |
  |---|---|
  | `0x00` | CP Link establishment wake |
  | `0x01` | Power outage recovery |
  | `0x02` | Connectionless control |

  The frame *optionally* carries a **Thread Header IE** (`0x2d`) to filter by WakeupId:

  | Header IE LTV Type | Meaning | Present? |
  |---|---|---|
  | `0x01` | Target ID (WakeupId filter; §16.5.7.5) | Optional — only when WakeupId addressing is used |

- **`0x02` — CP Link Command** (§16.9.1) — *WL → WI*

  Remaining payload bytes: `[Link Parameter Mask] [Supervision Interval] [Services bitmap] [Short Address]`. The frame *always* carries a **Thread Header IE** (`0x2d`); the Enh-ACK reply from the WI carries one too (§16.7.3):

  | Header IE LTV Type | Meaning | Present? |
  |---|---|---|
  | `0x02` | SCA IE — SLW period + phase + CoEx RAM (§16.10.3) | Mandatory if CoEx-constrained; optional otherwise |
  | `0x03` | Thread Challenge IE — HMAC-SHA256 (§16.7.2) | Mandatory in CP Link Command; echoed verbatim in Enh-ACK |

A "CP Link wake" is therefore a frame with Thread MAC Command ID = `0x01` **and** Wake Frame Type = `0x00` — these are two distinct byte positions in the MAC payload. Full struct definitions and serialization detail for the Thread Header IE LTVs are in §2.2.

#### Spec-compliant frame layout (§16.5.7.5)

Per the spec the Wake Command parameters sit in the MAC Command payload after the Thread MAC Command ID byte:

**Wake Frame (spec layout):**
```
[ MHR: FCF (AR=0) | Seq | DstPAN | DstAddr | SrcAddr | SrcPAN ]
[ Aux Security Header ]
[ Header IEs: Thread Header IE 0x2d:
    [TargetId LTV (Type=0x01): WakeupId bytes]  — only if WakeupId addressing is used
]
[ MAC Command Payload:
    0x54                  — IEEE 802.15.4 Command ID
    0x01                  — Thread MAC Command ID = Wake Command
    Wake Frame Type       — 0x00 / 0x01 / 0x02 (see table above)
    Rendezvous Time       — in 10-symbol units
    Retry Interval (RI)
    Retry Count (RC)
    [Channel]             — optional, absent = use current channel
]
[ MFR: MIC32 | FCS ]
```

**CP Link Command (spec layout):**
```
[ MHR: FCF | Seq | DstPAN | DstAddr | SrcAddr | SrcPAN ]
[ Aux Security Header ]
[ Header IEs: Thread Header IE 0x2d:
    SCA IE LTV (Type=0x02): RAM header + [RAM Bits] + [SLW Period + SLW Phase]
    ChallengeIe LTV (Type=0x03): 16 bytes HMAC challenge
]
[ MAC Command Payload:
    0x54                  — IEEE 802.15.4 Command ID
    0x02                  — Thread MAC Command ID = CP Link Command
    [Link Parameter Mask] — bitmap of present optional fields
    [Supervision Interval]
    [Services bitmap]
    [Short Address]
]
[ MFR: MIC32 | FCS ]
```

**Mechanical changes to `mac_frame.*`:**

1. **`Frame::IsWakeupFrame()`** (currently checks `kTypeMultipurpose`):  
   Updated to check: `GetType() == kTypeMacCmd` AND command ID byte `== 0x54` AND Thread MAC Command ID `== 0x01` (Wake Command). The Wake Frame Type (0x00/0x01/0x02) is byte 2 of the MAC Command payload (immediately after the Thread MAC Command ID byte); it is read directly from the payload — not from any IE LTV.

2. **`TxFrame::GenerateWakeupFrame()`**:  
   Updated to assemble a MAC Command frame with command ID `0x54` and MAC Command payload bytes directly: `0x01` (Wake Command) `| WakeFrameType | RendezvousTime | RI | RC | [Channel]`. If WakeupId addressing is used, a `TargetId` LTV (`kTypeTargetId=0x01`, 1–8 bytes) is appended inside the Thread Header IE 0x2d; no `WakeFrameIe` LTV is written.

   The existing Multipurpose frame path **is retained** under the compile guard `OPENTHREAD_CONFIG_CPL_USE_LEGACY_WAKEUP_FRAME`. When that guard is enabled, a runtime boolean (e.g., `mUseCplFrameFormat` in `WakeupTxScheduler`) selects which format to use. Both `IsWakeupFrame()` and `GenerateWakeupFrame()` contain guarded branches for each format.

3. **New `TxFrame::GenerateCpLinkCommand()`**:  
   Generates the WL → WI CP Link Command (Thread MAC Command ID = `0x02`). Encodes the mandatory short address, optional supervision interval, and services bitmap in the MAC Command payload; SCA IE and Challenge IE go in the Thread Header IE 0x2d.

4. **`Frame::GetThreadMacCommandId()`**:  
   Returns the Thread MAC Command ID byte (byte 1 of the MAC Command payload, after `0x54`) when `GetType() == kTypeMacCmd`, `kErrorParse` otherwise.

### 2.2  Thread Header IE (Element ID = 0x2d)

The spec defines a new standard Header IE with element ID `0x2d` that carries LTV-encoded CPL-specific elements. (Not to be confused with the already existing `ThreadIe`, which is a Vendor IE with Thread Company OUI).

The new IE is a short-form Header IE in the non-IETF space.

New class in `mac_header_ie.hpp`:

```cpp
/**
 * Implements the Thread Header IE (IEEE 802.15.4 Header IE, element ID = 0x2d).
 *
 * The payload is a sequence of LTV-encoded elements (Thread §16.6).
 */
class ThreadHeaderIe
{
public:
    static constexpr uint8_t kHeaderIeId = 0x2d;

    // LTV element type codes (Thread spec §16.5.7.2 / §16.7.2 / §16.10.3)
    enum ElementType : uint8_t
    {
        kTypeTargetId    = 0x01, ///< Target ID LTV (WakeupId filter; spec §16.5.7.5 + §16.5.6)
        kTypeScaIe       = 0x02, ///< Scheduled Channel Access IE (SLW schedule; spec §16.10.3)
        kTypeChallengeIe = 0x03, ///< Thread Challenge IE (HMAC-SHA256; spec §16.7.2)
    };

    static constexpr uint8_t kMinIeSize = sizeof(HeaderIe);
};
```

#### Wire Format and Struct Model

Each LTV element in the Thread Header IE payload has the form `[L] [T] [V…]`:

- **L** — Length of the Value field in bytes. In the packed LTV format (§16.5.7.3) the number of bits used for L adapts as the remaining IE content shrinks, and unused Length bits carry bits of T in the same leading byte. Logically L and T are always separate.
- **T** — Type byte: `0x01` = Target ID, `0x02` = SCA IE, `0x03` = Thread Challenge.
- **V** — The type-specific value bytes; the T and L bytes are **not** part of this.

The packed structs below model **only the V (Value) bytes**. The T and L bytes never appear in the struct — the serialization helpers supply them externally:
- `kType` constants are passed to the serializer to write (or match) the T byte.
- The length (L) is evaluated based on each packed struct below (see comments).

The LTV elements modelled as packed structs:

```cpp
///////////////////////////////
// Type=0x01: Target ID LTV (spec §16.5.7.5 + §16.5.6)
// V field: 1–8 bytes — the raw WakeupId bytes (variable length).
// No packed struct is defined for this type because the V field has no fixed layout:
// the WakeupId length is 1–8 bytes depending on how many significant bytes the
// WakeupId uses. The serializer writes the low bytes of the uint64_t WakeupId
// directly as the V field, with L set to the actual byte length. The parser reads
// L bytes and zero-extends back to uint64_t.
//
// kTypeTargetId = 0x01 is defined in the ElementType enum above.

///////////////////////////////
// Type=0x02: SCA IE (Scheduled Channel Access, spec §16.10.3)
// V field: Variable length.
//   2 bytes  (fixed) : mRamHeader
//     bits [15:5]  RAM Offset   (11 bits, signed µs offset to start of radio schedule)
//     bits  [4:0]  RAM Duration (5-bit enum):
//                    0  = No change to previous RAM
//                    1  = No CoEx constraints (RAM Bits field absent)
//                    2–31 = Bitmap length; RAM Bits field present (1–4 bytes follow)
//   0–4 bytes (variable) : RAM Bits (only when RAM Duration >= 2)
//   0 or 3 bytes (optional) : SLW Period (12 bits) + SLW Phase (12 bits)
//                    May be omitted when the intent is to update RAM only.
//
//   Notes:
//   1. The struct below models only the fixed-prefix + SLW case (RAM Duration 0 or 1,
//      SLW fields present). The serializer/parser must inspect and retrieve the RAM
//      Duration bits.
//   2. SCA teardown is signalled by L = 0 (empty payload).
OT_TOOL_PACKED_BEGIN
struct ScaIe
{
    static constexpr uint8_t kType = 0x02;

    uint16_t mRamHeader;       // bits [15:5]=RAM Offset, bits [4:0]=RAM Duration
    // Variable RAM Bits (0–4 bytes) belong here in the wire format but cannot be
    // expressed in the fixed packed struct — see serializer.
    uint8_t  mSlwFields[3];    // 24 bits: [23:12]=SLW Period, [11:0]=SLW Phase (160 µs units)
} OT_TOOL_PACKED_END;

///////////////////////////////
// Type=0x03: Thread Challenge IE (spec §16.7.2)
// V field: kLength bytes of challenge material.
// kLength = 16: design choice for HMAC-SHA256 truncation length.
// The serializer writes L = kLength in the LTV header.
OT_TOOL_PACKED_BEGIN
struct ChallengeIe
{
    static constexpr uint8_t kType   = 0x03;
    static constexpr uint8_t kLength = 16;
    uint8_t mChallenge[kLength];
} OT_TOOL_PACKED_END;
///////////////////////////////
```

Helper methods can be added to `Mac::Frame` and `Mac::TxFrame` for reading/writing these LTV elements within a `ThreadHeaderIe` payload:

- `Frame::GetThreadHeaderIe() const`
- `Frame::GetScaIe(ScaIe &) const`
- `Frame::GetChallengeIe(ChallengeIe &) const`
- `TxFrame::AppendThreadHeaderIe(const ScaIe *, const ChallengeIe *)`

### 2.3  Wake Frame Security and Key Derivation

The spec (§16.5.9) defines a dedicated **Wake Key** for securing the communication between a WI and a WL. This is a **new key material requirement** distinct from the existing network MAC keys.

#### 2.3.1  Motivation: Guest Access

The guest access use case requires that the Network Key (and its derived link-layer keys) is **not** shared with the CP Device acting as guest. A Wake Key is therefore independently generated and distributed only to CP Device peers, keeping Thread network credentials isolated from the Wake signaling channel.

#### 2.3.2  Default Wake Key Derivation

For WLs that are commissioned with the Network Key (the common case), a network-wide default Wake Key is derived as:

```
defaultWakeKey = HMAC-SHA256(thrNetworkKey, "Thread-Wake")
```

This is directly analogous to the existing `ComputeKeys()` in `KeyManager`, which derives MAC keys via `HMAC-SHA256(NetworkKey, keySequence || "Thread")`.

The default Wake Key is installed as **Key Index 129** in Key Identifier Mode 1.

#### 2.3.3  Key Index Ranges

| Key range | Key Index values | Usage |
|---|---|---|
| Network-derived MAC keys | [0, 128] | Normal Thread MAC security |
| Default Wake Key | 129 | Derived from NetworkKey as above |
| Additional Wake Keys | [130, 192] | Individually provisioned Wake Keys |
| Group Wake Keys | KeyIdMode2 | Wake frame addressed to a group of CP Devices |
| Reserved | [193, 255] | Future use |

#### 2.3.4  `KeyManager` Changes

`KeyManager` gains a Wake Key derivation method modeled on the existing `ComputeKeys()` / `ComputeTrelKey()` pattern:

```cpp
// In key_manager.hpp
void ComputeWakeKey(Mac::Key &aWakeKey) const;
const Mac::Key &GetDefaultWakeKey(void);  // cached; recomputed when NetworkKey changes

// New member state:
Mac::Key mDefaultWakeKey;    // HMAC-SHA256(NetworkKey, "Thread-Wake"), Key Index 129
bool     mWakeKeyValid;      // invalidated when NetworkKey changes via SetNetworkKey()

static constexpr uint8_t kDefaultWakeKeyIndex = 129;
static const uint8_t     kWakeKeyString[];      // "Thread-Wake" (11 bytes)
```

```cpp
// In key_manager.cpp
const uint8_t KeyManager::kWakeKeyString[] = {
    'T', 'h', 'r', 'e', 'a', 'd', '-', 'W', 'a', 'k', 'e',
};

void KeyManager::ComputeWakeKey(Mac::Key &aWakeKey) const
{
    Crypto::HmacSha256        hmac;
    Crypto::Key               cryptoKey;
    Crypto::HmacSha256::Hash  hash;

#if OPENTHREAD_CONFIG_PLATFORM_KEY_REFERENCES_ENABLE
    cryptoKey.SetAsKeyRef(mNetworkKeyRef);
#else
    cryptoKey.Set(mNetworkKey.m8, NetworkKey::kSize);
#endif

    hmac.Start(cryptoKey);
    hmac.Update(kWakeKeyString, sizeof(kWakeKeyString));
    hmac.Finish(hash);

    static_assert(sizeof(aWakeKey.m8) <= sizeof(hash.m8));
    memcpy(aWakeKey.m8, hash.m8, sizeof(aWakeKey.m8));
}
```

`GetDefaultWakeKey()` returns the cached `mDefaultWakeKey`, recomputing if `!mWakeKeyValid`. The cache is invalidated in `SetNetworkKey()`.

#### 2.3.5  Wake Frame Security Processing

The spec (§16.6) mandates that a WL MUST accept Wake Frames secured by **either**:
- Its Wake Key (guest access scenario), OR
- A valid MAC Key derived from the Network Key.

The key used in the Wake Frame determines the security context for the subsequent CP Link Command and Enh-ACK:
- **Wake Key was used** → all subsequent CP Link messages use the Wake Key.
- **MAC Key was used** → all subsequent CP Link messages use the Current MAC Key.

`CplHandler` records which key type secured the incoming Wake Frame in `WakeupInfo`, and passes that context to `GenerateCpLinkCommand()` and `ComputeChallenge()`. `Mac::ProcessReceiveSecurity()` already performs key lookup by Key ID and Key ID Mode; the Wake Key at Key Index 129 should be found by the existing lookup automatically.

#### 2.3.6  Key ID Mode Selection for Wake Frames (WI TX side)

| Wake type | Key ID Mode | Key Index |
|---|---|---|
| Unicast (one WL) | Mode 1 | 129 (derived) or [130, 192] (provisioned) |
| Group (multiple WLs share same key) | Mode 2 | — (no Key Index field in frame) |

For the default unicast case, the WI transmits Wake Frames with `KeyIdMode=1, KeyIndex=129`. `TxFrame::GenerateWakeupFrame()` is updated to set the Key ID Mode and Key Index from the configured Wake Dataset.

---

## 3. Stack Layer Changes

### 3.1  `WakeupTxScheduler` — WI Side

`WakeupTxScheduler::PrepareWakeupFrame()` is updated:
- Calls `TxFrame::GenerateWakeupFrame()` (updated per §2.1): builds a MAC Command 0x54 frame when CPL format is selected, or a legacy Multipurpose frame when `OPENTHREAD_CONFIG_CPL_USE_LEGACY_WAKEUP_FRAME` is active and the runtime flag chooses legacy
- Directly writes Wake Command bytes into the MAC payload: Thread MAC Command ID `0x01`, `WakeFrameType=0x00` (CP Link wake), `RendezvousTime`, then `(RI << 4) | RC` packed into **one byte** (spec §16.5.7.5: RI is bits [7:4], RC is bits [3:0] of a single octet) — follows spec §16.5.7.5 MAC payload byte layout (CPL path)
- Sets security header: `KeyIdMode=1`, `KeyIndex=129` (default Wake Key), `FrameCounter` from MAC frame counter (CPL path)
- **ExtAddress unicast wake:** `DstAddr` = WL's extended address. No Target ID LTV needed. Uses KeyIdMode 1.
- **WakeupId wake (unicast-by-ID or group):** `DstAddr` = broadcast (0xFFFF). One or more Target ID LTVs (`kTypeTargetId = 0x01`, 1–8 bytes each) are appended inside the Thread Header IE 0x2d. Only WLs pre-configured with a matching WakeupId respond.
  - *Unicast-by-ID:* a unique WakeupId identifies a single WL; uses KeyIdMode 1, Key Index [129, 192].
  - *Group wake* (requires `OPENTHREAD_CONFIG_P2P_GROUP_WAKEUP_ENABLE`): a shared WakeupId wakes multiple WLs simultaneously; uses KeyIdMode 2 (no Key Index field). All group members share the same Wake Key. `CplHandler` must handle potentially concurrent CP Link Commands arriving from multiple WLs.

The `WakeupRequest` type (`uint64_t WakeupId` in `provisional/link.h`) covers all three addressing cases.

`GetConnectionWindowUs()` is updated to compute when the WI must stay in receive mode. Per spec §16.5.5, connection windows start at:

```
t_k = RendezvousTime + RI × k × WAKE_INTERVAL,  k = 0, 1, …, N  (N = Retry Count)
```

The WI must keep its radio in receive from the first window (`k=0`, i.e., at `RendezvousTime`) through the last window end (`k=N`), giving a total receive span of `RI × N × WAKE_INTERVAL` plus the minimum window duration (≥3 ms per spec):

```cpp
uint32_t GetTotalConnectionWindowSpanUs(void) const
{
    // Total span from first window start to last window end  
    // = RI × RetryCount × WAKE_INTERVAL_US + kMinWindowDurationUs
    // RendezvousTime is the offset FROM the last Wake Frame TX to the first window —
    // handled by the WakeupTxScheduler timer, not this function.
    constexpr uint32_t kWakeIntervalUs    = 7500; // WAKE_INTERVAL = 7.5 ms (spec §16.12)
    constexpr uint32_t kMinWindowDurationUs = 3000; // Minimum Connection Window = 3 ms (spec §16.5.5)
    return static_cast<uint32_t>(mRetryInterval) * mRetryCount * kWakeIntervalUs
           + kMinWindowDurationUs;
}
```

### 3.2  `sub_mac_wed.cpp` — Wake Listener

No changes to the WED listen scheduling logic itself (`HandleWedReceiveAt` / `HandleWedReceiveOrSleep`) — these handle `OT_RADIO_CAPS_RECEIVE_TIMING` capability and software fallback already.

Changes needed:
1. **`ShouldHandleWakeupFrame()`** is already upstream (with WakeupId table filtering). The WL checks:
   - CPL path: frame type == `kTypeMacCmd` AND IEEE 802.15.4 command ID == `0x54` AND Thread MAC Command ID (byte 1 of payload) == `0x01` (Wake Command). Optionally further filters on `WakeFrameType` == `0x00` (CP Link wake) by reading byte 2 of the MAC Command payload directly (no IE LTV lookup needed).
   - Legacy path (when `OPENTHREAD_CONFIG_CPL_USE_LEGACY_WAKEUP_FRAME` is active): frame type == `kTypeMultipurpose` (existing behavior)
   - If `WakeupId` is present (Thread Header IE 0x2d `TargetId` LTV for CPL; `ConnectionIe` for legacy), the WakeupId lookup runs against the pre-configured table

2. **`WakeupInfo` struct** (`mac_types.hpp`) gains fields indicating which key type was used:

```cpp
struct WakeupInfo
{
    ExtAddress mExtAddress;        // Extended address of WI
    uint32_t   mAttachDelayMs;     // Rendezvous Time converted to ms (offset to first Connection Window)
    uint8_t    mRetryInterval;     // Retry Interval (RI) — 4-bit wire field (stored as uint8_t in RAM)
    uint8_t    mRetryCount;        // Retry Count (RC) — 4-bit wire field (stored as uint8_t in RAM)
    bool       mIsGroupWakeup : 1; // Set if dest was broadcast with WakeupId
    bool       mWakeKeyUsed   : 1; // true = Wake Key; false = Current MAC Key
    uint8_t    mWakeKeyIndex;      // Key Index from the received Wake Frame security header
};
```

`Mac::HandleWakeupFrame()` extracts these fields (WakeFrameType, RendezvousTime, RI, RC) from the MAC Command payload bytes (after the Thread MAC Command ID byte) in place of the legacy `RendezvousTimeIe` + `ConnectionIe` approach.

### 3.3  CP Link Command and `CplHandler`

The final spec uses a **MAC-layer-only CP Link Command** (MAC Command 0x54, Command=2) followed by an **Enh-ACK** from the WI. The current P2P implementation, `mle_p2p.cpp` uses **MLE Link Request / Link Accept And Request** over UDP.

The mapping is:

| Spec step | Current upstream (MLE) | CPL chapter 16 spec (MAC-layer) |
|---|---|---|
| WL → WI response | MLE Link Request (UDP unicast) | MAC Cmd 0x54, Command=2 |
| WI → WL response | MLE Link Accept And Request (UDP unicast) | Enh-ACK with Thread Header IE 0x2d Challenge |
| WL → WI ack | MLE Link Accept (UDP unicast) | (implicit — Enh-ACK received = link established) |
| WL data params | MLE TLVs | CP Link Command fields: supervision, services bitmap, short addr |

**Decision B applied here (see §1.2)**: `CplHandler` is the spec-compliant path; the legacy MLE path remains available behind `OPENTHREAD_CONFIG_CPL_USE_LEGACY_P2P` for PoC deployments that depend on it.

The new handshake can be implemented in:

```
src/core/mac/cpl_handler.hpp   // CplHandler class
src/core/mac/cpl_handler.cpp   // 3-way handshake state machine
```

`CplHandler` is an `InstanceLocator` and `InstanceLocator::Locator<CplHandler>` inside `Instance` (guarded by `OPENTHREAD_CONFIG_P2P_ENABLE`), following the same pattern as `WakeupTxScheduler`.

#### 3.3.1  `CplHandler` - Wake Initiator

```
States: kIdle → kWakingUp → kWaitingCpLinkCmd → kIdle
```

- `kWakingUp`: `WakeupTxScheduler` is running; when it ends, timer fires to close the connection window
- `kWaitingCpLinkCmd`: WI radio is in receive; incoming MAC Command 0x54 frames are routed via `Mac::HandleMacCommand()` → new `case Frame::kMacCmdCplCommand:` → `CplHandler::HandleCplMacCommand()`
- On CP Link Command reception:
  1. Validate MAC security per §2.3.5 (Wake Key or current MAC Key)
  2. Parse Thread Header IE 0x2d: short address, supervision interval, services bitmap, Challenge IE
  3. **Immediately** push the **received** Challenge IE bytes to the PAL via `otPlatRadioConfigureCplEnhAckIe()`, so the platform can insert them verbatim in the Enh-ACK (spec §16.7.3: the WI *echoes* the challenge bytes, it does not recompute them). The Enh-ACK must go out within the ~256 µs hardware turnaround window — before the stack can complete HMAC verification. This timing constraint is the subject of Decision D (§1.2).
  4. **Verify ex-post:** compute the expected HMAC-SHA256 (`ComputeChallenge` in §3.3.3) and compare to the received Challenge IE bytes. The Enh-ACK has already been sent at this point.
     - If mismatch → send CPL teardown frame (per spec §16.7.1; Enh-ACK suppression is not possible after the fact)
     - If match → proceed
  5. Record peer in `PeerTable` with state `kStateValid`, short address, SCA schedule, supervision interval
  6. Fire `OT_P2P_EVENT_LINKED` callback

#### 3.3.2  `CplHandler` - Wake Listener

```
States: kIdle → kAttachDelay → kWaitingEnhAck → kIdle
```

- `kAttachDelay`: `mAttachDelayMs` timer scheduled; listens for additional wake frames
- `kWaitingEnhAck`: CP Link Command has been transmitted; expecting Enh-ACK with Challenge IE
- On CP Link Command TX completion:
  - The Enh-ACK will arrive via `otPlatRadioTxDone(aAckFrame)` populated with the ACK frame
  - Parse `aAckFrame` for Thread Header IE 0x2d / Challenge IE
  - Verify challenge: per spec §16.7.3 the Enh-ACK echoes the same challenge bytes the WL sent; WL checks that the echoed value matches what it originally computed. Additionally validate MAC MIC and Frame Counter.
  - On success: record WI in `PeerTable`, configure SCA schedule, fire `OT_P2P_EVENT_LINKED`
  - On failure (MIC failure, Frame Counter replay, or Challenge mismatch):
    - Discard silently per spec §16.7.1
    - **MUST randomize the start time of the next Wake Listening window** (spec §16.7.4 — defense against Covert DoS: prevents attacker from predicting when WL will be listening again)
    - Retry up to RetryCount, then fail
  - **Rate limiting (spec §16.7.4):** The WL MUST NOT enforce any rate limit on incoming Wake Frames even if they fail the subsequent challenge handshake. Rate limiting would enable DoS attacks that block legitimate wakes by triggering the rate limit.

`CplHandler::GenerateCpLinkCommand()` builds the frame:

```cpp
Error CplHandler::GenerateCpLinkCommand(TxFrame &aFrame, const Peer &aPeer)
{
    // Build MAC Command 0x54 frame with:
    // - Command=2 (CP Link)
    // - Short Address from our assigned range [0xfd00, 0xfdff]
    // - Supervision interval (derived from configured SLW period)
    // - Services bitmap (bit 0 = SRP Server present)
    // - Thread Header IE 0x2d with Challenge IE (HMAC-SHA256 computed here)
    // - Security matching the Wake Frame's key type (Wake Key or Current MAC Key)
    ...
}
```

#### 3.3.3  Challenge Computation

The HMAC-SHA256 challenge uses the key that secured the incoming Wake Frame. The `WakeupInfo.mWakeKeyUsed` field (§3.2) carries this context through.

Per spec §16.7.3:
- **WL sends (Link Frame):** `Challenge = HMAC-SHA256(Key, LinkFC ‖ WakeFC ‖ WakeID ‖ LinkSeq)`
- **WI verifies then echoes:** WI computes the same HMAC using its own known values and compares to the received challenge. If they match, WI pushes the **received bytes** (not a recomputed value) into the Enh-ACK. The Enh-ACK is not a new HMAC — it literally echoes the WL's challenge bytes back.
- **WL verifies echo:** WL confirms the Enh-ACK's challenge bytes match what it originally sent.

`ComputeChallenge()` is therefore called on both sides for **generation/verification**, but on the WI side the output is used for comparison only; the Enh-ACK IE bytes come from the received CP Link Command.

> **WakeID length:** The spec defines WakeID as 1–8 bytes variable-length (§16.5.6). `WakeupId` is stored as `uint64_t` (8 bytes). When computing the HMAC, if no Wake Identifier was present, WakeID = 0 (8 zero bytes). For short WakeIDs (< 8 bytes), the spec is silent on zero-padding vs. using the exact length — this design will pad to 8 bytes. Need to verify that this is okay.

```cpp
void CplHandler::ComputeChallenge(ChallengeIe          &aOut,
                                   uint32_t              aLinkFc,
                                   uint32_t              aWakeFc,
                                   const Mac::WakeupId  &aWakeId,
                                   uint8_t               aLinkSeq,
                                   bool                  aUseWakeKey)
{
    HmacSha256       hmac;
    HmacSha256::Hash hash;
    Crypto::Key      cryptoKey;

    if (aUseWakeKey)
    {
        const Mac::Key &wakeKey = Get<KeyManager>().GetDefaultWakeKey();
        cryptoKey.Set(wakeKey.m8, sizeof(wakeKey.m8));
    }
    else
    {
        // Use the current network MAC key (key material extraction follows
        // the same pattern used by existing code in mac.cpp)
        ...
    }

    hmac.Start(cryptoKey);
    hmac.Update(BigEndian::HostSwap32(aLinkFc));
    hmac.Update(BigEndian::HostSwap32(aWakeFc));
    hmac.Update(aWakeId.m8, sizeof(aWakeId.m8)); // WakeupId (uint64_t, 8 bytes)
    hmac.Update(aLinkSeq);
    hmac.Finish(hash);

    static_assert(sizeof(aOut.mChallenge) <= sizeof(hash.m8));
    memcpy(aOut.mChallenge, hash.m8, sizeof(aOut.mChallenge)); // First 16 bytes
}
```

### 3.4  `Peer` and `PeerTable` — SCA Schedule

The `Peer` class (`src/core/thread/peer.hpp`) gains SCA state:

```cpp
class Peer : public CslNeighbor  // CslNeighbor already tracks CSL period and phase
{
    ...
    // CPL SCA IE fields (stored after learning from CP Link Command or Enh-ACK)
    uint16_t mSlwPeriodSlots;   ///< SLW Period in 160µs slots (0 = not yet configured)
    uint16_t mSlwPhaseSlots;    ///< SLW Phase in 160µs slots
    bool     mSlwCoexEnabled;   ///< Whether CoEx constraints apply (period must be multiple of 60ms)
    bool     mHasScaSchedule;   ///< SCA IE has been received and applied

    // Services
    bool     mHasSrpServer : 1; ///< Peer advertises SRP server (services bitmap bit 0)

    // CPL short address
    uint16_t mCplShortAddress;  ///< Assigned CPL short address [0xfd00, 0xfdff], or 0xffff if unset
    ...
};
```

`PeerTable` is unchanged — `OPENTHREAD_CONFIG_P2P_MAX_PEERS` already controls the table size.

### 3.5  Short Address Assignment

The WL includes its self-assigned CPL short address in the CP Link Command. The address is selected from `[0xfd00, 0xfdff]`:

```cpp
// In CplHandler — mNextCplShortAddr is a uint16_t member of CplHandler,
// initialized to Mac::kCplShortAddrBase in the constructor.
// Note: a function-local static would be shared across all OT instances and
// would not reset correctly between link teardown/re-establishment cycles.
uint16_t CplHandler::AllocateCplShortAddress(void)
{
    uint16_t addr = mNextCplShortAddr;
    mNextCplShortAddr = (mNextCplShortAddr < Mac::kCplShortAddrMax)
                            ? (mNextCplShortAddr + 1)
                            : Mac::kCplShortAddrBase;
    return addr;
}
```

Constants in `mac_types.hpp`:
```cpp
static constexpr uint16_t kCplShortAddrBase = 0xfd00;
static constexpr uint16_t kCplShortAddrMax  = 0xfdff;
```

### 3.6  `CplHandler` state machine calls

The CPL 3-way handshake (Wake Frame → CP Link Command → Enh-ACK) is entirely a **MAC-layer operation**. It is different from `mle_p2p.cpp`, which is **not touched** by this feature.

`CplHandler::OnLinkEstablished()` calls `Get<PeerTable>().Add(peer)` directly. This follows the same cross-layer pattern already used in the `Mac` subsystem (e.g., `Mac` calls `Get<MeshForwarder>()` for TX scheduling). There is no MLE involvement in the CPL handshake path.

MAC command dispatch in `Mac::HandleMacCommand()` gains a new case:

```cpp
case Frame::kMacCmdCplCommand:  // Thread MAC Command ID = 0x02, inside a 0x54 MAC Command frame
#if OPENTHREAD_CONFIG_P2P_ENABLE
    Get<CplHandler>().HandleCplMacCommand(aFrame);
    didHandle = true;
#endif
    break;
```

Teardown: **SCA IE with empty payload** (Length=0 in the Thread Header IE 0x2d) signals teardown per spec §16.10. `CplHandler::SendTeardown()` constructs and transmits this as a MAC Command 0x54 frame with an empty SCA LTV. The existing `otP2pUnlink()` API triggers this path.

### 3.7  Post-Link Data Transfer and Supervision

CPL is a **symmetric peer-to-peer link between two CP Devices** (spec §16.1, §16.3 Design Goal "Symmetric"). Both WI and WL may be sleepy and/or CoEx-constrained; neither has its radio always on. After `OT_P2P_EVENT_LINKED` fires, each side knows the peer's SLW schedule and can schedule timed transmissions into the peer's receive windows accordingly.

**SCA IE exchange during handshake**

- **WL → WI:** The WL's SCA IE (SLW period + phase + optional RAM) is carried in the CP Link Command (§16.8). The WI records this in `Peer.mSlwPeriodSlots` / `Peer.mSlwPhaseSlots`.
- **WI → WL:** Per spec §16.8, the WI *may* include its own SCA IE in any frame — including the Enh-ACK and subsequent data frames. After receiving the CP Link Command, `CplHandler` on the WI side includes its own SCA IE in the Enh-ACK payload (if `OPENTHREAD_CONFIG_WAKEUP_COORDINATOR_ENABLE` is set and the WI itself has a configured SLW schedule). The WL records the WI's SLW schedule from the Enh-ACK or the first data frame.

**Data TX (WI → WL)**

Frames destined for the WL must arrive during the WL's SLW window. The mechanism is directly analogous to CSL enhanced indirect TX in `src/core/mac/csl_tx_scheduler.cpp`:

1. `MeshForwarder` has a pending frame addressed to the WL's CPL short address.
2. A new `CplTxScheduler` computes the next SLW window start time from `Peer.mSlwPhaseSlots` and `Peer.mSlwPeriodSlots`, using the same epoch-relative phase tracking that `CslTxScheduler::GetNextTxDelay()` uses for CSL.
3. The frame is submitted via `Mac::RequestDirectFrameTransmission()` with a TX timestamp targeting the SLW window, with the IEEE 802.15.4 *Frame Pending* bit set if further frames are queued.
4. The WL ACKs (clearing the pending bit when its RX queue is empty) or issues a MAC Data Poll to retrieve queued frames.

**Data TX (WL → WI)**

Symmetric to the above: the WL schedules outgoing frames to arrive during the WI's SLW windows, using the WI's SCA schedule learned from the Enh-ACK or a subsequent WI data frame. The WL's `CplTxScheduler` instance (on the WL side) uses `PeerSlwPeriodSlots` / `PeerSlwPhaseSlots` stored for the WI peer entry, in the same way as WI→WL.

If the WI has not yet advertised its own SCA IE (i.e., the WI has no SLW schedule configured), the WL falls back to standard IEEE 802.15.4 unicast TX with MAC retries, targeting the WI's extended address. The WI is expected to have its radio open at least long enough to ACK — matching the normal OpenThread behavior for a non-sleepy device in that role.

> **SLW phase reference:** The SCA IE gives period and phase in 160 µs slots but the spec draft does not define the absolute epoch reference for the phase. This is the same open issue as CSL phase tracking; resolution must be tracked against the spec and will follow the same pattern OpenThread uses for CSL phase exchange. See §8 open items.

**Supervision Interval**

Per spec §16.10.5: if a device does not transmit or receive a frame from its peer within the Supervision Interval, it enqueues and sends a link supervision message (MAC frame, zero-length payload, ACK requested). The spec currently states supervision frames are sent WI→WL, with a TBD note about whether the symmetric design requires both directions.

This design follows the spec's current (WI→WL only) statement:

- **WI side:** `CplHandler` starts a `mSupervisionTxTimer` (fires at `supervision_interval / 2` after each delivery). If no data is pending when the timer fires, `CplHandler::SendSupervisionFrame()` sends a zero-payload MAC frame to the WL's SLW window. This is directly analogous to `ChildSupervisor` (`child_supervisor.cpp`). *Note: the spec requires one supervision frame if no TX/RX occurred within the full Supervision Interval; firing at half the interval is an intentional conservative choice that sends extra keepalives at roughly 2× the required rate, trading a small overhead for earlier link-loss detection on the WL's RX timer.*
- **WL side:** `CplHandler` starts a `mSupervisionRxTimer`, reset on each received frame from the WI. On expiry, link loss detection triggers (§16.10.6): the WL drops the connection and returns to wake listen mode (see Teardown below).

**Link Loss Detection and Recovery (§16.10.6–16.10.7)**

If either side fails to get an IEEE 802.15.4 ACK after `macCslMaxFrameRetries` (7) retransmissions:
- **WL:** drops the connection, goes back to WED listen mode, fires `OT_P2P_EVENT_UNLINKED`.
- **WI:** triggers a fresh Wake Frame transmission cycle to re-establish the link.

**Teardown Conditions (§16.10.8)**

| Condition | Originating side | Action |
|---|---|---|
| `otP2pUnlink()` called | Either | `SendTeardown()` — SCA IE with L=0; fire `OT_P2P_EVENT_UNLINKED`; remove peer from `PeerTable` |
| Supervision RX timer expires on WL | WL | Link loss detection → drop connection; return to WED listen; fire `OT_P2P_EVENT_UNLINKED` |
| Max retransmission failures | Either | Link loss detection → see above per role |
| SCA teardown IE received from peer | Either | Parse empty SCA LTV; fire `OT_P2P_EVENT_UNLINKED`; remove peer from `PeerTable`; stop timers |

In all cases the `Peer` entry is removed from `PeerTable`, freeing the CPL short address slot for reuse.

---

## 4. Public API Surface

### 4.1  Provisional API

Some functions in `include/openthread/provisional/p2p.h` **already mirror the proposed CP Link API**:
- `otP2pWakeupAndLink()` — WI: wake + establish links
- `otP2pUnlink()` — either side: tear down
- `otP2pSetEventCallback()` — event notifications (`OT_P2P_EVENT_LINKED` / `OT_P2P_EVENT_UNLINKED`)

`otWakeupId` is currently defined as `uint64_t`.  The spec defines `WakeID` as an 8-byte field in the HMAC challenge, so **no type change is required.**

`otP2pRequest` with new fields:

```c
typedef struct otP2pRequest
{
    otWakeupRequest mWakeupRequest;               ///< Wake-up request (existing).
    uint16_t        mWakeupIntervalUs;            ///< Wake interval (0 = use config default).
    uint16_t        mWakeupDurationMs;            ///< Wake duration (0 = use config default).
} otP2pRequest;
```

### 4.2  New API for SCA Schedule (WL side)

The WL needs to advertise its SLW schedule to the WI.  This is application-configurable:

```c
// include/openthread/provisional/p2p.h

/**
 * Configures the SLW schedule that this device will advertise to Wake Initiators.
 *
 * Requires OPENTHREAD_CONFIG_WAKEUP_END_DEVICE_ENABLE.
 * A period of 0 clears the schedule (sends teardown SCA IE on next link establishment).
 *
 * @param[in] aInstance        The OpenThread instance.
 * @param[in] aSlwPeriodSlots  SLW Period in 160µs slots (must be multiple of 375 for CoEx,
 *                              or multiple of 250 for non-CoEx; spec §16.9.1).
 * @param[in] aSlwPhaseSlots   SLW Phase in 160µs slots.
 */
otError otP2pSetSlwSchedule(otInstance *aInstance, uint16_t aSlwPeriodSlots, uint16_t aSlwPhaseSlots);

/**
 * Represents the SCA (Scheduled Channel Access) schedule state of a CP peer.
 */
typedef struct otP2pScaState
{
    uint16_t mSlwPeriodSlots; ///< SLW Period in 160 µs slots (0 = not configured).
    uint16_t mSlwPhaseSlots;  ///< SLW Phase in 160 µs slots.
    bool     mHasSchedule;    ///< true if an SCA schedule has been received from this peer.
    bool     mCoexEnabled;    ///< true if peer reported CoEx constraints in the SCA IE.
} otP2pScaState;

/**
 * Gets the SCA schedule of a CP peer.
 *
 * @param[in]  aInstance     The OpenThread instance.
 * @param[in]  aExtAddress   The peer's extended address.
 * @param[out] aState        The peer's SCA state.
 */
otError otP2pGetPeerScaState(otInstance *aInstance, const otExtAddress *aExtAddress, otP2pScaState *aState);
```

---

## 5. Platform Abstraction Layer (PAL) Design

### 5.1  Enh-ACK Thread Header IE Injection

> **Note:** The design of this API is subject to Decision D in §1.2. The API below represents the current draft; the final form depends on which implementation option reviewers select.

The spec's 3-way handshake requires the WI's **Enh-ACK** to carry a Thread Header IE 0x2d containing a `ChallengeIe`. The stack can pre-compute the IE bytes and push them to the platform to inject them into the Enh-ACK autonomously (for example, this follows the **existing `otPlatRadioConfigureEnhAckProbing()` pattern** already established in the upstream codebase for Link Metrics probing).

```c
// include/openthread/platform/radio.h

/**
 * Configures the Thread Header IE data to be included in the Enh-ACK sent to
 * a specific source address.
 *
 * This API is called by the stack after computing the HMAC-SHA256 challenge response.
 * The platform MUST include the @p aIeData bytes as a Thread Header IE (element ID 0x2d)
 * in the Enh-ACK generated for the next received frame from @p aSrcExtAddress that has
 * the AckRequest bit set.
 *
 * The IE data is consumed after a single Enh-ACK. Setting @p aIeDataLength to 0 clears
 * any pending configuration for @p aSrcExtAddress.
 *
 * @note Platforms that delegate Enh-ACK generation to the RCP use the corresponding
 *       Spinel property (SPINEL_PROP_RCP_CPL_ENH_ACK_IE) to push the configuration over
 *       the transport.
 *
 * @param[in] aInstance        OpenThread instance.
 * @param[in] aSrcExtAddress   Extended address of the CP Link Command sender (peer WL).
 * @param[in] aIeData          Pointer to the Thread Header IE payload (LTV bytes only,
 *                              not including the IE header itself). May be NULL to clear.
 * @param[in] aIeDataLength    Length of @p aIeData in bytes.
 *
 * @retval OT_ERROR_NONE          Configured successfully.
 * @retval OT_ERROR_INVALID_ARGS  @p aIeDataLength exceeds platform maximum.
 * @retval OT_ERROR_NOT_CAPABLE   Platform does not support this API.
 */
otError otPlatRadioConfigureCplEnhAckIe(otInstance         *aInstance,
                                         const otExtAddress *aSrcExtAddress,
                                         const uint8_t      *aIeData,
                                         uint16_t            aIeDataLength);
```

No new radio capability bits are being introduced.

**Platform implementation:**

The platform can store the pre-computed IE bytes in a small per-peer buffer. When generating an Enh-ACK for an incoming frame, the platform checks whether a pending IE configuration exists for the frame's source address, appends the Thread Header IE if so, then clears the entry. For group wake (when `OPENTHREAD_CONFIG_P2P_GROUP_WAKEUP_ENABLE` is set), this can scale to a small table keyed by source address.

### 5.2  MAC Command 0x54 Frame Acceptance

On platforms where the MAC frame filter must be explicitly configured during WED listen mode, the platform's WED receive configuration must accept `kTypeMacCmd` frames (and optionally existing Multipurpose frames if they indicate support for the legacy method).

### 5.3  WL listening

WL listen windows continue to use `otPlatRadioReceiveAt()` as-is. No changes needed.

### 5.4  Wake Frame TX Security

The platform's transmit security path already handles KeyIdMode1 and KeyIdMode2 frames. The only change visible to the platform is that `IsWakeupFrame()` now returns true for MAC Command 0x54 frames **in addition to** the legacy Multipurpose type check (assuming both are accepted, when `OPENTHREAD_CONFIG_CPL_USE_LEGACY_WAKEUP_FRAME` is active). The key material lookup by Key Index 129 flows through the standard `Mac::ProcessTransmitSecurity()` path unchanged.

---

## 6. Spinel Commands (NCP / RCP Architecture)

### 6.1  NCP Architecture (stack on NCP, host communicates via Spinel)

For the NCP case, the host application sends Spinel `PROP_VALUE_SET` / `PROP_VALUE_GET` commands over UART/SPI; the NCP translates these internally to `otP2pWakeupAndLink()` etc. and fires async `PROP_VALUE_IS` notifications back to the host for events.  The following new properties are added to Spinel, proposed under the `SPINEL_PROP_VENDOR__BEGIN` range initially and nominated for standardization as a Thread-defined property range:

```
SPINEL_PROP_THREAD_CPL__BEGIN = SPINEL_PROP_THREAD__BEGIN + 0x60  (provisional range)
```

| Property | ID | Format | Description |
|---|---|---|---|
| `SPINEL_PROP_THREAD_CPL_ENABLE` | +0x00 | `b` | Enable/disable CPL (mirrors `OPENTHREAD_CONFIG_P2P_ENABLE`) |
| `SPINEL_PROP_THREAD_CPL_MAX_PEERS` | +0x01 | `S` | Max peers (read-only) |
| `SPINEL_PROP_THREAD_CPL_WAKEUP_CHANNEL` | +0x02 | `C` | Wake channel (maps to `otLinkSetWakeupChannel`) |
| `SPINEL_PROP_THREAD_CPL_WL_LISTEN_ENABLE` | +0x03 | `b` | Enable WL listening (`otLinkSetWakeUpListenEnabled`) |
| `SPINEL_PROP_THREAD_CPL_WL_LISTEN_PARAMS` | +0x04 | `LL` | interval_us, duration_us (`otLinkSetWakeupListenParameters`) |
| `SPINEL_PROP_THREAD_CPL_WI_WAKEUP` | +0x05 | `t(...)` | Trigger WI wake+link, returns done asynchronously |
| `SPINEL_PROP_THREAD_CPL_UNLINK` | +0x06 | `E` | Trigger WL unlink by ExtAddress |
| `SPINEL_PROP_THREAD_CPL_PEER_TABLE` | +0x07 | `A(t(...))` | Read peer table (array of peer info) |
| `SPINEL_PROP_THREAD_CPL_EVENT` | +0x08 | `CE` | Async notify: event code + peer ExtAddress |
| `SPINEL_PROP_THREAD_CPL_SLW_SCHEDULE` | +0x09 | `SS` | period_slots, phase_slots (`otP2pSetSlwSchedule`) |

**`SPINEL_PROP_THREAD_CPL_WI_WAKEUP` format** (TLV encoded in `t()`):

```
Field         Type    Description
WakeupType    uint8   0=ExtAddress, 1=WakeupId, 2=GroupId
WakeupTarget  bytes   ExtAddress (8B) or WakeupId (8B) depending on WakeupType
IntervalUs    uint16  Wake interval (0=default)
DurationMs    uint16  Wake duration (0=default)
```

Response: `SPINEL_CMD_PROP_VALUE_IS` on `SPINEL_PROP_THREAD_CPL_EVENT` with link-done event.

### 6.2  RCP Architecture (host runs OT stack, RCP is the radio)

In the RCP architecture, the host stack handles all P2P / CPL logic.  The RCP only needs standard radio operations.  The **sole new Spinel property required for the RCP** is the Enh-ACK IE injection, defined as a direct parallel to the existing `SPINEL_PROP_RCP_ENH_ACK_PROBING` (at `SPINEL_PROP_RCP_EXT__BEGIN + 3`):

```c
// spinel.h — in the SPINEL_PROP_RCP_EXT range
SPINEL_PROP_RCP_CPL_ENH_ACK_IE = SPINEL_PROP_RCP_EXT__BEGIN + 6,
```

| Property | Format | Direction | Description |
|---|---|---|---|
| `SPINEL_PROP_RCP_CPL_ENH_ACK_IE` | `Ed` | host → RCP | Configure Enh-ACK IE: ExtAddress (8B) + raw Thread Header IE payload bytes. (Spinel type codes: `E` = EUI-64 8-byte address, `d` = length-prefixed raw data.) |

**`radio_spinel.cpp` binding** — new method maps to `otPlatRadioConfigureCplEnhAckIe()`:

```cpp
otError RadioSpinel::ConfigureCplEnhAckIe(const otExtAddress &aSrcExtAddress,
                                           const uint8_t      *aIeData,
                                           uint16_t            aIeDataLength)
{
    return Set(SPINEL_PROP_RCP_CPL_ENH_ACK_IE,
               SPINEL_DATATYPE_EUI64_S SPINEL_DATATYPE_DATA_S,
               aSrcExtAddress.m8,
               aIeData, aIeDataLength);
}
```

**RCP firmware side**: The property handler decodes the `Ed` frame (EUI-64 + data), stores the IE bytes keyed by source address, then injects them into the next Enh-ACK generated for that peer — the same mechanism as the `SPINEL_PROP_RCP_ENH_ACK_PROBING` handler.

No other new Spinel properties are needed for the RCP case — all other CPL operations (wake frame TX, WED ReceiveAt, standard data TX to peer) use existing Spinel mechanisms.

---

## 7. CLI Extensions

New CLI subcommands under `wakeup` / `p2p` follow the existing CLI pattern in `src/cli/cli.cpp`:

```
# WI side
p2p connect ext <extaddr> [interval_us] [duration_ms]     # otP2pWakeupAndLink (ExtAddress)
p2p connect id <wakeupid_hex> [interval_us] [duration_ms] # otP2pWakeupAndLink (WakeupId)
p2p disconnect <extaddr>                                  # otP2pUnlink

# WL side  
wakeup listen enable / disable
wakeup parameters <interval_us> <duration_us>
wakeup channel <channel>
wakeup wakeupid add <hex64> / remove <hex64> / clear

# SCA schedule (WL advertises to WI)
p2p slw period <slots>   # SLW Period in 160µs slots
p2p slw phase <slots>    # SLW Phase in 160µs slots

# Peer table
p2p peers                # List all valid CP peers
```

---

## 8. Open Items Tracked Against the Spec

| Item | Spec section | Status |
|---|---|---|
| IEEE 802.15.4 MAC Command ID 0x54 formal allocation | §16.14 | Pending IEEE; using `0x54` provisional |
| Thread MAC Command sub-namespace: Wake Frame Type 0x02 (Connectionless) vs. CP Link Command 0x02 | §16.5.7.4 gap | Gap acknowledged in spec; using `0x01`=Wake Command and `0x02`=CP Link Command in the Thread MAC Command ID field; within Wake Command payload, Wake Frame Type `0x02` (connectionless) does not collide since it is at a different byte position. Connectionless Wake is still a well-defined type per §16.5.2. |
| Group wake TX/RX (`OPENTHREAD_CONFIG_P2P_GROUP_WAKEUP_ENABLE`) | §16.5 | Is group wake spec final? |
| eCSL slot contention between CPL SLW windows and regular CSL windows | §16.9 | Slot algorithm needs to be finalized in spec. Can follow `GetNeededShift()` pattern |
| Multi-protocol CoEx RAM encoding (RAM Bits) | §16.10.2 | Single-protocol path first: RAM Duration=1 (no CoEx constraints), RAM Bits absent |
| Wake Key provisioning for guest access (custom Wake Keys outside default) | §16.5.9 | Default Wake Key (Key Index 129) covered; custom key provisioning API can be deferred |
| WakeID HMAC input byte length: spec defines WakeID as 1–8 bytes variable length; this implementation always passes 8 bytes (zero-padded) | §16.7.3 | Track against spec clarification |
| Replay protection: WL MUST NOT enforce rate limits on Wake Frames even if challenge fails | §16.7.4 | Behavioral requirement — must be enforced in `sub_mac_wed.cpp` WL receive path; documented in §3.3.2 |
| Randomize listen window after failed challenge: WL MUST randomize start time of its next listen window to prevent Covert DoS | §16.7.4 | Behavioral requirement — documented in §3.3.2; implementation: add jitter to `WakeListenTimer` restart |
