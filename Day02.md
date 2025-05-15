# Security Risk of EIP-7702

## 1. **Private Key Leakage Risk**
- **Essence**: Still a problem, of course. Although EIP-7702 allows EOAs to delegate contract logic, ultimate control of the account remains with the private key holder.
- **Risk**: Regardless of how restrictive the delegated code is, if the private key is leaked, an attacker can sign transactions and directly control account assets, bypassing the contract logic.
- **Mitigation**:
  - Strengthen private key management (e.g., MPC, hardware wallets).
  - Implement multi-signature or guardian mechanisms to distribute risk.

---

## 2. **Delegated Contract Vulnerabilities Affect All Delegators**
- **Essence**: Multiple accounts can delegate to the same contract, expanding the attack surface.
- **Risk**: If the delegated contract has vulnerabilities, all accounts pointing to it are exposed to the same risks (e.g., logic flaws, asset theft).
- **Mitigation**:
  - Only delegate to thoroughly audited contracts.
  - Support per-account contract clones (template clones) to reduce centralization risk.

---

## 3. **Storage Collision and Incompatibility Risk**
- **Essence**: EIP-7702 only changes the account's code field without modifying storage.
- **Risk**: Delegating to a new contract with an incompatible storage layout can lead to data corruption, asset loss, or abnormal behavior.
- **Mitigation**:
  - Follow ERC-7201 namespace standards to isolate storage variables.
  - Establish storage versioning and backward compatibility mechanisms.

---

## 4. **Cross-Chain Replay Attack**
- **Essence**: Setting `chain_id` to 0 during signing allows the transaction to be replayed across all EIP-7702-supported chains.
- **Risk**: Attackers can replay the same transaction on different chains where the delegated contract behaves differently, leading to unexpected asset loss.
- **Mitigation**:
  - Wallets should enforce specifying the correct `chain_id` when signing.
  - Users are advised not to set `chain_id` to 0.

---

## 5. **Fake Deposit Risk**
- **Essence**: Delegated EOAs gain contract-like behavior, which can be abused to fake deposit transactions.
- **Risk**: Centralized exchanges (CEX) that do not perform extra verification for contract deposits may be exploited by fake deposit attacks.
- **Mitigation**:
  - CEXes should trace actual asset movements, not just rely on Transfer events.
  - Enhance deposit validation mechanisms.

---

## 6. **Account Conversion Breaks Security Assumptions**
- **Essence**: EIP-7702 allows accounts to flexibly switch between EOAs and contract accounts.
- **Risk**: Traditional assumptions like `tx.origin` always being an EOA no longer hold, breaking security mechanisms (e.g., reentrancy protections) based on such logic.
- **Mitigation**:
  - Developers should avoid using `tx.origin` for security checks.
  - Adopt more secure permission verification methods (e.g., ERC-1271).

---

## 7. **No Constructor Initialization Risk**
- **Essence**: EIP-7702 updates code without executing constructor or initializer functions as done in contract deployment.
- **Risk**: If the contract is not properly initialized, it may lead to security vulnerabilities or asset mismanagement during use.
- **Mitigation**:
  - Use `ecrecover` to verify signatures and ensure initialization is triggered by the correct user.
  - Implement one-time initialization protection logic.

---

## 8. **Phishing and User Misguidance Risk**
- **Essence**: Users can delegate code themselves, but without understanding contract risks, they are prone to phishing sites or malicious contracts.
- **Risk**: Once an account delegates to a malicious contract, its assets are at high risk of being stolen.
- **Mitigation**:
  - Wallet UIs must clearly display target contract risk information.
  - Introduce automated risk analysis (e.g., open-source status, permission checks, access behavior warnings).

## References:
1. https://www.nethermind.io/blog/eip-7702-attack-surfaces-what-developers-should-know
2. https://slowmist.medium.com/in-depth-discussion-on-eip-7702-and-best-practices-968b6f57c0d5