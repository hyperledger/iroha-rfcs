| Status  | Stakeholders | Outcome | Due date |   Owner   |
| :-----: | :----------: | :-----: | :------: | :-------: |
| Started |              |         |          | @arjentix |



## Background

"Oracle" is a common name for different technologies which basically do one thing – bring off-chain data on chain. The possibilities are almost limitless:

1. Real wold currencies
2. Random numbers generation
3. Timestamp retrieving
4. Events in other blockchains
5. Election results
6. Geo-data i.e. to track delivery chains
7. Complex calculations
8. Weather data
9. Statistical data
10. The central bank's key rate

Obviously integrating Oracles into Iroha 2 based blockchains will expand the horizon on possibilities for our users. By providing a clear way to integrate your WASM Trigger with Oracles we encourage people to use Iroha 2 in really complex systems and bring their ideas to life.

## Problem

We need to add Oracles support into Iroha 2. Oracles should be a flexible part of a blockchain and should not be hardcoded anywhere on our side. There should be a clear mechanism to interact with them and respective documentation.

### Example scenarios

 Scenario 1:

1. End-user interacts with a trigger to buy some asset which is tied to a real world, i.e. gold, for some other asset A;
2. Trigger calls Oracle 1 to get gold price in dollars, and then calls Oracle 2 to get asset A price in dollars;
3. Trigger computes the result and transfers money to the buyer.

Scenario 2:

1. Weather sensors have registered a hurricane
2. Trigger has been activated to transfer insurance benefits to the clients

 Scenario 3:

1. End-user interacts with a trigger, requesting some permission token to be given to their account. This permission can be given only to account which corresponds to University "X" students.
2. Trigger calls Oracle of University "X" to ensure that account corresponds to a student
3. On success trigger sends permission token to the account

## Solution

Proposed solution is based on already built parts of Iroha and does not require much efforts to implement.
It consists of the two parts:

1. `EmitOracleRequest` Iroha Special Instruction;
2. `OracleRequest` event type.

`EmitOracleRequest`  – is a new ISI that does one thing – emits `OracleRequest` with provided payload.

`OracleRequest`  – special event which directly says that blockchain asks for off-chain data from some Oracle. Such events will be transmitted in event pipeline like all other Iroha 2 events.
These events should be caught by oracle off-chain workers. Such workers should use one of the Iroha 2 SDK to properly initialize event-listening connection or implement it by their own.

After the capture of `OracleRequest` event worker analyses the payload, makes the calculations it needs and writes the response to some *store* asset. Trigger can read the response from the *store* asset and proceed with further logic.

To make the process more secure trigger owner can grant the asset access only to a special Oracle account known beforehand.

Described solution can be simplified in some edge cases. For example there can be an off-chain worker that always transmits recent data (i.e. currencies). In that case there is no need in a special `OracleRequest`, trigger can read the info directly from the asset.

### Implementing example scenarios

Scenario 1 (quick request):

1. End-user transfers some amount of asset A to a predefined technical account;
2. Trigger which listens for incoming asset transfers to its technical account reads the *store* assets where two oracles consistently write data. It reads gold price in dollars and asset A price in dollars. After doing some calculations trigger transfers computed amount of synthetic gold to the end-user account.

Scenario 2 (publisher-consumer):

1. Off-chain Oracle worker receive messages about hurricane from sensors
2. Worker calls `ExecuteTrigger` instruction on-chain to start benefits transferring

Scenario 3 (request-response):

1. End-user calls trigger T1 using existing `ExecuteTrigger` instruction
2. Trigger T1 calls `EmitOracleRequest` instruction, providing payload with data like oracle id, request, user data, asset id to write response into and etc. That's all what this trigger does.
3. Off-chain oracle worker receives the `OracleRequest` event, retrieves user data, makes a request to a University "X" server, and writes the response to an asset specified in original request.
4. Trigger T2 being triggered on that asset modification. It reads the response from the asset, identifies if end-user is actually a student of University "X" and then grants a permission if so.

So technically Iroha 2 already supports first two scenarios. This document aims to find a solution for the third scenario.

### Implementing timeouts

For *request-response* cases there may be a need to handle timeout errors.

In such case *Trigger T1* (see scenario 3 for a reference) after emitting `OracleRequest` to *Oracle O* will also emit another `OracleRequest` to a special *Timeout Oracle* which tracks timeouts. Payload will contain the timeout value.

*Trigger T2* will have more complex activation filter, which also tracks store asset related to *Timeout Oracle*. When execution is started it analyzes triggering event if it got response from *Oracle O* or just awakened by *Timeout Oracle*.

Obviously this solution still relies on some Oracle but just not the original one. As we are in the blockchain nothing can happen without a documented reason or *transaction* to be more precise. So any solution which requires some error checking like that will require another Oracle, just a more trusted one.

### Implementing payment

Payment is not a big deal with centralized solutions, because in that case it can be done off-chain. This section is about automatic decentralized on-chain *Oracles* and payments.

There are a couple of ways to implement that:

1. Pay when the exact Oracle result is retrieved. Such result should contain the amount of money to pay. Probably not very usable solution, because *Oracle* should have a white-list or it can be abused with multiple one-shot accounts.
2. Basic pre-pay inside trigger. If there will be some change, that *Oracle* transfers it back. The quest is how much should trigger pre-pay? There can be even another *free* `OracleRequest` to retrieve the amount of estimated payment.
3. Permission to withdraw some amount of money from trigger account. That way *Oracle* can withdraw the exact amount of money it wants for its computations. Also such permission can contain the upper limit of withdrawal.

There are no best solution here. Users should choose whatever they find useful for their exact cases. There can be even more possible ways to process payments.

### Decisions

List all decisions you and other stakeholders made

### Alternatives

1. Make one peer (i.e. *Leader*) directly call some external *API* when trigger submits `EmitOracleRequest` instruction. This is a more on-chain variant and allows us to create integrations with popular oracle providers on our side. However it' not so secure and performant.
2. Implement our own IBC, make Oracle providers to run their own Iroha 2 based networks and make triggers use this IBC to retrieve data from Oracle blockchains. How exactly Oracle blockchains get data from off-chain world is out of the scope.

### Concerns

There is one unresolved problem:

1. Data signing. Most of the time we want to be sure that the actual computation was done or that the real off-chain server was requested for a specific data. There are some ways to check that, i.e. `TLSNotary`. It's unclear if we should supported it and how.

### Assumptions

There are no specific assumptions made for this document.

### Risks

There are no currently observable risks for Iroha 2 core system if we proceed with the provided solution.

## Additional Information

[Mastering Ethereum. Oracles](https://github.com/ethereumbook/ethereumbook/blob/develop/11oracles.asciidoc)

[Polkadot docs. Oracles](https://wiki.polkadot.network/docs/build-oracle)