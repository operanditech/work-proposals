# Streamer nodeos plugin

> Work Proposal Document. Version 1.0-rc1

This work proposal will explain the details of the project for building a `streamer_plugin` for `nodeos`. Since this project is already under development, it will describe the work already done, and detail the roadmap for the work ahead, goals, motivations, features, and use cases.

## About the authors

Mario Silva and Andres Berrios are both software engineers, each with over a decade of experience in designing, building, and scaling software systems and infrastructure for startups and corporations. Together they have started Operandi, a blockchain consultancy firm that aims to contribute as much value as possible to the EOSIO software community through open source projects and initiatives, as well as offering professional consultancy and software development services.

They have already made many contributions to the community, including various open source projects such as eos-sharp, scatter-sharp, the MVP of the Tungsten bond management dapp, various example smart contracts, tools for managing local development testnets and nodeos processes in remote servers, as well as actively helping to educate the community in [eosio.stackexchange.com](https://eosio.stackexchange.com/) and the Telegram channels, giving presentations and workshops at meetup events, and more.

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

### Communication

To send messages to the receiver process, the plugin will use the widely supported and popular [ZeroMQ library](http://zeromq.org/). It supports multiple different underlying transports, such as TCP or IPC, and it takes care of most of the potential issues when managing communications. It's well supported on almost all languages and platforms, and provides a high-speed asynchronous I/O engine. It sends the messages through the underlying connection using a background thread, which allows it not to take CPU cycles away from the main blockchain processing thread of `nodeos`. It also provides smart communication patterns that allow for completely reliable message passing (no lost messages).

### Utilities

We will also provide a Node.js library with plug-and-play implementations for many useful functionalities to build your receiver process very easily. The initially supported ones will be:

- Storing transaction and action history into MongoDB - an offloaded equivalent to `mongodb_plugin`.
- Mirroring smart contract state database tables into MongoDB to provide more powerful querying capabilities, and high scalability read performance.
- A horizontally scalable HTTP API server that provides access to the historical and state data stored in MongoDB.
- A generic data store adaptor implementation that will allow you to replace MongoDB for any other data store of your choice (such as Postgres, MySQL, or any other database system).
- A horizontally scalable WebSocket server to feed real time updates to your dapp frontend.
- An `ActionReader` for [demux-js](https://github.com/EOSIO/demux-js) that also provides the state DB operations associated to each action.

These utilities will be properly documented and we will provide examples of how to use them.

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

The Streamer plugin and receiver library will provide the necessary functionality to enable these and many more use cases:

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
- Running off-chain or on-chain side-effects for certain monitored actions or state DB conditions.
  - Sending an email when a user makes a purchase in your dapp.
  - Sending a blockchain transaction in response to a user action automatically.
  - Calling an external API when a user accumulates a certain amount of points.

## Alternatives

The existing alternatives provide only limited functionality, and in hard-to-scale ways:

- **`history_plugin`**: This plugin is already deprecated and not supported anymore. It was not reliable and not scalable, and it kept only account history in a chainbase database within the nodeos shared memory.
- **`mongodb_plugin`**: As many node operators report, this plugin has proven unreliable and unstable. It also doesn't handle the blockchain forks properly. Big part of the issue, as explained in the [motivation](#motivation) section, is that it is adding too much work on top of what `nodeos` already needs to do. This work involves managing the connection to the MongoDB server, inserting the data into it, handling connection and database errors, tracking the blockchain data that needs to be inserter, processing and structuring it appropriately, and serializing it for the MongoDB server to understand it. It also obviously only support MongoDB as a data store, and it tracks only historical data, meaning there is no support for state DB tables.
- **`state_history_plugin`**: This new plugin from Block.one improves the functionality of previous plugins by providing external processes with a way to track chain state (tables), transaction history, and blocks. It however has the following drawbacks:
  - It stores all the data in additional files managed by `nodeos`, and exposes this data only through HTTP endpoints that need to be handled by `nodeos` itself (see [scaling difficulties](#scaling-difficulties)).
  - Equally, it provides a WebSocket interface to receive updates when a block is added to the chain, but exposing this publicly is not recommended since there are no abuse mitigation.
  - It can only provide the state DB operations that happened in a block, and not separated by transactions or actions.

## Work

A lot of work has been done so far, and some amount is left to do. We need to ensure that this software will be stable and reliable, and ready to be used in production settings.

### Previous work

The current status of the development of this plugin (and the provided receiver) is the result of several weeks of work and research, and several iterations on the approach and technology solution, thanks to testing and discussions with dapp developers as well as with Block.one engineers about the needs, philosophy, constraints, goals, and use cases that should be supported.

> **We would like to donate the work done so far to the EOSIO community.** We want to see the amazing things that people will build with these tools.

### Remaining work

The majority of the remaining work is related to the following points (with estimated hours of work required):

#### Polishing the code and functionalities: 20 hours

The current proof-of-concept version needs to be polished and some functionalities are yet to be implemented, such as:

- Full reliability of the ZeroMQ communication model.
- Standardization of the message formats and structure.
- Potentially also grouping undo state DB operations (when there's a fork) per action if we find a valid use case for that.

#### Functional testing and stress testing: 40 hours

To ensure everything is ready for production deployments, we need to test all the functionalities and and make sure to handle edge cases. We will also test a full replay of the EOS main net blockchain and measure the performance both during the replay and after the node starts tracking the live blockchain activity.

#### Optimizing performance: 20 hours

We need to ensure the resource usage is kept to a minimum, and delivery of messages to the receiver is fast and efficient. We will use the results of the stress tests to identify potential bottlenecks in the system and apply the necessary optimizations. After the optimizations are applied, we will measure performance once more and publish the results in a report.

#### Implementing additional features: 40 hours

Once the plugin and receiver are thoroughly tested and the fundamental functionalities are in place, we will implement the additional features in the receiver library. These will include [utilities](#utilities) for covering many of the [example use cases](#use-cases).

#### Documentation: 16 hours

We will provide documentation for each component of the system, as well as code samples to explain their usage.

## Timeline

We expect the remaining of the work to be done in around one month. Frequent updates about the progress will be published so the community can follow the development and testing.

## Costs

The cost estimation is based on the lower end of the range for the market rate for this type of development work, which would be 80 USD per hour. At the current market price (1.8 USD), this would be equivalent to around 45 EOS per hour.

The hosting costs for the infrastructure necessary to run the tests is estimated to be 2000 USD, or around 1111 EOS.

The total work estimate is of 136 hours, which translates to 6120 EOS. With infrastructure hosting costs, the total budget would be 7231 EOS.

## Financing

We would like to offer organizations and members of the community the opportunity to jointly sponsor the development of this suite of functionalities. Please contact us through Telegram by direct message to [@andresberrios](https://t.me/andresberrios) or through email at [hello@operandi.io](mailto:hello@operandi.io) if you would like to help finance this work proposal. Please share this document with any parties that you think would be interested in the described functionalities and would like to provide feedback, suggestions, opinions, or funding.

### Fund management

We are evaluating the following options for managing the sponsorship funds:

- Receiving the sponsorship funds into the company's EOS account (`operanditech`).
- Setting up a dedicated EOS account controlled by a multi-signature configuration of the set of sponsors.
- Creating a smart contract (with a web frontend) to allow the community to contribute funds and publish their own work proposals, providing escrow functionality.

Since this is not a big ICO or anything like that, but an open source project for the community with a limited budget, we think that the first option could be enough this time, and that way we can gain insight into the process of managing this kind of sponsorships so we can eventually implement the more ambitious third option.

### What sponsors can expect

- Prominent credit mentions in all the project's communications.
- Ongoing support and assistance in setting up their usage of the project's software.
- Priority in the development of specific features they are interested in.
- Potentially the addition of the [utilities](#utilities) necessary for a use case they are interested in.
- Discounts in custom integration services of the software with their dapps or systems, consultations, and infrastructure setup.
- Please contact us with ideas of what else you would be interested to receive as a sponsor to the project.
