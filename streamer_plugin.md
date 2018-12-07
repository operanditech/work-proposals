# Streamer nodeos plugin

This work proposal will explain the details of the project for building a `streamer_plugin` for `nodeos`. Since this project is already being developed, it will describe the work already done, and detail the roadmap for the work ahead, goals, motivations, features, and use cases.

## Motivation

The `nodeos` process already needs to do quite a bit of work, including validating blocks, network connections management, sending and receiving data to other nodes, processing transactions, managing isolated instances of WASM virtual machines, maintaining data files for blocks history, contract state databases, fork database, and many other tasks.

Many plugins have been developed for `nodeos` to extend this set of tasks even further. Some of them relate to interacting with the `nodeos` process, exposing HTTP endpoints to handle requests, querying the blockchain data, or storing the transaction history data in some other data store such as MongoDB or Postgres.
These plugins generally put a higher load on the `nodeos` process and make use of more resources, including CPU time, RAM memory, and additional storage requirements.

In particular, there is a need among developers and users to be able to more efficiently access the blockchain history and the state database of the smart contracts, while node operators need a way to provide these functionalities in a scalable way and without incurring in excessive costs.

### Scaling difficulties

- Dapp developers and users want features provided by plugins (such as HTTP endpoints for querying historical data, or WebSocket connections to get live updates).
- Serving more users means higher resource usage (serving HTTP requests, managing WebSocket connections and sending data to each, etc).
- Running a `nodeos` instance already requires considerable resources just to run the blockchain. We'll call these resources "fixed", since they don't depend on the number of users that require the plugin features.
- Plugin features require more or fewer resources depending on the number of users, so we will call these resources "variable".
- Scaling vertically (using a more powerful machine for the node) is considerably more expensive than scaling horizontally (spreading the load among multiple, less powerful machines).
- Adding plugins with the desired functionalities to a node creates an economic inefficiency (waste), since the benefit of scaling horizontally (using less powerful machines) is eroded by the fact that each machine will need to run a full node.
- Each full node would duplicate the "fixed" resource requirements unnecessarily, when we are actually after the features associated to the "variable" resource requirements.
- The node operators are forced to choose between scaling vertically and incurring in higher costs, or scaling horizontally and using most of the resources to do duplicate work unnecessarily, thus incurring in similarly higher costs.
- It's hard to tell which option is less inconvenient, and both are wasteful.

## Approach

In order to separate the fixed and variable resource usages, it is necessary to let `nodeos` do what it's meant to be doing (running the blockchain) and separate the rest of the functionalities into other processes that can run on different machines and be scaled horizontally and independently.

The `streamer_plugin` will focus exclusively on tracking relevant blockchain events and their associated data, and streaming them outside of `nodeos` for any receiver to consume and process however they see fit. This would reduce the load on `nodeos` while providing the receiver with the most advanced and comprehensive set of data about the blockchain activity, in real time.

We will also provide a Node.js library with plug-and-play implementations for many useful functionalities to build your receiver process very easily. The initially supported ones will be:

- Storing transaction and action history into MongoDB - an offloaded equivalent to `mongodb_plugin`.
- Mirroring smart contract state database tables into MongoDB to provide more powerful querying capabilities, and high scalability read performance.
- A horizontally scalable HTTP API server that provides access to the historical and state data stored in MongoDB.
- A generic data store adaptor implementation that will allow you to replace MongoDB for any other data store of your choice (such as Postgres, MySQL, or any other database system).
- A horizontally scalable WebSocket server to feed real time updates to your dapp frontend.
- An `ActionReader` for [demux-js](https://github.com/EOSIO/demux-js) that also provides the state DB operations associated to each action.

## Desired features

- Provide the following event messages:
  - Transaction
    - Since transactions are the atomic unit of work, we use them as the basic event type to ensure we are giving a consistent view of the state of the blockchain.
    - Data:
      - Transaction details
      - List of actions executed within the transaction (including inline actions)
        - Action details
        - List of state DB operations executed within the action
  - Block
    - We provide a block event that simply marks the end of each applied block.
    - Data:
      - Block ID
      - Block height (block number)
  - Pop block
    - This event is sent when a reversible block is removed from the head of the chain due to a fork.
    - Data:
      - Block ID
      - Block height (block number)
      - State DB undo operations (allows receiver to roll back its state DB mirror to the consistent state)
  - Irreversible block
    - Marks a previously applied block as irreversible.
    - Data:
      - Block ID
      - Block height (block number)
- Reliability
  - Send messages through a connection and communication scheme that will ensure that no messages are lost.
- Filtering
  - Allow the node operator to configure what to track and receive messages for:
    - Event types
    - Accounts and contracts
    - Actions
    - State DB tables and scopes

## Use cases

The Streamer plugin and receiver library will provide the necessary functionality to support these and many more use cases:

- Efficiently querying the full history of the blockchain.
  - Getting the actions associated to an account or contract.
  - Getting all known token balances for a given account.
  - Getting all the token transfers for a given account.
  - Getting all the interactions a given account has had with your contract or dapp.
- Efficient and complex querying of your contract's state DB data.
  - Track as many contract accounts as you like, or all of them.
  - No need to duplicate your smart contract logic in order to insert the right data into your data store.
- Aggregating values from action parameters or state DB rows.
  - To derive statistical information.
  - To calculate total transfer volumes.
  - To track all public keys that have ever been associated to an account.
  - To track all accounts that have ever been linked to a public key.
  - To compute advanced producer voting statistics.
  - Any other thing you can think of.
- Providing real time updates to your frontend.
  - To enable multiplayer games.
  - To update the price in your exchange or trading bot.
  - To show notifications in your dapp.
  - To update any values in your user interface.
  - Make your block explorer efficiently update in real time.

## Alternatives

The existing alternatives provide only limited functionality, and in hard-to-scale ways:

- **`history_plugin`**: This plugin is already deprecated and not supported anymore. It was not reliable and not scalable, and it kept only account history in a chainbase database within the nodeos shared memory.
- **`mongodb_plugin`**: As many node operators report, this plugin has proven unreliable and unstable. It also doesn't handle the blockchain forks properly. Big part of the issue, as explained in the [motivation](#motivation) section, is that it is adding too much work on top of what `nodeos` already needs to do. This work involves managing the connection to the MongoDB server, inserting the data into it, handling connection and database errors, tracking the blockchain data that needs to be inserter, processing and structuring it appropriately, and serializing it for the MongoDB server to understand it. It also obviously only support MongoDB as a data store, and it tracks only historical data, meaning there is no support for state DB tables.
- **`state_history_plugin`**: This new plugin from Block.one improves the functionality of previous plugins by providing external processes with a way to track chain state (tables), transaction history, and blocks. It however has the following drawbacks:
  - It stores all the data in additional files managed by `nodeos`, and exposes this data only through HTTP endpoints that need to be handled by `nodeos` itself (see [scaling difficulties](#scaling-difficulties)).
  - Equally, it provides a WebSocket interface to receive updates when a block is added to the chain, but exposing this publicly is not recommended since there are no abuse mitigation.
  - It can only provide the state DB operations that happened in a block, and not separated by transactions or actions.

## Work

### Previous work

### Remaining work

<List the features that still need to be implemented>
<Detail what the acceptance criteria is for each feature, and the deliverables, including test results, benchmarks, reports, usage code samples, documentation, etc>

## Costs

<Details of the costs in terms of development hours and testing infrastructure>

## Financing

### What sponsors can expect

<List the things they can expect, such as some ongoing support and assistance in setting up their usage of the software, crediting of sponsors in various marketing places, etc>

### Sponsorship plan

<Describe the sponsorship proposal>
<Maybe propose a system in which we need the total sum to be covered by a group of sponsors, and the more sponsors participate, the less each one needs to pay>
