# 🛰 Hubble Public API

Hubble Public API is a TypeScript API (using Express) that serves public data of the Hubble Protocol.

## Table of contents

- [Usage](#usage)
  * [Metrics](#metrics)
  * [Metrics History](#metrics-history)
  * [Config](#config)
  * [IDL](#idl)
  * [Circulating Supply Value (HBB)](#circulating-supply-value-hbb)
  * [Circulating Supply (HBB)](#circulating-supply-hbb)
  * [API Version](#api-version)
  * [Maintenance mode](#maintenance-mode)
  * [Borrowing version](#borrowing-version)
  * [Loans](#loans)
  * [Staking](#staking)
  * [Stability](#stability)
  * [Strategies (Kamino)](#strategies-kamino)
  * [Whirlpools (Kamino)](#whirlpools-kamino)
  * [Prices](#prices)
  * [Borrowing Market State](#borrowing-market-state)
  * [Transactions](#transactions)
  * [Kamino Market Lending](#kamino-market-lending)

## Usage

### Metrics

#### Get `mainnet-beta` metrics of Hubble Protocol:

```http request
GET https://api.hubbleprotocol.io/metrics
```

You may also specify the environment (`mainnet-beta`[default],`devnet`,`localnet`,`testnet`) by using an `env` query parameter:

```http request
GET https://api.hubbleprotocol.io/metrics?env=devnet
```

### Metrics History

#### Get `mainnet-beta` metrics of Hubble Protocol:

```http request
GET https://api.hubbleprotocol.io/history
```

You may also specify the environment (`mainnet-beta`[default],`devnet`,`localnet`,`testnet`) by using an `env` query parameter:

```http request
GET https://api.hubbleprotocol.io/history?env=devnet
```

History endpoint will only return the historical data of the current year by default.
**Warning:** the maximum allowed period for the history endpoint is 1 year per request.

```http request
GET https://api.hubbleprotocol.io/history?year=2022
```

### Config

Runtime config specifies all of the public configuration used by Hubble (accounts, public keys, program ids...). Also available as a [NPM package](https://www.npmjs.com/package/@hubbleprotocol/hubble-config).

#### Get runtime config of all environments:

```http request
GET https://api.hubbleprotocol.io/config
```

You may also filter by environment (`mainnet-beta`[default],`devnet`,`localnet`,`testnet`) by using an `env` query parameter:

```http request
GET https://api.hubbleprotocol.io/config?env=devnet
```

### IDL

#### Get a list of all Hubble IDLs (Interface Description Language) generated by Anchor. Also available as a [NPM package](https://www.npmjs.com/package/@hubbleprotocol/hubble-idl).

```http request
GET https://api.hubbleprotocol.io/idl
```

### Circulating Supply Value HBB

#### Get circulating supply value of HBB (number of HBB issued * HBB price).
This is also included in the `/metrics` endpoint, but we need this for external services like CoinMarketCap.

```http request
GET https://api.hubbleprotocol.io/circulating-supply-value
```

You may also filter by environment (`mainnet-beta`[default],`devnet`,`localnet`,`testnet`) by using an `env` query parameter:

```http request
GET https://api.hubbleprotocol.io/circulating-supply-value?env=devnet
```

### Circulating Supply HBB

#### Get circulating supply of HBB (number of HBB issued).
This is also included in the `/metrics` endpoint, but we need this for external services like CoinGecko.

```http request
GET https://api.hubbleprotocol.io/circulating-supply
```

You may also filter by environment (`mainnet-beta`[default],`devnet`,`localnet`,`testnet`) by using an `env` query parameter:

```http request
GET https://api.hubbleprotocol.io/circulating-supply?env=devnet
```

### API Version

#### Get current version of the API.

```http request
GET https://api.hubbleprotocol.io/version
```

### Maintenance mode

#### Get maintenance mode parameter that specifies if Hubble webapp/smart contracts are in maintenance mode.

```http request
GET https://api.hubbleprotocol.io/maintenance-mode
```

### Borrowing version

#### Get borrowing version parameter that specifies the current version of the borrowing market state (smart contracts).

```http request
GET https://api.hubbleprotocol.io/borrowing-version
```

### Loans

You may use the `env` query param for all of the methods specified below (`mainnet-beta`[default],`devnet`,`localnet`,`testnet`).

#### Get a list of all loans and their values from on-chain Hubble data:

```http request
GET https://api.hubbleprotocol.io/loans?env=mainnet-beta
```

#### Get specific loan data:

```http request
// GET https://api.hubbleprotocol.io/loans/:pubkey
GET https://api.hubbleprotocol.io/loans/HrwbdQYwSnAyVpVHuGQ661HiNbWmGjDp5DdDR9YMw7Bu
```

#### Get a specific user's list of loans by specifying their public key:

```http request
// GET https://api.hubbleprotocol.io/owners/:pubkey/loans
GET https://api.hubbleprotocol.io/owners/HrwbdQYwSnAyVpVHuGQ661HiNbWmGjDp5DdDR9YMw7Bu/loans
```

#### Get specific loan's history data:

```http request
// GET https://api.hubbleprotocol.io/loans/:pubkey/history
GET https://api.hubbleprotocol.io/loans/HrwbdQYwSnAyVpVHuGQ661HiNbWmGjDp5DdDR9YMw7Bu/history
```

#### Get all unique loan pubkeys that were created by a specific user:

```http request
// GET https://api.hubbleprotocol.io/owners/:pubkey/loans/created?env={cluster}
GET https://api.hubbleprotocol.io/owners/HrwbdQYwSnAyVpVHuGQ661HiNbWmGjDp5DdDR9YMw7Bu/loans/created?env=mainnet-beta
```

### Staking

You may use the `env` query param for all of the methods specified below (`mainnet-beta`[default],`devnet`,`localnet`,`testnet`).

#### Get HBB and USDH staking stats (APR, APY):

```http request
GET https://api.hubbleprotocol.io/staking
```

#### Get all HBB stakers (grouped by owner public key):

```http request
GET https://api.hubbleprotocol.io/staking/hbb/users
```

#### Get all USDH stakers (grouped by owner public key):

```http request
GET https://api.hubbleprotocol.io/staking/usdh/users
```

#### Get LIDO staking rewards (APR + APY):

```http request
GET https://api.hubbleprotocol.io/staking/lido
```

#### Get eligible loans for LIDO staking rewards for a specified month and year.
Months are to be input from 1 (January) - 12 (December) and years from 2022 and above.

```http request
GET https://api.hubbleprotocol.io/staking/lido/eligible-loans/years/2022/months/07?env=devnet
```

#### Get single loan data for LIDO staking rewards for a specified month and year.
Months are to be input from 1 (January) - 12 (December) and years from 2022 and above.

```http request
// GET https://api.hubbleprotocol.io/staking/lido/eligible-loans/:loanPubkey/years/:year/months/:month?env={cluster}
GET https://api.hubbleprotocol.io/staking/lido/eligible-loans/6Hfhr9d9NRkhiPLbpRbHRLWLHBrCfMKDzdkv1sKd1S9M/years/2022/months/07?env=devnet
```

#### Get eligible loans for LIDO staking rewards for a specified time interval (start date is inclusive, end date is exclusive).
If no start/end date is specified it will use `start date: today - 14 days` and `end date = today`.

Please note: This route is not exposed to the public and requires basic authentication.
Please use the route above to get monthly data instead.

```http request
GET https://api.hubbleprotocol.io/staking/lido/eligible-loans?env=devnet&start=2022-06-01&end=2022-07-01
```

### Stability

You may use the `env` query param for all the methods specified below (`mainnet-beta`[default],`devnet`,`localnet`,`testnet`).

#### Get stability liquidation rewards (APR + APY):

```http request
GET https://api.hubbleprotocol.io/stability/liquidation-rewards
```

#### Get stability pool state history:

Returns 15 days of stability pool state history if no start/end date query params are specified.

```http request
GET https://api.hubbleprotocol.io/stability/history?env=mainnet-beta&start=2020-01-01&end=2023-01-10
```

#### Get stability provider history for a wallet:

Returns 15 days of stability provider history if no start/end date query params are specified.

```http request
GET https://api.hubbleprotocol.io/stability/providers/FDiwbqCeDFWcDq7EdQBLc8KtdW8zMmFWwZpf6T8dFFbW/history?env=mainnet-beta&start=2020-01-01&end=2023-01-09'
```

### Strategies Kamino

You may use the `env` query param for all the methods specified below (`mainnet-beta`[default],`devnet`,`localnet`,`testnet`).

#### Get frontend-enabled strategies:

```http request
// GET https://api.hubbleprotocol.io/strategies/enabled?env={cluster}
GET https://api.hubbleprotocol.io/strategies/enabled?env=mainnet-beta
```

#### Get specific strategy stats (tvl/pnl/shares..):

```http request
// GET https://api.hubbleprotocol.io/strategies/:strategy_pubkey/metrics?env={cluster}
GET https://api.hubbleprotocol.io/strategies/ByXB4xCxVhmUEmQj3Ut7byZ1Hbva1zhKjaVcv3jBMN7E/metrics?env=mainnet-beta
```

#### Get strategy stats for all frontend-enabled strategies (tvl/pnl/shares..):

```http request
// GET https://api.hubbleprotocol.io/strategies/metrics?env={cluster}
GET https://api.hubbleprotocol.io/strategies/metrics?env=mainnet-beta
```

#### Get strategy reward stats for all frontend-enabled strategies:

```http request
// GET https://api.hubbleprotocol.io/strategies/rewards?env={cluster}
GET https://api.hubbleprotocol.io/strategies/rewards?env=mainnet-beta
```

#### Get strategy stats for rewards:

```http request
// GET https://api.hubbleprotocol.io/strategies/:strategy_pubkey/rewards?env={cluster}&year={year}
GET https://api.hubbleprotocol.io/strategies/ByXB4xCxVhmUEmQj3Ut7byZ1Hbva1zhKjaVcv3jBMN7E/rewards?env=mainnet-beta
```

#### Get rewards for eligible shareholders of a specific strategy:

This will fetch the last 14 days of data by default, you can use the `start` and `end` query parameter dates to change it.

Please note: This route is not exposed to the public and requires basic authentication.

Please use the public route to get rewards for a specific shareholder.

```http request
// GET https://api.hubbleprotocol.io/strategies/:strategy_pubkey/rewards?env={cluster}&start={start}&end={end}
GET https://api.hubbleprotocol.io/strategies/ByXB4xCxVhmUEmQj3Ut7byZ1Hbva1zhKjaVcv3jBMN7E/rewards/eligible-shareholders?env=mainnet-beta&start=2022-09-01&end=2022-09-30
```

#### Get amount of rewards for a specific Kamino shareholder (all strategies):

```http request
// GET https://api.hubbleprotocol.io/strategies/shareholders/:shareholder_pubkey/rewards/history?env={cluster}
GET https://api.hubbleprotocol.io/strategies/shareholders/GkeDRfHcACap2CM9oaWrb3QUMG7pk6F6EMYca8Lu52t8/rewards/history?env=mainnet-beta
```

#### Get strategy state history aggregated by the hour for a specific year (default current year, or use query param `year`):

```http request
// GET https://api.hubbleprotocol.io/strategies/:strategy_pubkey/history?env={cluster}&year={year}
GET https://api.hubbleprotocol.io/strategies/ByXB4xCxVhmUEmQj3Ut7byZ1Hbva1zhKjaVcv3jBMN7E/history?env=devnet&year=2022
```

#### Get full non-aggregated strategy state history for a specific year (default current year, or use query param `year`):

Please note: This route is not exposed to the public and requires basic authentication.

Please use the public route to get hourly aggregated history for a specific strategy.

```http request
// GET https://api.hubbleprotocol.io/strategies/:strategy_pubkey/full-history?env={cluster}&year={year}
GET https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/full-history?env=mainnet-beta&year=2022
```

#### Get user's shares history aggregated by the hour for a specific year (default current year, or use query param `year`):

```http request
// GET https://api.hubbleprotocol.io/owners/:owner_pubkey/strategies/:strategy_pubkey/shares/history?env={cluster}&year={year}
GET https://api.hubbleprotocol.io/owners/BabJ4KTDUDqaBRWLFza3Ek3zEcjXaPDmeRGRwusQyLPS/strategies/ByXB4xCxVhmUEmQj3Ut7byZ1Hbva1zhKjaVcv3jBMN7E/shares/history?env=mainnet-beta&year=2022
```

#### Get fees and rewards earned for a strategy shareholder:

```http request
// GET https://api.hubbleprotocol.io/strategies/:strategyPubkey/shareholders/:shareholderPubkey/fees-and-rewards?env={cluster}
GET https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L/fees-and-rewards?env=mainnet-beta
```

Sample response:

```json
{
    "feesAEarned": "31.32619353486741639998339868",
    "feesBEarned": "24.70081628219520072196336092",
    "feesAEarnedUsd": "31.256576407153470503340282981001181676",
    "feesBEarnedUsd": "24.698716826795158937664218970228174",
    "rewards0Earned": "0",
    "rewards1Earned": "0",
    "rewards2Earned": "0",
    "rewards0EarnedUsd": "0",
    "rewards1EarnedUsd": "0",
    "rewards2EarnedUsd": "0",
    "kaminoRewards0Earned": "712379.5022312449919715809869996",
    "kaminoRewards1Earned": "0",
    "kaminoRewards2Earned": "0",
    "kaminoRewards0EarnedUsd": "0.61236142011797819509877101642485616",
    "kaminoRewards1EarnedUsd": "0",
    "kaminoRewards2EarnedUsd": "0",
    "lastCalculated": "2023-01-12T13:10:21.212Z"
}
```

- fees/rewards earned represent the amount of tokens earned from the first deposit
- fees/rewards earned in USD represent the USD amount earned from the first deposit
- rewards represent vault rewards, kamino rewards represent autocompounded rewards
- last calculated refers to the last calculation date (fees and rewards are calculated hourly)

#### Get all strategies with filters

* The current filters that are supported are:
  * `strategyType` which can be:
    * `NON_PEGGED`: e.g. SOL-BONK
    * `PEGGED`: e.g. BSOL-JitoSOL
    * `STABLE`: e.g. USDH-USDC

  * `strategyCreationStatus` which can be:
    * `IGNORED`
    * `SHADOW`
    * `LIVE`
    * `DEPRECATED`
    * `STAGING`

If no filters are provided all strategies are fetched
```http request
GET https://api.hubbleprotocol.io/strategies?env={cluster}&type={type}&status={status}
```

Examples:

Get all `NON_PEGGED` strategies
```http request
GET https://api.hubbleprotocol.io/strategies?type=NON_PEGGED
```

Get all `STAGING` strategies
```http request
GET https://api.hubbleprotocol.io/strategies?status=STAGING
```

#### Get total token amounts in live strategies

```http request
GET https://api.hubbleprotocol.io/strategies/tokens/{tokenMint}/amounts?env={cluster}
```

Example request to get total MSOL stored in Kamino:
https://api.hubbleprotocol.io/strategies/tokens/mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So/amounts?env=mainnet-beta

Example response:
```json
{
    "totalTokenAmount": "2643,805199154",
    "vaults": [
        {
            "address": "9zBNQtnenpQY6mCoRqbPpeePeSy17h34DZP82oegt1fL",
            "frontendUrl": "https://app.kamino.finance/liquidity/9zBNQtnenpQY6mCoRqbPpeePeSy17h34DZP82oegt1fL",
            "amount": "1641.613204799"
        },
        {
            "address": "F3v6sBb5gXL98kaMkaKm5GfEoBNUaSd3ZGErbjqgzTho",
            "frontendUrl": "https://app.kamino.finance/liquidity/F3v6sBb5gXL98kaMkaKm5GfEoBNUaSd3ZGErbjqgzTho",
            "amount": "1002.191994355"
        }
    ],
    "timestamp": "2023-02-21T12:09:21.304Z"
}
```

### Whirlpools Kamino

You may use the `env` query param for all the methods specified below (`mainnet-beta`[default],`devnet`,`localnet`,`testnet`).

#### Get Orca whirlpool history for a specific year (default current year, or use query param `year`):

```http request
// GET https://api.hubbleprotocol.io/whirlpools/:whirlpool_pubkey/history?env={cluster}&year={year}
GET https://api.hubbleprotocol.io/whirlpools/BabJ4KTDUDqaBRWLFza3Ek3zEcjXaPDmeRGRwusQyLPS/history?env=mainnet-beta&year=2022
```

### kToken metadata (Kamino)

You may use the `env` query param for all the methods specified below (`mainnet-beta`[default],`devnet`,`localnet`,`testnet`).

All mints must be valid kToken mints.

#### Get kToken metadata for mint:

```http request
// GET https://api.hubbleprotocol.io/ktokens/:mint/metadata
GET https://api.hubbleprotocol.io/ktokens/BabJ4KTDUDqaBRWLFza3Ek3zEcjXaPDmeRGRwusQyLPS/metadata
```

#### Get kToken image for mint:

```http request
// GET https://api.hubbleprotocol.io/ktokens/:mint/metadata/image.svg
GET https://api.hubbleprotocol.io/ktokens/BabJ4KTDUDqaBRWLFza3Ek3zEcjXaPDmeRGRwusQyLPS/metadata/image.svg
```

### Prices

#### Get all prices:

```http request
GET https://api.hubbleprotocol.io/prices?env={cluster}&source={priceSource:scope(default)}
```

Example request: https://api.hubbleprotocol.io/prices?env=mainnet-beta&source=scope
Example response:

```json
[
    {
        "usdPrice": "22.382246552631578",
        "token": "SOL"
    },
    {
        "usdPrice": "1580.8847893915756",
        "token": "ETH"
    }
]
```


#### Get specific token price:

```http request
GET https://api.hubbleprotocol.io/prices?env={cluster}&source={priceSource:scope(default)}&token={tokenName}
```

Example request: https://api.hubbleprotocol.io/prices?env=mainnet-beta&source=scope&token=SOL
Example response:

```json
[
    {
        "usdPrice": "22.382246552631578",
        "token": "SOL"
    }
]
```

#### Get token price history:

Please note: This route is not exposed to the public and requires basic authentication.

You can specify start/end date range with query params `start` and `end`. Otherwise, it will return 24 hours of prices by default.

```http request
GET https://api.hubbleprotocol.io/prices/history?env=mainnet-beta&token=SOL&start=2020-01-01&end=2023-01-01
```

Example response:

```js
[
    [
        "17.35422124",  // <-- token price
        1668162327      // <-- seconds since epoch
    ],
    [
        "16.51238765",
        1668172327
    ]
]
```

#### Get token price moving averages:

```http request
GET https://api.hubbleprotocol.io/prices/moving-averages?env={cluster:optional, default mainnet}&token={tokenName:required}&durationInSec={duration:optional, default 3600}
```
Example request:
https://api.hubbleprotocol.io/prices/moving-averages?env=mainnet-beta&token=SOL&durationInSec=3600

Example response:

```json
[
    {
        "pair": "SOL/USD",
        "sma": "21.8084071408819895",
        "priceCount": "2",
        "start": "2023-02-14T15:36:42.498Z",
        "end": "2023-02-14T15:39:03.759Z",
        "ema": "21.8084071408819895",
        "source": "scope"
    }
]
```

### Borrowing Market State

Borrowing market state = BMS

#### Get BMS history

Please note: This route is not exposed to the public and requires basic authentication.

You can specify start/end date range with query params `start` and `end`. Otherwise, it will return 24 hours of BMS by default.

```http request
// GET https://api.hubbleprotocol.io/bms/:bmsPubkey/history?env={cluster}&start={startDate}&end={endDate}
GET https://api.hubbleprotocol.io/bms/FqkHHpETrpfgcA5SeH7PKKFDLGWM4tM7ZV31HfutTXNV/history?env=mainnet-beta&start=2020-01-01&end=2023-01-01
```

#### Get BMS loans closest to a snapshot timestamp

Please note: This route is not exposed to the public and requires basic authentication.

```http request
// GET https://api.hubbleprotocol.io/bms/:bmsPubkey/closest-loans?env={cluster}&timestamp={timestamp}
GET https://api.hubbleprotocol.io/bms/FqkHHpETrpfgcA5SeH7PKKFDLGWM4tM7ZV31HfutTXNV/closest-loans?env=mainnet-beta&timestamp=2022-11-22T12:00:00.000Z
```

### Debug

Endpoints to help with debugging by using the Carpool service.

#### Get program accounts

```http request
// GET https://api.hubbleprotocol.io/debug/accounts/:programId/:accountName?env={cluster}
GET https://api.hubbleprotocol.io/debug/accounts/HubbLeXBb7qyLHt3x7gvYaRrxQmmgExb7fCJgDqFuB6T/BorrowingMarketState?env=mainnet-beta
```

### Transactions

#### Get all kamino transactions

Get shareholder's Kamino transactions (`withdraw`, `deposit` and `depositAndInvest` instructions).
Returns the last 1000 transactions ordered by timestamp descending.

```http request
// GET https://api.hubbleprotocol.io/shareholders/:shareholderPubkey/transactions?env={cluster}
GET https://api.hubbleprotocol.io/shareholders/9y7uLMUMW6EiRwH1aJFSp9Zka7dVx2JdZKA3858u6YHT/transactions?env=mainnet-beta
```

Example response:

```json
{
  "transactions": [
    {
      "timestamp": "2023-01-12T20:39:48.758Z",
      "transactionSignature": "3Qmbxcuhc58QDe2xSsKhVQTLRACbuyisdayMTA6sCpMVqX34fPFnqiB8d1BSYCqbNNGxZ5ekdrMk3CsZ4cV7gUqf",
      "transactionName": "depositAndInvest",
      "strategy": "HWg7yB3C1BnmTKFMU3KGD7E96xx2rUhv4gxrwbZLXHBt",
      "tokenA": "SOL",
      "tokenAAmount": "0.162654437",
      "tokenAPrice": "0",
      "tokenB": "BONK",
      "tokenBAmount": "2854564.78389",
      "tokenBPrice": "0",
      "usdValue": "0",
      "numberOfShares": "7.879718"
    },
    {
      "timestamp": "2023-01-17T18:00:07.820Z",
      "transactionSignature": "3GruXM2CJtpeoP9rbG8QGAEqeCCAGcpyVmD6k5A4ezAuZpSDgLB5HPKAWFWDa6bPiDKiWDYk7mmLoF4w3ZKRhpsU",
      "transactionName": "withdraw",
      "strategy": "Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN",
      "tokenA": "USDH",
      "tokenAAmount": "3.928922",
      "tokenAPrice": "10",
      "tokenB": "USDC",
      "tokenBAmount": "2.738213",
      "tokenBPrice": "0",
      "usdValue": "39.28922",
      "numberOfShares": "6.604654"
    }
  ],
  "lastUpdatedOn": "2023-01-18T14:49:31.344Z"
}
```

### Kamino Market Lending

**⚠️ This is still work in progress, endpoints return mock data for testing. ⚠️**

#### Get Kamino Market config

```http request
GET https://api.hubbleprotocol.io/kamino-market?env={cluster}
```
Example:
https://api.hubbleprotocol.io/kamino-market?env=mainnet-beta

#### Get Kamino Market metrics history

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/metrics/history?env={cluster}&start={date}&end={date}'
```
Example:
https://api.hubbleprotocol.io/kamino-market/HcHkvZEDechu7bruV8zvMN11v9yHg3iY4QHNgrYUATmg/metrics/history?env=mainnet-beta&start=2023-01-01&end=2023-01-02

#### Get Kamino Market reserve history

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/reserves/:reservePubkey/metrics/history?env={cluster}&start={date}&end={date}'
```
Example:
https://api.hubbleprotocol.io/kamino-market/HcHkvZEDechu7bruV8zvMN11v9yHg3iY4QHNgrYUATmg/reserves/HcHkvZEDechu7bruV8zvMN11v9yHg3iY4QHNgrYUATmm/metrics/history?env=mainnet-beta&start=2023-01-01&end=2023-01-02

#### Get Kamino Market loan history

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/loans/:loanPubkey/metrics/history?env={cluster}&start={date}&end={date}'
```
Example:
https://api.hubbleprotocol.io/kamino-market/HcHkvZEDechu7bruV8zvMN11v9yHg3iY4QHNgrYUATmg/loans/HcHkvZEDechu7bruV8zvMN11v9yHg3iY4QHNgrYUATmm/metrics/history?env=mainnet-beta&start=2023-01-01&end=2023-01-02
  
