# VFX Core CLI

C# .NET 6.0 blockchain node, CLI wallet, and validator for the VerifiedX network. This is Tyler's fork for development/testing — changes here get validated before merging upstream to `VerifiedXBlockchain/VerifiedX-Core`.

## Quick Reference

| Property | Value |
|----------|-------|
| Language | C# 10.0, .NET 6.0 |
| Database | LiteDB (embedded NoSQL) |
| P2P | SignalR (WebSocket) |
| Crypto | ECDSA secp256k1, BIP32/39, FROST MPC |
| API Port | 7292 |
| P2P Port | 3338 |

## Project Structure

```
ReserveBlockCore/
├── Program.cs                    # Entry point
├── Controllers/                  # REST API endpoints
│   ├── TXV1Controller.cs         # Transaction APIs (raw TX, fee, hash, send)
│   ├── SCV1Controller.cs         # Smart contract APIs
│   ├── BTCV2Controller.cs        # V1 tokenization APIs
│   └── Bitcoin/Controllers/
│       └── VBTCController.cs     # vBTC V2 APIs (ceremony, transfer, withdraw, Raw endpoints)
├── Services/                     # Business logic
│   ├── SmartContractService.cs   # SC mint/transfer/burn (beacon upload + asset queue)
│   ├── TransactionValidatorService.cs  # TX verification
│   ├── FeeCalcService.cs         # Fee calculation
│   └── StartupService.cs         # Initialization, beacon loading
├── Bitcoin/
│   ├── Services/
│   │   ├── VBTCService.cs        # vBTC V2 operations (transfer, withdraw, ownership)
│   │   ├── FrostMPCService.cs    # FROST DKG + signing ceremonies
│   │   └── BitcoinTransactionService.cs  # BTC TX building + FROST signing
│   └── FROST/
│       └── Models/               # FROST message types, PreSignedLeaderAuth
├── Privacy/                      # Shielded transactions (PLONK, Poseidon, Pedersen)
├── Models/                       # Data models
│   ├── Transaction.cs            # TX types enum (0-38)
│   └── Privacy/                  # Shielded wallet, commitments, nullifiers
├── Utilities/
│   └── SmartContractUtility.cs   # Gzip compress/decompress for SC data
└── Data/
    ├── StateData.cs              # State Trei processing
    └── DbContext.cs              # LiteDB collections
```

## Key Concepts

### Transaction Types
Types 0-38. The ones we work with most:
- 25: VBTC_V2_CONTRACT_CREATE (mint)
- 26: VBTC_V2_TRANSFER
- 27: VBTC_V2_WITHDRAWAL_REQUEST
- 28: VBTC_V2_WITHDRAWAL_COMPLETE
- 29: VBTC_V2_WITHDRAWAL_CANCEL
- 31-36: Shielded/privacy transactions

### Raw Endpoints (Web Wallet)
All vBTC V2 write operations for the web wallet use a two-step pattern:
1. `Get*Data` — CLI builds unsigned TX, stores in `_pendingRawVbtcTxs`, returns Hash
2. `Send*Tx` — Client signs Hash, CLI stamps signature and broadcasts

This avoids the `GetSingleAccount` / local key requirement.

### vBTC V2 Operations
- **MPC Ceremony**: `PrepareMPCCeremonyRaw` → `ExecuteMPCCeremonyRaw` (DKG with 93 validators)
- **Transfer**: `GetRawTransferVBTCData` → `SendRawTransferVBTCTx`
- **Withdrawal**: 4-step flow (request → FROST signing → BTC broadcast → completion TX)
- **Ownership Transfer**: Beacon upload → `GetVBTCOwnershipTransferData` → raw TX with TKNZ_TX type

### FROST Signing
- Pre-signed leader auth: web wallet signs `{sessionId}.{ownerAddress}.{timestamp}` messages
- CLI coordinates with validators using these signatures instead of local `AddressSignature()`
- `SignatureService.AddressSignature()` is the function that requires local keys — pre-signed auth bypasses it

### Smart Contract Data Encoding
SC code must be gzip + base64 encoded before going into TX data:
```csharp
var bytes = Encoding.Unicode.GetBytes(scCode);
var encoded = Convert.ToBase64String(SmartContractUtility.Compress(bytes));
```
The `SmartContractUtility.Compress()` method is in `Utilities/SmartContractUtility.cs`.

### Beacon System
- Beacons propagate SC assets to recipients during transfers
- `CreateBeaconUploadRequest` → uploads assets → returns Locator
- Beacon connections are lazily initialized (not at startup)
- Asset queue (`rsrvassetqueue.db`) tracks pending transfers — can get stuck if TX fails mid-flight

## Hard Rules

1. **Never modify consensus logic** without Aaron's review
2. **Test on the headless CLI instance** (`44.254.72.141:7292`) before merging upstream
3. **Raw endpoints must not require local keys** — that's the whole point of the two-step pattern
4. **SC data must always be gzip+base64 encoded** in TX payloads (the SCVersion/encoding bug taught us this)
5. **FROST signing can take 30-60 seconds** — callers must handle timeouts

## Known Issues

- First beacon upload after CLI restart fails (lazy init, `Globals.Beacon` starts empty)
- Asset queue gets stuck if transfer fails mid-flight (no cleanup on success path in `SmartContractService.TransferSmartContract` line 449)
- `VBTCContractV2.Balance` (local DB) doesn't reflect actual BTC balance — use State Trei or Spyglass instead
- FROST signing fails on tokens created by older CLI versions (DKG incompatibility)

## Upstream

- **Upstream repo**: `VerifiedXBlockchain/VerifiedX-Core`
- **This fork**: `tylersavery/VerifiedX-Core`
- Changes validated here, then merged to upstream or sent as PRs to Aaron
