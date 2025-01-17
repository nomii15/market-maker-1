# Zigzag Market Maker

This is the reference market maker for Zigzag zksync markets. It works on both Rinkeby and Mainnet.

This market maker uses existing price feeds to set bids and asks for a market. For now, in order to provide liquidity for a market, there must be an existing market with **greater** liquidity listed on Cryptowatch, via either Uniswap or some other centralized exchange. It is crucial that the oracle market have more liquidity than the Zigzag one so that you are not prone to oracle attacks. 

Soon we will add the ability to run standalone markets and this will not be an issue. 

## Requirements

* Activated zkSync account
* Ethereum private key of that account
* Funds in that account corresponding to the pairs you want to market make
* [Cryptowatch API key](https://cryptowat.ch/account/api-access) (free for limited time)
* [Node.js](https://nodejs.org/en/download/)
* Node.js 16 works on macOS, Windows and Linux (17 seems not)
* Optional: VPS when you have high ping running the bot

## Setup

Copy the `config.json.EXAMPLE` file to `config.json` to get started. 

Set your `eth_privkey` to be able to relay transactions. The ETH address with that private key should be loaded up with adequate funds for market making.

Currently zkSync needs around 5 seconds to process a single swap and generate the receipt. So  there is a upper limit of 12 swaps per wallet per minute. To circumvent this, there is also the option to use the `eth_privkeys` array. Here you can add any number of private keys. Each should be loaded up with adequate funds for market making. The founds will be handled separately, therefor each additional wallet has the opportunity to process (at least) 12 more swaps per minute.

###### External price feed
You have two options to get a price feed for your market maker. It is not recommended to use different price feeds sources as primary and secondary feed, as those might have some difference and could stop the market maker.

**Warning:** Make sure your price feed is close to the price you see on zigzag. **Otherwise, your mm can lose money!**

**Cryptowatch**
You need a Cryptowatch API key to use the market maker. Once you obtain one, you can set the `cryptowatchApiKey` field in `config.json`. And set it to your public key.

You can use [this link](https://api.cryptowat.ch/markets) to download a JSON with all available market endpoints. Add those to you pair config as "cryptowatch:<id>".

Example:
```
"ETH-USDC": {
    "mode": "pricefeed",
    "side": "d",
    "initPrice": null,
    "priceFeedPrimary": "cryptowatch:6631",
    "priceFeedSecondary": "cryptowatch:588",
    ....
}
```

**Chainlink**
With chainlink you have access to price oracles via blockchain. The requests are read calls to a smart contract. The public ethers provider might be too slow for a higher number of pairs or at times of high demand. Therefore, it might be needed to have access to an Infura account (100000 Requests/Day for free). There you can get an endpoint for your market maker (like https://mainnet.infura.io/v3/...), You can add this with the `infuraUrl` field in `config.json`, like this:
```
....
"infuraUrl": "https://mainnet.infura.io/v3/xxxxxxxx",
"zigzagChainId": 1,
"zigzagWsUrl": "wss://zigzag-exchange.herokuapp.com",
....
```
You can get the available market contracts [here.](https://docs.chain.link/docs/ethereum-addresses/)Add those to you pair config as "chainlink:<address>", like this:
```
"ETH-USDC": {
    "mode": "pricefeed",
    "side": "d",
    "initPrice": null,
    "priceFeedPrimary": "chainlink:0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419",
    "priceFeedSecondary": null,
    ....
}
```


To run the marketmaker:

```bash
node marketmaker.js
```

## Settings

You can add, remove, and configure pair settings in the `pairs` section. A pair setting looks like this:

```
"ETH-USDC": {
    "mode": "pricefeed",
    "side": "d",
    "initPrice": null,
    "priceFeedPrimary": "cryptowatch:6631",
    "priceFeedSecondary": "cryptowatch:588",
    "slippageRate": 1e-5,
    "maxSize": 100,
    "minSize": 0.0003,
    "minSpread": 0.0005,
    "active": true
}
```

There are 2 modes available with a 3rd on the way. 

* `pricefeed`: Follows an external price oracle and updates indicated bids and asks based on that. 
* `constant`: Sets an `initPrice` and market makes around that price. Can be combined with single-sided liquidity to simulate limit orders.
* `independent`: Under development. The price is set independent of a price feed. 

For all modes the `slippageRate`, `maxSize`, `minSize`, `minSpread`, and `active` settings are mandatory.

For `pricefeed` mode, the `priceFeedPrimary` is mandatory. 

For `independent` and `constant` mode, the `initPrice` is mandatory. 

The `side` setting can be toggled for single-sided liquidity. By default, the side setting is set to `d`, which stands for double-sided liquidity. To toggle single-sided liquidity, the value can be set to `b` or `s` for buy-side only or sell-side only.

The primary price feed is the price feed used to determine the bids and asks of the market maker. The secondary price feed is used to validate the first price feed and make sure the market isn't returning bad data. If the primary and secondary price feeds vary by more than 1%, the market maker will not fill orders. 

The slippage rate is the rate at which the spread increases as the base unit increases. For the example above, the spread goes up by 1e-5 for every 1 ETH in size added to an order. That's the equivalent of 0.1 bps / ETH in slippage. 

Orders coming in below the `minSpread` from the price feed will not be filled. The spread is calculated as a decimal value. 0.01 is 1%, and 0.0002 is 2 basis points (bps).

A market can be set inactive by flipping the active switch to `false`. 

## Pair Setting Examples 

Stable-Stable constant price:

```
"DAI-USDC": {
    "mode": "constant",
    "initPrice": 1,
    "slippageRate": 1e-9,
    "maxSize": 100000,
    "minSize": 1,
    "minSpread": 0.0003,
    "active": true
}
```

Single-sided accumulation:

```
"ETH-USDC": {
    "mode": "pricefeed",
    "side": "b",
    "priceFeedPrimary": "cryptowatch:6631",
    "priceFeedSecondary": "cryptowatch:588",
    "slippageRate": 1e-5,
    "maxSize": 100,
    "minSize": 0.0003,
    "minSpread": 0.0005,
    "active": true
}
```

Sell the rip:

```
"DYDX-USDC": {
    "mode": "constant",
    "side": "s",
    "initPrice": 20,
    "slippageRate": 1e-5,
    "maxSize": 1000,
    "minSize": 0.5,
    "minSpread": 0,
    "active": true
}
```

## Configuration Via Environment Variables

If your hosting service requires you to pass in configs via environment variables you can compress `config.json`:

```
cat config.json | tr -d ' ' | tr -d '\n'
```

and set it to the value of the `MM_CONFIG` environment variable to override the config file.

You can also override the private key in the config file with the `ETH_PRIVKEY` environment variable, and the cryptowatch API key with the `CRYPTOWATCH_API_KEY` environment variable, and the Infura provider url with `INFURA_URL`
