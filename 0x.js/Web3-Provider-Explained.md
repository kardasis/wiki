When instantiating a [new instance of the 0x.js library](https://0xproject.com/docs/0xjs#zeroEx), we require that you pass in a Web3 provider. Since there doesn't seem to be great documentation on what exactly a Web3 Provider is, we thought we'd fill the gap.

A provider can be any module or class instance that implements the `sendAsync` method (simply `send` in web3 V1.0 Beta). That's it. What this `sendAsync` method does is take JSON RPC payload requests and handles them properly.

The simplest example of a provider is:

```ts
const provider = new web3.providers.HttpProvider('http://localhost:8545')
```

This provider simply takes each JSON RPC payload it receives and sends it on to the Ethereum node running on port 8545.

### Examples of complex providers

Providers can have much more complex however, and [ProviderEngine](https://github.com/MetaMask/provider-engine) is a great tool for building providers that do all kinds of cool stuff.

In 0x Portal we use a custom built provider called [RedundantRPCSubprovider](https://github.com/0xProject/website/blob/development/ts/subproviders/redundant_rpc_subprovider.ts) that has a fallback to sending the payload to several Ethereum nodes until one successfully received it. During our token sale, the huge amount of traffic caused some of our nodes to fail and luckily each request would fallback to hitting Infura's nodes when ours were unresponsive.

Another great use case for custom providers is to support additional ways for your users to sign requests and send transactions. We want to add hardware wallet signing support to 0x Portal so we wrote a [LedgerWalletSubproviderFactory](https://github.com/0xProject/website/blob/development/ts/subproviders/ledger_wallet_subprovider_factory.ts) that would send all message and transaction signing requests to a Ledger Nano S attached to a users computer. All other requests would be sent to a backing Ethereum node.

One final example is the custom provider that allows 0x.js to play nicely with Infura shown in [this wiki article](https://0xproject.com/wiki#Infura-Setup-Guide). This uses the [ProviderEngine](https://github.com/MetaMask/provider-engine)'s [FilterSubprovider](https://github.com/MetaMask/provider-engine/blob/master/subproviders/filters.js) to filter logs locally rather then on the remote Ethereum node.

#### Updating the provider used by 0x.js

If at some point the provider used by your dApp changes (e.g if your user wants to use their Ledger Nano S to sign orders), it is important that you update the provider used by your 0x.js instance. You can do this by calling [setProviderAsync](https://0xproject.com/docs/0xjs#setProviderAsync).

```ts
await zeroEx.setProviderAsync(newProvider);
```
