## Introduction to DynamoDB Transactions - atomicity, write isolation, idempotency and cost.
A basic demo to show some of the benefits and properties of DynamoDB Transactions. The scenario is an online bank, where customers have accounts, and there's a capability allowing for direct transfers from one customer account to another. It's important to maintain valid data and to never allow for a situation where a partially completed transaction is visible.

### Pre-requisites
 * An AWS Account with administrator access
 * A bash command-line environment such as Mac Terminal or Windows 10 bash shell
 * The [AWS CLI](https://aws.amazon.com/cli/) setup and configured
 * The [NoSQL Workbench for Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/workbench.html)
 
### Setup
1. Navigate to the project's folder.
2. Import model [OnlineBank.json](./tx/OnlineBank.json) into NoSQL Workbench for Amazon DynamoDB.
3. Use NoSQL Workbench Visualizer to "commit" the model to your AWS account in *us-east-1*.

You now have two tables called *Accounts* and *Transactions*.

Scan the tables to look at the existing data - accounts with balances, and a transaction
recorded already.

### Demo steps
1.  Perform a transfer.  

 ```balance_transfer.sh c3e67497-fcb0-4881-8477-b0cbedab7240 transfer1-allowed```

This succeeds - all conditions are met.  Notice the consumed writes, and re-scan the tables to see the new and changed items.


2.  Attempt a transfer where the payer has insufficient funds.

```balance_transfer.sh 4510ba8a-518b-4701-88b5-3db78e618f71 transfer2-underfunded```

This fails - the condition is not met on the first action, which is to verify
adequate funding in the source account.  Writes are consumed anyway.

3.  Attempt the same transfer again using the same idempotency key.

```balance_transfer.sh 4510ba8a-518b-4701-88b5-3db78e618f71 transfer2-underfunded```

Because the prior attempt failed due to a condition exception, the idempotency
token is not tracked by DynamoDB.  We try again, get the same exception, and
we consume the same writes.

4.  Attempt a transaction that uses a *txid* that was used in the past

```balance_transfer.sh 7d622075-f2f1-4dd4-8aaf-fb29e87c2b9a transfer3-txidused```

Now we try to make a transfer with a **txid** which matches the one that was
already recorded some time ago - it was in our initial sample data.  This fails
because the third condition is not matched - that every new transaction must
have its own unique txid.  This consumes writes.  
The validation check against historical use of *txid* was part of our application business logic, 
and did not involve the idempotency token.  Idempotency tokens are only tracked for around 10 mins).


5.  Perform a successful transfer, while using an idempotency token.

```balance_transfer.sh e896d9e5-818c-43b2-a139-59fd63fbcd12 transfer4-allowed```

This is a successful transfer - see the writes consumed, balances updated
and new transaction recorded.  But what if our client never received the 200
response from DynamoDB saying the transaction was committed?  It must retry, which could
be a problem - the exception would be raised due to seeing an existing *txid*,
preventing a repeat transfer, but to the client it still seems like a fail.
No way to know if this is because of retry in the short term or if this is a
genuine *txid* clash from separate balance transfer requests.  This sounds messy.


6.  Repeat the transfer

```balance_transfer.sh e896d9e5-818c-43b2-a139-59fd63fbcd12 transfer4-allowed```

Thankfully, if we retry within 10 minutes, DynamoDB will return a successful
response code to the client, so it knows it actually succeeded. The
transfer is not actually made again; no updates are made to the account balances and
the transaction is not recorded again.  You'll notice that capacity was
consumed - but look carefully.  It is read units.  No writes were made,
but read units are consumed in checking and confirming that the transaction was
in fact already successfully committed.  This adds a great deal of resilience
and integrity.  Clients can retry and ascertain the exact status of any
transaction.

## Cleanup
You can delete your tables if desired.

---

Please contribute to this code sample by issuing a Pull Request or creating an Issue.

Share your feedback at [@pj_naylor](https://twitter.com/pj_naylor)