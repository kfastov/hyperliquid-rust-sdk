# Hyperliquid API Spec — OI Cap relevant parts

## 1. REST: perpsAtOpenInterestCap

`POST https://api.hyperliquid.xyz/info`

Request:
```json
{"type": "perpsAtOpenInterestCap", "dex": "<optional dex name>"}
```

Response (200):
```json
["BADGER","CANTO","FTM","LOOM","PURR"]
```

Returns array of coin names that are currently at their open interest cap.

## 2. WS: webData3 subscription

Subscription message:
```json
{"method": "subscribe", "subscription": {"type": "webData3", "user": "<address>"}}
```

Data format:
```typescript
interface WebData3 {
  userState: {
    agentAddress: string | null;
    agentValidUntil: number | null;
    serverTime: number;
    cumLedger: number;
    isVault: boolean;
    user: string;
    optOutOfSpotDusting?: boolean;
    dexAbstractionEnabled?: boolean;
  };
  perpDexStates: Array<PerpDexState>;
}

interface PerpDexState {
  totalVaultEquity: number;
  perpsAtOpenInterestCap?: Array<string>;  // ← THIS IS WHAT WE NEED
  leadingVaults?: Array<LeadingVault>;
}

interface LeadingVault {
  address: string;
  name: string;
}
```

Notes from docs:
- "Additional undocumented fields in WebData3 will be removed on a future update"
- `perpDexStates` is an array (one per dex)
- `perpsAtOpenInterestCap` is optional (may be absent)
- Contains same data as REST endpoint but pushed via WS

## 3. Existing SDK WS types (for reference)

The SDK currently has WebData2 support:
```rust
// src/ws/ws_manager.rs
pub enum Subscription {
    WebData2 { user: Address },
    // ... others
}

pub enum Message {
    WebData2(WebData2),
    // ... others
}

// src/ws/message_types.rs
pub struct WebData2 {
    pub data: WebData2Data,
}

// src/ws/sub_structs.rs
pub struct WebData2Data {
    // ... fields
}
```

## 4. Existing SDK REST InfoRequest (for reference)

```rust
// src/info/info_client.rs
enum InfoRequest {
    ActiveAssetData { user: Address, coin: String },
    // ... others
    // NO perpsAtOpenInterestCap variant exists
}
```

## 5. Order rejection statuses related to OI cap

From order status endpoint:
- `openInterestCapCanceled` — Canceled due to order being too aggressive when open interest was at cap
- `positionIncreaseAtOpenInterestCapRejected` — Rejected due to open interest cap
- `positionFlipAtOpenInterestCapRejected` — Rejected due to open interest cap
- `tooAggressiveAtOpenInterestCapRejected` — Rejected due to price too aggressive at open interest cap
- `openInterestIncreaseRejected` — Rejected due to open interest cap

## 6. Meta response includes isDelisted

```json
{
    "name": "LOOM",
    "szDecimals": 1,
    "maxLeverage": 3,
    "isDelisted": true,
    "marginMode": "strictIsolated",
    "onlyIsolated": true
}
```

Already supported in our fork via `AssetMeta.is_delisted: Option<bool>`.
