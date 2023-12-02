---
sidebar_position: 4
---
# Closer Look at Snapshots

Pooler is a Uniswap V2 specific implementation in the Powerloom Protocol, designed to capture, process, and aggregate blockchain data. This documentation provides an in-depth look at how Pooler operates and how developers can utilize it for building data-rich applications.

## Data Points Overview

Pooler focuses on several key data points:

- **Total Value Locked (TVL):** The cumulative value locked in Uniswap V2.
- **Trade Volume, Liquidity Reserves, and Fees Earned:** These are categorized by:
  - Pair Contracts
  - Individual Tokens within Pair Contracts
- **Time Aggregation:** Data is aggregated over 24-hour and 7-day periods.
- **Transaction Types:** Including Swap, Mint, and Burn events.


## Base Snapshots

:::info
Before you dive into this section, please make sure you take a look into the [Snapshot Generation Section](docs/protocol/specifications/snapshotter/snapshot-build#base-snapshots)
:::

Snapshotter node has several interfaces defined to handle the heavy lifting so that you can focus on just writing computes modules. For example, `TradeVolumeProcessor`, located in the **Snapshotter-computes** [`snapshotter-computer/trade_volume.py`](https://github.com/Powerloom/snapshotter-computes/blob/eth_uniswapv2/trade_volume.py) is one of the base computes for Pooler. This class uses the `GenericProcessorSnapshot` structure found in [`snapshotter/utils/callback_helpers.py`](https://github.com/Powerloom/pooler/blob/main/snapshotter/utils/callback_helpers.py).

Any compute for base snapshots basically needs to implement the `compute` function.

```python reference
https://github.com/Powerloom/snapshotter-computes/blob/74b2eaa452bfac8c0e4e0a7ed74a4d2748e9c224/trade_volume.py#L23-L28
```

The `compute` function is the main part where we create and process snapshots. It uses these inputs:
- `epoch`: Epoch details for which the snapshot is being generated.
- `redis`: Redis connection object.
- `rpc_helper`: RPC Helper object.

`epoch` is `PowerloomSnapshotProcessMessage` object which contains the following information:
```python reference
https://github.com/PowerLoom/pooler/blob/main/snapshotter/utils/models/message_models.py#L46-L50
```

The `TradeVolumeProcessor` collects and stores information about trades that happen within a specific range of blocks in the blockchain, known as the epoch. This range is defined by the lowest block number (`min_chain_height`) and the highest block number (`max_chain_height`) in that epoch.

The format of the output data can vary based on what you need it for. However, the return type must always be a [`pydantic`](https://pypi.org/project/pydantic/) model, it helps organize and define the data structure clearly.


:::info
Pydantic Model is a Python Library that helps data validation and parsing, by using Python type annotations.
:::

## Aggregates Snapshots

-  As we explored in the previous section, the  `TradeVolumeProcessor`  logic takes care of capturing a snapshot of information regarding Uniswap v2 trades between the block heights of  `min_chain_height`  and  `max_chain_height`.
    
-   The epoch size as described in the prior section on  [epoch generation](/docs/protocol/specifications/epoch)  can be considered to be constant for this specific implementation of the Uniswap v2 use case on Powerloom Protocol, and by extension, the time duration captured within the epoch.
    
-   The finalized state and data CID corresponding to each epoch can be accessed on the smart contract on the anchor chain that holds the protocol state. The corresponding helpers for that can be found in  `get_project_epoch_snapshot()`  in  [`pooler/snapshotter/utils/data_utils.py`](hhttps://github.com/Powerloom/pooler/blob/main/snapshotter/utils/data_utils.py)

```python reference

https://github.com/Powerloom/pooler/blob/fc08cdd951166ab0cea669d233cd28d0639f628d/snapshotter/utils/data_utils.py#L273-L295

```

To figure out the end point (or tail) for a 24-hour period of snapshots and trade data, starting from a given epoch ID (the beginning or head of this time span), we use a specific formula.

```
time_in_seconds = 86400
tail_epoch_id = current_epoch_id - int(time_in_seconds / (source_chain_epoch_size * source_chain_block_time))
```

```python reference 

https://github.com/Powerloom/pooler/blob/fc08cdd951166ab0cea669d233cd28d0639f628d/snapshotter/utils/data_utils.py#L507-L546
```

The worker class for such aggregation is defined in  `config/aggregator.json`  in the following manner:

```json reference 
https://github.com/Powerloom/snapshotter-configs/blob/ae77941311155a9126205af08735c3dfa5d72ac2/aggregator.example.json#L3-L10

```

`AggregateTradeVolumeProcessor`, located in the **Snapshotter-computes** [`snapshotter-computer/aggregate/single_uniswap_trade_volume_24h.py`](https://github.com/Powerloom/snapshotter-computes/blob/eth_uniswapv2/aggregate/single_uniswap_trade_volume_24h.py) is one of the aggregate computes for Pooler. This class uses the `GenericProcessorAggregate` structure found in [`snapshotter/utils/callback_helpers.py`](https://github.com/Powerloom/pooler/blob/main/snapshotter/utils/callback_helpers.py).

Any compute for base snapshots basically needs to implement the `compute` function.

```python reference
https://github.com/PowerLoom/snapshotter-computes/blob/74b2eaa452bfac8c0e4e0a7ed74a4d2748e9c224/aggregate/single_uniswap_trade_volume_24h.py#L58-L66
```

The `compute` function is the main part where we create and process snapshots. It uses these inputs:
- `msg_obj`: Underlying message object that contains the details of the aggregate snapshot composition.
- `redis`: Redis connection object.
- `rpc_helper`: RPC Helper object.
- `anchor_rpc_helper`: RPC Helper object for the anchor chain.
- `ipfs_reader`: IPFS Client object.
- `protocol_state_contract`: Protocol state contract Web3 object.
- `project_id`: Project ID for which the aggregate snapshot is being generated.
  
`msg_object` is either of type `PowerloomSnapshotSubmittedMessage` or `PowerloomCalculateAggregateMessage` for even more complex aggregats.

PowerloomSnapshotSubmittedMessage contains the following information:
```python reference
https://github.com/PowerLoom/pooler/blob/main/snapshotter/utils/models/message_models.py#L46-L50
```

PowerloomCalculateAggregateMessage contains the following information:
```python reference
https://github.com/PowerLoom/pooler/blob/main/snapshotter/utils/models/message_models.py#L90-L93
```

Detailed implementation for Pooler computes can be found in the [snapshotter-computes](https://github.com/PowerLoom/snapshotter-computes/tree/eth_uniswapv2) repository.

## Extending Pooler for more Datapoints. 

Implementing custom data points on top of existing pooler is easy. We have a section in [Build with Powerloom](/docs/build-with-powerloom) which covers this in detail. 