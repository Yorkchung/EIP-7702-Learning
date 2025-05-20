# EIP-7702 Modular Architecture with PREP

## 1. What is PREP (Programmable Runtime EOA Proxy)

**PREP** is a proxy-based architecture proposed by Biconomy to enhance EIP-7702 with modularity and composability. Instead of directly delegating to a single contract, the EOA delegates to a **proxy contract**, which then routes execution to a **logic module** based on the input.

> **PREP = EOA temporarily delegates to a dispatcher proxy → proxy invokes logic module.**

### Key Characteristics:
- Runtime delegation via EIP-7702 `SetCode`  
- Stateless and reusable modules  
- Modular design allows feature plug-ins (e.g., session keys, spending limits)

This enables **runtime composability** without deploying a full smart contract wallet, preserving the EOA address.

---

## 2. Stateless Delegation and Modular Logic

PREP emphasizes **stateless delegation**, meaning:

> Logic contracts do not maintain state. All behavior is driven by calldata, not persistent storage.

### Benefits of Stateless Modules:
- **Security**: Less prone to reentrancy and state corruption  
- **Reusable**: Modules can be shared across EOAs  
- **Composable**: Multiple modules can be chained in one transaction  

### Common Module Examples:
- `SessionKeyModule`: Temporary key-based authorization  
- `SpendLimitModule`: Enforces daily or per-tx limits  
- `WhitelistTransferModule`: Restricts interaction to approved contracts

PREP serves as the glue that routes to one or more of these modules at runtime.

---

## 3. Why Modular Smart Accounts Matter

Modular smart accounts provide a **scalable**, **user-friendly**, and **secure** path for account abstraction:

### Key Advantages:
1. **Flexibility & Customization**  
   Users and apps can mix-and-match functionality without over-deploying heavy logic.

2. **Lower Deployment Cost**  
   Shared modules reduce on-chain storage and gas usage. No need for a full contract wallet.

3. **Ecosystem Interoperability**  
   Wallets or SDKs (e.g., ZyFi, Stackup, Biconomy) can maintain module registries for public use.

4. **Security & Auditability**  
   Individual modules are small and focused, making audits and testing easier.

---


It represents a middle ground between the heavy infra of ERC-4337 and the minimalism of EOAs, enabling powerful use cases with minimal overhead.

---

## References:
1. [PREP Deep Dive – Biconomy](https://blog.biconomy.io/prep-deep-dive/)
2. [EIP-7702: A Comprehensive Guide for Apps – Biconomy](https://blog.biconomy.io/a-comprehensive-eip-7702-guide-for-apps/)