# Hydration Onboarding Specification for NEAR Intents

**Version:** 1.0  
**Date:** January 1, 2026  
**Status:** Draft  
**Author:** Based on NEAR Intents ecosystem research  

---

## Executive Summary

Discussion document for integrating Hydration (Polkadot parachain) into NEAR Intents. Presents technical specifications, bridge options, implementation approaches, and recommended paths forward for evaluation by both Hydration and NEAR Intents teams.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Prerequisites & Requirements](#2-prerequisites--requirements)
3. [Architecture Overview](#3-architecture-overview)
4. [Signature Support - Sr25519](#4-signature-support---sr25519)
5. [Bridge Integration](#5-bridge-integration)
6. [Verifier Smart Contract Integration](#6-verifier-smart-contract-integration)
7. [SDK & Tooling Integration](#7-sdk--tooling-integration)
8. [1-Click API Integration](#8-1-click-api-integration)
9. [Message Bus Integration](#9-message-bus-integration)
10. [Testing & Validation](#10-testing--validation)
11. [Documentation Updates](#11-documentation-updates)
12. [Deployment Checklist](#12-deployment-checklist)
13. [Open Questions](#13-open-questions)

---

## 1. Introduction

### 1.1 Components

**NEAR Intents**: Multichain transaction protocol with three components:
- Distribution Channels (wallets, apps)
- Market Makers (liquidity providers)
- Verifier Contract (`intents.near`)

**Hydration**: Polkadot parachain DEX (Parachain ID: 2034) with dual architecture:
- Native Substrate runtime (WASM-based)
- EVM compatibility layer via Frontier pallet (EVM Chain ID: 222222)
- Cross-chain messaging via Polkadot XCM

### 1.2 Integration Scope

- Sr25519 signature verification
- Bridge integration (POA/HOT/Wormhole/Custom)
- Token deposits and withdrawals
- SDK support for Hydration addresses and signatures

---

## 2. Prerequisites & Requirements

### 2.1 Technical Requirements

**Blockchain Specifications:**
- **Genesis Hash**: `0xafdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d`
- **CAIP-2 Identifier**: `polkadot:afdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d`
- **Parachain ID**: 2034
- **EVM Chain ID**: 222222
- **Consensus**: Aura consensus as Polkadot parachain (collator-based block production)
- **Architecture**: Substrate-based parachain with EVM compatibility (Frontier pallet)
- **Block Time**: 6 seconds (average)
- **Finality**: Polkadot relay chain finality (~12-18 seconds, deterministic via GRANDPA)
- **Native Token**: HDX (Asset ID: 0, 12 decimals)
- **RPC Endpoints**: See Section 2.1.1 below

#### 2.1.1 Public RPC Endpoints (All Archive Nodes)

**Primary (Mainnet):**
- `wss://rpc.hydradx.cloud`

**Alternative Providers:**
- `wss://hydration-rpc.n.dwellir.com` (Dwellir)
- `wss://hydration.dotters.network`
- `wss://rpc.helikon.io/hydradx`
- `wss://hydration.ibp.network`
- `wss://rpc.cay.hydration.cloud`
- `wss://rpc.parm.hydration.cloud`
- `wss://rpc.roach.hydration.cloud`
- `wss://rpc.zipp.hydration.cloud`
- `wss://rpc.sin.hydration.cloud`
- `wss://rpc.coke.hydration.cloud`

**Testnet:**
- Paseo testnet available for testing

**Address & Account Format:**
- Substrate SS58 address format
- Network prefix: 0 (Polkadot/generic format)
- Account derivation methods

**Cryptographic Specifications:**

**Native (Substrate side):**
- Primary: Sr25519 (Schnorrkel/Ristretto) - PR #171 in progress
- SS58 address encoding
- Public key: 32 bytes
- Signature: 64 bytes
- Wallets: Polkadot.js, Talisman, SubWallet, Nova Wallet

**EVM side:**
- secp256k1 (ECDSA) - standard Ethereum signatures
- Keccak-256 hashing
- Standard Ethereum addresses (H160, 0x-prefixed 20-byte)
- Wallets: MetaMask and other Ethereum-compatible wallets

**Dual Account System:**
- Both signature schemes supported on Hydration
- Users can interact with either Substrate or EVM accounts
- Account binding capability between Substrate and EVM addresses
- `pallet_evm_accounts` manages linking and conversion
- Bidirectional ERC-20 mapping via `HydraErc20Mapping`

### 2.2 Bridge Requirements

Determine which bridge solution to use:
- **Wormhole (via Moonbeam)**: Operational via Moonbeam Routed Liquidity (MRL) with XCM
- **POA Bridge**: Currently supports BTC, ETH, SOL, TRON (no Substrate support)
- **HOT Bridge (Omni)**: Currently supports EVM chains, TON, Stellar (no Substrate support)

See Section 5 for detailed bridge options analysis.

---

## 3. Architecture Overview

### 3.1 User Intent Flow (NEAR Intents Protocol)

```
1. User expresses intent (swap, transfer, withdraw)
   ↓
2. Frontend/App calls Message Bus API (quote)
   ↓
3. Message Bus forwards to connected Solvers
   ↓
4. Solvers respond with quotes
   ↓
5. User selects quote, signs intent (NEP-413/ERC-191/sr25519)
   ↓
6. Frontend publishes signed intent to Message Bus
   ↓
7. Solver responds with counter-intent
   ↓
8. Message Bus bundles intents, submits to Verifier
   ↓
9. Verifier contract (intents.near) executes atomically
```

### 3.2 Hydration Integration Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   Message Bus (Solver Relay v2)                  │
│         https://solver-relay-v2.chaindefuser.com/rpc            │
│                                                                  │
│  • Matches user intents with solver quotes                      │
│  • Supports all Defuse Asset Identifiers                        │
│  • Bundles signed intents for atomic execution                  │
└──────────────┬──────────────────────────┬────────────────────────┘
               │                          │
               │ User Intents             │ Solver Quotes
               │                          │
      ┌────────▼─────────┐       ┌───────▼────────┐
      │  Hydration Users │       │    Solvers     │
      │                  │       │  (Any Chain)   │
      │ Sign with:       │       │                │
      │ • sr25519        │       │ Monitor Bus    │
      │ • ECDSA (EVM)    │       │ Provide quotes │
      └────────┬─────────┘       └───────┬────────┘
               │                         │
               │ Both submit to          │
               └─────────┬───────────────┘
                         │
                         ▼
      ┌──────────────────────────────────┐
      │    NEAR Intents Verifier         │
      │    (intents.near contract)       │
      │                                  │
      │ • Validates signatures           │
      │ • Executes atomic swaps          │
      │ • Manages token balances         │
      │ • Emits events                   │
      └──────────┬──────────────────┬────┘
                 │                  │
        (Deposit)│                  │(Withdrawal)
                 │                  │
      ┌──────────▼─────┐   ┌────────▼────────┐
      │ Bridge to      │   │ Bridge from     │
      │ Hydration      │   │ Hydration       │
      └──────────┬─────┘   └────────▲────────┘
                 │                  │
                 │  Token Flow      │
                 │                  │
      ┌──────────▼──────────────────┴────────┐
      │         Hydration Parachain          │
      │                                      │
      │ Users can:                           │
      │ • Hold tokens                        │
      │ • Create intents to trade            │
      │ • Receive filled orders              │
      │                                      │
      │ Solvers can:                         │
      │ • Provide liquidity via Omnipool     │
      │ • Execute swaps                      │
      │ • Fulfill user intents               │
      │                                      │
      │ Chain Features:                      │
      │ • Asset Registry (u32 IDs)          │
      │ • Omnipool DEX                       │
      │ • EVM compatibility (Frontier)       │
      │ • Multi-asset fee payment            │
      │ • Dual signatures (sr25519/ECDSA)   │
      └──────────────────────────────────────┘
```

### 3.3 Bridge Options

**Option C (Recommended): Wormhole via MRL**
```
Hydration ←→ XCM ←→ Moonbeam ←→ Wormhole ←→ NEAR
         (Operational)    (19 Guardians)
```

**Option A: POA Bridge** (No Substrate support yet)  
**Option B: HOT Bridge** (No Substrate support yet)

### 3.4 Key Integration Points

1. **Signature Support**
   - Sr25519 for Hydration Substrate accounts (Polkadot.js, Talisman, etc.)
   - ECDSA for Hydration EVM accounts (MetaMask, etc.)
   - Verifier contract must support sr25519 (PR #171)

2. **Asset Identifiers**
   - Format: `polkadot:afdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d:{assetId}`
   - Examples: HDX (0), USDT (10), USDC (21), HOLLAR (222)

3. **Bridge Integration**
   - Bidirectional token flow between NEAR and Hydration
   - Wrapped tokens on NEAR side after bridging
   - MRL provides proven Hydration ↔ Moonbeam ↔ Wormhole path

### 3.1 Chain Identifier (CAIP-2)

Hydration uses the CAIP-2 format for Substrate-based chains:
```
polkadot:<genesis-hash>
```

**Confirmed Identifier:**
```typescript
Chains.Hydration: "polkadot:afdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d"
```

**Genesis Hash**: `0xafdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d`

---

## 4. Signature Support - Sr25519

### 4.1 Current Status

**PR #171**: "Add support for sr25519 signatures"  
**Repository**: https://github.com/near/intents/pull/171  
**Status**: In review (as of research date)

This PR adds comprehensive sr25519 support including:
- Sr25519 curve implementation using `schnorrkel` library
- SignedSr25519Payload type
- Integration into MultiPayload enum
- Public key and signature parsing
- Verification logic

### 4.2 Message Wrapping

Polkadot wallets (like Talisman, Polkadot.js) wrap signed messages with:
```
<Bytes>...message content...</Bytes>
```

The verification implementation must handle this wrapping at the verification stage, not in the JSON payload.

### 4.3 Required Actions

1. **Complete PR #171 Review & Merge**
   - Ensure all test cases pass, especially with real Polkadot/Substrate wallets
   - Verify wallet signature test with Talisman or PolkadotJS wallet
   - Document signing context requirements

2. **Wallet Integration Testing**
   - Test with Talisman wallet
   - Test with Polkadot.js extension
   - Test with SubWallet
   - Test with Nova Wallet

3. **Add to Chain Support Documentation**
   - Update `chain-address-support.md` with Hydration support
   - Document address format requirements
   - Add signing standard (Sr25519/Substrate) to table

---

## 5. Bridge Integration

### 5.1 Bridge Options Analysis

#### Option A: POA Bridge
**Current Support**: BTC, ETH, SOL, TRON, Cardano, 15+ chains  
**Characteristics:**
- Operated by Defuse Labs
- No current Substrate/Polkadot support
- Repository: `defuse-protocol/sdk-monorepo`

#### Option B: HOT Bridge
**Current Support**: EVM chains, TON, Stellar  
**Characteristics:**
- Operated by HOT Labs
- No Substrate support
- Requires HOT Labs coordination

#### Option C: Wormhole Bridge via MRL (Recommended)
**Route**: Hydration → XCM → Moonbeam → Wormhole → NEAR

**Current Status**: Hydration actively uses Moonbeam Routed Liquidity (MRL) with XCM + Wormhole

**NEAR Contracts:**
- Core Bridge: `contract.wormhole_crypto.near`
- Token Bridge: `contract.portalbridge.near`

**Characteristics:**
- 30+ chains, 19 guardian validators
- Multi-hop: XCM (Hydration → Moonbeam) + Wormhole (Moonbeam → NEAR)
- XCM channel to Moonbeam: **Operational**
- Existing Wormhole assets on Hydration: WETH (ID 20), USDC (ID 21)
- Wrapped tokens on NEAR: `{counter}.contract.portalbridge.near`
- VAA (Verified Action Approvals) mechanism
- Fee structure: XCM fee + Wormhole fee + NEAR gas
- Latency: ~6s (Hydration) + XCM transfer + Wormhole confirmation + NEAR finality
- **Proven technology stack** for Hydration team

### 5.2 Bridge Requirements

Regardless of which bridge is chosen, the following must be supported:

**Deposit Flow (Hydration → NEAR Intents):**
1. User deposits tokens on Hydration
2. Bridge locks/burns tokens on Hydration
3. Bridge mints wrapped representation in NEAR Intents Verifier
4. Token ID format: `nep141:hydration-<token-address>.bridge.near`

**Withdrawal Flow (NEAR Intents → Hydration):**
1. User initiates withdrawal intent
2. Bridge burns wrapped tokens in Verifier
3. Bridge unlocks/mints tokens on Hydration
4. User receives tokens on Hydration

### 5.3 Token Representation

**Native Hydration Token:**
```
Token ID in Verifier: nep141:hydration-native.<bridge-name>.near
Example: nep141:hydration-native.omft.near (if POA Bridge)
```

**Other Hydration Tokens:**
```
Format: nep141:hydration-<substrate-asset-id>.<bridge-name>.near
```

### 5.4 Required Repository Changes for Bridge

#### If using POA Bridge:

**Repository**: `defuse-protocol/sdk-monorepo`

**Files to modify:**

1. **`packages/internal-utils/src/poaBridge/constants/blockchains.ts`**
   ```typescript
   export const PoaBridgeNetworkReference = {
     // ... existing chains
     HYDRATION: "hydration:mainnet",
   } as const;
   ```

2. **`packages/intents-sdk/src/bridges/poa-bridge/poa-bridge-utils.ts`**
   - Add Hydration to `caip2Mapping`
   - Add token prefix mapping for Hydration
   ```typescript
   const tokenPrefixMapping = {
     // ... existing
     hydration: Chains.Hydration,
   };
   ```

3. **`packages/intents-sdk/src/lib/caip2.ts`**
   ```typescript
   export const Chains = {
     // ... existing chains
     Hydration: "polkadot:<genesis-hash>",
   } as const;
   ```

4. **`packages/intents-sdk/src/bridges/poa-bridge/poa-bridge.ts`**
   - Update `supports()` method to include Hydration token patterns
   - Implement withdrawal methods for Hydration
   - Add fee calculation for Hydration network

#### If using HOT Bridge:

**Files to modify:**

1. **`packages/intents-sdk/src/bridges/hot-bridge/hot-bridge-chains.ts`**
   ```typescript
   export const HotBridgeChains = [
     // ... existing
     Chains.Hydration,
   ];
   ```

2. **`packages/intents-sdk/src/bridges/hot-bridge/hot-bridge-utils.ts`**
   - Add native token mapping
   - Add network ID mapping

### 5.5 Required Actions

1. **Determine Bridge Strategy**
   - Evaluate POA Bridge extensibility for Substrate chains
   - Consult with POA Bridge team on feasibility
   - If needed, explore custom bridge development

2. **Bridge Development**
   - Implement bridge contracts/relayers
   - Set up monitoring and relayer infrastructure
   - Conduct security audits

3. **Testing**
   - Test deposit flow (Hydration → Verifier)
   - Test withdrawal flow (Verifier → Hydration)
   - Verify token wrapping/unwrapping
   - Test with various token amounts
   - Stress test for edge cases

---

## 5A. Hydration Asset System Details

### 5A.1 Asset Identification System

Hydration uses **numeric Asset IDs (u32)** with on-chain metadata stored in the Asset Registry pallet.

**Asset ID Range**: 0 to 1,001,219+ (dynamically growing)

### 5A.2 Asset Metadata Structure

```rust
pub struct AssetMetadata {
    name: Option<Vec<u8>>,           // Hex-encoded UTF-8 name
    assetType: AssetType,            // Token | Erc20 | XYK | StableSwap | External
    existentialDeposit: u128,        // Minimum balance required
    symbol: Option<Vec<u8>>,         // Hex-encoded symbol (e.g., 0x484458 = "HDX")
    decimals: Option<u8>,            // Token precision
    xcmRateLimit: Option<u128>,      // Maximum XCM transfer per block
    isSufficient: bool               // Asset can exist independently without requiring another token
}
```

### 5A.3 Asset Type Categories

1. **Token**: Native blockchain assets (HDX, USDT, USDC, KSM, etc.)
2. **Erc20**: Assets from Hydration's EVM environment (Frontier/EVM pallet): HOLLAR, HUSDe, HUSDT, GETH, aDOT
3. **XYK**: Liquidity pool share tokens (XYK AMM model)
4. **StableSwap**: Stable pool share tokens
5. **External**: Cross-chain assets received via XCM

### 5A.4 Priority Assets for Initial Integration

**Test Tokens** (specified by Hydration team):

| Asset ID | Symbol | Name | Decimals | Type | Use Case |
|----------|--------|------|----------|------|----------|
| 0 | HDX | Hydration | 12 | Token | Native token, fee payment |
| 10 | USDT | Tether | 6 | Token | Stablecoin |
| 21 | USDC | USDC (Moonbeam Wormhole) | 6 | Token | Stablecoin (via Wormhole) |
| 222 | HOLLAR | Hydrated Dollar | 18 | Erc20 | Native stablecoin (Hydration EVM) |

### 5A.5 Fee Payment in Multiple Assets

**Multi-Asset Fee Payment**: Hydration supports paying fees in **any Omnipool asset**

**Fee Conversion Mechanism**:
1. User selects fee payment token (e.g., USDT, USDC, HOLLAR)
2. Omnipool automatically converts fee token to HDX
3. Transaction fees paid in HDX internally
4. Conversion rate determined by Omnipool spot price

**Initial Test Configuration**:
- **Supported fee tokens**: USDT (ID 10), USDC (ID 21), HOLLAR (ID 222), HDX (ID 0)
- **Default fee token**: The first asset an account receives becomes its default fee payment asset

### 5A.6 NEAR Token Identifier Mapping

**Format**: `polkadot:{genesis-hash}:{assetId}`

**Genesis Hash**: `afdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d`

**Examples**:
```typescript
// HDX (Native token)
"polkadot:afdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d:0"

// USDT
"polkadot:afdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d:10"

// USDC (Wormhole)
"polkadot:afdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d:21"

// HOLLAR
"polkadot:afdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d:222"
```

### 5A.7 Querying Asset Metadata

**RPC Method**: `assetRegistry.assets(u32): Option<PalletAssetRegistryAssetDetails>`

**Example Call**:
```typescript
const metadata = await api.query.assetRegistry.assets(10); // USDT
// Returns Option<PalletAssetRegistryAssetDetails>
```

---

## 6. Verifier Smart Contract Integration

### 6.1 Repository

**Repository**: https://github.com/near/intents  
**Contract**: Deployed at `intents.near` on NEAR mainnet

### 6.2 Required Changes

The Verifier contract needs to support:

1. **Sr25519 Signature Verification** (PR #171)
   - Already in progress
   - Ensure integration with account abstraction

2. **Token Standards Support**
   - **NEP-141**: NEAR's fungible token standard (similar to ERC-20)
     * Bridged Hydration tokens will appear as NEP-141 tokens on NEAR
     * Format: `nep141:hydration-<assetId>.<bridge>.near`
     * Example: `nep141:hydration-0.omft.near` for HDX
   - **NEP-245**: Multi-token standard used internally by Verifier contract
     * Allows uniform handling of all token types (NEP-141, NEP-171, NEP-245)
     * No contract changes needed - existing implementation supports bridged tokens
     * Users interact with tokens using NEP-141/NEP-171 standards, Verifier manages them as NEP-245

3. **Account Abstraction**
   - Hydration addresses need to be mappable to implicit accounts
   - Sr25519 public keys → implicit account derivation

### 6.3 Account Derivation

**Implicit Account for Sr25519:**
```
Format: Sr25519 public key (32 bytes) → hex string
Example: 1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b
```

### 6.4 Required Actions

1. **Verify Account Abstraction Compatibility**
   - Ensure sr25519 public keys properly derive to implicit accounts
   - Test account key addition for Hydration users
   - Document the account creation flow

2. **Hydration Chain Configuration**
   - Whitelist Hydration chain ID in verifier
   - Configure bridge mappings

---

## 7. SDK & Tooling Integration

### 7.1 Intents SDK

**Repository**: `defuse-protocol/sdk-monorepo/packages/intents-sdk`

#### 7.1.1 Files to Modify

1. **`src/lib/caip2.ts`**
   ```typescript
   export const Chains = {
     // ... existing
     Hydration: "polkadot:<genesis-hash>",
   } as const;
   ```

2. **`src/lib/configure-rpc-config.ts`**
   - Add Hydration RPC configuration
   - Create `configureHydrationRpcUrls()` if special handling needed

3. **`src/constants/public-rpc-urls.ts`**
   ```typescript
   export const PUBLIC_HYDRATION_RPC_URLS = ["https://rpc.hydration.network"];
   ```

4. **`src/lib/route-config-factory.ts`**
   - Add route config creation for Hydration bridge
   ```typescript
   export function createHydrationBridgeRoute(chain?: Chain): HydrationBridgeRouteConfig {
     return { name: RouteEnum.HydrationBridge, chain };
   }
   ```

5. **Bridge Implementation**
   - Create `src/bridges/hydration-bridge/` directory
   - Implement `HydrationBridge` class extending `Bridge` interface
   - Implement `supports()`, `parseAssetId()`, `makeAssetInfo()`, `withdraw()` methods

#### 7.1.2 Type Definitions

**`src/shared-types.ts`**
```typescript
export interface HydrationBridgeRouteConfig {
  name: RouteEnum.HydrationBridge;
  chain?: Chain;
}
```

**`src/constants/route-enum.ts`**
```typescript
export enum RouteEnum {
  // ... existing
  HydrationBridge = "hydration-bridge",
}
```

#### 7.1.3 Asset ID Format

Define Hydration asset ID parsing:
```typescript
// Example for Hydration native token
"nep141:hydration-native.bridge.near"

// Example for other Hydration assets
"nep141:hydration-<asset-id>.bridge.near"
```

### 7.2 Cross-Chain Asset ID Package

**Repository**: `defuse-protocol/sdk-monorepo/packages/crosschain-assetid`

Add Hydration chain slug and namespace:

**`src/gen.ts`**
```typescript
const stringSchema = {
  examples: [
    // ... existing
    "1cs_v1:hydration:substrate-asset:native",
    "1cs_v1:hydration:substrate-asset:<asset-id>",
  ],
};
```

### 7.3 Required Actions

1. **Add RPC Configuration**
   - Provide default public RPC URLs (see Section 2.1.1)
   - Allow custom RPC configuration

2. **Update Type Definitions**
   - Add Hydration to chain types
   - Export all new types

3. **Documentation**
   - Add inline code documentation
   - Update SDK documentation with Hydration support

**Note**: Bridge-specific implementation depends on bridge selection (see Section 5). If Wormhole/MRL is chosen, SDK integration focuses on asset identifier mapping rather than custom bridge implementation.

---

## 8. 1-Click API Integration

### 8.1 Overview

The 1-Click API provides a simplified REST API for executing cross-chain swaps. It abstracts away the complexity of intent creation, solver coordination, and transaction execution.

**Base URL**: `https://1click.chaindefuser.com/`

### 8.2 Required Changes

#### 8.2.1 Token Registry

**Endpoint**: `GET /v0/tokens`

Add Hydration tokens to the supported tokens list:
```json
{
  "blockchain": "hydration",
  "symbol": "HDX",
  "assetId": "nep141:hydration-native.bridge.near",
  "contractAddress": "native",
  "price": "0.05",
  "decimals": 12
}
```

#### 8.2.2 Quote Endpoint

**Endpoint**: `POST /v0/quote`

Ensure Hydration tokens are supported in:
- `originAsset` parameter
- `destinationAsset` parameter
- Deposit address generation for Hydration network
- Fee calculation for Hydration

#### 8.2.3 Deposit Address Generation

The 1-Click API must be able to generate:
- Hydration-compatible deposit addresses (SS58 format)
- Unique addresses per quote for tracking

#### 8.2.4 Status Monitoring

**Endpoint**: `GET /v0/status`

Ensure status tracking works for Hydration transactions:
- Monitor Hydration blockchain for deposits
- Track bridge transaction status
- Update intent execution status

### 8.3 TypeScript SDK

**Repository**: `https://github.com/defuse-protocol/one-click-sdk-typescript`

#### Required Changes:

1. **Add Hydration to Chain Types**
   - Update enums/types to include Hydration

2. **Update OpenAPI Spec**
   - Ensure API spec includes Hydration in supported chains

3. **Examples**
   - Add example swap involving Hydration

### 8.4 Required Actions

1. **Backend Integration**
   - Add Hydration blockchain indexing
   - Implement Hydration RPC interaction
   - Configure deposit address generation

2. **Testing**
   - Test token listing endpoint
   - Test quote generation with Hydration assets
   - Test end-to-end swap flow

3. **SDK Updates**
   - Update TypeScript SDK
   - Update Go SDK
   - Update Rust SDK

---

## 9. Message Bus Integration

### 9.1 Overview

The Message Bus is an off-chain component that optimizes price discovery and matches user intents with market maker quotes. It consists of the Solver Relay service that connects distribution channels (wallets, apps) with market makers (solvers).

**Service**: Solver Relay v2  
**Endpoint**: `https://solver-relay-v2.chaindefuser.com/rpc` (JSON-RPC)  
**WebSocket**: `wss://solver-relay-v2.chaindefuser.com/ws`  
**Documentation**: See `market-makers/bus/solver-relay.md` in this repository

### 9.2 Solver Relay API Changes

The Solver Relay API uses Defuse Asset Identifiers in the format used throughout the system.

#### 9.2.1 Quote Request Support

**Method**: `get_quote_response`

**Hydration Asset Identifiers**:
```json
{
  "defuse_asset_identifier_in": "polkadot:afdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d:0",
  "defuse_asset_identifier_out": "near:mainnet:wrap.near",
  "exact_amount_in": "1000000000000"
}
```

**Example Quote Request** (Swap HDX to NEAR):
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "get_quote_response",
  "params": {
    "defuse_asset_identifier_in": "polkadot:afdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d:0",
    "defuse_asset_identifier_out": "near:mainnet:wrap.near",
    "exact_amount_in": "1000000000000",
    "min_deadline_ms": 60000
  }
}
```

**Example Quote Response**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": [
    {
      "quote_hash": "0x1234...",
      "defuse_asset_identifier_in": "polkadot:afdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d:0",
      "defuse_asset_identifier_out": "near:mainnet:wrap.near",
      "amount_in": "1000000000000",
      "amount_out": "500000000000000000000000",
      "expiration_time": 1735689600000,
      "solver_id": "solver1.near"
    }
  ]
}
```

#### 9.2.2 Intent Execution Support

**Method**: `execute_intent`

**Hydration Intent Example**:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "execute_intent",
  "params": {
    "quote_hash": "0x1234...",
    "recipient_id": "alice.near",
    "slippage_bps": 50
  }
}
```

### 9.3 Required Changes

#### 9.3.1 Solver Configuration

Solvers need to support Hydration asset pricing and liquidity:

1. **Add Hydration RPC Connection**
   ```typescript
   // Solver configuration
   {
     chainId: "polkadot:afdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d",
     rpcUrl: "wss://hydration-rpc.n.dwellir.com",
     supportedAssets: [
       { assetId: 0, symbol: "HDX", decimals: 12 },
       { assetId: 10, symbol: "USDT", decimals: 6 },
       { assetId: 21, symbol: "USDC", decimals: 6 },
       { assetId: 222, symbol: "HOLLAR", decimals: 18 }
     ]
   }
   ```

2. **Implement Price Discovery**
   - Use Hydration Trade Router SDK for optimal pricing across all liquidity sources
   - **Trade Router**: Automatically finds best route across Omnipool, Stableswap, XYK pools
   - **SDK**: `@galacticcouncil/sdk` ([GitHub](https://github.com/galacticcouncil/sdk/))
   - Calculate cross-chain swap rates (Hydration → Moonbeam → Wormhole → NEAR)
   - Account for multi-hop fees and slippage

3. **Implement Settlement Monitoring**
   - Monitor XCM transfers from Hydration to Moonbeam
   - Track Wormhole VAA confirmations
   - Verify NEAR token bridge mints

#### 9.3.2 Message Bus Updates

**No code changes required** - the Solver Relay is asset-agnostic and uses Defuse Asset Identifiers. However, operational configuration may need updates:

1. **Asset Registry**
   - Add Hydration assets to internal asset registry
   - Configure bridge mappings for Hydration tokens

2. **Monitoring**
   - Add Hydration RPC health checks
   - Monitor bridge transaction status

### 9.4 Solver Onboarding Documentation

Market makers integrating Hydration support need:

1. **Technical Guide**
   - Hydration Trade Router SDK documentation
   - XCM transfer implementation guide
   - Wormhole bridge integration steps
   - Settlement verification procedures

2. **Example Implementation**
   ```typescript
   // Example: Solver getting best price using Trade Router
   import { PolkadotApiPoolService, TradeRouter, PoolType } from '@galacticcouncil/sdk';
   
   const poolService = new PolkadotApiPoolService('wss://hydration-rpc.n.dwellir.com');
   const router = new TradeRouter(poolService, {
     includeOnly: [PoolType.Omni, PoolType.Stable, PoolType.XYK]
   });
   
   // Get best sell route for HDX -> USDT
   const assetIn = '0';   // HDX
   const assetOut = '10'; // USDT
   const amountIn = '1000000000000'; // 1 HDX (12 decimals)
   
   const bestRoute = await router.getBestSell(assetIn, assetOut, amountIn);
   console.log('Best route:', bestRoute.swaps);
   console.log('Amount out:', bestRoute.amountOut);
   ```
   
   See [SDK examples](https://github.com/galacticcouncil/sdk/blob/main/packages/sdk/test/script/examples/router/getBestSell.ts) for more details.

3. **Liquidity Requirements**
   - Minimum liquidity thresholds per asset
   - Expected quote response times
   - Settlement time SLAs

### 9.5 Required Actions

1. **Message Bus Configuration**
   - Add Hydration chain to asset registry
   - Configure bridge route mappings
   - Update monitoring dashboards

2. **Solver Coordination**
   - Share Hydration integration documentation with existing solvers
   - Provide test environment access
   - Coordinate liquidity bootstrap for initial launch

3. **Testing**
   - Test quote requests for all Hydration test tokens
   - Validate intent execution flow
   - Verify settlement monitoring

---

## 10. Testing & Validation

### 10.1 Unit Tests

**Repositories to Add Tests:**

1. **`near/intents` (Verifier Contract)**
   - Test sr25519 signature verification
   - Test account derivation from sr25519 public keys
   - Test Hydration token deposits/withdrawals

2. **`defuse-protocol/sdk-monorepo/packages/intents-sdk`**
   - Test Hydration bridge `supports()` method
   - Test asset ID parsing for Hydration tokens
   - Test withdrawal intent creation
   - Test fee calculation

3. **`defuse-protocol/sdk-monorepo/packages/crosschain-assetid`**
   - Test Hydration asset ID parsing
   - Test asset ID stringification

### 10.2 Integration Tests

1. **End-to-End Swap Flow**
   ```
   Test: NEAR → Hydration HDX
   - User deposits NEAR to Verifier
   - Create intent to swap NEAR for HDX
   - Market maker fulfills intent
   - HDX bridged to Hydration
   - User receives HDX on Hydration address
   ```

2. **End-to-End Swap Flow (Reverse)**
   ```
   Test: Hydration HDX → NEAR
   - User deposits HDX via Hydration bridge
   - Create intent to swap HDX for NEAR
   - Market maker fulfills intent
   - User withdraws NEAR
   ```

3. **Multi-Hop Swap**
   ```
   Test: Hydration HDX → USDC (ETH) → ARB
   - Test complex multi-chain swaps involving Hydration
   ```

### 10.3 Wallet Testing

Test with actual Hydration-compatible wallets:
- **Talisman Wallet** (Substrate universal wallet)
- **Polkadot.js Extension**
- **SubWallet**
- **Nova Wallet** (mobile)

### 10.4 Mainnet Testing Checklist

Before full mainnet launch:
- [ ] Small test deposits (<$10 value)
- [ ] Small test withdrawals
- [ ] Verify correct token amounts after bridge
- [ ] Verify fees are calculated correctly
- [ ] Verify transaction finality on both chains
- [ ] Monitor for any failed transactions
- [ ] Verify refund mechanism works

---

## 11. Documentation Updates

### 11.1 Files to Update

**Repository**: `https://github.com/defuse-protocol/docs`

#### 1. **`chain-address-support.md`**

Add Hydration to the chain support table:

```markdown
| Chain        | Address Types                           | Support Status | Example Address                                    |
|--------------|-----------------------------------------|----------------|----------------------------------------------------|
| **Hydration**| - SS58 (Substrate)                      | ✅ Supported    | `7L53bUTBopuwFt3mKUfmkzgGLayYa1Yvn1hAg9v5UMrQzTfh` |
```

Add to signing standards table:

```markdown
| Signing Standard | Supported Wallets / Apps                                  | Implementation Status |
|------------------|-----------------------------------------------------------|-----------------------|
| **Sr25519**      | Talisman, Polkadot.js, SubWallet, Nova Wallet             | ✅ Implemented         |
```

#### 2. **`README.md`**

Update supported chains mention to include Hydration.

#### 3. **`SUMMARY.md`**

No changes needed unless adding dedicated Hydration documentation page.

#### 4. **Create `hydration-integration.md`** (Optional)

Dedicated page explaining:
- How to use Hydration with NEAR Intents
- Supported tokens
- Wallet connection guide
- Example swap flows

### 11.2 SDK Documentation

**Repository**: `defuse-protocol/sdk-monorepo/packages/intents-sdk`

#### **`README.md`**

Add Hydration examples:
```typescript
import { Chains, IntentsSDK } from '@defuse-protocol/intents-sdk';

const sdk = new IntentsSDK({
  rpc: {
    [Chains.Hydration]: ['https://rpc.hydration.network'],
  }
});

// Withdraw HDX to Hydration
await sdk.withdraw({
  assetId: 'nep141:hydration-native.bridge.near',
  amount: '1000000000000', // 1 HDX (12 decimals)
  recipientAddress: '7L53bUTBopuwFt3mKUfmkzgGLayYa1Yvn1hAg9v5UMrQzTfh',
  routeConfig: createHydrationBridgeRoute(),
});
```

### 11.3 Required Actions

1. **Update Chain Support Documentation**
   - Add Hydration to all relevant tables
   - Include address format examples
   - Document signature requirements

2. **Create Integration Guide**
   - Step-by-step guide for developers
   - Example code snippets
   - Common pitfalls and solutions

3. **Update API Documentation**
   - 1-Click API docs should mention Hydration support
   - Update OpenAPI specifications

---

## 12. Deployment Checklist

### 12.1 Pre-Deployment

- [ ] **Signature Support**
  - [ ] PR #171 reviewed and merged
  - [ ] Sr25519 verification tested with real wallets
  - [ ] Account derivation validated

- [ ] **Bridge Selection & Implementation**
  - [ ] Bridge strategy decided (POA/HOT/Custom)
  - [ ] Bridge contracts deployed (if new)
  - [ ] Relayer infrastructure operational
  - [ ] Security audit completed (if custom bridge)

- [ ] **Smart Contract**
  - [ ] Verifier contract supports sr25519 (via PR #171)
  - [ ] Token standards compatible
  - [ ] Account abstraction tested

- [ ] **SDK Integration**
  - [ ] Hydration chain added to CAIP-2 constants
  - [ ] Bridge implementation complete
  - [ ] RPC configuration added
  - [ ] Unit tests passing
  - [ ] Integration tests passing

- [ ] **1-Click API**
  - [ ] Token registry updated
  - [ ] Quote endpoint supports Hydration
  - [ ] Deposit address generation working
  - [ ] Status monitoring operational

- [ ] **Message Bus**
  - [ ] Solvers configured
  - [ ] Intent broadcasting tested
  - [ ] Settlement monitoring operational

### 12.2 Deployment Steps

1. **Initial Deployment**
   - Deploy bridge infrastructure
   - Run comprehensive test suite
   - Fix any issues

2. **Limited Mainnet Beta**
   - Deploy to mainnet with limits
   - Whitelist select users/testers
   - Monitor closely for issues
   - Gradually increase limits

3. **Full Mainnet Launch**
   - Remove limits
   - Announce public availability
   - Monitor performance and security

### 12.3 Post-Deployment

- [ ] **Monitoring**
  - [ ] Set up alerts for bridge failures
  - [ ] Monitor transaction success rates
  - [ ] Track gas/fee costs
  - [ ] Monitor liquidity depth

- [ ] **Documentation**
  - [ ] Publish integration guides
  - [ ] Update all relevant docs
  - [ ] Create video tutorials (optional)

- [ ] **Community**
  - [ ] Announce Hydration support
  - [ ] Engage with Hydration community
  - [ ] Gather feedback
  - [ ] Onboard market makers

---

## 13. Open Questions

### 13.1 Bridge Strategy

**Recommended Approach**: Moonbeam Routed Liquidity (MRL) via Wormhole

**Rationale**: 
- Hydration actively uses MRL with XCM + Wormhole
- XCM channel to Moonbeam is operational
- Existing Wormhole assets on Hydration: WETH (ID 20), USDC (ID 21)
- Leverages proven infrastructure already deployed

**Bridge Path**:
```
Hydration → XCM Message → Moonbeam → Wormhole Contracts → NEAR
         (Asset Transfer)  (Parachain) (Token Bridge)    (intents.near)
```

**Alternative Options**:
1. **POA Bridge**: Extend for Substrate support (8-12 weeks development)
2. **HOT Bridge**: Coordinate with HOT Labs for Substrate support  
3. **Custom Bridge**: Direct integration (12+ weeks, audit required)

**Discussion Points**:
- Multi-hop complexity vs development timeline
- Fee structure across multiple hops
- Operational dependencies on Moonbeam
- Long-term scalability for other Polkadot parachains

---

### 13.2 Token Representation

**Asset System**: Numeric Asset IDs (u32) with on-chain metadata

**Initial Test Tokens**:
- HDX (ID 0): 12 decimals
- USDT (ID 10): 6 decimals
- USDC (ID 21): 6 decimals
- HOLLAR (ID 222): 18 decimals

**NEAR Token Mapping Format**: `polkadot:{genesis-hash}:{assetId}`

See Section 5A for complete asset system details.

---

### 13.3 Finality & Confirmation

**Block Time**: 6 seconds (average)

**Finality Mechanism**: Polkadot parachain finality
- Parachain blocks included in Relay Chain blocks
- Finalized when Relay Chain finalizes (GRANDPA finality gadget)
- **Typical finality**: 12-18 seconds (2-3 Relay Chain blocks)

**Recommended Confirmation**: 1 finalized block
- Use RPC: `chain_getFinalizedHead` to check finality status
- Sufficient for high-value transactions

**Reorg Risk**: Extremely low post-finalization (backed by Relay Chain security)

---

### 13.4 Fees

**Native Fee Token**: HDX (Asset ID 0)

**Multi-Asset Fee Payment**: Supported for any Omnipool asset
- **Initial test tokens**: USDT, USDC, HOLLAR, HDX
- Fee conversion via Omnipool exchange rates
- Automatic asset-to-HDX conversion for fee payment

**Fee Structure**:
- Base fee: Weight-based (Substrate standard)
- Fee multiplier: Dynamic based on block fullness
- Length fee: Per-byte transaction size

**Fee Estimation**: Use `payment_queryInfo` RPC
- Provide unsigned transaction hex
- Returns weight, base fee, and adjusted fee

---

### 13.5 RPC Infrastructure

**Recommended Primary Provider**: Dwellir
- Endpoint: `wss://hydration-rpc.n.dwellir.com`
- Archive node: Yes (full historical state)

**All Endpoints Are Archive Nodes**: Full historical data available

**Backup Endpoints**: 9 additional endpoints (see Section 2.1.1)

**Failover Strategy**:
1. Primary: Dwellir
2. Secondary: Round-robin across backup endpoints
3. Health check: `system_health` RPC call
4. Retry: 3 attempts per endpoint before failover

---

### 13.6 Wallet Support

Hydration supports both Substrate and EVM wallets due to its dual architecture:

**Substrate Wallets** (sr25519 signatures):
- Polkadot.js Extension (recommended primary test wallet)
- Talisman (multi-chain wallet)
- SubWallet (mobile + browser)
- Nova Wallet (mobile-focused)

**EVM Wallets** (ECDSA/secp256k1 signatures):
- MetaMask
- All Ethereum-compatible wallets

**Integration Approaches**:
- Substrate: Use `@polkadot/extension-dapp` for browser wallets
- EVM: Standard Ethereum wallet connection (Web3/ethers.js)
- Handle sr25519 signatures via `@polkadot/util-crypto`
- Test with Polkadot.js first, then expand

---

### 13.7 XCM Support

**Moonbeam Routed Liquidity (MRL)**: Currently operational on Hydration
- **Status**: Hydration actively uses MRL to access Wormhole on Moonbeam
- **XCM Channel to Moonbeam**: Established and operational
- **Wormhole Access**: Via Moonbeam's deployed contracts
- **Existing Wormhole Assets**: 
  - WETH (Asset ID 20) - Moonbeam Wormhole, 18 decimals
  - USDC (Asset ID 21) - Moonbeam Wormhole, 6 decimals

**Bridge Path to NEAR**:
```
Hydration → XCM Message → Moonbeam Parachain → Wormhole Contract → NEAR
                         (asset transfer)      (token bridge)     (intents.near)
```

**Fee Structure**:
1. XCM transfer fee (paid on Hydration)
2. Moonbeam execution fee (via XCM fee asset)
3. Wormhole bridge fee (paid on Moonbeam)
4. NEAR gas (paid on destination)

**Other Parachain Connections**: Not relevant for NEAR integration

---

## 14. Success Criteria

The Hydration integration will be considered successful when:

1. ✅ Users can deposit Hydration tokens via bridge
2. ✅ Users can withdraw to Hydration addresses
3. ✅ Cross-chain swaps involving Hydration work reliably
4. ✅ Hydration wallet signatures (sr25519) verify correctly
5. ✅ 1-Click API supports Hydration tokens
6. ✅ SDK examples demonstrate Hydration integration
7. ✅ Documentation is complete and accurate
8. ✅ Transaction success rate > 95%
9. ✅ Average swap completion time < 5 minutes
10. ✅ Security audits pass (if custom bridge)

---

## 15. Contact & Next Steps

### Recommended Actions

1. **Immediate**:
   - Review and merge PR #171 (sr25519 support)
   - Decide on bridge strategy (Wormhole/MRL recommended)
   - Review this specification document with NEAR Intents core team

2. **Short Term (1-2 weeks)**:
   - Begin bridge development/integration
   - Set up development environment
   - Start SDK modifications

3. **Medium Term (1-2 months)**:
   - Complete integration testing
   - Update documentation
   - Begin limited beta testing

4. **Long Term (3 months)**:
   - Full mainnet launch
   - Community announcement
   - Onboard market makers for Hydration liquidity

### Questions for Stakeholders

**For NEAR Intents Team**:
- What is the preferred bridge strategy?
- What are the security requirements for new chain integrations?
- What is the timeline priority for Hydration?

---

## Appendix A: Glossary

- **CAIP-2**: Chain Agnostic Improvement Proposal for blockchain identification
- **Intent**: A user's desired outcome (e.g., "swap X for Y")
- **Market Maker**: Entity providing liquidity to fulfill intents
- **Sr25519**: Schnorrkel signature scheme using Ristretto group
- **Substrate**: Blockchain framework used by Polkadot parachains
- **Verifier**: NEAR smart contract (`intents.near`) that settles transactions
- **XCM**: Cross-Consensus Messaging (Polkadot's cross-chain protocol)

---

## Appendix B: Key Discussion Points for Joint Review

### 1. Technical Specifications Provided

**Chain Identification**
- Genesis Hash: `0xafdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d`
- Proposed CAIP-2 Identifier: `polkadot:afdc188f45c71dacbaa0b62e16a91f726c7b8699a9748cdf715459de6b7f366d`
- Block Time: 6 seconds (average)
- Finality: Polkadot parachain finality (12-18 seconds typical)

**RPC Infrastructure**
- 10 public RPC endpoints available (all archive nodes)
- Recommended primary: Dwellir (`wss://hydration-rpc.n.dwellir.com`)
- 9 backup endpoints for redundancy

**Asset System**
- Asset IDs: Numeric (u32), current range 0-1,001,219+
- Metadata: On-chain via Asset Registry pallet
- Proposed initial test tokens: HDX (0), USDT (10), USDC (21), HOLLAR (222)
- Fee payment capability: Multi-asset support via Omnipool

### 2. Bridge Options for Evaluation

**Recommended: Wormhole via MRL**
- Leverages existing Hydration infrastructure (MRL operational)
- XCM channel to Moonbeam already active
- Existing Wormhole assets on Hydration: WETH (ID 20), USDC (ID 21)
- Trade-offs: Multi-hop complexity, multiple fee layers, dependency on Moonbeam

**Alternatives for Consideration**
- POA Bridge extension (8-12 weeks, requires Substrate support development)
- HOT Bridge coordination (requires HOT Labs engagement)
- Custom bridge (12+ weeks, full audit, reusable for other parachains)

**Questions for NEAR Intents Team**:
- Preference on bridge approach given timeline and complexity trade-offs?
- Experience with multi-hop bridge architectures?
- Willingness to extend POA/HOT bridges vs leverage Wormhole?

### 3. Implementation Requirements

**Sr25519 Signature Support**
- PR #171 in near/intents repository (in progress)
- Required libraries: schnorrkel (Rust), @polkadot/util-crypto (TypeScript)
- Question: Timeline for PR #171 completion?

**Verifier Contract Updates**
- Sr25519 signature verification integration needed
- Hydration chain ID whitelisting approach
- Account abstraction for sr25519 public keys
- Question: Any specific constraints or preferences for implementation?

**SDK Integration**
- Add Hydration to CAIP-2 chain registry
- Implement bridge module (structure depends on bridge choice)
- RPC configuration and failover logic
- Token identifier mapping strategy
- Question: Preferred SDK architecture for new chains?

### 4. Testing & Validation Plan

**Proposed Approach**
- Multi-asset fee payment testing
- Bridge deposit/withdrawal flow validation
- Wallet integration testing (Polkadot.js primary)
- Question: NEAR Intents standard testing requirements for new chains?

### 5. Open Questions for Joint Discussion

1. **Bridge Strategy**: Which bridge approach aligns best with NEAR Intents architecture and roadmap?
2. **Timeline**: What are realistic milestones given sr25519 PR status and bridge choice?
3. **Token Scope**: Start with 4 test tokens (HDX, USDT, USDC, HOLLAR) or broader asset support initially?
4. **Fee Handling**: How should multi-asset fees integrate with NEAR Intents fee structure?
5. **Maintenance**: Operational responsibilities for RPC infrastructure and bridge monitoring?
6. **Documentation**: What documentation updates needed for Hydration support?

---

## Appendix C: Useful Links

- **NEAR Intents Docs**: https://docs.near-intents.org
- **Verifier Contract Repo**: https://github.com/near/intents
- **SDK Monorepo**: https://github.com/defuse-protocol/sdk-monorepo
- **1-Click API**: https://docs.near-intents.org/near-intents/integration/distribution-channels/1click-api
- **Sr25519 PR**: https://github.com/near/intents/pull/171
- **Hydration Website**: https://hydration.net/
- **Polkadot Wiki**: https://wiki.polkadot.network/
- **Wormhole Docs**: https://docs.wormhole.com/
- **Polkadot.js Apps**: https://polkadot.js.org/apps/

---
