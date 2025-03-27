# DarkPool

## Table of Contents

- [Overview](#overview)
- [Techstack](#techstack)
- [Modules](#modules)
- [Workflow](#workflow)
- [Circuit-related Procedure](#circuit-related-procedure)
- [Foundry Usage](#foundry-usage)

## Overview
`DarkPool` is an innovative decentralized exchange (DEX) that leverages zk-SNARKs technology to address two critical issues in decentralized finance (DeFi): `privacy` and `scalability`. As a leading DEX, DarkPool introduces the concept of a `dark pool`, a trading environment where transaction details remain undisclosed to the public until after the trade is executed. This approach offers unparalleled security and privacy protection, especially for users executing large transactions.

### Core Objectives
The primary goal of DarkPool is to provide users with a `secure`, `efficient`, and `privacy-protected` trading environment. Leveraging zk-SNARKs technology, DarkPool ensures that transaction details, including prices and volumes, remain private during the verification process.

### Benefits of DarkPool

1. Large-Scale Private Transaction
2. Resistance to Front-running and MEV
3. Optimized Trading Environment

### Key Features

- **Concealing Price Information**: In limit orders, only the required balance for the order is exposed, while the price remains concealed.
- **Preventing Market Reactions**: Users' trading intentions remain hidden, preventing market reactions that could otherwise affect prices, ensuring price stability.


## Techstack

### Smart Contract Development
- `Foundry`: A smart contract development toolchain
- `OpenZeppelin Contracts`: `ERC20`, `SafeERC20`, `Ownable`

### Zero-Knowledge Proof (zk-SNARKs)
- `Circom`: zkSnark circuit compiler
- `SnarkJS`: zkSNARK implementation in JavaScript & WASM

## Modules

### Circuit
The provided code implements a circuit that computes the Keccak hash of the input data and outputs it as two 128-bit values. The main purpose of the circuit is to prove that the user knows the inputs `a0e`, `a1m`, and `salt`, and can compute their Keccak hash.

```
- Function: Proves the plaintext of `H(a0e, a1m, salt)`
- Public Input: `a0e` (The amount of tokens being spent), `a1m` (The minimum acceptable amount of tokens to be received)
- Private Input: `salt` (A private input used to add randomness and prevent brute-force attacks)
- Output: `H(a0e, a1m, salt)`
```

1. `Num2Bits (Number to Bits Conversion):` The values a0e, a1m, and salt are converted into 256-bit binary numbers.
2. `Reversing Bit Order`: The `reverse[]` array is used to reverse the bit order of `a0e`, `a1m`, and `salt`, typically to match the bit order used by certain hash algorithms like Keccak.
3. `Byte-Level Bit Reordering`: The hash_input[] array swaps the bit order at the byte level, ensuring that the bit order within each byte is correct for the hash calculation.
4. `Keccak Hashing`: The Keccak module computes the Keccak hash of the 768-bit data, producing a 256-bit hash output.
5. `Splitting the Hash Value`: The hash value is split into two 128-bit parts as  `left[128]` and `right[128]`
6. `Bits2Num (Bits to Number Conversion)`: The Bits2Num(128) module converts the high 128 bits and low 128 bits back into numbers.
7. `Outputs`: The circuit outputs two 128-bit values, `out[0]` and `out[1]`, which are the Keccak hash of the inputs a0e, a1m, and salt.

### Agent Backend
1. Call the `swapForward` function on the contract to execute the swap.

### Delegate Contract
1. Stores the plaintext order information
2. Forwards orders to the pool
3. Validate inputs from the agent

This function is the most critical part of the contract, responsible for order forwarding and verification. It uses zk-SNARKs (Groth16 proof) to ensure the correctness and privacy of the transaction.

```Solidity
    function swapForward(
        uint256[2] calldata _proofA,
        uint256[2][2] calldata _proofB,
        uint256[2] calldata _proofC,
        address swapper,
        uint256 index,
        uint256 a0e,
        uint256 a1m,
        uint256 _gasFee,
        OrderType _type
    ) external onlyAgent {
        ...

        require(pendingOrder.t.exchangeRate <= (a0e * 1 ether) / (a1m), "bad exchangeRate");
        require(!pendingOrder.t.OrderIsExecuted, "already executed");
        require(block.timestamp <= pendingOrder.t.deadline, "order expired");

        uint256[4] memory signals;
        signals[0] = uint256(uint128(pendingOrder.HOsF));
        signals[1] = uint256(uint128(pendingOrder.HOsE));
        signals[2] = a0e;
        signals[3] = a1m;

        require(verifier.verifyProof(_proofA, _proofB, _proofC, signals), "Proof is not valid");

        ...

        orderbook[swapper][index].t.OrderIsExecuted = true;
        takeFeeInternal(pendingOrder.t.swapper, _gasFee);
        emit OrderExecuted(
            pendingOrder.t.swapper,
            index,
            pendingOrder.t.tokenIn,
            pendingOrder.t.tokenOut,
            pendingOrder.t.exchangeRate,
            pendingOrder.t.deadline,
            pendingOrder.t.OrderIsExecuted,
            pendingOrder.t.isMultiPath,
            pendingOrder.t.encodedPath
        );
    }
```

The zk-SNARK proof is verified via Groth16Verifier, using `_proofA`, `_proofB`, `_proofC`, and `signals` to ensure that the off-chain computed hash (`HOsF` and `HOsE`) matches the on-chain data and prevents the agent from maliciously tampering with the order details.

### Plaintext Order O
O consists of t and s.

### Shielded Order oBar
oBar consists of t and H(s).

### Purchase Order (i.e., details of the plaintext order O)
Where t is a septuple and s is a triplet:
```solidity
struct OrderDetails {
        address swapper;
        address recipient;
        address tokenIn;
        address tokenOut;
        uint256 exchangeRate;
        uint256 deadline;
        bool OrderIsExecuted;
        bool isMultiPath; 
        bytes encodedPath; 
    }
struct Os {
        uint256 a0e;
        uint256 a1m;
        bytes32 salt;
    }
```

<details>
<summary>
Explanation of t:
</summary>

  - `swapper`: The user address that wants to initiate the swap.
  - `recipient`: The receiving address.
  - `tokenIn`: The contract address of the payment token (e.g., USDC contract address if exchanging with USDC)
  - `tokenOut`: The contract address of the receiving token (e.g., ETH contract address if receiving ETH)
  - `exchangeRate`: The exchange rate, i.e., a0e/a1m.
  - `deadline`: The order expiration time. Orders that are not executed before this time will be discarded.
  - `OrderIsExecuted`: Indicates whether the current order has been executed.
  - `isMultiPath`: Indicates whether the order is a multi-path order.
  - `encodedPath`: Encoded path for multi-path orders.
</details>

<details>
<summary>
Explanation of s:
</summary>

  - `a0e`: The amount of USDC the user is willing to spend.
  - `a1m`: The minimum amount of ETH acceptable to the user.
  - `salt`: A random value to prevent brute force attacks when a0e and a1m are small. Note that the first bit of salt must be 0 (to avoid errors in the circuit operation).
</details>

For instance, if Alice wants to swap 10,000 USDC for 10 ETH, the constructed order O should be:
```JSON
O:{
    "t": {
        "swapper": "swapper/sender",
        "recipient": "receiver",
        "tokenIn": "0x176211869cA2b568f2A7D4EE941E073a821EE1ff",
        "tokenOut": "0xe5D7C2a44FfDDf6b295A15c148167daaAf5Cf34f",
        "exchangeRate": "1000",
        "deadline": "12345678",
        "OrderIsExecuted": "false",
        "isMultiPath": "false",
        "encodedPath": ""
    },
    "s": {
        "a0e": "100000",
        "a1m": "10",
        "salt": "0x56b1a323c72b42888beb02627b6befb3f170bc7aa9eaa7bb563b0eb46ac1b939"
    }
}
```
- `O.t.swapper`: user, the person who wants to perform the token swap.

- `O.t.recipient`: receiver, the address where the swapped ETH will be sent.

- `O.t.tokenIn`: `0x176211869cA2b568f2A7D4EE941E073a821EE1ff`, the contract address for USDC.

- `O.t.tokenOut`: `0xe5D7C2a44FfDDf6b295A15c148167daaAf5Cf34f`, the contract address for wETH.

- `O.t.exchangeRate`: 1000, the exchange rate, used by the off-chain script to determine when to forward the order.

- `O.t.exchangeRate`: timeStamp, the expiration time for the transaction. If the transaction hasn't been forwarded by the time the deadline (ddl) is reached, it will be discarded.

- `O.t.OrderIsExecuted`: flag, indicates whether the transaction has been executed. **false** means it hasn’t been executed, `true` means it has been.

- `O.t.isMultiPath`: flag, indicates whether the transaction is a multi-path transaction. **false** means it isn’t, `true` means it is.

- `O.t.encodedPath`: empty, used for multi-path transactions.

- `O.s.a0e`: 100000, the amount of USDC Alice is willing to spent.

- `O.s.a1m`: 10, the minimum acceptable amount of ETH to be received.

- `O.s.salt`: random large number, used to prevent brute-force attacks (as attackers could easily try brute-forcing when `O.s.a0e` and `O.s.a1m` are small).

## Workflow

### User Approves Amount to Delegate Contract:
The user approves a large amount to the delegate contract (the user can decide the amount themselves, but it's recommended not to match the exact swap amount, so the approved amount and the actual swap amount a0e remain uncorrelated). The user constructs `oBar` using `O (oBar.t == O.t, oBar.s == H(a0e, a1m, salt))`.

### User Sends Data to Agent:
When a user places an order, the agent will synchronously transfer O.s to zerobase-prover and obtain the returned proof and oBar.s. The user then initiates a transaction to store oBar in the delegate contract (to prevent oBar.t from being maliciously modified by the agent before the order is forwarded, because the on-chain data cannot be changed). The delegate contract forces the initial value of oBar.t.f to be "false" and triggers an event, indicating that the contract has received oBar.

### Agent Monitors Exchange Rate:
The agent listens to the event on the contract, queries oBar.t.exchangeRate and continuously fetches real-time exchange rates using a pricing function or script until the rate meets the user's expectations. 

### Agent Executes Swap When Conditions Are Met:
When the exchange rate is met, the agent calls the swapForward function on the contract (through flashBots to prevent frontrunning). The agent ensures:
- The exchange rate matches the user’s expectations.
- The shielded order has not been executed before.
- The shielded order has not expired.
- The proof is valid `(H(a0e, a1m, salt) == oBar.s`, preventing the agent from maliciously modifying O.s).

### Agent Collects Gas Fees:
Within the function, `takeFeeInternal` function is called to collect gas fees paid by the agent. The fees can be calculated using the average network gas fee or estimated with estimateGas from web3py/web3js.

### Order Forwarding:
Inside the function, swap is called to forward the order and complete the transaction.

### Project Team Collects Gas Fees:
Finally, the project team calls the profit function in the contract to collect gas fees paid on their behalf.

## Circuit-related Procedure

> In this section, we use the below path as an example

`circuit/keccak-circuit/example1`

### 1. Compile the Circuit

The circuit has already been compiled, and the `.wasm` file is ready for use.

### 2. Generate Witness

Before generating the witness (e.g. `/example1/generate_witness.js`), the `input.json` needs to be constructed. You can generate the witness using the following command:

```bash
$ node generate_witness.js main.wasm input.json witness.wtns
```

### 3. Generate ZKP

Since the trusted setup has already been completed, you can directly proceed to the proof generation step.

### 4. Generate Proof

Use the following command to generate the proof:

```bash
$ snarkjs groth16 prove main_0001.zkey witness.wtns proof.json public.json
```

### 5. Submit Proof

The generated `proof.json` and `public.json` can then be submitted to the contract for verification.

## Foundry Usage

### Build

```shell
$ forge build --via-ir
```

Result

<img width="245" alt="forge_build" src="https://github.com/user-attachments/assets/752a622d-70e0-4ea1-9f0a-870d380f159d">

### Test

```shell
$ forge test --via-ir
```

Result

<img width="689" alt="forge_test" src="https://github.com/user-attachments/assets/90b7cc83-7ec5-43ef-b904-b00d4369c0a3">

### Deploy

In `script/Deploy.s.sol`, please replace the below parameters:
1. **owner**
2. **agent**

#### The following address is used for the test deployment:

https://chainscan-newton.0g.ai/address/0x78376f00f9d61cd99d62e1b8db34ffe22e23937d

```Solidity
// SPDX-License-Identifier: UNLICENSED
contract ZDPScript is Script {
    function setUp() public {}

    function run() public {
        ...
        ZDPc zdp = new ZDPc(0x2b2E23ceC9921288f63F60A839E2B28235bc22ad, payable(0x610D2f07b7EdC67565160F587F37636194C34E74), agent, 0xe5D7C2a44FfDDf6b295A15c148167daaAf5Cf34f);
        ...
    }
}
```

```shell
$ forge script script/Deploy.s.sol --rpc-url <your_rpc_url> --private-key <your_private_key> --broadcast --via-ir -vv
```

### Help

```shell
$ forge --help
$ anvil --help
$ cast --help
```
