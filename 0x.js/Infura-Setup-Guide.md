[Infura](https://infura.io) is a service that provides scalable Ethereum node infrastructure for dApps with significant traffic. Ethereum nodes were not built to act as server backends and have significant performance shortcomings when used in this way.

In order to optimize Ethereum nodes for a server-like environment, Infura has had to change/omit some of the expected behavior of an Ethereum node in key ways. This document will outline what these changes were and how you can get Infura to play nicely with 0x.js.

### Event watching

Infura has a set of Ethereum nodes hosted behind a non-sticky load balancer. Because of this, subsequent requests are not guaranteed to hit the same Ethereum node. When you call `zeroEx.exchange.subscribeAsync` (or any other 0x.js `subscribeAsync` call) to subscribe to events, a `eth_newFilter` request is sent to the backing Ethereum node, setting up a filter for the events you are interested in watching. Since this request would be routed to one of Infura's Ethereum nodes, if any subsequent `eth_getFilterLogs` request are routed to a different node, no events would be returned. Because of this non-determistic behavior, Infura currently disallows requests attempting to set up filters.

They do intend on adding support for stateful transactions and you can track their progress with [this issue](https://github.com/INFURA/infura/issues/10). The current workaround is to install a filter client-side by using [Provider Engine](https://github.com/MetaMask/provider-engine)'s `filterSubprovider`.

#### Provider engine

Provider engine works similarly to middleware in a server stack. What this means is that the providerEngine instance is made up for a stack of subProviders. When your dApp makes a JSON RPC request, the providerEngine instance hands the request to the top-most subProvider. This provider looks at the request and either decides to handle the request or to pass it on to the next-in-line subProvider. This continues until the request is handled or bottoms out after passing through each subProvider. Using this architecture, the RPC request `eth_newFilter` is caught by the `FilterSubprovider` before reaching the `RpcSubprovider`, and instead of attempting to install it on Infura, it installs a filter client-side. All other requests pass through the `FilterSubprovider` and are sent to Infura.


##### Example setup:

```javascript
import ProviderEngine = require('web3-provider-engine');
import FilterSubprovider = require('web3-provider-engine/subproviders/filters');
import RpcSubprovider = require('web3-provider-engine/subproviders/rpc');
import { ZeroEx } from "0x.js";


const KOVAN_ENDPOINT = 'https://kovan.infura.io';

const providerEngine = new ProviderEngine();
providerEngine.addProvider(new FilterSubprovider());
providerEngine.addProvider(new RpcSubprovider({rpcUrl: KOVAN_ENDPOINT}));
providerEngine.start();
const zeroEx = new ZeroEx(providerEngine);
```

##### Things to note

- Make sure your imports are in this exact order. Because of a [global object leak in Web3](https://github.com/ethereum/web3.js/issues/844) and Provider Engine using this same [global to detect it's running environment](https://github.com/MetaMask/provider-engine/blob/master/subproviders/rpc.js#L1), your code will hang if the imports are ordered differently.
- The order in which ProviderEngine subProviders are added matters.

#### Performance considerations

The most inefficient call implemented by Geth and Parity is `eth_getLogs`. This call is used when you call  an 0x.js `subscribeAsync` method with [SubscriptionOpts](https://0xproject.com/docs/0xjs#SubscriptionOpts).fromBlock set to a block in the past. Your dApp sends a slew of `eth_getLogs` calls to fetch all historical events. It is therefore important that the `fromBlock` param is set only as far back as necessary. Instead of setting it to the genesis block `0`, to retrieve all exchange contract events ever, you can set it to the block including the 0x genesis order:

- Mainnet: `4145578`
- Kovan: `3117574`

If you only care about events related to your dApp's users, even better performance can be reached by setting the `fromBlock` to the block number at which your service became available. Alternatively you could also limit historical trade data to the last two months, and only query the last two months of events.

This optimization is equally if not more important when hosting your own Ethereum node infrastructure.
