# Crypto currency demo

This is demo of crypto currency implemented on Exonum blockchain.

It demonstrates the very basic operations:

- create a new wallet

- add funds into a wallet

- transfer funds from the one wallet to another

- monitor blocks status

### Backend

TODO

### Frontend

Frontend is a lightweight single page application implemented on [riotjs](https://github.com/riot/riot).

Application is served by Node.js and communicates directly with backends REST api and uses Exonum client to convert data into appropriate format and parse it into JSON.

All business logic is can be found in the file `cryptocurrency.js`.

#### Submit transaction

To create transaction of each type you need to declare the new entity of `newMessage` type.

##### Create a new wallet transaction

Here is an example of how `create a new wallet` transaction is declared:

```javascript
var CreateWalletTransaction = {
    size: 144,
    service_id: 128,
    message_id: 130,
    fields: {
        pub_key: {type: Exonum.PublicKey, size: 32, from: 0, to: 32},
        login: {type: Exonum.String, size: 8, from: 32, to: 40},
        sec_key_enc: {type: Exonum.String, size: 80, from: 40, to: 120},
        nonce: {type: Exonum.Nonce, size: 24, from: 120, to: 144}
    }
};
```

Then new random pair of publicKey and secretKey is generated:

```javascript
var pair = Exonum.keyPair(); 
```

Then random nonce is generated:

```javascript
var nonce = Exonum.randomNonce();
```

Then secretKey is encrypted using password as key:

```
var secretKeyEncrypted = Exonum.encryptDigest(pair.secretKey, nonce, password);
```

Then transaction data can be signed:

```javascript
var data = {
    pub_key: pair.publicKey,
    login: ...,
    sec_key_enc: secretKeyEncrypted,
    nonce: nonce
};

var signature = CreateWalletTransaction.sign(data, pair.secretKey);
```

Finally, signed data and signature can be submitted to server:

```javascript
$.ajax({
    method: 'POST',
    url: '...',
    contentType: 'application/json',
    data: JSON.stringify({
        service_id: 128,
        message_id: 130,
        body: data,
        signature: signature
    }),
    success: ...,
    error: ...
});
```

##### Add funds transaction

Here is an example of how `add funds` transaction is declared:

```javascript
var AddFundsTransaction = {
    size: 48,
    service_id: 128,
    message_id: 129,
    fields: {
        wallet: {type: Exonum.PublicKey, size: 32, from: 0, to: 32},
        amount: {type: Exonum.Int64, size: 8, from: 32, to: 40},
        seed: {type: Exonum.Uint64, size: 8, from: 40, to: 48}
    }
};
```

Then transaction data can be signed:

```javascript
var seed = Exonum.randomUint64();

var data = {
    wallet: ...,
    amount: ...,
    seed: seed
};

var signature = TransferTransaction.sign(data, secretKey);
```

Finally, signed data and signature can be submitted to server:

```javascript
$.ajax({
    method: 'POST',
    url: '...',
    contentType: 'application/json',
    data: JSON.stringify({
        service_id: 128,
        message_id: 129,
        body: data,
        signature: signature
    }),
    success: ...,
    error: ...
});
```

##### Transfer transaction

Here is an example of how `transfer` transaction is declared:

```javascript
var TransferTransaction = {
    size: 80,
    service_id: 128,
    message_id: 128,
    fields: {
        from: {type: Exonum.PublicKey, size: 32, from: 0, to: 32},
        to: {type: Exonum.PublicKey, size: 32, from: 32, to: 64},
        amount: {type: Exonum.Int64, size: 8, from: 64, to: 72},
        seed: {type: Exonum.Uint64, size: 8, from: 72, to: 80}
    }
};
```

Then transaction data can be signed:

```javascript
var seed = Exonum.randomUint64();

var data = {
    from: ...,
    to: ...,
    amount: ...,
    seed: seed
};

var signature = TransferTransaction.sign(data, secretKey);
```

Finally, signed data and signature can be submitted to server:

```javascript
$.ajax({
    method: 'POST',
    url: '...',
    contentType: 'application/json',
    data: JSON.stringify({
        service_id: 128,
        message_id: 128,
        body: data,
        signature: signature
    }),
    success: ...,
    error: ...
});
```

#### Get wallet

Wallet data is encoded inside next structure:

```json
{
    "block_info": {
        "block": {...},
        "precommits": [...]
    },
    "wallet": {
        "mpt_proof": {...},
        "value": {...}
    },
    "wallet_history": {
        "mt_proof": {...},
        "values": [...]
    }
}
```

Here the list of a necessary steps to get wallet data:

1) Verify block:

```javascript
Exonum.verifyBlock(data.block_info, validators);
```

`validators` is the array of validators.

2) Find wallets table hash at Merkle Patricia tree stored in `wallet.mpt_proof`. Key of this value is generated using `service_id` and `table_index`:

```javascript
var TableKey = Exonum.newType({
    size: 4,
    fields: {
        service_id: {type: Exonum.Uint16, size: 2, from: 0, to: 2},
        table_index: {type: Exonum.Uint16, size: 2, from: 2, to: 4}
    }
});

var tableKeyData = {
    service_id: serviceId,
    table_index: 0
};

var tableKey = TableKey.hash(tableKeyData);

var walletsHash = Exonum.merklePatriciaProof(data.block_info.block.state_hash, data.wallet.mpt_proof, tableKey);
```

3) Find wallet's data at Merkle Patricia tree stored in `wallet.value`. Wallets table hash from previous step is used as key.

```javascript
var wallet = Exonum.merklePatriciaProof(walletsHash, data.wallet.value, publicKey, Wallet);
```

`publicKey` is the public key of wallet;

`Wallet` is the custom type:

```javascript
var Wallet = Exonum.newType({
    size: 88,
    fields: {
        pub_key: {type: Exonum.PublicKey, size: 32, from: 0, to: 32},
        name: {type: Exonum.String, size: 8, from: 32, to: 40},
        balance: {type: Exonum.Uint64, size: 8, from: 40, to: 48},
        history_len: {type: Exonum.Uint64, size: 8, from: 48, to: 56},
        history_hash: {type: Exonum.Hash, size: 32, from: 56, to: 88}
    }
});
```

4) Find hashes of all transactions at Merkle tree in `wallet_history.mt_proof`.

```javascript
var hashes = Exonum.merkleProof(wallet.history_hash, wallet.history_len, data.wallet_history.mt_proof, [0, wallet.history_len]);
```

5) Validate each transaction and its hash, append hash to transaction data.

```javascript
var transactions = [];
for (var i in hashes) {
    if (!hashes.hasOwnProperty(i)) {
        continue;
    }

    if (!validateTransaction(data.wallet_history.values[i], hashes[i])) {
        return undefined;
    }

    var transaction = data.wallet_history.values[i];
    transaction.hash = hashes[i];

    transactions.push(transaction);
}
```

The steps from above guarantees all wallet info reliability and consistency.

#### Build

First, install npm dependencies:

```
npm install
```

Then install bower dependencies:

```
bower install
```

#### Run

To run application:

```
node frontend/app.js
```
