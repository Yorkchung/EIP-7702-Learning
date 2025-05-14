# EIP-7702 Explained

## 1. What is EIP-7702

EIP-7702 is an Ethereum Improvement Proposal designed to allow **Externally Owned Accounts (EOAs)** to temporarily gain smart contract-like capabilities within a single transaction. After the transaction, the EOA returns to its original state.

### Key Concepts:
- **EOA remains an EOA**: Still controlled by a private key, never permanently becomes a contract wallet.
- **Temporary empowerment**: EOAs can execute custom logic for a specific transaction.
- **Minimized attack surface**: Since the code only exists during execution, security risks are contained.

### Example:
Users can batch transactions, enable social recovery, or use gas sponsorship features without migrating to smart contract wallets, using a one-time "SetCode" transaction.

---

## 2. How it works?

EIP-7702 introduces a new transaction type called **SetCode Transaction**.

### Process Flow

1. **Authorization signing**
   - The EOA owner signs an authorization message to delegate execution to a specific smart contract.
   - The message includes:
     - `chainId`: Limits where the authorization is valid.
     - `address`: The contract address to delegate to.
     - `nonce`: Prevents replay attacks.
   - This ensures the delegation is explicit, scoped, and only valid for this transaction.

2. **Submit SetCode transaction (type: 0x04)**
   - A special transaction is submitted with `TransactionType = 0x04`.
   - The transaction includes:
     - Standard fields (`to`, `value`, `gasLimit`, `data`, etc.).
     - `authorization_list`: Contains the signed delegation proofs.
   - Upon execution, the EOA’s code is temporarily set to:
     ```plaintext
     code = 0xef0100 || address
     ```
     - `0xef0100`: A 3-byte prefix indicating delegation logic.
     - `address`: The contract address from the authorization.
   - For this transaction, all calls to the EOA will delegate to the specified contract logic.

3. **Execute logic**
   - During this transaction:
     - Calls to the EOA execute the delegated contract logic.
     - Enables features like:
       - Custom signature validation.
       - Batch token transfers.
       - Sponsored gas payments.
       - Programmable transaction hooks.
   - The EOA behaves like a smart contract **only during this transaction**.

4. **Revert to clean EOA**
   - After the transaction completes:
     - The temporary code is automatically cleared.
     - The EOA reverts to its original, code-free state.
   - Ensures no persistent changes, maintaining compatibility and security.

---

## 3. What is Set Code?

**Set Code** refers to the transaction-level mechanism of EIP-7702, which temporarily adds executable code to an EOA.

### Characteristics:
- **No contract deployment**: This is not about deploying a contract, just attaching code temporarily.
- **One-time usage**: The code only exists during this transaction.
- **Minimal state impact**: Once the transaction ends, everything reverts.

### Example:
Alice wants to batch approve tokens, transfer assets, and use a gas sponsor — all in one transaction.  
With EIP-7702, she can send a SetCode transaction, execute this complex logic, and her wallet remains a plain EOA afterward.

---

## 4. Difference between EIP-4337 and EIP-3074

### Compared to EIP-4337:
- ERC-4337 is a comprehensive account abstraction framework using an EntryPoint contract to manage smart wallets.
- Requires migrating assets to smart contract wallets.
- EIP-7702 enhances EOAs directly, no asset migration needed.
- 4337 is like building a **full-featured smart vehicle**, while 7702 adds a **temporary rocket booster to your existing bike**.

### Compared to EIP-3074:
- EIP-3074 proposes new EVM opcodes (AUTH / AUTHCALL) to delegate control to an Invoker contract.
- It involves persistent delegation and protocol-level changes.
- EIP-7702 avoids persistent delegation risks by limiting code effects to a single transaction.
- 3074 is like **handing over your car keys to someone else**, 7702 is like **activating a one-time turbo for yourself**.

---

## References:
1. [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702)
2. [EIP-4337](https://eips.ethereum.org/EIPS/eip-4337)
3. [EIP-3074](https://eips.ethereum.org/EIPS/eip-3074)
4. https://www.youtube.com/watch?v=uZTeYfYM6fM
5. https://eip7702.io/
6. https://www.nethermind.io/blog/eip-7702-attack-surfaces-what-developers-should-know