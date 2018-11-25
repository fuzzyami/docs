---
title: Add Kin To Your Exchange
---

This guide describes how to add kins from the Kin network to your exchange. Because the Kin network is a fork of the Stellar network, the terminology in this guide will refer to the Core and Horizon components as Kin/Stellar core and Kin/Stellar Horizon which are the forked versions of the original Stellar core and horizon components. This example uses Node.js and the [JS Stellar SDK](https://github.com/stellar/js-stellar-sdk), but it should be easy to adapt to other languages. Note that unlike Stellar, The Kin network does not support assets other than the native currency, Kin.

There are many ways to architect an exchange. This guide uses the following design:
 - `issuing account`: One Kin account that holds the majority of customer deposits offline.
 - `base account`: One Kin account that holds a small amount of customer deposits online and is used to payout to withdrawal requests.
 - `customerID`: Each user has a customerID, used to correlate incoming deposits with a particular user's account on the exchange.

The two main integration points to Kin for an exchange are:<br>
1) Listening for deposit transactions from the Kin network<br>
2) Submitting withdrawal transactions to the Kin network

## Setup

### Operational
* *(optional)* Set up Kin/Stellar Core. [Stellar's Doc](https://www.stellar.org/developers/stellar-core/software/admin.html)[Kin/Stellar forked code](https://github.com/kinecosystem/stellar-core)
* *(optional)* Set up Kin/Stellar Horizon. [Stellar's Doc](https://www.stellar.org/developers/horizon/reference/index.html)[Kin/Stellar forked code](https://github.com/kinecosystem/go/tree/kinecosystem/master/services/horizon)

It's recommended, though not strictly necessary, to run your own instances of Kin/Stellar Core and Kin/Horizon. If you choose not to, it's possible to use the Kinecosystem.com public-facing Horizon servers. Our test and live networks are listed below: 

```
  test net: {hostname:'horizon-playground.kininfrastructure.com', secure:true, port:443};
  live: {hostname:'horizon-ecosystem.kininfrastructure.com', secure:true, port:443};
```

### Issuing account
An issuing account is typically used to keep the bulk of customer funds secure. An issuing account is a Kin account whose secret keys are not on any device that touches the Internet. Transactions are manually initiated by a human and signed locally on the offline machine—a local install of `js-stellar-sdk` creates a `tx_blob` containing the signed transaction. This `tx_blob` can be transported to a machine connected to the Internet via offline methods (e.g., USB or by hand). This design makes the issuing account secret key much harder to compromise.

### Base account
A base account contains a more limited amount of funds than an issuing account. A base account is a Kin account used on a machine that is connected to the Internet. It handles the day-to-day sending and receiving of kins. The limited amount of funds in a base account restricts loss in the event of a security breach.

### Database
- Need to create a table for pending withdrawals, `StellarTransactions`.
- Need to create a table to hold the latest cursor position of the deposit stream, `StellarCursor`.
- Need to add a row to your users table that creates a unique `customerID` for each user.
- Need to populate the customerID row.

```
CREATE TABLE StellarTransactions (UserID INT, Destination varchar(56), KINAmount INT, state varchar(8));
CREATE TABLE StellarCursor (id INT, cursor varchar(50)); // id - AUTO_INCREMENT field
```

Possible values for `StellarTransactions.state` are "pending", "done", "error".


### Code

Use this code framework to integrate Kin into your exchange. For this guide, we use placeholder functions for reading/writing to the exchange database. Each database library connects differently, so we abstract away those details. The following sections describe each step:


```js
// Config your server
var config = {};
config.baseAccount = "your base account address";
config.baseAccountSecret = "your base account secret key";

// You can use KinEcosystem.com's instance of Horizon or your own
config.horizon = 'https://horizon-playground.kininfrastructure.com';

// Include the JS Stellar SDK
// It provides a client-side interface to Horizon
var StellarSdk = require('stellar-sdk');

// Initialize the Stellar SDK with the Horizon instance
// You want to connect to
var server = new StellarSdk.Server(config.horizon);

// Get the latest cursor position
var lastToken = latestFromDB("StellarCursor");

// Listen for payments from where you last stopped
// GET https://horizon-playground.kininfrastructure.com/accounts/{config.baseAccount}/payments?cursor={last_token}
let callBuilder = server.payments().forAccount(config.baseAccount);

// If no cursor has been saved yet, don't add cursor parameter
if (lastToken) {
  callBuilder.cursor(lastToken);
}

callBuilder.stream({onmessage: handlePaymentResponse});

// Load the account sequence number from Horizon and return the account
// GET https://horizon-playground.kininfrastructure.com/accounts/{config.baseAccount}
server.loadAccount(config.baseAccount)
  .then(function (account) {
    submitPendingTransactions(account);
  })
```

## Listening for deposits
When a user wants to deposit kins in your exchange, instruct them to send them to your base account address with the customerID in the memo field of the transaction.

You must listen for payments to the base account and credit any user that sends kins there. Here's code that listens for these payments:

```js
// Start listening for payments from where you last stopped
var lastToken = latestFromDB("StellarCursor");

// GET https://horizon-playground.kininfrastructure.com/accounts/{config.baseAccount}/payments?cursor={last_token}
let callBuilder = server.payments().forAccount(config.baseAccount);

// If no cursor has been saved yet, don't add cursor parameter
if (lastToken) {
  callBuilder.cursor(lastToken);
}

callBuilder.stream({onmessage: handlePaymentResponse});
```


For every payment received by the base account, you must:<br>
 - check the memo field to determine which user sent the deposit.<br>
 - record the cursor in the `StellarCursor` table so you can resume payment processing where you left off.<br>
 - credit the user's account in the DB with the number of kins they sent to deposit.

So, you pass this function as the `onmessage` option when you stream payments:

```js
function handlePaymentResponse(record) {

  // GET https://horizon-playground.kininfrastructure.com/transaction/{id of transaction this payment is part of}
  record.transaction()
    .then(function(txn) {
      var customer = txn.memo;

      // If this isn't a payment to the baseAccount, skip
      if (record.to != config.baseAccount) {
        return;
      }
      if (record.asset_type != 'native') {
         // If you are a KIN exchange and the customer sends
         // you a non-native asset, you can safely ignore it as non-native assets are not supported on the Kin network.
         // 2. Send it back to the customer  
      } else {
        // Credit the customer in the memo field
        if (checkExists(customer, "ExchangeUsers")) {
          // Update in an atomic transaction
          db.transaction(function() {
            // Store the amount the customer has paid you in your database
            store([record.amount, customer], "StellarDeposits");
            // Store the cursor in your database
            store(record.paging_token, "StellarCursor");
          });
        } else {
          // If customer cannot be found, you can raise an error,
          // add them to your customers list and credit them,
          // or do anything else appropriate to your needs
          console.log(customer);
        }
      }
    })
    .catch(function(err) {
      // Process error
    });
}
```


## Submitting withdrawals
When a user requests a kin withdrawal from your exchange, you must generate a Kin transaction to send them kins. See following Stellar document[building transactions](https://www.stellar.org/developers/js-stellar-base/learn/building-transactions.html) for more information.

The function `handleRequestWithdrawal` will queue up a transaction in the exchange's `StellarTransactions` table whenever a withdrawal is requested.

```js
function handleRequestWithdrawal(userID,amountKins,destinationAddress) {
  // Update in an atomic transaction
  db.transaction(function() {
    // Read the user's balance from the exchange's database
    var userBalance = getBalance('userID');

    // Check that user has the required kins
    if (amountKins <= userBalance) {
      // Debit the user's internal kin balance by the amount of kins they are withdrawing
      store([userID, userBalance - amountKins], "UserBalances");
      // Save the transaction information in the StellarTransactions table
      store([userID, destinationAddress, amountKins, "pending"], "StellarTransactions");
    } else {
      // If the user doesn't have enough KIN, you can alert them
    }
  });
}
```

Then, you should run `submitPendingTransactions`, which will check `StellarTransactions` for pending transactions and submit them.

```js
StellarSdk.Network.useTestNetwork();
// This is the function that handles submitting a single transaction

function submitTransaction(exchangeAccount, destinationAddress, amountKins) {
  // Update transaction state to sending so it won't be
  // resubmitted in case of the failure.
  updateRecord('sending', "StellarTransactions");

  // Check to see if the destination address exists
  // GET https://horizon-playground.kininfrastructure.com/accounts/{destinationAddress}
  server.loadAccount(destinationAddress)
    // If so, continue by submitting a transaction to the destination
    .then(function(account) {
      var transaction = new StellarSdk.TransactionBuilder(exchangeAccount)
        .addOperation(StellarSdk.Operation.payment({
          destination: destinationAddress,
          asset: StellarSdk.Asset.native(),
          amount: amountKins
        }))
        // Sign the transaction
        .build();

      transaction.sign(StellarSdk.Keypair.fromSecret(config.baseAccountSecret));

      // POST https://horizon-testnet.stellar.org/transactions
      return server.submitTransaction(transaction);
    })
    //But if the destination doesn't exist...
    .catch(StellarSdk.NotFoundError, function(err) {
      // create the account and fund it
      var transaction = new StellarSdk.TransactionBuilder(exchangeAccount)
        .addOperation(StellarSdk.Operation.createAccount({
          destination: destinationAddress,
          // Creating an account requires funding it with kins
          startingBalance: amountKins
        }))
        .build();

      transaction.sign(StellarSdk.Keypair.fromSecret(config.baseAccountSecret));

      // POST https://horizon-testnet.stellar.org/transactions
      return server.submitTransaction(transaction);
    })
    // Submit the transaction created in either case
    .then(function(transactionResult) {
      updateRecord('done', "StellarTransactions");
    })
    .catch(function(err) {
      // Catch errors, most likely with the network or your transaction
      updateRecord('error', "StellarTransactions");
    });
}

// This function handles submitting all pending transactions, and calls the previous one
// This function should be run in the background continuously

function submitPendingTransactions(exchangeAccount) {
  // See what transactions in the db are still pending
  // Update in an atomic transaction
  db.transaction(function() {
    var pendingTransactions = querySQL("SELECT * FROM StellarTransactions WHERE state =`pending`");

    while (pendingTransactions.length > 0) {
      var txn = pendingTransactions.pop();

      // This function is async so it won't block. For simplicity we're using
      // ES7 `await` keyword but you should create a "promise waterfall" so
      // `setTimeout` line below is executed after all transactions are submitted.
      // If you won't do it will be possible to send a transaction twice or more.
      await submitTransaction(exchangeAccount, tx.destinationAddress, tx.amountKins);
    }

    // Wait 30 seconds and process next batch of transactions.
    setTimeout(function() {
      submitPendingTransactions(sourceAccount);
    }, 30*1000);
  });
}
```
