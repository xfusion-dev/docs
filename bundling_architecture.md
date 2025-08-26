# XFusion Token Bundling Architecture

## Overview

XFusion implements a sophisticated Chain-Key token bundling system that allows users to mint and burn index tokens representing diversified portfolios of underlying assets. The system uses a resolver network of semi-centralized liquidity providers to ensure optimal pricing and liquidity.

## Core Components

- **User Interface**: Frontend for bundle creation and trading
- **Bundler Canister**: Main smart contract managing bundle tokens
- **Resolver Network**: Semi-centralized network of automated software for liquidity providers
- **Router**: Intelligent routing system for optimal LP selection
- **Liquidity Providers (LPs)**: Entities providing liquidity for individual ck-Assets
- **Intermediary Wallets**: LP-controlled wallets with pre-approved allowances

---

## Process 1: Minting Bundle Tokens (Buying)

### Scenario: User wants to mint 1 bundle token containing 5 underlying assets

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant Router
    participant ResolverNetwork as Resolver Network
    participant LP1 as Liquidity Provider 1
    participant LP2 as Liquidity Provider 2
    participant LPn as Liquidity Provider N
    participant Wallet1 as LP1 Intermediary Wallet
    participant Wallet2 as LP2 Intermediary Wallet
    participant BundlerCanister as Bundler Canister

    User->>Frontend: Request to mint bundle token (5 assets)
    Frontend->>Router: Submit mint request with asset allocation
    
    Note over Router,ResolverNetwork: Quote Discovery Phase
    Router->>ResolverNetwork: Request quotes for 5 individual ck-Assets
    ResolverNetwork->>LP1: Get quote for Asset A & B
    ResolverNetwork->>LP2: Get quote for Asset C & D  
    ResolverNetwork->>LPn: Get quote for Asset E
    
    LP1-->>ResolverNetwork: Quote: Asset A ($100), Asset B ($200)
    LP2-->>ResolverNetwork: Quote: Asset C ($150), Asset D ($50)
    LPn-->>ResolverNetwork: Quote: Asset E ($300)
    ResolverNetwork-->>Router: Aggregated quotes
    
    Note over Router: Optimization Phase
    Router->>Router: Calculate best LP combination
    Router->>User: Present total price & LP breakdown
    User->>Router: Approve transaction
    
    Note over Router,BundlerCanister: Asset Acquisition Phase
    Router->>LP1: Request Asset A & B (order_id: 001)
    Router->>LP2: Request Asset C & D (order_id: 002)
    Router->>LPn: Request Asset E (order_id: 003)
    
    LP1->>Wallet1: Top up with Asset A & B
    LP1->>Router: confirmDeposit(order_id: 001)
    Router->>Router: Mark Asset A & B as 'bought'
    
    LP2->>Wallet2: Top up with Asset C & D
    LP2->>Router: confirmDeposit(order_id: 002)
    Router->>Router: Mark Asset C & D as 'bought'
    
    LPn->>Router: confirmDeposit(order_id: 003)
    Router->>Router: Mark Asset E as 'bought'
    
    Note over Router,BundlerCanister: Bundle Minting Phase
    Router->>BundlerCanister: Send all 5 acquired assets
    BundlerCanister->>BundlerCanister: Mint new bundle token
    BundlerCanister->>User: Transfer bundle token
```

### Detailed Steps:

1. **User Initiation**: User selects desired bundle composition (5 tokens with specific allocations)

2. **Quote Discovery**: 
   - Router queries resolver network for individual ck-Asset prices
   - Multiple LPs provide competitive quotes for different assets
   - Router receives aggregated pricing data

3. **Route Optimization**:
   - Router calculates optimal combination of LPs to minimize cost
   - Considers factors: price, liquidity depth, LP reliability
   - Presents final quote to user

4. **Asset Acquisition**:
   - Router distributes purchase orders to selected LPs
   - Each LP must have intermediary wallet with pre-approved allowances
   - LPs top up wallets and confirm deposits with unique order IDs
   - Router tracks completion status for each asset

5. **Bundle Minting**:
   - Once all assets are acquired, router transfers them to bundler canister
   - Bundler canister mints new bundle token representing the portfolio
   - Bundle token is transferred to user's account

---

## Process 2: Burning Bundle Tokens (Selling)

### Scenario: User wants to sell 1 bundle token and receive USDC

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant Router
    participant ResolverNetwork as Resolver Network
    participant DutchAuction as Dutch Auction Engine
    participant LP1 as Liquidity Provider 1
    participant LP2 as Liquidity Provider 2
    participant LPn as Liquidity Provider N
    participant BundlerCanister as Bundler Canister

    User->>Frontend: Request to sell bundle token
    Frontend->>Router: Submit burn request
    
    Note over Router,ResolverNetwork: Price Discovery Phase
    Router->>ResolverNetwork: Request USDC quotes for underlying assets
    ResolverNetwork->>LP1: Get USDC quote for Asset A & B
    ResolverNetwork->>LP2: Get USDC quote for Asset C & D
    ResolverNetwork->>LPn: Get USDC quote for Asset E
    
    LP1-->>ResolverNetwork: USDC quotes for A & B
    LP2-->>ResolverNetwork: USDC quotes for C & D
    LPn-->>ResolverNetwork: USDC quote for E
    ResolverNetwork-->>Router: Aggregated USDC receivable
    
    Router->>User: Present minimum receivable USDC amount
    User->>Router: Accept minimum price
    
    Note over Router,DutchAuction: Asset Liquidation Phase
    Router->>BundlerCanister: Burn bundle token & release assets
    BundlerCanister-->>Router: Transfer underlying 5 assets
    
    Router->>DutchAuction: Start Dutch auction for Asset A
    Note over DutchAuction: Stop price = (Asset A allocation %) × (Min USDC)
    DutchAuction->>LP1: Auction Asset A (decreasing price)
    LP1->>DutchAuction: Buy at acceptable price
    DutchAuction-->>Router: Asset A sold for X USDC
    
    Router->>DutchAuction: Start Dutch auction for Asset B
    DutchAuction->>LP2: Auction Asset B
    LP2->>DutchAuction: Buy at acceptable price
    DutchAuction-->>Router: Asset B sold for Y USDC
    
    Note over Router,DutchAuction: Continue for remaining assets...
    
    Router->>DutchAuction: Start Dutch auction for Asset E
    DutchAuction->>LPn: Auction Asset E
    LPn->>DutchAuction: Buy at acceptable price
    DutchAuction-->>Router: Asset E sold for Z USDC
    
    Router->>Router: Aggregate total USDC received
    Router->>User: Transfer total USDC amount
```

### Detailed Steps:

1. **Sell Request**: User initiates sale of bundle token for USDC

2. **Price Discovery**:
   - Router queries resolver network for current USDC value of each underlying asset
   - Multiple LPs provide competitive buy quotes
   - Router calculates minimum receivable USDC amount

3. **User Acceptance**: User reviews and accepts minimum receivable price

4. **Bundle Burning**:
   - Router instructs bundler canister to burn the bundle token
   - Bundler canister releases the 5 underlying assets to router

5. **Dutch Auction Liquidation**:
   - Router initiates Dutch auctions for each asset individually
   - Stop price for each auction = (Asset allocation %) × (Minimum USDC)
   - Auctions start high and decrease until LPs bid
   - Each asset is sold to highest bidder above stop price

6. **USDC Distribution**: Router aggregates all USDC received and transfers to user

---

## Key Architecture Benefits

### For Minting (Buying):
- **Competitive Pricing**: Multiple LPs compete for each asset
- **Optimal Routing**: Router selects best LP combination
- **Atomic Execution**: All assets must be acquired before minting
- **Transparency**: Clear order tracking and confirmation system

### For Burning (Selling):
- **Price Protection**: Minimum receivable price protects users
- **Market Efficiency**: Dutch auctions discover fair market prices
- **Proportional Liquidation**: Stop prices reflect asset allocations
- **Immediate Settlement**: USDC received upon completion

### For Liquidity Providers:
- **Flexible Participation**: LPs can provide liquidity for specific assets
- **Risk Management**: Intermediary wallets with controlled allowances
- **Competitive Environment**: Quote-based selection process
- **Clear Settlement**: Confirmed deposit system with order IDs

### For the Ecosystem:
- **Decentralized Liquidity**: No single point of failure
- **Efficient Price Discovery**: Market-driven pricing through competition
- **Scalable Architecture**: Can handle multiple simultaneous transactions
- **Transparent Operations**: All transactions recorded on-chain
