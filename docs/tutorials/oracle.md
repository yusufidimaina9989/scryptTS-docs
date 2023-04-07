---
sidebar_position: 4
---

# Tutorial 4: Oracle

## Overview

In this tutorial, we will go over how to build a smart contract that consumes off-chain data from an oracle. Specifically, we will implement a smart contract that lets two players bet on the price of BSV at some point in the future. It retrieves prices from an oracle.

### What is an Oracle?
A blockchain oracle is a third-party service or agent that provides external data to a blockchain network. It is a bridge between the blockchain and the external world, enabling smart contracts to access, verify, and incorporate data from outside the blockchain. This allows smart contracts to execute based on real-world events and conditions, enhancing their utility and functionality.
![](../../static/img/oracle.jpeg) [Credit: bitnovo](https://blog.bitnovo.com/en/what-is-a-blockchain-oracle/)

The data supplied by oracles can include various types of information, such as stock prices, weather data, election results, and sports scores.

### Rabin Signatures
A digital signature is required to verify the authenticity and integrity of arbitrary data provided by known oracles in a smart contract. Instead of ECDSA used in Bitcoin, we use an alternative digital signature algorithm called [Rabin signatures](https://en.wikipedia.org/wiki/Rabin_signature_algorithm). This is because Rabin signature verification is orders of magnitude cheaper than ECDSA.
We have implemented [Rabin signature](https://github.com/sCrypt-Inc/scrypt-ts-lib/blob/master/src/rabinSignature.ts) as part of the standard libraries [`scrypt-ts-lib`](https://www.npmjs.com/package/scrypt-ts-lib), which can be imported and used directly. 

## Contract Properties

Our contract will take signed pricing data from the [WitnessOnChain oracle](https://witnessonchain.com). Depending if the price target is reached or not, it will pay out a reward to one of the two players.

There are quite a few properties which our price betting smart contract will require:

```ts
// Price target that needs to be reached.
@prop()
targetPrice: bigint

// Symbol of the pair, e.g. "BSV_USDC"
@prop()
symbol: ByteString

// Timestamp window in which the price target needs to be reached.
@prop()
timestampFrom: bigint
@prop()
timestampTo: bigint

// Oracles Rabin public key.
@prop()
oraclePubKey: RabinPubKey

// Addresses of both players.
@prop()
alicePkh: PubKeyHash
@prop()
bobPkh: PubKeyHash
```

Notice that the type `RabinPubKey`, which represents a Rabin public key, is not a standard type. You can import it the following way:

```ts
import { RabinPubKey } from 'scrypt-ts-lib'
```

## Public Method - `unlock`

The contract will have only a single public method, namely `unlock`. As parameters, it will take the oracles signature and the message signed:

```ts
@method()
public unlock(msg: ByteString, sig: RabinSig) {
    // Verify oracle signature.
    assert(
        RabinVerifierWOC.verifySig(msg, sig, this.oraclePubKey),
        'Oracle sig verify failed.'
    )

    // Decode data.
    // 4 bytes timestamp (LE) + 8 bytes rate (LE) + 1 byte decimal + 16 bytes symbol
    const timestamp = Utils.fromLEUnsigned(msg.slice(0, 8))
    const price = Utils.fromLEUnsigned(msg.slice(8, 24))
    const symbol: ByteString = msg.slice(26, 58)

    // Validate data.
    assert(timestamp >= this.timestampFrom, 'Timestamp too early.')
    assert(timestamp <= this.timestampTo, 'Timestamp too late.')
    assert(symbol == this.symbol, 'Wrong symbol.')

    // Include output that pays the winner.
    const outAmount = this.ctx.utxo.value // Include all sats from contract instance.
    const winner = price >= this.targetPrice ? this.alicePkh : this.bobPkh
    const out = Utils.buildPublicKeyHashOutput(winner, outAmount)
    assert(this.ctx.hashOutputs == hash256(out), 'hashOutputs mismatch')
}
```

Let's walk through each part.

First, we verify that the passed signature is correct. For that we use the `RabinVerifierWOC` library from the [`scrypt-ts-lib`](https://www.npmjs.com/package/scrypt-ts-lib) package

```ts
import { RabinPubKey, RabinSig, RabinVerifierWoc } from 'scrypt-ts-lib'
```

Now, we can call the `verifySig` method of the verification library:
```ts
// Verify oracle signature.
assert(
    RabinVerifierWOC.verifySig(msg, sig, this.oraclePubKey),
    'Oracle sig verify failed.'
)
``` 
The verification method requires the message signed by the oracle, the oracles signature for the message, and the oracle's public key, which we already set via the constructor.

Next, we need to parse information from the chunk of data that is the signed message and assert on it. For a granular description of the message format check out the `"Exchange Rate"` section in the [WitnessOnChain API docs](https://witnessonchain.com).

```ts
// Decode data.
// 4 bytes timestamp (LE) + 8 bytes rate (LE) + 1 byte decimal + 16 bytes symbol
const timestamp = Utils.fromLEUnsigned(msg.slice(0, 8))
const price = Utils.fromLEUnsigned(msg.slice(8, 24))
const symbol: ByteString = msg.slice(26, 58)
```

We parse out the following data:
- `timestamp` - The time at which this exchange rate is present.
- `price` - The exchange rate encoded as an integer -> (priceFloat * (10^decimal)).
- `symbol` - The symbol of the token pair, e.g. `BSV_USDC`.

Now we can validate the data. First, we check if the timestamp of the exchange rate is within our specified range that we bet on:

```ts
assert(timestamp >= this.timestampFrom, 'Timestamp too early.')
assert(timestamp <= this.timestampTo, 'Timestamp too late.')
```

Additionally, we check if the exchange rate is actually for the correct token pair:

```ts
assert(symbol == this.symbol, 'Wrong symbol.')
```

Lastly, upon having all the necessary information, we can choose the winner and pay to his address accordingly:

```ts
// Include output that pays the winner.
const outAmount = this.ctx.utxo.value // Include all sats from contract instance.
const winner = price >= this.targetPrice ? this.alicePkh : this.bobPkh
const out = Utils.buildPublicKeyHashOutput(winner, outAmount)
assert(this.ctx.hashOutputs == hash256(out), 'hashOutputs mismatch')
```

As we can see, if the target price us reached, Alice gets paid; otherwise Bob gets the funds. The contract ensures the winner gets paid via a standard [`P2PKH`](https://wiki.bitcoinsv.io/index.php/Bitcoin_Transactions#Pay_to_Public_Key_Hash_.28P2PKH.29) output.


## Conclusion

Congratulations! You have completed the oracle tutorial!

The full code along with [tests](https://github.com/sCrypt-Inc/boilerplate/blob/master/tests/local/priceBet.test.ts) can be found in sCrypt's [boilerplate repository](https://github.com/sCrypt-Inc/boilerplate/blob/master/src/contracts/priceBet.ts).
