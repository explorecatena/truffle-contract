# truffle-contract

Better Ethereum contract abstraction, for Node and the browser.

### Install

```
$ npm install truffle-contract
```

### Features

* Synchronized transactions for better control flow (i.e., transactions won't finish until you're guaranteed they've been mined).
* Promises. No more callback hell. Works well with `ES6` and `async/await`.
* Default values for transactions, like `from` address or `gas`.
* Returning logs, transaction receipt and transaction hash of every synchronized transaction.

### Usage

First, set up a new web3 provider instance and initialize your contract, then `require("truffle-contract")`. The input to the `contract` function is a JSON blob defined by [truffle-contract-schema](https://github.com/trufflesuite/truffle-contract-schema). This JSON blob is structured in a way that can be passed to all truffle-related projects.

```javascript
var provider = new Web3.providers.HttpProvider("http://localhost:8545");
var contract = require("truffle-contract");

var MyContract = contract({
  abi: ...,
  unlinked_binary: ...,
  address: ..., // optional
  // many more
})
MyContract.setProvider(provider);
```

You now have access to the following functions on `MyContract`, as well as many others:

* `at()`: Create an instance of `MyContract` that represents your contract at a specific address.
* `deployed()`: Create an instance of `MyContract` that represents the default address managed by `MyContract`.
* `new()`: Deploy a new version of this contract to the network, getting an instance of `MyContract` that represents the newly deployed instance.

Each instance is tied to a specific address on the Ethereum network, and each instance has a 1-to-1 mapping from Javascript functions to contract functions. For instance, if your Solidity contract had a function defined `someFunction(uint value) {}` (solidity), then you could execute that function on the network like so:

  ```javascript
  var deployed;
  MyContract.deployed().then(function(instance) {
    var deployed = instance;
    return instance.someFunction(5);
  }).then(function(result) {
    // Do something with the result or continue with more transactions.
  });
  ```

### Browser Usage

In your `head` element, include Web3 and then include truffle-contract:

```
<script type="text/javascript" src="./path/to/web3.min.js"></script>
<script type="text/javascript" src="./dist/truffle-contract.min.js"></script>
```

Alternatively, you can use the non-minified versions for easier debugging.

With this usage, `truffle-contract` will be available via the `TruffleContract` object:

```
var MyContract = TruffleContract(...);
```

### Full Example

Let's use `truffle-contract` with an example contract from [Dapps For Beginners](https://dappsforbeginners.wordpress.com/tutorials/your-first-dapp/). In this case, the abstraction has been saved to a `.sol.js` file by [truffle-artifactor](https://github.com/trufflesuite/truffle-artifactor):

```javascript
// Require the package that was previosly saved by truffle-artifactor
var MetaCoin = require("./path/to/MetaCoin.sol.js");

// Remember to set the Web3 provider (see above).
MetaCoin.setProvider(provider);

// In this scenario, two users will send MetaCoin back and forth, showing
// how truffle-contract allows for easy control flow.
var account_one = "5b42bd01ff...";
var account_two = "e1fd0d4a52...";

// Note our MetaCoin contract exists at a specific address.
var contract_address = "8e2e2cf785...";
var coin;

MetaCoin.at(contract_address).then(function(instance) {
  coin = instance;

  // Make a transaction that calls the function `sendCoin`, sending 3 MetaCoin
  // to the account listed as account_two.
  return coin.sendCoin(account_two, 3, {from: account_one});
}).then(function(result) {
  // This code block will not be executed until truffle-contract has verified
  // the transaction has been processed and it is included in a mined block.
  // truffle-contract will error if the transaction hasn't been processed in 120 seconds.

  // Since we're using promises, we can return a promise for a call that will
  // check account two's balance.
  return coin.balances.call(account_two);
}).then(function(balance_of_account_two) {
  alert("Balance of account two is " + balance_of_account_two + "!"); // => 3

  // But maybe too much was sent. Let's send some back.
  // Like before, will create a transaction that returns a promise, where
  // the callback won't be executed until the transaction has been processed.
  return coin.sendCoin(account_one, 1.5, {from: account_two});
}).then(function(result) {
  // Again, get the balance of account two
  return coin.balances.call(account_two)
}).then(function(balance_of_account_two) {
  alert("Balance of account two is " + balance_of_account_two + "!") // => 1.5
}).catch(function(err) {
  // Easily catch all errors along the whole execution.
  alert("ERROR! " + err.message);
});
```

# API

There are two API's you'll need to be aware of. One is the static Contract Abstraction API and the other is the Contract Instance API. The Abstraction API is a set of functions that exist for all contract abstractions, and those function exist on the abstraction itself (i.e., `MyContract.at()`). In contrast, the Instance API is the API available to contract instances -- i.e., abstractions that represent a specific contract on the network -- and that API is created dynamically based on functions available in your Solidity source file.

### Contract Abstraction API

Each contract abstraction -- `MyContract` in the examples above -- have the following useful functions:

#### `MyContract.new([arg1, arg2, ...], [tx params])`

This function take whatever contructor parameters your contract requires and deploys a new instance of the contract to the network. There's an optional last argument which you can use to pass transaction parameters including the transaction from address, gas limit and gas price. This function returns a Promise that resolves into a new instance of the contract abstraction at the newly deployed address.

#### `MyContract.at(address)`

This function creates a new instance of the contract abstraction representing the contract at the passed in address. Returns a "thenable" object (not yet an actual Promise for backward compatibility). Resolves to a contract abstraction instance after ensuring code exists at the specified address.

#### `MyContract.deployed()`

Creates an instance of the contract abstraction representing the contract at its deployed address. The deployed address is a special value given to truffle-contract that, when set, saves the address internally so that the deployed address can be inferred from the given Ethereum network being used. This allows you to write code referring to a specific deployed contract without having to manage those addresses yourself. Like `at()`, `deployed()` is thenable, and will resolve to a contract abstraction instance representing the deployed contract after ensuring that code exists at that location and that that address exists on the network being used.

#### `MyContract.getTransaction(txHash)`

Given a transaction hash for a transaction done with an instance of the contract abstracted by this contract abstraction, this returns a Promise which resolves to a object with three properties: `tx` which is the transaction hash passed in, `receipt` which is the transaction receipt as reported by the underlying Web3 instance, and `logs` which are the decoded logs attached with the transaction. Note that this is the same object which is returned when making a transaction with a contract function on a contract abstraction instance. If the transaction has not been confirmed, the promise will resolve to `null`.

#### `MyContract.syncTransaction(txHash)`

Returns a promise that resolves to the same object as `MyContract.getTransaction(txHash)` but will wait up to `MyContract.synchronization_timeout` ms for the transaction to confirm. If the transaction does not confirm within the timeout window, the promise will reject with an Error.

#### `MyContract.link(instance)`

Link a library represented by a contract abstraction instance to MyContract. The library must first be deployed and have its deployed address set. The name and deployed address will be inferred from the contract abstraction instance. When this form of `MyContract.link()` is used, MyContract will consume all of the linked library's events and will be able to report that those events occurred during the result of a transaction.

Libraries can be linked multiple times and will overwrite their previous linkage.

Note: This method has two other forms, but this form is recommended.

#### `MyContract.link(name, address)`

Link a library with a specific name and address to MyContract. The library's events will not be consumed using this form.

#### `MyContract.link(object)`

Link multiple libraries denoted by an Object to MyContract. The keys must be strings representing the library names and the values must be strings representing the addresses. Like above, libraries' events will not be consumed using this form.

#### `MyContract.networks()`

View a list of network ids this contract abstraction has been set up to represent.

#### `MyContract.setProvider(provider)`

Sets the web3 provider this contract abstraction will use to make transactions.

#### `MyContract.setNetwork(network_id)`

Sets the network that MyContract is currently representing.

#### `MyContract.hasNetwork(network_id)`

Returns a boolean denoting whether or not this contract abstraction is set up to represent a specific network.

#### `MyContract.defaults([new_defaults])`

Get's and optionally sets transaction defaults for all instances created from this abstraction. If called without any parameters it will simply return an Object representing current defaults. If an Object is passed, this will set new defaults. Example default transaction values that can be set are:

```javascript
MyContract.defaults({
  from: ...,
  gas: ...,
  gasPrice: ...,
  value: ...
})
```

Setting a default `from` address, for instance, is useful when you have a contract abstraction you intend to represent one user (i.e., one address).

#### `MyContract.clone(network_id)`

Clone a contract abstraction to get another object that manages the same contract artifacts, but using a different `network_id`. This is useful if you'd like to manage the same contract but on a different network. When using this function, don't forget to set the correct provider afterward.

```javascript
var MyOtherContract = MyContract.clone(1337);
```

### Contract Instance API

Each contract instance is different based on the source Solidity contract, and the API is created dynamically. For the purposes of this documentation, let's use the following Solidity source code below:

```javascript
contract MyContract {
  uint public value;
  event ValueSet(uint val);
  function setValue(uint val) {
    value = val;
    ValueSet(value);
  }
  function getValue() constant returns (uint) {
    return value;
  }
}
```

From Javascript's point of view, this contract has three functions: `setValue`, `getValue` and `value`. This is because `value` is public and automatically creates a getter function for it.

#### Making a transaction via a contract function

When we call `setValue()`, this creates a transaction. From Javascript:

```javascript
instance.setValue(5).then(function(result) {
  // result object contains important information about the transaction
});
```

Note that this is the same as:
```javascript
instance.setValue.sendTransaction(5).then(MyContract.syncTransaction).then(function(result) {
  // same result object as above
});
```

#### Explicitly making a call instead of a transaction

We can call `setValue()` without creating a transaction by explicitly using `.call`:

```javascript
instance.setValue.call(5).then(...);
```

This isn't very useful in this case, since `setValue()` sets things, and the value we pass won't be saved since we're not creating a transaction.

#### Calling getters

However, we can *get* the value using `getValue()`, using `.call()`. Calls are always free and don't cost any Ether, so they're good for calling functions that read data off the blockchain:

```javascript
instance.getValue.call().then(function(val) {
  // val reprsents the `value` storage object in the solidity contract
  // since the contract returns that value.
});
```

Even more helpful, however is we *don't even need* to use `.call` when a function is marked as `constant`, because `truffle-contract` will automatically know that that function can only be interacted with via a call:

```javascript
instance.getValue().then(function(val) {
  // val reprsents the `value` storage object in the solidity contract
  // since the contract returns that value.
});
```

#### Processing transaction results

When you make a transaction, you're given a `result` object that gives you a wealth of information about the transaction. You're given the transaction has (`result.tx`), the decoded events (also known as logs; `result.logs`), and a transaction receipt (`result.receipt`). In the below example, you'll recieve the `ValueSet()` event because you triggered the event using the `setValue()` function:

```javascript
instance.setValue(5).then(function(result) {
  // result.tx => transaction hash, string
  // result.logs => array of trigger events (1 item in this case)
  // result.receipt => receipt object
});
```

#### Sending Ether / Triggering the fallback function

You can trigger the fallback function by sending a transaction to this function:

```javascript
instance.sendTransaction({...}).then(function(result) {
  // Same result object as above.
});
```

This is promisified like all available contract instance functions, and has the same API as `web3.eth.sendTransaction` without the callback. The `to` value will be automatically filled in for you.

If you only want to send Ether to the contract a shorthand is available:

```javascript
instance.send(web3.toWei(1, "ether")).then(function(result) {
  // Same result object as above.
});
```

#### Estimating gas usage

Run this function to estimate the gas usage:

```javascript
instance.setValue.estimateGas(5).then(function(result) {
  // result => estimated gas for this transaction
});
```

# Testing

This package is the result of breaking up EtherPudding into multiple modules. Tests currently reside within [truffle-artifactor](https://github.com/trufflesuite/truffle-artifactor) but will soon move here.
