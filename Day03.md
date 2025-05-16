# Follow EIP-7702 sample code

## 1. Prepare the environment
1. [**Foundry**](https://book.getfoundry.sh/):
- Install the Foundry development toolkit.
- Configure the `forge` and `anvil` commands in your environment variables.
2. [**Sample code**](https://github.com/quiknode-labs/qn-guide-examples/tree/main/ethereum/eip-7702):
- Clone the sample code from QuickNode
- Open the `eip-7702` directory
3. Install Required Dependencies:
    ```bash
    forge install foundry-rs/forge-std
    forge install OpenZeppelin/openzeppelin-contracts
    forge remappings > remappings.txt
    ```
    If you encounter git repository issues during installation, you can remove the lib directory and reinstall the dependencies.

## 2. Sample Code Review
### `BatchCallAndSponsor.sol`
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/utils/cryptography/MessageHashUtils.sol";

/**
 * @title BatchCallAndSponsor
 * @notice An educational contract that allows batch execution of calls with nonce and signature verification.
 *
 * When an EOA upgrades via EIP‑7702, it delegates to this implementation.
 * Off‑chain, the account signs a message authorizing a batch of calls. The message is the hash of:
 *    keccak256(abi.encodePacked(nonce, calls))
 * The signature must be generated with the EOA’s private key so that, once upgraded, the recovered signer equals the account’s own address (i.e. address(this)).
 *
 * This contract provides two ways to execute a batch:
 * 1. With a signature: Any sponsor can submit the batch if it carries a valid signature.
 * 2. Directly by the smart account: When the account itself (i.e. address(this)) calls the function, no signature is required.
 *
 * Replay protection is achieved by using a nonce that is included in the signed message.
 */
contract BatchCallAndSponsor {
    using ECDSA for bytes32;

    /// @notice A nonce used for replay protection.｀
    uint256 public nonce;

    /// @notice Represents a single call within a batch.
    struct Call {
        address to;
        uint256 value;
        bytes data;
    }

    /// @notice Emitted for every individual call executed.
    event CallExecuted(address indexed sender, address indexed to, uint256 value, bytes data);
    /// @notice Emitted when a full batch is executed.
    event BatchExecuted(uint256 indexed nonce, Call[] calls);

    /**
     * @notice Executes a batch of calls using an off–chain signature.
     * @param calls An array of Call structs containing destination, ETH value, and calldata.
     * @param signature The ECDSA signature over the current nonce and the call data.
     *
     * The signature must be produced off–chain by signing:
     * The signing key should be the account’s key (which becomes the smart account’s own identity after upgrade).
     */
    function execute(Call[] calldata calls, bytes calldata signature) external payable {
        // Compute the digest that the account was expected to sign.
        bytes memory encodedCalls;
        for (uint256 i = 0; i < calls.length; i++) {
            encodedCalls = abi.encodePacked(encodedCalls, calls[i].to, calls[i].value, calls[i].data);
        }
        bytes32 digest = keccak256(abi.encodePacked(nonce, encodedCalls));
        
        bytes32 ethSignedMessageHash = MessageHashUtils.toEthSignedMessageHash(digest);

        // Recover the signer from the provided signature.
        address recovered = ECDSA.recover(ethSignedMessageHash, signature);
        require(recovered == address(this), "Invalid signature");

        _executeBatch(calls);
    }

    /**
     * @notice Executes a batch of calls directly.
     * @dev This function is intended for use when the smart account itself (i.e. address(this))
     * calls the contract. It checks that msg.sender is the contract itself.
     * @param calls An array of Call structs containing destination, ETH value, and calldata.
     */
    function execute(Call[] calldata calls) external payable {
        require(msg.sender == address(this), "Invalid authority");
        _executeBatch(calls);
    }

    /**
     * @dev Internal function that handles batch execution and nonce incrementation.
     * @param calls An array of Call structs.
     */
    function _executeBatch(Call[] calldata calls) internal {
        uint256 currentNonce = nonce;
        nonce++; // Increment nonce to protect against replay attacks

        for (uint256 i = 0; i < calls.length; i++) {
            _executeCall(calls[i]);
        }

        emit BatchExecuted(currentNonce, calls);
    }

    /**
     * @dev Internal function to execute a single call.
     * @param callItem The Call struct containing destination, value, and calldata.
     */
    function _executeCall(Call calldata callItem) internal {
        (bool success,) = callItem.to.call{value: callItem.value}(callItem.data);
        require(success, "Call reverted");
        emit CallExecuted(msg.sender, callItem.to, callItem.value, callItem.data);
    }

    // Allow the contract to receive ETH (e.g. from DEX swaps or other transfers).
    fallback() external payable {}
    receive() external payable {}
}
```

The overall design reflects the core philosophy of EIP-7702: lightweight, transaction-level account abstraction that eliminates the need for additional infrastructure like EntryPoint or Forwarders, while still enabling practical use cases such as batch operations and gas sponsorship.

In this implementation, the most distinctive EIP-7702 features include:
- Delegating logic execution directly to the EOA’s own address.
- Using msg.sender == address(this) to verify execution authority through self-calling behavior.
- Allowing sponsors to submit authorized transactions on behalf of the EOA without relying on external relayers.

### `BatchCallAndSponsor.s.sol`
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import {Vm} from "forge-std/Vm.sol";
import {BatchCallAndSponsor} from "../src/BatchCallAndSponsor.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {MessageHashUtils} from "@openzeppelin/contracts/utils/cryptography/MessageHashUtils.sol";

contract MockERC20 is ERC20 {
    constructor() ERC20("Mock Token", "MOCK") {}

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

contract BatchCallAndSponsorScript is Script {
    // Alice's address and private key (EOA with no initial contract code).
    address payable ALICE_ADDRESS = payable(0x70997970C51812dc3A010C7d01b50e0d17dc79C8);
    uint256 constant ALICE_PK = 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d;

    // Bob's address and private key (Bob will execute transactions on Alice's behalf).
    address constant BOB_ADDRESS = 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC;
    uint256 constant BOB_PK = 0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a;

    // The contract that Alice will delegate execution to.
    BatchCallAndSponsor public implementation;

    // ERC-20 token contract for minting test tokens.
    MockERC20 public token;

    function run() external {
        // Start broadcasting transactions with Alice's private key.
        vm.startBroadcast(ALICE_PK);

        // Deploy the delegation contract (Alice will delegate calls to this contract).
        implementation = new BatchCallAndSponsor();

        // Deploy an ERC-20 token contract where Alice is the minter.
        token = new MockERC20();

        // // Fund accounts
        token.mint(ALICE_ADDRESS, 1000e18);

        vm.stopBroadcast();

        // Perform direct execution
        performDirectExecution();

        // Perform sponsored execution
        performSponsoredExecution();
    }

    function performDirectExecution() internal {
        BatchCallAndSponsor.Call[] memory calls = new BatchCallAndSponsor.Call[](2);

        // ETH transfer
        calls[0] = BatchCallAndSponsor.Call({to: BOB_ADDRESS, value: 1 ether, data: ""});

        // Token transfer
        calls[1] = BatchCallAndSponsor.Call({
            to: address(token),
            value: 0,
            data: abi.encodeCall(ERC20.transfer, (BOB_ADDRESS, 100e18))
        });

        vm.signAndAttachDelegation(address(implementation), ALICE_PK);
        vm.startPrank(ALICE_ADDRESS);
        BatchCallAndSponsor(ALICE_ADDRESS).execute(calls);
        vm.stopPrank();

        console.log("Bob's balance after direct execution:", BOB_ADDRESS.balance);
        console.log("Bob's token balance after direct execution:", token.balanceOf(BOB_ADDRESS));
    }

    function performSponsoredExecution() internal {
        console.log("Sending 1 ETH from Alice to a random address, the transaction is sponsored by Bob");

        BatchCallAndSponsor.Call[] memory calls = new BatchCallAndSponsor.Call[](1);
        address recipient = makeAddr("recipient");
        calls[0] = BatchCallAndSponsor.Call({to: recipient, value: 1 ether, data: ""});

        // Alice signs a delegation allowing `implementation` to execute transactions on her behalf.
        Vm.SignedDelegation memory signedDelegation = vm.signDelegation(address(implementation), ALICE_PK);

        // Bob attaches the signed delegation from Alice and broadcasts it.
        vm.startBroadcast(BOB_PK);
        vm.attachDelegation(signedDelegation);

        // Verify that Alice's account now temporarily behaves as a smart contract.
        bytes memory code = address(ALICE_ADDRESS).code;
        require(code.length > 0, "no code written to Alice");
        // console.log("Code on Alice's account:", vm.toString(code));

        bytes memory encodedCalls = "";
        for (uint256 i = 0; i < calls.length; i++) {
            encodedCalls = abi.encodePacked(encodedCalls, calls[i].to, calls[i].value, calls[i].data);
        }
        bytes32 digest = keccak256(abi.encodePacked(BatchCallAndSponsor(ALICE_ADDRESS).nonce(), encodedCalls));
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(ALICE_PK, MessageHashUtils.toEthSignedMessageHash(digest));
        bytes memory signature = abi.encodePacked(r, s, v);

        // As Bob, execute the transaction via Alice's temporarily assigned contract.
        BatchCallAndSponsor(ALICE_ADDRESS).execute(calls, signature);

        vm.stopBroadcast();

        console.log("Recipient balance after sponsored execution:", recipient.balance);
    }
}
```
### Scenario 1: Alice Executes Batch Transfer (ETH + Token)

#### Without EIP-7702:
- Alice needs to send **two separate transactions**:
    1. The first transaction transfers 1 ETH to Bob.
    2. The second transaction calls the token contract’s `transfer` function to send 100 tokens to Bob.

- If Alice wants a single batch transaction:
    - Alice has to rely on a **BatchCall contract**.
    - For example: Gnosis Multisend, Multicall contract, etc.
    - Alice would send a transaction to the BatchCall contract, which then executes multiple calls on her behalf.
    - This means she still depends on an **external contract** to handle batch logic, as an EOA alone cannot perform such operations.

#### With EIP-7702:
- Alice signs a message containing (nonce + calls hash).
- She sends a single EIP-7702 transaction.
- This transaction temporarily gives Alice's account "smart contract logic" to execute the authorized batch calls.
- No need to deploy a BatchCall contract or any additional infrastructure.
- A single EIP-7702 transaction completes the batch call natively.

### Scenario 2: Bob Sponsors a Transaction for Alice

#### Without EIP-7702:
- Alice signs a MetaTx message specifying:
  - recipient, value, data, nonce, etc.
- Bob submits this message to a **Forwarder contract**.
- The Forwarder verifies Alice’s signature.
- The Forwarder then executes the actual transaction logic on Alice’s behalf.
- This requires a Forwarder contract, and every project typically implements its own MetaTx framework.
- Bob pays the gas fee, but the overall process is relatively complex.

#### With EIP-7702:
- Alice simply signs a lightweight authorization message (nonce + calls hash).
- Bob directly submits a single EIP-7702 transaction.
- The EVM treats this transaction as if Alice herself is executing the logic.
- No need for a Forwarder contract or a complex MetaTx infrastructure.
- Bob submits the transaction, effectively sponsoring Alice’s gas fees, while the logic remains fully authorized by Alice.

## 3. Local Testing and Deployment
1. Run a Local Network
    ```bash
    anvil --hardfork prague
    ```
2. Install Dependencies
    ```bash
    forge install
    ```
3. Build the Contract
    ```bash
    forge build
    ```
4. Run the Test Cases
    ```bash
    forge test -vvv
    ```
5. Run the Script

    Deploy the BatchCallAndSponsor contract using the provided deployment script.
    ```bash
    forge script ./script/BatchCallAndSponsor.s.sol --tc BatchCallAndSponsorScript --broadcast --rpc-url 127.0.0.1:8545
    ```

## 4. Thoughts
This EIP-7702 example uses a simple and intuitive design to clearly demonstrate how an EOA can temporarily gain smart contract capabilities within a single transaction.

By leveraging mechanisms such as `address(this)` signature verification, nonce-based replay protection, and batch call execution, it implements a lightweight smart account logic that requires no additional infrastructure. This makes it especially suitable for use cases like gas sponsorship and batch operations. Although the example is simple, it effectively conveys the core concepts of EIP-7702 and serves not only as a learning reference but also as a solid foundation for practical applications.

For testing EIP-7702 features, Foundry provides three cheatcodes that simulate the delegation execution process:

- **`signDelegation`**: Generates a signature from an EOA (Externally Owned Account) that authorizes a specific contract to execute logic on its behalf.
- **`attachDelegation`**: Attaches the generated signature to the next transaction, allowing the EVM to recognize it as a valid delegated execution.
- **`signAndAttachDelegation`**: Combines the two previous steps, signing and attaching the delegation in a single command to simplify the testing workflow.

## Reference:
1. https://www.quicknode.com/guides/ethereum-development/smart-contracts/eip-7702-smart-accounts
2. https://github.com/quiknode-labs/qn-guide-examples/tree/main/ethereum/eip-7702
3. https://book.getfoundry.sh/cheatcodes/sign-delegation