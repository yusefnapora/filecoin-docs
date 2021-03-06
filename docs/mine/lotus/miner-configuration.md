---
title: 'Lotus Miner: configuration reference'
description: 'This guide covers the Lotus Miner configuration files, detailing the meaning of the options contained in them.'
breadcrumb: 'Configuration reference'
---

# {{ $frontmatter.title }}

{{ $frontmatter.description }}

The Lotus Miner configutation is created after the [initialization step](miner-setup.md) during setup and placed in `~/.lotusminer/config.toml` or `$LOTUS_MINER_PATH/config.toml` when defined.

The _default configuration_ has all the items commented, so in order to customize one of the items the leading `# ` need to be removed.

::: tip
For any configuration changes to take effect, the miner must be [restarted](miner-lifecycle.md).
:::

[[TOC]]

## API section

The API section controls the settings of the [miner API](../../reference/lotus-api.md):

```toml
[API]
  # Binding address for the miner API
  ListenAddress = "/ip4/127.0.0.1/tcp/2345/http"
  # This should be set to the miner API address as seen externally
  RemoteListenAddress = "127.0.0.1:2345"
  # General network timeout value
  Timeout = "30s"
```

As you see, the listen address is bound to the local loopback interface by default. If you need to open access to the miner API to other machines, you will need to set this to the IP address of the network interface you want to use, or to `0.0.0.0` (which means "all interfaces"). Note that API access is protected by [JWT tokens](../../build/lotus/api-tokens.md), but it should not be open to the internet.

Configure `RemoteListenAddress` to the value that a different node would have to use to reach this API. Usually it is the miner's IP address and API port, but depending on your setup (proxies, public IPs etc.), it might be a different IP.

## Libp2p section

This section configures the miner's embedded Libp2p node. As noted in the [setup instructions](miner-setup.md#connectivity-to-the-miner), it is very important to adjust this section with the miner's public IP and a fixed port:

```toml
[Libp2p]
  # Binding address for the libp2p host. 0 means random port.
  # Type: Array of multiaddress strings
  ListenAddresses = ["/ip4/0.0.0.0/tcp/0", "/ip6/::/tcp/0"]
  # Insert any addresses you want to explicitally
  # announce to other peers here. Otherwise, they are
  # guessed.
  # Type: Array of multiaddress strings
  AnnounceAddresses = []
  # Insert any addresses to avoid announcing here.
  # Type: Array of multiaddress strings
  NoAnnounceAddresses = []
  # Connection manager settings, decrease if your
  # machine is overwhelmed by connections.
  ConnMgrLow = 150
  ConnMgrHigh = 180
  ConnMgrGrace = "20s"
```

The connection manager will start to prune the existing connections if the number of established crosses the value set for `ConnMgrHigh` until it hits the value set for `ConnMgrLow`. Connections younger than `ConnMgrGrace` will be kept.

## Pubsub section

This section controls some Pubsub settings. Pubsub is used to distribute messages in the network:

```toml
[Pubsub]
  # Usually you will not run a pubsub bootstrapping node, so leave this as false
  Bootstrapper = false
  # FIXME
  RemoteTracer = ""
  # DirectPeers specifies peers with direct peering agreements. These peers are
  # connected outside of the mesh, with all (valid) message unconditionally 
  # forwarded to them. The router will maintain open connections to these peers.
  # Note that the peering agreement should be reciprocal with direct peers
  # symmetrically configured at both ends.
  # Type: Array of multiaddress peerinfo strings, must include peerid (/p2p/12D3K...)
  DirectPeers = []
```

## Dealmaking section

This section controls parameters for making storage and retrieval deals:

```toml
[Dealmaking]
  # When enabled, the miner can accept online deals
  ConsiderOnlineStorageDeals = true
  # When enabled, the miner can accept offline deals
  ConsiderOfflineStorageDeals = true
  # When enabled, the miner can accept retrieval deals
  ConsiderOnlineRetrievalDeals = true
  # When enabled, the miner can accept offline retrieval deals
  ConsiderOfflineRetrievalDeals = true
  # When enabled, the miner can accept verified deals
  ConsiderVerifiedStorageDeals = true
  # When enabled, the miner can accept unverified deals
  ConsiderUnverifiedStorageDeals = true
  # A list made of Data CIDs to reject when making deals
  PieceCidBlocklist = []
  # Maximum expected amount of time getting the deal into a sealed sector will take
  # This includes the time the deal will need to get transfered and published
  # before being assigned to a sector
  # for more see below
  ExpectedSealDuration = "24h0m0s"
  # When a deal is ready to publish, the amount of time to wait for more
  # deals to be ready to publish, before publishing them all as a batch
  PublishMsgPeriod = "1h0m0s"
  # The maximum number of deals to include in a single publish deals message
  MaxDealsPerPublishMsg = 8

  # A command used for fine-grained evaluation of storage deals (see below)
  Filter = "/absolute/path/to/storage_filter_program"

  # A command used for fine-grained evaluation of retrieval deals (see below)
  RetrievalFilter = "/absolute/path/to/retrieval_filter_program"
```

`ExpectedSealDuration` is an estimate of how long sealing will take, and is used to reject deals whose start epoch might be earlier than the expected completion of sealing. It can be estimated by [benchmarking](benchmarks.md) or by [pledging a sector](sector-pledging.md).

:::warning
The final value of `ExpectedSealDuration` should equal `(TIME_TO_SEAL_A_SECTOR + WaitDealsDelay) * 1.5`. This equation ensures that the miner does not commit to having the sector sealed too soon.
:::

### Publishing several deals in one message

The `PublishStorageDeals` message can publish many deals in a single message.
When a deal is ready to be published, lotus will wait up to `PublishMsgPeriod`
for other deals to be ready before sending the `PublishStorageDeals` message.

However once `MaxDealsPerPublishMsg` are ready, lotus will immediately publish all the deals.

For example if `PublishMsgPeriod` is 1 hour:

- At 1:00pm Deal 1 is ready to publish.
  Lotus will wait until 2:00pm for other deals to be ready before sending `PublishStorageDeals`
- At 1:30pm Deal 2 is ready to publish
- At 1:45pm Deal 3 is ready to publish
- At 2:00pm lotus publishes Deals 1, 2 and 3 in a single `PublishStorageDeals` message.

If `MaxDealsPerPublishMsg` is 2, then in the above example when deal 2 is ready to be published at 1:30,
lotus would immediately publish Deals 1 & 2 in a single `PublishStorageDeals` message.
Deal 3 would be published in a subsequent `PublishStorageDeals` message.

> Note: If any of the deal in the `PublishStorageDeals` fails validation upon executation, i.e: start epoch has passed, all deals will fail to be published.

## Using filters for fine-grained storage and retrieval deal acceptance

Your use-case might demand very precise and dynamic control over a combination of deal parameters.

Lotus provides two IPC hooks allowing you to name a command to execute for every deal before the miner accepts it:

- `Filter` for storage deals.
- `RetrievalFilter` for retrieval deals.

The executed command receives a JSON representation of the deal parameters on standard input, and upon completion its exit code is interpreted as:

- `0`: success, proceed with the deal.
- `non-0`: failure, reject the deal.

The most trivial filter rejecting any retrieval deal would be something like:
`RetrievalFilter = "/bin/false"`. `/bin/false` is binary that immediately exits with a code of `1`.

[This Perl script](https://gist.github.com/ribasushi/53b7383aeb6e6f9b030210f4d64351d5/9bd6e898f94d20b50e7c7586dc8b8f3a45dab07c#file-dealfilter-pl) lets the miner deny specific clients and only accept deals that are set to start relatively soon.

You can also use a third party content policy framework like `bitscreen` by Murmuration Labs:

```sh
# grab filter program
go get -u -v github.com/Murmuration-Labs/bitscreen

# add it to both filters
Filter = "/path/to/go/bin/bitscreen"
RetrievalFilter = "/path/to/go/bin/bitscreen"
```

## Sealing section

This section controls some of the behaviour around sector sealing:

```toml
[Sealing]
  # Upper bound on how many sectors can be waiting for more deals to be packed in it before it begins sealing at any given time.
  # If the miner is accepting multiple deals in parallel, up to MaxWaitDealsSectors of new sectors will be created.
  # If more than MaxWaitDealsSectors deals are accepted in parallel, only MaxWaitDealsSectors deals will be processed in parallel
  # Note that setting this number too high in relation to deal ingestion rate may result in poor sactor packing efficiency
  MaxWaitDealsSectors = 2
  # Upper bound on how many sectors can be sealing at the same time when creating new CC sectors (0 = unlimited)
  MaxSealingSectors = 0
  # Upper bound on how many sectors can be sealing at the same time when creating new sectors with deals (0 = unlimited)
  MaxSealingSectorsForDeals = 0
  # Period of time that a newly created sector will wait for more deals to be packed in to before it starts to seal.
  # Sectors which afe fully filled will start sealing immediately
  WaitDealsDelay = "6h0m0s"
  # Whether to keep unsealed copies of deal data regardless of whether the client requested that. This lets the miner
  # avoid the relatively high cost of unsealing the data later, at the cost of more storage space
  AlwaysKeepUnsealedCopy = true
```

## Storage section

The storage sector controls whether the miner can perform certain sealing actions. Depending on the setup and the use of additional [seal workers](seal-workers.md), you may want to modify some of the options.

```toml
[Storage]
  # Upper bound on how many sectors can be fetched in parallel by the storage system at a time
  ParallelFetchLimit = 10
  # Sealing steps that the miner can perform itself. Sometimes we have a dedicated seal worker to do them and do not want the miner to commit any resources for this.
  AllowAddPiece = true
  AllowPreCommit1 = true
  AllowPreCommit2 = true
  AllowCommit = true
  AllowUnseal = true
```

## Fees section

The fees section allows to set limits to the gas consumption for the different messages that are submitted to the chain by the miner:

```toml
[Fees]
  # Maximum fees to pay
  MaxPreCommitGasFee = "0.025 FIL"
  MaxCommitGasFee = "0.05 FIL"
  MaxTerminateGasFee = "0.5 FIL"
  # This is a high-value operation, so the default fee is higher.
  MaxWindowPoStGasFee = "5 FIL"
  MaxPublishDealsFee = "0.05 FIL"
  MaxMarketBalanceAddFee = "0.007 FIL"
```

Depending on the network congestion the base fee for a transaction may grow or decrease. Your gas limits will have to be at any case larger than the base fee for the messages to be included. A very large max fee can however result in the quick burning of funds when the base fees are very high, as the miner automatically submits messages during normal operation, so be careful about this. It is also necessary to have more funds available then any max fee set, even if the actual fee will be far less then the max fee set. \*MaxWindowPostGasFee is currently reduced, but the setting used should remain fairly high, eg. 2FIL.

## Addresses section

The addresses section allows users to specify additional addresses to send messages from. This helps mitigate head-of-line blocking for important messages when network fees are high. For more details see the [Miner addresses](miner-addresses.md) section.

```toml
[Addresses]
  # Addresses to send PreCommit messages from
  PreCommitControl = []
  # Addresses to send Commit messages from
  CommitControl = []
  # Disable the use of the owner address for messages which are sent automatically.
  # This is useful when the owner address is an offline/hardware key
  DisableOwnerFallback = false
  # Disable the use of the worker address for messages for which it's possible to use other control addresses
  DisableWorkerFallback = false
```
