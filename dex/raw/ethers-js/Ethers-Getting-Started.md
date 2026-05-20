---
created: 2026-05-18T10:09:17 (UTC +08:00)
tags: [ethers, ethereum, javascript]
source: https://docs.ethers.org/v6/getting-started/
author: ethers.js
title: Ethers.js Getting Started
---

# Ethers.js Getting Started

> Documentation for ethers, a complete, tiny and simple Ethereum library.

This is a very short introduction to Ethers, but covers many of the most common operations that developers require and provides a starting point for those newer to Ethereum.

## Getting Ethers

If using NPM, you must first install Ethers.

### Installing via NPM

```bash
npm install ethers
```

Everything in Ethers is exported from its root as well as on the `ethers` object. There are also exports in the `package.json` to facilitate more fine-grained importing.

Generally this documentation will presume all exports from `ethers` have been imported in the code examples, but you may import the necessary objects in any way you wish.

### Importing in Node.js

```javascript
import { ethers } from "ethers";
import { BrowserProvider, parseUnits } from "ethers";
import { HDNodeWallet } from "ethers/wallet";
```

### Importing ESM in a browser

```html
<script type="module">
  import { ethers } from "https://cdnjs.cloudflare.com/ajax/libs/ethers/6.7.0/ethers.min.js";
</script>
```

## Some Common Terminology

To begin, it is useful to have a basic understanding of the types of objects available and what they are responsible for, at a high level.

### Provider

A [Provider](https://docs.ethers.org/v6/api/providers/#Provider) is a read-only connection to the blockchain, which allows querying the blockchain state, such as account, block or transaction details, querying event logs or evaluating read-only code using `call`.

If you are coming from Web3.js, you are used to a Provider offering both read and write access. In Ethers, all write operations are further abstracted into another object, the **Signer**.

### Signer

A [Signer](https://docs.ethers.org/v6/api/providers/#Signer) wraps all operations that interact with an account. An account generally has a private key located somewhere, which can be used to sign a variety of types of payloads.

The private key may be located in memory (using a [Wallet](https://docs.ethers.org/v6/api/wallet/#Wallet)) or protected via some IPC layer, such as MetaMask which proxies interaction from a website to a browser plug-in, which keeps the private key out of the reach of the website and only permits interaction after requesting permission from the user and receiving authorization.

### Transaction

To make any state changes to the blockchain, a transaction is required, which requires a fee to be paid, where the fee covers the associated costs with executing the transaction (such as reading the disk and performing maths) and storing the updated information.

If a transaction reverts, a fee must still be paid, since the validator still had to expend resources to try running the transaction to determine that it reverted and the details of its failure are still recorded.

Transactions include sending ether from one user to another, deploying a Contract or executing a state-changing operation against a Contract.

### Contract

A [Contract](https://docs.ethers.org/v6/api/contract/#Contract) is a program that has been deployed to the blockchain, which includes some code and has allocated storage which it can read from and write to.

It may be read from when it is connected to a [Provider](https://docs.ethers.org/v6/api/providers/#Provider) or state-changing operations can be called when connected to a [Signer](https://docs.ethers.org/v6/api/providers/#Signer).

### Receipt

Once a Transaction has been submitted to the blockchain, it is placed in the memory pool (mempool) until a validator decides to include it.

A transaction's changes are only made once it has been included in the blockchain, at which time a receipt is available, which includes details about the transaction, such as which block it was included in, the actual fee paid, gas used, all the events that it emitted and whether it was successful or reverted.

## Connecting to Ethereum

The very first thing needed to begin interacting with the blockchain is connecting to it using a [Provider](https://docs.ethers.org/v6/api/providers/#Provider).

### MetaMask (and other injected providers)

The quickest and easiest way to experiment and begin developing on Ethereum is to use [MetaMask](https://metamask.io/), which is a browser extension that injects objects into the window, providing:

- read-only access to the Ethereum network (a [Provider](https://docs.ethers.org/v6/api/providers/#Provider))
- authenticated write access backed by a private key (a [Signer](https://docs.ethers.org/v6/api/providers/#Signer))

When requesting access to the authenticated methods, such as sending a transaction or even requesting the private key address, MetaMask will show a pop-up to the user asking for permission.

```javascript
let signer = null;
let provider;

if (window.ethereum == null) {
  console.log("MetaMask not installed; using read-only defaults");
  provider = ethers.getDefaultProvider();
} else {
  provider = new ethers.BrowserProvider(window.ethereum);
  signer = await provider.getSigner();
}
```

### Custom RPC Backend

If you are running your own Ethereum node (e.g. [Geth](https://geth.ethereum.org/)) or using a custom third-party service (e.g. [INFURA](https://infura.io/)), you can use the [JsonRpcProvider](https://docs.ethers.org/v6/api/providers/jsonrpc/#JsonRpcProvider) directly, which communicates using the [JSON-RPC](https://github.com/ethereum/wiki/wiki/JSON-RPC) protocol.

When using your own Ethereum node or a developer-base blockchain, such as Hardhat or Ganache, you can get access to the accounts with `JsonRpcProvider.getSigner()`.

#### Connecting to a JSON-RPC URL

```javascript
provider = new ethers.JsonRpcProvider(url);
signer = await provider.getSigner();
```

## User Interaction

All units in Ethereum tend to be integer values, since dealing with decimals and floating points can lead to imprecise and non-obvious results when performing mathematic operations.

As a result, the internal units used (e.g. wei) which are suited for machine-readable purposes and maths are often very large and not easily human-readable.

For example, imagine dealing with dollars and cents; you would show values like `"$2.56"`. In the blockchain world, we would keep all values as cents, so that would be `256` cents, internally.

So, when accepting data that a user types, it must be converted from its decimal string representation (e.g. `"2.56"`) to its lowest-unit integer representation (e.g. `256`). And when displaying a value to a user the opposite operation is necessary.

In Ethereum, one ether is equal to `10 ** 18` wei and one gwei is equal to `10 ** 9` wei, so the values get very large very quickly. Some convenience functions are provided to help convert between representations.

```javascript
eth = parseEther("1.0");           // 1000000000000000000n
feePerGas = parseUnits("4.5", "gwei"); // 4500000000n
formatEther(eth);                  // '1.0'
formatUnits(feePerGas, "gwei");    // '4.5'
```

## Interacting with the Blockchain

### Querying State

Once you have a [Provider](https://docs.ethers.org/v6/api/providers/#Provider), you have a read-only connection to the data on the blockchain. This can be used to query the current account state, fetch historic logs, look up contract code and so on.

```javascript
await provider.getBlockNumber();              // 23929462
balance = await provider.getBalance("ethers.eth"); // 4085267032476673080n
formatEther(balance);                         // '4.08526703247667308'
await provider.getTransactionCount("ethers.eth");  // 2
```

### Sending Transactions

To write to the blockchain you require access to a private key which controls some account. In most cases, those private keys are not accessible directly to your code, and instead you make requests via a [Signer](https://docs.ethers.org/v6/api/providers/#Signer), which dispatches the request to a service (such as [MetaMask](https://metamask.io/)) which provides strictly gated access and requires feedback to the user to approve or reject operations.

```javascript
tx = await signer.sendTransaction({
  to: "ethers.eth",
  value: parseEther("1.0")
});
receipt = await tx.wait();
```

## Contracts

A Contract is a meta-class, which means that its definition is derived at run-time, based on the ABI it is passed, which then determines what methods and properties are available on it.

### Application Binary Interface (ABI)

Since all operations that occur on the blockchain must be encoded as binary data, we need a concise way to define how to convert between common objects (like strings and numbers) and its binary representation, as well as encode the ways to call and interpret the Contract.

For any method, event or error you wish to use, you must include a [Fragment](https://docs.ethers.org/v6/api/abi/abi-coder/#Fragment) to inform Ethers how it should encode the request and decode the result.

Any methods or events that are not needed can be safely excluded.

There are several common formats available to describe an ABI. The Solidity compiler usually dumps a JSON representation but when typing an ABI by hand it is often easier (and more readable) to use the human-readable ABI, which is just the Solidity signature.

#### Simplified ERC-20 ABI

```javascript
abi = [
  "function decimals() view returns (string)",
  "function symbol() view returns (string)",
  "function balanceOf(address addr) view returns (uint)"
];
contract = new Contract("dai.tokens.ethers.eth", abi, provider);
```

### Read-only methods (i.e. view and pure)

A read-only method is one which cannot change the state of the blockchain, but often provide a simple interface to get important data about a Contract.

#### Reading the DAI ERC-20 contract

```javascript
abi = [
  "function decimals() view returns (uint8)",
  "function symbol() view returns (string)",
  "function balanceOf(address a) view returns (uint)"
];
contract = new Contract("dai.tokens.ethers.eth", abi, provider);
sym = await contract.symbol();                    // 'DAI'
decimals = await contract.decimals();             // 18n
balance = await contract.balanceOf("ethers.eth"); // 4000000000000000000000n
formatUnits(balance, decimals);                   // '4000.0'
```

### State-changing Methods

#### Change state on an ERC-20 contract

```javascript
abi = ["function transfer(address to, uint amount)"];
contract = new Contract("dai.tokens.ethers.eth", abi, signer);
amount = parseUnits("1.0", 18);
tx = await contract.transfer("ethers.eth", amount);
await tx.wait();
```

#### Forcing a call (simulation) of a state-changing method

```javascript
abi = ["function transfer(address to, uint amount) returns (bool)"];
contract = new Contract("dai.tokens.ethers.eth", abi, provider);
amount = parseUnits("1.0", 18);
await contract.transfer.staticCall("ethers.eth", amount); // true

other = new VoidSigner("0x643aA0A61eADCC9Cc202D1915D942d35D005400C");
contractAsOther = contract.connect(other.connect(provider));
await contractAsOther.transfer.staticCall("ethers.eth", amount); // true
```

### Listening to Events

When adding event listeners for a named event, the event parameters are destructured for the listener.

There is always one additional parameter passed to a listener, which is an [EventPayload](https://docs.ethers.org/v6/api/utils/events/#EventPayload), which includes more information about the event including the filter and a method to remove that listener.

#### Listen for ERC-20 events

```javascript
abi = ["event Transfer(address indexed from, address indexed to, uint amount)"];
contract = new Contract("dai.tokens.ethers.eth", abi, provider);

contract.on("Transfer", (from, to, _amount, event) => {
  const amount = formatEther(_amount, 18);
  console.log(`${from} => ${to}: ${amount}`);
  event.removeListener();
});

contract.on(contract.filters.Transfer, (from, to, amount, event) => {});

filter = contract.filters.Transfer(null, "ethers.eth");
contract.on(filter, (from, to, amount, event) => {});

contract.on("*", (event) => {});
```

### Query Historic Events

When querying within a large range of blocks, some backends may be prohibitively slow, may return an error or may truncate the results without any indication. This is at the discretion of each backend.

#### Query historic ERC-20 events

```javascript
abi = ["event Transfer(address indexed from, address indexed to, uint amount)"];
contract = new Contract("dai.tokens.ethers.eth", abi, provider);
filter = contract.filters.Transfer;
events = await contract.queryFilter(filter, -100);
events.length; // 319

// events[0] example:
// EventLog {
//   address: '0x6B175474E89094C44Da98b954EedeAC495271d0F',
//   args: Result(3) [
//     '0xC2e9F25Be6257c210d7Adf0D4Cd6E3E881ba25f8',
//     '0xfBd4cdB413E45a52E2C8312f670e9cE67E794C37',
//     370136547842778449664n
//   ],
//   blockHash: '0x9d343d0880968c61e2285ebbeecf02a73e535010cdcef97c4823a69bb78e57f8',
//   blockNumber: 23929363,
//   ...
// }

filter = contract.filters.Transfer(null, "ethers.eth");
events = await contract.queryFilter(filter);

// events[0] example:
// EventLog {
//   address: '0x6B175474E89094C44Da98b954EedeAC495271d0F',
//   args: Result(3) [
//     '0xaB7C8803962c0f2F5BBBe3FA8bf41cd82AA1923C',
//     '0x047218ea2AE0Ce6be5052AC99AfB59B26d3e5059',
//     4000000000000000000000n
//   ],
//   blockNumber: 12385295,
//   ...
// }
```

## Signing Messages

A private key can do a lot more than just sign a transaction to authorize it. It can also be used to sign other forms of data, which are then able to be validated for other purposes.

For example, signing a message can be used to prove ownership of an account which a website could use to authenticate a user and log them in.

```javascript
signer = new Wallet(id("test"));
// Wallet {
//   address: '0xC08B5542D177ac6686946920409741463a15dDdB',
//   provider: null
// }

message = "sign into ethers.org?";
sig = await signer.signMessage(message);
// '0xefc6e1d2f21bb22b1013d05ecf1f06fd73cdcb34388111e4deec58605f3667061783be1297d8e3bee955d5b583bac7b26789b4a4c12042d59799ca75d98d23a51c'

verifyMessage(message, sig);
// '0xC08B5542D177ac6686946920409741463a15dDdB'
```

Many other more advanced protocols built on top of signed messages are used to allow a private key to authorize other users to transfer their tokens, allowing the transaction fees of the transfer to be paid by someone else.
