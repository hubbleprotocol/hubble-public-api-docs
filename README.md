# 🛰 Hubble Public API

Hubble Public API is a TypeScript API (using Express) that serves public data of the Hubble Protocol.  

## Table of contents

- [Development](#development)
    * [Database](#database)
    * [Local API Setup](#local-api-setup)
    * [Tests](#tests)
    * [Deployment](#deployment)
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
    * [Kamino Lending](#kamino-lending)
    * [Trades](#trades)
    * [Simulator](#simulator)
    * [Leaderboard](#leaderboard)

## Development

### Database

We use Flyway for database migrations and PostgreSQL for data storage.

Lint your migration files:

```shell
cd migrations
./lint.sh
```

Run migrations with docker:

```shell
# Run postgresql at localhost:5432 and apply flyway migrations
docker-compose up db flyway
```

### Local API Setup
You will need to use [npm](https://www.npmjs.com/) to install the dependencies.

We are using private nodes without rate limit for fetching mainnet-beta and devnet chain data.
You will have to add an `.env` file in the root of this repository with the correct environment variables inside.
Please take a look at the example `.env.example` file:

```shell
cd hubble-public-api
cp .env.example .env
# edit .env with actual endpoints with your favorite editor
# nano .env  
# code .env
# ...
```

We also use Redis for caching API responses from PostgreSQL database. You can use docker-compose to run an instance of Redis locally.

Run the API with yarn (for debugging) and dependencies with docker-compose:

```shell
cd hubble-public-api
docker-compose up -d redis db flyway
yarn start
```

Run everything with docker-compose:

```shell
cd hubble-public-api
docker-compose up -d
```

API will be available at http://localhost:8888.

Some routes are protected by basic authentication - for local dev API you can use these credentials:
- user: `hubble`
- password: `development`

### Tests

We use a test database and migrations for integration tests that needs to be launched before running the tests:

```shell
docker-compose up -d test-db test-flyway
yarn test
yarn test -- -t 'should calculate the fees and rewards of strategy shareholder 2 with race condition'

```

### How to run locally

- Set this up for prod: https://github.com/hubbleprotocol/hubble-infrastructure/blob/master/cluster/docs/database_access.md#connecting
- Ssh into the prod cluster in a separate window: (this creates an RDS tunnel from localhost -> prod db): `ssh <tunnel>`
- Set up the .env to have POSTGRES_CONNECTION_STRING=.....localhost:5432/hubble (this connects to the localhost tunnel that points to prod db)
- Start redis locally: `docker-compose up redis`
- Start the server locally: `yarn start`

### Deployment

Deployments are done automatically, everything that gets pushed to the `master` branch will be packaged as a docker container and pushed to Hubble's DockerHub.

## Cache

We cache all the endpoint responses with Redis. 

### Local development cache

To access the local cache, you will need to install [redis-cli](https://redis.io/docs/ui/cli/) and run:

```shell
# make sure local redis is running first:
docker-compose up redis
# connect to redis-cli
redis-cli
# a few examples below

# list all keys:
KEYS *
# get specific redis key contents:
GET strategies-leaderboard-mainnet-beta
# delete specific redis key:
DEL strategies-leaderboard-mainnet-beta
# check when a specific redis key will expire:
TTL strategies-leaderboard-mainnet-beta
# clear entire cache - delete all redis keys (useful when testing locally, not enabled in production)
FLUSHALL
```

### Production cache

:warning: This can be potentially be very dangerous, proceed with caution! :warning:

Make sure you have everything required to access the Kubernetes clusters: [Developer Setup](https://github.com/hubbleprotocol/hubble-infrastructure/blob/6e53b122a6b74a6a53c019b9fadd3364ffb4c718/README.md#L8).

To access the production redis cache:
```shell
# connect to the prod kubernetes cluster 
export NAME=k8s.hubbleprotocol.io
export KOPS_STATE_STORE=s3://k8s.hubbleprotocol.io-kops-state-store
kops export kubecfg k8s.hubbleprotocol.io --admin

# reverse-proxy the redis instance in the api namespace to localhost:6380
kubectl -n api port-forward redis-hubble-public-api-master-0 6380:6379

# access the prod redis cache by using localhost:6380 with redis-cli
redis-cli -p 6380
```

A very common use case is to clear certain redis keys in production to force a refresh:

```shell
# the commands below support wildcard (*) usage 

# WARNING - POTENTIALLY DANGEROUS, DOUBLE CHECK BEFORE EXECUTING, MAKE A BACKUP IF NECESSARY

# check how many keys you're deleting beforehand - make sure the keys returned are expected
redis-cli -p 6380 KEYS "*transactions*"
# clear all redis keys that have "transactions" in their name
redis-cli -p 6380 KEYS "*transactions*" | xargs redis-cli DEL

# clear all redis keys that start with "strategies" in their name
redis-cli -p 6380 KEYS "strategies*" | xargs redis-cli DEL
```


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

#### Get staking yields

```http request
GET https://api.hubbleprotocol.io/staking-yields
```

Example request:
https://api.hubbleprotocol.io/staking-yields

Example response:

```json
[
    {
        "apy": "0.076269480000000005",
        "token": "cgntSOL",
        "tokenMint": "CgnTSoL3DgY9SFHxcLj6CgCgKKoTBr6tp4CPAEWy25DE"
    },
    {
        "apy": "6.88",
        "token": "LDO",
        "tokenMint": "HZRCwxP2Vq9PCpPXooayhJ2bxTpo5xfpQrwB1svh332p"
    }
]
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

#### Get strategy stats for strategies by status (tvl/pnl/shares..):

```http request
// GET https://api.hubbleprotocol.io/strategies/metrics?env={cluster}&status={status}
GET https://api.hubbleprotocol.io/strategies/metrics?env=mainnet-beta&status=IGNORED
```

#### Get strategy reward stats for all live strategies:

```http request
// GET https://api.hubbleprotocol.io/strategies/rewards?env={cluster}
GET https://api.hubbleprotocol.io/strategies/rewards?env=mainnet-beta
```

Example response:

```json
[
    {
        "apr": "0.03089658127477581433161189171198501610943",
        "apy": "0.031377485860868284369015169698693427029",
        "totalReturn": "9959.75762697216",
        "totalInvestment": "322357.9184504589828735999999999999999998",
        "strategy": "2dczcMRpxWHZTcsiEjPT4YBcSseTaUmWFzw24HxYMFod",
        "token": "RAY",
        "rewardsPerDay": "137.8944",
        "rewardsPerHour": "5.7456",
        "rewardsPerSecond": "0.001596",
        "rewardIsOption": false      
    },
    {
        "apr": "0.1837059453843344336978185004649421142565",
        "apy": "0.201606883863164306468378664554523674214",
        "totalReturn": "39084.02775504",
        "totalInvestment": "212753.2000843610114423811349599999999999",
        "strategy": "6K4jM79yijUEFxdFhCFZSjav1nZji1gsxUWQE6XrC8YD",
        "token": "LDO",
        "rewardsPerDay": "48.386592",
        "rewardsPerHour": "2.016108",
        "rewardsPerSecond": "0.00056003",
        "rewardIsOption": false      
    }
]
```

#### Get strategy stats for rewards:

```http request
// GET https://api.hubbleprotocol.io/strategies/:strategy_pubkey/rewards?env={cluster}&year={year}
GET https://api.hubbleprotocol.io/strategies/ByXB4xCxVhmUEmQj3Ut7byZ1Hbva1zhKjaVcv3jBMN7E/rewards?env=mainnet-beta
```

Example response:

```json
    {
        "apr": "0.03089658127477581433161189171198501610943",
        "apy": "0.031377485860868284369015169698693427029",
        "totalReturn": "9959.75762697216",
        "totalInvestment": "322357.9184504589828735999999999999999998",
        "strategy": "2dczcMRpxWHZTcsiEjPT4YBcSseTaUmWFzw24HxYMFod",
        "token": "RAY",
        "rewardsPerDay": "137.8944",
        "rewardsPerHour": "5.7456",
        "rewardsPerSecond": "0.001596",
        "rewardIsOption": false  
    }
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

:warning: **DEPRECATED, PLEASE USE /v2/strategies/:pubkey/history INSTEAD!** :warning:

```http request
// GET https://api.hubbleprotocol.io/strategies/:strategy_pubkey/history?env={cluster}&year={year}
GET https://api.hubbleprotocol.io/strategies/ByXB4xCxVhmUEmQj3Ut7byZ1Hbva1zhKjaVcv3jBMN7E/history?env=devnet&year=2022
```

#### Get full non-aggregated strategy state history for a specific year (default current year, or use query param `year`):

:warning: **DEPRECATED, PLEASE USE /v2/strategies/:pubkey/history INSTEAD!** :warning:

Please note: This route is not exposed to the public and requires basic authentication.

Please use the public route to get hourly aggregated history for a specific strategy.

```http request
// GET https://api.hubbleprotocol.io/strategies/:strategy_pubkey/full-history?env={cluster}&year={year}
GET https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/full-history?env=mainnet-beta&year=2022
```

#### Get strategy state history v2:

```http request
GET https://api.hubbleprotocol.io/v2/strategies/:strategyPubkey/history?env={cluster}&start={start}&end={end}&frequency={frequency}
```

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
* start: start date (inclusive), e.g. `2023-05-01T00:00:00.000Z` (optional, default since beginning of strategy)
* end: end date (exclusive), e.g. `2023-05-02T00:00:00.000Z` (optional, default now)
* frequency: frequency of the snapshots, e.g. `"hour" (default) | "day"`


Example requests:

- https://api.hubbleprotocol.io/v2/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/history?env=mainnet-beta
- https://api.hubbleprotocol.io/v2/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/history?env=mainnet-beta&start=2023-01-01&end=2023-02-01&frequency=hour
- https://api.hubbleprotocol.io/v2/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/history?env=mainnet-beta&start=2023-01-01&end=2023-02-01&frequency=day


Example response:

```js
[
  {
    "timestamp": "2023-06-06T00:00:00.000Z",
    "feesCollectedCumulativeA": "4528.851182",
    "feesCollectedCumulativeB": "4388.399699",
    "rewardsCollectedCumulative0": "6816.973791",
    "rewardsCollectedCumulative1": "869.925191",
    "rewardsCollectedCumulative2": "0",
    "kaminoRewardsIssuedCumulative0": "0",
    "kaminoRewardsIssuedCumulative1": "0",
    "kaminoRewardsIssuedCumulative2": "0",
    "sharePrice": "1.009166844496339887986660123337354133388",
    "sharesIssued": "2000370.229713",
    "tokenAAmounts": "1941835.079462",
    "tokenBAmounts": "83285.638712",
    "tokenAPrice": "0.9966972458",
    "tokenBPrice": "0.99999998",
    "reward0Price": "0.042612354",
    "reward1Price": "0.6421835081",
    "reward2Price": "0.99999998",
    "kaminoReward0Price": "0",
    "kaminoReward1Price": "0",
    "kaminoReward2Price": "0",
    "totalValueLocked": "2018707.312543886771519599999999999999998983602957644",
    "solPrice": "0",
    "profitAndLoss": "0"
  }
]
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
- last calculated refers to the last calculation date (fees and rewards are calculated every 5 minutes)

#### Get latest fees and rewards earned for a strategy shareholder:

Fetch strategy shareholder fees and rewards for only the latest position.
This endpoint takes a look at when the user's position was last closed (fully withdrawn)
and opened again and only calculates the fees and rewards after that time.

For all-time fees and rewards of a strategy shareholder use endpoint
mentioned in the previous section (`Get fees and rewards earned for a strategy shareholder` above).

```http request
// GET https://api.hubbleprotocol.io/strategies/:strategyPubkey/shareholders/:shareholderPubkey/fees-and-rewards/latest-position?env={cluster}
GET https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L/fees-and-rewards/latest-position?env=mainnet-beta
```

Example request:

https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L/fees-and-rewards/latest-position?env=mainnet-beta

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

- fees/rewards earned represent the amount of tokens earned from the last deposit
- fees/rewards earned in USD represent the USD amount earned from the last deposit
- rewards represent vault rewards, kamino rewards represent autocompounded rewards
- last calculated refers to the last calculation date (fees and rewards are calculated every 5 minutes)

#### Get fees and rewards for strategies:

```http request
GET https://api.hubbleprotocol.io/strategies/fees-and-rewards?env={cluster}&period={timePeriod}&status={strategyStatus}
```

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
* period: time period for fees and rewards `"24h" (default) | "7d" | "30d"`
* status: strategy status `"LIVE" (default) | "STAGING" | "SHADOW" | "IGNORED" | "DEPRECATED"`, status query param can also be used multiple times, for example:
  * https://api.hubbleprotocol.io/strategies/fees-and-rewards?status=LIVE&status=STAGING&status=SHADOW

Example response:

```json
[
  {
    "strategyPubkey": "2VQaDuSqqxeX2h9dS9WgpvN6ShaBxd8JjaaWEvbmTDY1",
    "feesAEarned": "1.037294",
    "feesBEarned": "2.753151",
    "rewards0Earned": "0",
    "rewards1Earned": "0",
    "rewards2Earned": "0",
    "kaminoRewards0Earned": "0",
    "kaminoRewards1Earned": "0",
    "kaminoRewards2Earned": "0",
    "feesAEarnedUsd": "1.0344169674816346",
    "feesBEarnedUsd": "2.75657612228428",
    "rewards0EarnedUsd": "0",
    "rewards1EarnedUsd": "0",
    "rewards2EarnedUsd": "0",
    "kaminoRewards0EarnedUsd": "0",
    "kaminoRewards1EarnedUsd": "0",
    "kaminoRewards2EarnedUsd": "0",
    "lastCalculated": "2023-03-24T14:14:13.885Z"
  },
  {
    "strategyPubkey": "2dczcMRpxWHZTcsiEjPT4YBcSseTaUmWFzw24HxYMFod",
    "feesAEarned": "1.389828787",
    "feesBEarned": "1.182662835",
    "rewards0Earned": "681.574997269",
    "rewards1Earned": "272.629657",
    "rewards2Earned": "0",
    "kaminoRewards0Earned": "351.579127",
    "kaminoRewards1Earned": "0",
    "kaminoRewards2Earned": "0",
    "feesAEarnedUsd": "29.78482623351582254",
    "feesBEarnedUsd": "27.9253762204614832164",
    "rewards0EarnedUsd": "31.2709271924846080985",
    "rewards1EarnedUsd": "66.55395386232317",
    "rewards2EarnedUsd": "0",
    "kaminoRewards0EarnedUsd": "85.68865812406762",
    "kaminoRewards1EarnedUsd": "0",
    "kaminoRewards2EarnedUsd": "0",
    "lastCalculated": "2023-03-24T14:14:13.885Z"
  }
]
```

- fees/rewards earned represent the amount of tokens earned from the first deposit
- fees/rewards earned in USD represent the USD amount earned from the first deposit
- rewards represent vault rewards, kamino rewards represent autocompounded rewards
- last calculated refers to the last calculation date (fees and rewards are calculated every 5 minutes)

#### Get PnL for a strategy shareholder position:

Caclulate profit and loss and cost basis for a Kamino strategy shareholder's latest position.

```http request
// GET https://api.hubbleprotocol.io/strategies/:strategyPubkey/shareholders/:shareholderPubkey/pnl?env={cluster}
GET https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L/pnl?env=mainnet-beta
```

Example request:

https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L/pnl?env=mainnet-beta

Sample response:

```json
{
  "totalPnl": {
    "sol": "0.821586623506697901642320602103",
    "usd": "15.932660735531558716134077510914",
    "a": "0.821586623506697901642320602103",
    "b": "0.742615763210195652674132993392"
  },
  "totalCostBasis": {
    "sol": "1.263745586773755885725088551614",
    "usd": "24.507251109021206639863158576972",
    "a": "1.263745586773755885725088551614",
    "b": "1.142274431659923954552610113376"
  }
}
```

`"a"` and `"b"` properties in the response refer to the token A and token B that the strategy contains.

#### Get PnL history for a strategy shareholder:

Return PnL history data for strategy shareholder with mark to market analysis. 

Returns hourly timeseries if user's latest position is under 15 days old, otherwise it returns daily timeseries data.
This is an optimization for frontend charts. 

```http request
// GET https://api.hubbleprotocol.io/strategies/:strategyPubkey/shareholders/:shareholderPubkey/pnl/history?env={cluster}&start={date}&end={date}
GET https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L/pnl/history?env=mainnet-beta
```

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
* start: start date (inclusive), e.g. `2023-05-01T00:55:00.000Z`
* end: end date (exclusive), e.g. `2023-05-01T00:55:00.000Z`

Example request:

https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L/pnl/history?env=mainnet-beta

Sample response:

```json
{
  "history": [
    {
      "timestamp": 1672051577137,
      "type": "buy",
      "price": {
        "sol": "13.707781891346918813283886561673",
        "usd": "264.432819438594964488396234360437",
        "a": "13.707781891346918813283886561673",
        "b": "12.38943455028558047998876234889"
      },
      "quantity": "0.042891188",
      "positionPnl": {
        "sol": "0",
        "usd": "0",
        "a": "0",
        "b": "0"
      },
      "realizedPnl": {
        "sol": "0",
        "usd": "0",
        "a": "0",
        "b": "0"
      },
      "allHistoryPnl": {
        "sol": "0",
        "usd": "0",
        "a": "0",
        "b": "0"
      },
      "position": "0.042891188",
      "positionValue": {
        "sol": "0.587943050164756268041296075887",
        "usd": "11.341837771910831077725126706445",
        "a": "0.587943050164756268041296075887",
        "b": "0.531397566509994286056328243793"
      },
      "costBasis": {
        "sol": "0.587943050164756268041296075887",
        "usd": "11.341837771910831077725126706445",
        "a": "0.587943050164756268041296075887",
        "b": "0.531397566509994286056328243793"
      },
      "investment": {
        "sol": "0.587943050164756268041296075887",
        "usd": "11.341837771910831077725126706445",
        "a": "0.587943050164756268041296075887",
        "b": "0.531397566509994286056328243793"
      }
    }
  ],
  "totalPnl": {
    "sol": "0.815290185426638259315771871017",
    "usd": "15.727525000165956333285426076312",
    "a": "0.815290185426638259315771871017",
    "b": "0.736879567525786810587921709799"
  },
  "totalCostBasis": {
    "sol": "1.270417392486119200058849813594",
    "usd": "24.507251109021206639863158576972",
    "a": "1.270417392486119200058849813594",
    "b": "1.148234868376991871955894984218"
  },
  "strategy": "ByXB4xCxVhmUEmQj3Ut7byZ1Hbva1zhKjaVcv3jBMN7E",
  "wallet": "2VGzusQTEFJneuTWd7RQXx53vXiTi2qNYXx4ftj26Vvb"
}
```

`"a"` and `"b"` properties in the response refer to the token A and token B that the strategy contains.

Transaction type can be `buy`, `sell` or `mark-to-market`. Buys and sells are actual transactions of the user.
Mark to market represents potential position and PnL if they bought/sold at that specific time.

#### Get volume for strategies:

:warning: **DEPRECATED, PLEASE USE /v2/strategies/volume INSTEAD!** :warning:

```http request
GET https://api.hubbleprotocol.io/strategies/volume?env={cluster}&status={strategyStatus}
```

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
* status: strategy status `"LIVE" (default) | "STAGING" | "SHADOW" | "IGNORED" | "DEPRECATED"`, status query param can also be used multiple times, for example:
  * https://api.hubbleprotocol.io/strategies/volume?status=LIVE&status=STAGING&status=SHADOW

Example response:

```json
[
  {
    "strategy": "6K4jM79yijUEFxdFhCFZSjav1nZji1gsxUWQE6XrC8YD",
    "kaminoVolume": [
      {
        "period": "24h",
        "amount": "0"
      },
      {
        "period": "7d",
        "amount": "4943.5832904232213864595352"
      },
      {
        "period": "30d",
        "amount": "70711922.7175338705593670146676"
      }
    ],
    "poolVolume": [
      {
        "period": "24h",
        "amount": "32449.432601658486"
      },
      {
        "period": "7d",
        "amount": "199221.15394896772"
      },
      {
        "period": "30d",
        "amount": "1493200.352996167"
      }
    ]
  }
]
```

#### Get volume for all strategies v2:

Return 24h/7d/30d volume (in USD) for every single Kamino strategy. 

```http request
GET https://api.hubbleprotocol.io/v2/strategies/volume?env={cluster}
```

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

Example request: https://api.hubbleprotocol.io/v2/strategies/volume?env=mainnet-beta

Example response:

```json
[
  {
    "strategy": "BfyQYYr2T9eJfMfq5gPXcq3SUkJSh2ahtk7ZNUCzkx9e",
    "kaminoVolume": [
      {
        "period": "24h",
        "amount": "110492.5009755243"
      },
      {
        "period": "7d",
        "amount": "2720522.381172522"
      },
      {
        "period": "30d",
        "amount": "5617787.72039797"
      }
    ]
  },
  {
    "strategy": "9zBNQtnenpQY6mCoRqbPpeePeSy17h34DZP82oegt1fL",
    "kaminoVolume": [
      {
        "period": "24h",
        "amount": "110492.5009755243"
      },
      {
        "period": "7d",
        "amount": "2720522.381172522"
      },
      {
        "period": "30d",
        "amount": "5617787.72039797"
      }
    ]
  }
]
```

#### Get volume for specific strategy v2:

Return 24h/7d/30d volume (in USD) for specified Kamino strategy.

```http request
GET https://api.hubbleprotocol.io/v2/strategies/:strategyPubkey/volume?env={cluster}
```

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

Example request: https://api.hubbleprotocol.io/v2/strategies/BfyQYYr2T9eJfMfq5gPXcq3SUkJSh2ahtk7ZNUCzkx9e/volume?env=mainnet-beta

Example response:

```json
{
  "strategy": "BfyQYYr2T9eJfMfq5gPXcq3SUkJSh2ahtk7ZNUCzkx9e",
  "kaminoVolume": [
    {
      "period": "24h",
      "amount": "110492.5009755243"
    },
    {
      "period": "7d",
      "amount": "2720522.381172522"
    },
    {
      "period": "30d",
      "amount": "5617787.72039797"
    }
  ]
}
```

#### Get all-time Kamino volume:

```http request
GET https://api.hubbleprotocol.io/strategies/all-time-volume?env={cluster}
```

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

Example request:

https://api.hubbleprotocol.io/strategies/all-time-volume

Example response:

```json
{
  "volumeUsd": "106015327.76",
  "lastCalculated": "2023-04-13T15:55:31.761Z"
}
```

#### Get all-time Kamino fees and rewards:

```http request
GET https://api.hubbleprotocol.io/strategies/all-time-fees-and-rewards?env={cluster}
```

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

Example request:

https://api.hubbleprotocol.io/strategies/all-time-fees-and-rewards

Example response:

```json
{
  "feesAEarnedUsd": "59246.89656897732967689080867366185019",
  "feesBEarnedUsd": "57080.94920621374904358352501138520562",
  "rewards0EarnedUsd": "5927.1681652971537634193",
  "rewards1EarnedUsd": "4404.4392311526895494322",
  "rewards2EarnedUsd": "0",
  "kaminoRewards0EarnedUsd": "39648.8949681613351787107",
  "kaminoRewards1EarnedUsd": "1231.803977675133",
  "kaminoRewards2EarnedUsd": "0",
  "totalUsd": "167540.15211747739021203653368504705581",
  "lastCalculated": "2023-04-14T09:52:59.292Z"
}
```

If you want to get a breakdown of all-time Kamino fees and rewards for each strategy, you can use this endpoint instead:

```http request
GET https://api.hubbleprotocol.io/strategies/all-time-fees-and-rewards/breakdown?env={cluster}
```

Example request:

https://api.hubbleprotocol.io/strategies/all-time-fees-and-rewards/breakdown

Example response:

```json
{
  "lastCalculated": "2023-06-27T13:58:16.619Z",
  "strategies": [
    {
      "strategyPubkey": "12iZna9cRnhSY85cDTvas3mav36bYE9WeDuL9uFzH2Zw",
      "feesAEarned": "0.08458176",
      "feesBEarned": "0.00064853",
      "rewards0Earned": "0",
      "rewards1Earned": "0",
      "rewards2Earned": "0",
      "kaminoRewards0Earned": "0",
      "kaminoRewards1Earned": "0",
      "kaminoRewards2Earned": "0",
      "feesAEarnedUsd": "1.492410700820064",
      "feesBEarnedUsd": "1.114493941025",
      "rewards0EarnedUsd": "0",
      "rewards1EarnedUsd": "0",
      "rewards2EarnedUsd": "0",
      "kaminoRewards0EarnedUsd": "0",
      "kaminoRewards1EarnedUsd": "0",
      "kaminoRewards2EarnedUsd": "0",
      "totalUsd": "2.606904641845064"
    },
    {
      "strategyPubkey": "13GDKPRjvK5GWtk8YNTMGj7S4PzhQ6gpQdHJpmCSmWn6",
      "feesAEarned": "0",
      "feesBEarned": "0",
      "rewards0Earned": "0",
      "rewards1Earned": "0",
      "rewards2Earned": "0",
      "kaminoRewards0Earned": "0",
      "kaminoRewards1Earned": "0",
      "kaminoRewards2Earned": "0",
      "feesAEarnedUsd": "0",
      "feesBEarnedUsd": "0",
      "rewards0EarnedUsd": "0",
      "rewards1EarnedUsd": "0",
      "rewards2EarnedUsd": "0",
      "kaminoRewards0EarnedUsd": "0",
      "kaminoRewards1EarnedUsd": "0",
      "kaminoRewards2EarnedUsd": "0",
      "totalUsd": "0"
    }
  ]
}
```

#### Get Kamino TVL:

```http request
GET https://api.hubbleprotocol.io/strategies/tvl?env={cluster}
```

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

Example request:

https://api.hubbleprotocol.io/strategies/tvl?env=mainnet-beta

Example response:

```json
{
  "tvl": "8360625.990205809513007916645145375207353"
}
```

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

#### Get strategy historical ranges

```http request
GET https://api.hubbleprotocol.io/strategies/:strategyPubkey/ranges/history?env={cluster}&period={24h/7d/30d}&start={timestamp}&end={timestamp}
```

Example requests:
  - https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/ranges/history?env=mainnet-beta&start=2020-01-01&end=2024-01-01
  - https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/ranges/history?env=mainnet-beta&period=24h

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
* period: time period for fees and rewards `"24h" (default) | "7d" | "30d"`
* start: start date (inclusive), e.g. `2023-05-01T00:55:00.000Z`
* end: end date (exclusive), e.g. `2023-05-01T00:55:00.000Z`

You can either use a custom range with start/end query params, or a fixed range with period query param.
Fixed range should generally be faster due to caching.

Example response:
```json
[
  {
    "poolPrice": "0.9972366173254501385554518847257254077598",
    "priceLower": "0.9967056034608861435985512952707255012191",
    "priceUpper": "0.9991004498350494870014376094211437370195",
    "oraclePrice": "0.9971830421584632669044453843263363536864",
    "rebalanceParams": {
      "rangePriceLower": "0.9967056034608861435985512952707255012191",
      "rangePriceUpper": "0.9991004498350494870014376094211437370195"
    },
    "date": "2023-06-25T12:00:00.000Z"
  },
  {
    "poolPrice": "0.9972366173254501385554518847257254077598",
    "priceLower": "0.9967056034608861435985512952707255012191",
    "priceUpper": "0.9991004498350494870014376094211437370195",
    "oraclePrice": "0.9971830421584632669044453843263363536864",
    "rebalanceParams": {
      "rangePriceLower": "0.9967056034608861435985512952707255012191",
      "rangePriceUpper": "0.9991004498350494870014376094211437370195"
    },
    "date": "2023-06-25T13:00:00.000Z"
  }
]
```

#### Get strategy metrics history

```http request
GET https://api.hubbleprotocol.io/strategies/:strategyPubkey/metrics/history?env={cluster}&period={24h/7d/30d}&start={timestamp}&end={timestamp}
```

Example requests:
- https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/metrics/history?env=mainnet-beta&start=2020-01-01&end=2024-01-01
- https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/metrics/history?env=mainnet-beta&period=24h

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
* period: time period for fees and rewards `"24h" (default) | "7d" | "30d"`
* start: start date (inclusive), e.g. `2023-05-01T00:00:00.000Z`
* end: end date (exclusive), e.g. `2023-06-01T00:00:00.000Z`

You can either use a custom range with start/end query params, or a fixed range with period query param.
Fixed range should generally be faster due to caching.

Example response:
```json
[
  {
    "date": "2023-06-28T13:00:00.000Z",
    "feesAndRewards24hUsd": "6.892236874",
    "volume24hUsd": "137960909.25357357",
    "cumulativeFeesAndRewardsUsd": "65507.881823738092895",
    "apy24h": "0.001245986450885626907330639945496746179"
  },
  {
    "date": "2023-06-28T12:00:00.000Z",
    "feesAndRewards24hUsd": "6.3897214358",
    "volume24hUsd": "137922148.76631424",
    "cumulativeFeesAndRewardsUsd": "65507.881823738092895",
    "apy24h": "0.001186575692116333812242675851871094793"
  }
]
```

#### Get strategy shareholders history

```http request
GET https://api.hubbleprotocol.io/strategies/:strategyPubkey/shareholders/history?env={cluster}&start={start}&end={end}&frequency={frequency}
```

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
* start: start date (inclusive), e.g. `2023-05-01T00:00:00.000Z` (optional, default since beginning of strategy)
* end: end date (exclusive), e.g. `2023-05-02T00:00:00.000Z` (optional, default now)
* frequency: frequency of the snapshots, e.g. `"hour" (default) | "day"`

Example requests:

- https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/history?env=mainnet-beta
- https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/history?env=mainnet-beta&start=2023-01-01&end=2023-02-01&frequency=hour
- https://api.hubbleprotocol.io/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/history?env=mainnet-beta&start=2023-01-01&end=2023-02-01&frequency=day


Example response:

```json
[
    {
        "timestamp": "2023-06-06T11:00:00.000Z",
        "shareholders": [
            {
                "sharesAmount": "25215.549995",
                "sharePrice": "1.067670377475403913338573081893682618365",
                "sharesUsd": "26921.89578141156925960743690845138333804",
                "tokenAAmount": "820.4214362237481540135710513269",
                "tokenAPrice": "19.98425069",
                "tokenAUsd": "16395.507652945230041231933551843626740561",
                "tokenBAmount": "0.408879105380424526340758204",
                "tokenBPrice": "25744.5",
                "tokenBUsd": "10526.388128466339218379649582878",
                "wallet": "7tH1k4PsMu3sNUYJxD5ezxhAmYYXyVVZz5c3dbaxcvUV"
            },
            {
                "sharesAmount": "0.853582",
                "sharePrice": "1.067670377475403913338573081893682618365",
                "sharesUsd": "0.9113442161462102231553658883889733967492",
                "tokenAAmount": "0.027772424972431754280993115411983",
                "tokenAPrice": "19.98425069",
                "tokenAUsd": "0.55501110291829251646784712055717090201827",
                "tokenBAmount": "0.00001384113551352396460160423028",
                "tokenBPrice": "25744.5",
                "tokenBUsd": "0.35633311322791770668600010644346",
                "wallet": "BUfcRmRPQNqMbPSKh6a7Pp3tSTneNMt2zWYr5DefW5hd"
            }
        ]
    }
]
```

#### Get strategies start date overrides

```http request
GET https://api.hubbleprotocol.io/strategies/start-date-overrides?env={cluster}
```

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

Example request:

- https://api.hubbleprotocol.io/strategies/start-date-overrides?env=mainnet-beta


Example response:

```json
[
  {
    "strategy": "Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN",
    "start": "2022-09-01T00:00:00.000Z"
  },
  {
    "strategy": "ByXB4xCxVhmUEmQj3Ut7byZ1Hbva1zhKjaVcv3jBMN7E",
    "start": "2022-09-01T00:00:00.000Z"
  },
  {
    "strategy": "98kNMp1aqWoYAaUU8m5REBAYVwhFb4aX9yoSpgq8kUFu",
    "start": "2022-09-01T00:00:00.000Z"
  }
]
```

### Whirlpools Kamino

You may use the `env` query param for all the methods specified below (`mainnet-beta`[default],`devnet`,`localnet`,`testnet`).

#### Get Orca whirlpool history for a specific year (default current year, or use query param `year`):

```http request
// GET https://api.hubbleprotocol.io/whirlpools/:whirlpool_pubkey/history?env={cluster}&year={year}
GET https://api.hubbleprotocol.io/whirlpools/BabJ4KTDUDqaBRWLFza3Ek3zEcjXaPDmeRGRwusQyLPS/history?env=mainnet-beta&year=2022
```

#### Get whirlpools TVL

Get all unique Orca/Raydium pools from Kamino strategies and return their TVL: 

```http request
GET https://api.hubbleprotocol.io/whirlpools/tvl?env={cluster}
```

Example request:

https://api.hubbleprotocol.io/whirlpools/tvl

Response example:
```json
[
    {
        "tvl": "10628.86524130715",
        "pool": "3BScXnPjT4hut1G5yJ5UGQWhUmoYxyBFQf3juLBeMH2S"
    },
    {
        "tvl": "522635.86139358365",
        "pool": "4nFbdT7DeXATvaRZfR3WqALGJnogMjqe9vf2H6C1WXBr"
    }
]
```

#### Get whirlpool fees

Get all unique Orca/Raydium pools from Kamino strategies and return their fees:

```http request
GET https://api.hubbleprotocol.io/whirlpools/fees?env={cluster}
```

Example request:

https://api.hubbleprotocol.io/whirlpools/fees

Response example:
```json
[
  {
    "fees": [
      {
        "period": "24h",
        "amount": "0.13107745873661961"
      },
      {
        "period": "7d",
        "amount": "0.4323762174032862"
      },
      {
        "period": "30d",
        "amount": "0.52013355740328621"
      }
    ],
    "pool": "H1fREbTWrkhCs2stH3tKANWJepmqeF9hww4nWRYrM7uV"
  },
  {
    "fees": [
      {
        "period": "24h",
        "amount": "437.0303313444236"
      },
      {
        "period": "7d",
        "amount": "2578.2959811239195"
      },
      {
        "period": "30d",
        "amount": "7032.52314803323"
      }
    ],
    "pool": "BVXNG6BrL2Tn3NmppnMeXHjBHTaQSnSnLE99JKwZSWPg"
  }
]
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
GET https://api.hubbleprotocol.io/prices?env={cluster}&source={priceSource:scope(default)|birdeye}
```

Example request: https://api.hubbleprotocol.io/prices?env=mainnet-beta&source=scope
Example response:

```json
[
    {
        "usdPrice": "22.382246552631578",
        "token": "SOL",
        "mint": "So11111111111111111111111111111111111111112"
    },
    {
        "usdPrice": "1580.8847893915756",
        "token": "ETH",
        "mint": "7vfCXTUXx5WJV5JADk17DUJ4ksgau7utNKj4b963voxs"
    }
]
```


#### Get specific token price:

```http request
GET https://api.hubbleprotocol.io/prices?env={cluster}&source={priceSource:scope(default)|birdeye}&token={tokenName or mint}
```

Example requests:
- https://api.hubbleprotocol.io/prices?env=mainnet-beta&source=scope&token=SOL
- https://api.hubbleprotocol.io/prices?env=mainnet-beta&source=scope&token=So11111111111111111111111111111111111111112
Example response:

```json
[
    {
        "usdPrice": "22.382246552631578",
        "token": "SOL",
        "mint": "So11111111111111111111111111111111111111112"
    }
]
```

#### Get token price history:

```http request
GET https://api.hubbleprotocol.io/prices/history?env={cluster}&token={name or mint}&start={start}&end={end}&frequency={frequency}&type={type}
```

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
* token: name (deprecated soon) or mint (**use of mint recommended!**) pubkey, e.g. `"SOL"` or `"So11111111111111111111111111111111111111112"`
* start: start date (inclusive), e.g. `2023-05-01T00:55:00.000Z` (optional, default 1 day ago) 
* end: end date (inclusive), e.g. `2023-05-01T00:55:00.000Z` (optional, default now)
* frequency: frequency of the prices, e.g. `"minute" (default) | "hour" | "day"` for price every minute/hour/day for the specified timeseries
* type: price type, e.g. `"spot" (default) | "TWAP"`


Example requests:

- https://api.hubbleprotocol.io/prices/history?env=mainnet-beta&token=USDH1SM1ojwWUga67PGrgFWUHibbjqMvuMaDkRJTgkX
- https://api.hubbleprotocol.io/prices/history?env=mainnet-beta&token=So11111111111111111111111111111111111111112&start=2020-01-01&end=2023-01-01&frequency=hour&type=spot
- https://api.hubbleprotocol.io/prices/history?env=mainnet-beta&token=USDH1SM1ojwWUga67PGrgFWUHibbjqMvuMaDkRJTgkX&type=TWAP
- https://api.hubbleprotocol.io/prices/history?env=mainnet-beta&token=So11111111111111111111111111111111111111112&start=2020-01-01&end=2023-01-01&frequency=day
- (not recommended, deprecated soon): https://api.hubbleprotocol.io/prices/history?env=mainnet-beta&token=USDH


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
        "source": "birdeye"
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

### Kamino Lending

#### Get All Kamino Markets config

:warning: **DEPRECATED, PLEASE USE /v2/kamino-market/ INSTEAD!** :warning:

```http request
GET https://api.hubbleprotocol.io/kamino-market?env={cluster}
```
Example: 
https://api.hubbleprotocol.io/kamino-market?env=mainnet-beta

#### Get Kamino Market config

:warning: **DEPRECATED, PLEASE USE /v2/kamino-market/:marketPubkey INSTEAD!** :warning:

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey?env={cluster}
```
Example: 
https://api.hubbleprotocol.io/kamino-market/J5ndTP1GJe6ZWzGiZQR2UKJmWWMJojbWxCxZ2yUXwakR?env=mainnet-beta

#### Get All Kamino Markets config V2

```http request
GET https://api.hubbleprotocol.io/kamino-market?programId={programId}
```
Example: 
https://api.hubbleprotocol.io/kamino-market?programId=KLend2g3cP87fffoy8q1mQqGKjrxjC8boSyAYavgmjD

#### Get Kamino Market config V2

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey?programId={programId}
```
Example: 
https://api.hubbleprotocol.io/kamino-market/6WVSwDQXrBZeQVnu6hpnsRZhodaJTZBUaC334SiiBKdb?programId=SLendK7ySfcEzyaFqy93gDnD3RtrpXJcnRwb6zFHJSh

#### Get KLend Market metrics history

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/metrics/history?env={cluster}&start={date}&end={date}'
```
Example: https://api.hubbleprotocol.io/kamino-market/9pMFoVgsG2cNiUCSBEE69iWFN7c1bz9gu9TtPeXkAMTs/metrics/history?env=mainnet-beta&start=2023-01-01&end=2023-01-02

Example response:

```json
[
  {
    "market": "9pMFoVgsG2cNiUCSBEE69iWFN7c1bz9gu9TtPeXkAMTs",
    "timestamp": "2023-06-29T15:15:26.464Z",
    "metrics": { "tvl": 234.14377328764223, "obligations": 38 }
  },
  {
    "market": "9pMFoVgsG2cNiUCSBEE69iWFN7c1bz9gu9TtPeXkAMTs",
    "timestamp": "2023-06-29T16:50:08.601Z",
    "metrics": { "tvl": "234.14377328764223401824", "obligations": 38 }
  },
  {
    "market": "9pMFoVgsG2cNiUCSBEE69iWFN7c1bz9gu9TtPeXkAMTs",
    "timestamp": "2023-06-30T18:02:39.430Z",
    "metrics": { "tvl": "234.14377328764223401824", "obligations": 38 }
  }
]
```

#### Get KLend reserve history

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/reserves/:reservePubkey/metrics/history?env={cluster}&start={date}&end={date}&frequency={frequency}'
```

Query params:

* env: solana cluster, e.g. "mainnet-beta" (default) | "devnet"
* start: start date (inclusive), e.g. 2023-05-01T00:00:00.000Z (optional, default since beginning of strategy)
* end: end date (exclusive), e.g. 2023-05-02T00:00:00.000Z (optional, default now)
* frequency: frequency of the snapshots, e.g. "hour" (default) | "day"

Example: https://api.hubbleprotocol.io/kamino-market/9pMFoVgsG2cNiUCSBEE69iWFN7c1bz9gu9TtPeXkAMTs/reserves/HcHkvZEDechu7bruV8zvMN11v9yHg3iY4QHNgrYUATmm/metrics/history?env=mainnet-beta&start=2023-01-01&end=2023-01-02&frequency=hour

```json
{
  "reserve": "5ZQCZt4UV3b7SzR3nPRgqbFE1XAUmo1yVfNanGM7qFVj",
  "market": "9pMFoVgsG2cNiUCSBEE69iWFN7c1bz9gu9TtPeXkAMTs",
  "history": [
    {
      "metrics": {
        "tvl": "21.58894844114247816444",
        "symbol": "USDH",
        "decimals": 6,
        "borrowCurve": [
          [0, 0.0001],
          [0.7, 0.1],
          [1, 1.5]
        ],
        "mintAddress": "USDH1SM1ojwWUga67PGrgFWUHibbjqMvuMaDkRJTgkX",
        "borrowFactor": 0,
        "exchangeRate": 0.9749784180009508,
        "assetPriceUSD": 0.9971944372197178,
        "mintTotalSupply": "21.833854",
        "totalSupplyWads": "22.411148742143389807977042",
        "liquidationBonus": 0,
        "loanToValueRatio": 0.75,
        "protocolTakeRate": 0.1,
        "totalBorrowsWads": "0.761460742143389807977042",
        "utilizationRatio": 0.033976872444360214,
        "borrowInterestAPY": 0.004961253732466142,
        "supplyInterestAPY": 0.00016816974038857957,
        "reserveBorrowLimit": "10000000000",
        "totalLiquidityWads": "21.649688",
        "borrowFeePercentage": 0.001,
        "reserveDepositLimit": "10000000000",
        "liquidationThreshold": 0.8,
        "borrowLimitCrossedSlot": 0,
        "flashLoanFeePercentage": 0.003,
        "accumulatedProtocolFees": "0.01695662785321651304",
        "depositLimitCrossedSlot": 0,
        "cumulativeBorrowRateWads": "1033653581204157935"
      }
    }
  ]
}
```

#### Get KLend obligation history

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/obligations/:obligationPubkey/metrics/history?env={cluster}&start={date}&end={date}&frequency={frequency}'
```

Query params:

* env: solana cluster, e.g. "mainnet-beta" (default) | "devnet"
* start: start date (inclusive), e.g. 2023-05-01T00:00:00.000Z (optional, default since beginning of strategy)
* end: end date (exclusive), e.g. 2023-05-02T00:00:00.000Z (optional, default now)
* frequency: frequency of the snapshots, e.g. "hour" (default) | "day"

Example: https://api.hubbleprotocol.io/kamino-market/9pMFoVgsG2cNiUCSBEE69iWFN7c1bz9gu9TtPeXkAMTs/obligations/63QrAB1okxCc4FpsgcKYHjYTp1ua8ch6mLReyKRdc22o/metrics/history?env=mainnet-beta&start=2023-01-01&end=2023-01-02&frequency=day

Example response:

```json
{
  "obligation": "63QrAB1okxCc4FpsgcKYHjYTp1ua8ch6mLReyKRdc22o",
  "market": "9pMFoVgsG2cNiUCSBEE69iWFN7c1bz9gu9TtPeXkAMTs",
  "history": [
    {
      "timestamp": "2023-07-11T09:51:40.498Z",
      "stats": {
        "leverage": 1.8849400779844114,
        "positions": 2,
        "borrowLimit": 0.20884206942858852,
        "loanToValue": 0.46947915656326233,
        "netAccountValue": 0.14772676109110466,
        "userTotalBorrow": 0.13072933148034668,
        "userTotalDeposit": 0.2784560925714514,
        "borrowUtilization": 0.6259722087510164,
        "liquidationThreshold": 0.22276487405716108
      },
      "deposits": [
        {
          "amount": "11442889",
          "marketValue": "0.27845609257145136",
          "mintAddress": "mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So"
        }
      ],
      "borrows": [
        {
          "amount": "6024369",
          "marketValue": "0.13072933148034669",
          "mintAddress": "So11111111111111111111111111111111111111112"
        }
      ],
      "tag": 1
    }
  ]
}
```

#### Get KLend tx history per user (for all obligations)

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/users/:userPubkey/transactions/
```

Example response:

```json
{
  "allTransactions": {
    "8th5rmiEfHUdPRs1fFRzxisjhHMTPmotAeBsAGGvFGG": {
      "transactions": [
        {
          "createdOn": "2023-05-12T11:41:31.818Z",
          "timestamp": 1683891691818,
          "transactionSignature": "5xtF68Ra53aqYvrCZnXQ2bRVUaR2BiiKLvSsZhEXLr2F2fRbeJMR6FaxWCDRcxUzetBrod8i7kre3651L7ohxRDL",
          "transactionName": "initObligation",
          "reserve": "",
          "liquidityToken": "",
          "liquidityTokenAmount": "0",
          "liquidityTokenPrice": "0",
          "solPrice": "0",
          "liquidityUsdValue": "0",
          "isLiquidationWithdrawal": false
        },
        {
          "createdOn": "2023-05-12T11:41:31.818Z",
          "timestamp": 1683891691818,
          "transactionSignature": "5xtF68Ra53aqYvrCZnXQ2bRVUaR2BiiKLvSsZhEXLr2F2fRbeJMR6FaxWCDRcxUzetBrod8i7kre3651L7ohxRDL",
          "transactionName": "depositReserveLiquidityAndObligationCollateral",
          "reserve": "455pvamji39kgFkiZScbLFnqFuReFQJLvv8pd1nmzfRp",
          "liquidityToken": "JITOSOL",
          "liquidityTokenAmount": "2",
          "liquidityTokenPrice": "20.99153922345559845933295",
          "solPrice": "20.28813173",
          "liquidityUsdValue": "41.9830784469111969186659",
          "isLiquidationWithdrawal": false
        },
        {
          "createdOn": "2023-05-12T11:41:31.818Z",
          "timestamp": 1683891691818,
          "transactionSignature": "5xtF68Ra53aqYvrCZnXQ2bRVUaR2BiiKLvSsZhEXLr2F2fRbeJMR6FaxWCDRcxUzetBrod8i7kre3651L7ohxRDL",
          "transactionName": "initObligation",
          "reserve": "",
          "liquidityToken": "",
          "liquidityTokenAmount": "0",
          "liquidityTokenPrice": "0",
          "solPrice": "0",
          "liquidityUsdValue": "0",
          "isLiquidationWithdrawal": false
        },
        {
          "createdOn": "2023-05-12T11:41:31.818Z",
          "timestamp": 1683891691818,
          "transactionSignature": "5xtF68Ra53aqYvrCZnXQ2bRVUaR2BiiKLvSsZhEXLr2F2fRbeJMR6FaxWCDRcxUzetBrod8i7kre3651L7ohxRDL",
          "transactionName": "depositReserveLiquidityAndObligationCollateral",
          "reserve": "455pvamji39kgFkiZScbLFnqFuReFQJLvv8pd1nmzfRp",
          "liquidityToken": "JITOSOL",
          "liquidityTokenAmount": "2",
          "liquidityTokenPrice": "20.99153922345559845933295",
          "solPrice": "20.28813173",
          "liquidityUsdValue": "41.9830784469111969186659",
          "isLiquidationWithdrawal": false
        }
      ],
      "lastUpdatedOn": "2023-08-04T06:55:07.391Z"
    },
    "8th5rmiEfHUdPRs1fFRzxisjhHMTPmotAeBsAGGvFGJ": {
      "transactions": [
        {
          "createdOn": "2023-07-26T03:10:32.190Z",
          "timestamp": 1690341032190,
          "transactionSignature": "5xtF68Ra53aqYvrCZnXQ2bRVUaR2BiiKLvSsZhEXLr2F2fRbeJMR6FaxWCDRcxUzetBrod8i7kre3651L7ohxRDL",
          "transactionName": "initObligation",
          "reserve": "",
          "liquidityToken": "",
          "liquidityTokenAmount": "0",
          "liquidityTokenPrice": "0",
          "solPrice": "0",
          "liquidityUsdValue": "0",
          "isLiquidationWithdrawal": false
        },
        {
          "createdOn": "2023-07-28T00:03:15.368Z",
          "timestamp": 1690502595368,
          "transactionSignature": "5xtF68Ra53aqYvrCZnXQ2bRVUaR2BiiKLvSsZhEXLr2F2fRbeJMR6FaxWCDRcxUzetBrod8i7kre3651L7ohxRDL",
          "transactionName": "depositReserveLiquidityAndObligationCollateral",
          "reserve": "FdSLxHXZLDrAxTwQ9ztWBw6Djueu6KbCvViJvf1Ck5Av",
          "liquidityToken": "BSOL",
          "liquidityTokenAmount": "2",
          "liquidityTokenPrice": "0",
          "solPrice": "25.07601599",
          "liquidityUsdValue": "0",
          "isLiquidationWithdrawal": false
        },
        {
          "createdOn": "2023-07-26T03:10:32.190Z",
          "timestamp": 1690341032190,
          "transactionSignature": "5xtF68Ra53aqYvrCZnXQ2bRVUaR2BiiKLvSsZhEXLr2F2fRbeJMR6FaxWCDRcxUzetBrod8i7kre3651L7ohxRDL",
          "transactionName": "depositReserveLiquidityAndObligationCollateral",
          "reserve": "FdSLxHXZLDrAxTwQ9ztWBw6Djueu6KbCvViJvf1Ck5Av",
          "liquidityToken": "BSOL",
          "liquidityTokenAmount": "0.396843479",
          "liquidityTokenPrice": "0",
          "solPrice": "23.62",
          "liquidityUsdValue": "0",
          "isLiquidationWithdrawal": false
        },
        {
          "createdOn": "2023-07-26T03:10:32.190Z",
          "timestamp": 1690341032190,
          "transactionSignature": "5xtF68Ra53aqYvrCZnXQ2bRVUaR2BiiKLvSsZhEXLr2F2fRbeJMR6FaxWCDRcxUzetBrod8i7kre3651L7ohxRDL",
          "transactionName": "initObligation",
          "reserve": "",
          "liquidityToken": "",
          "liquidityTokenAmount": "0",
          "liquidityTokenPrice": "0",
          "solPrice": "0",
          "liquidityUsdValue": "0",
          "isLiquidationWithdrawal": false
        },
        {
          "createdOn": "2023-07-28T00:03:15.368Z",
          "timestamp": 1690502595368,
          "transactionSignature": "5xtF68Ra53aqYvrCZnXQ2bRVUaR2BiiKLvSsZhEXLr2F2fRbeJMR6FaxWCDRcxUzetBrod8i7kre3651L7ohxRDL",
          "transactionName": "depositReserveLiquidityAndObligationCollateral",
          "reserve": "FdSLxHXZLDrAxTwQ9ztWBw6Djueu6KbCvViJvf1Ck5Av",
          "liquidityToken": "BSOL",
          "liquidityTokenAmount": "2",
          "liquidityTokenPrice": "0",
          "solPrice": "25.07601599",
          "liquidityUsdValue": "0",
          "isLiquidationWithdrawal": false
        },
        {
          "createdOn": "2023-07-26T03:10:32.190Z",
          "timestamp": 1690341032190,
          "transactionSignature": "5xtF68Ra53aqYvrCZnXQ2bRVUaR2BiiKLvSsZhEXLr2F2fRbeJMR6FaxWCDRcxUzetBrod8i7kre3651L7ohxRDL",
          "transactionName": "depositReserveLiquidityAndObligationCollateral",
          "reserve": "FdSLxHXZLDrAxTwQ9ztWBw6Djueu6KbCvViJvf1Ck5Av",
          "liquidityToken": "BSOL",
          "liquidityTokenAmount": "0.396843479",
          "liquidityTokenPrice": "0",
          "solPrice": "23.62",
          "liquidityUsdValue": "0",
          "isLiquidationWithdrawal": false
        }
      ],
      "lastUpdatedOn": "2023-08-04T06:55:04.134Z"
    },
    "8th5rmiEfHUdPRs1fFRzxisjhHMTPmotAeBsAGGvFGH": {
      "transactions": [
        {
          "createdOn": "2023-05-12T12:10:48.695Z",
          "timestamp": 1683893448695,
          "transactionSignature": "5xtF68Ra53aqYvrCZnXQ2bRVUaR2BiiKLvSsZhEXLr2F2fRbeJMR6FaxWCDRcxUzetBrod8i7kre3651L7ohxRDL",
          "transactionName": "initObligation",
          "reserve": "",
          "liquidityToken": "",
          "liquidityTokenAmount": "0",
          "liquidityTokenPrice": "0",
          "solPrice": "0",
          "liquidityUsdValue": "0",
          "isLiquidationWithdrawal": false
        },
        {
          "createdOn": "2023-07-23T13:45:58.881Z",
          "timestamp": 1690119958881,
          "transactionSignature": "5xtF68Ra53aqYvrCZnXQ2bRVUaR2BiiKLvSsZhEXLr2F2fRbeJMR6FaxWCDRcxUzetBrod8i7kre3651L7ohxRDL",
          "transactionName": "depositReserveLiquidityAndObligationCollateral",
          "reserve": "7SsmK3wj4V6cg4hqKFrSAdRYirUU5UyJPY2oQ2Bjgvcw",
          "liquidityToken": "STSOL",
          "liquidityTokenAmount": "-0.002682361",
          "liquidityTokenPrice": "27.4789254927168",
          "solPrice": "24.57688671",
          "liquidityUsdValue": "-0.0737083980635693283648",
          "isLiquidationWithdrawal": false
        }
      ],
      "lastUpdatedOn": "2023-07-23T13:46:56.790"
    }
  }
}
```

#### Get KLend tx history per obligation

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/obligations/:obligationPubkey/transactions/
```

Example response:

```json
{
  "transactions": [
    {
      "createdOn": "2023-07-24T06:30:53.785Z",
      "timestamp": 1690180253785,
      "transactionSignature": "5AHBE2oupPNnwQnpzHSqyfW2xMKhkvR8bduBDqbRtuuLRKRMhscKAPv8pwxx75Na6KNMJUCKvhw7EXDz9XLnc4Xp",
      "transactionName": "depositReserveLiquidityAndObligationCollateral",
      "reserve": "3zqgH952VPgRWw9Xwa9q9Smt4cPwPaF5LTqHGGxYEYgi",
      "kTokenAmount": "0",
      "liquidityToken": "USDC",
      "liquidityTokenAmount": "1",
      "liquidityTokenPrice": "0.9999",
      "solPrice": "24.3694875",
      "liquidityUsdValue": "0.9999"
    },
    {
      "createdOn": "2023-07-24T06:10:43.489Z",
      "timestamp": 1690179043489,
      "transactionSignature": "2MRCGnGQcgRmzgkmp48C4ngkSwUxFSXujEVJXt4zxjb3BJ23i3QEtcZTVpe396nULHKBNXvbiGwCmkSyojLvRwgi",
      "transactionName": "depositReserveLiquidityAndObligationCollateral",
      "reserve": "3zqgH952VPgRWw9Xwa9q9Smt4cPwPaF5LTqHGGxYEYgi",
      "kTokenAmount": "0",
      "liquidityToken": "USDC",
      "liquidityTokenAmount": "2",
      "liquidityTokenPrice": "0.9999",
      "solPrice": "24.3694875",
      "liquidityUsdValue": "1.9998"
    }
  ]
}
```

#### Get PnL per obligation

You can specify pnl mode with query param `positionMode` with one of these values: {`obligation_all_time`, `current_obligation`}. By default, pnl mode is set to `current_obligation` position

These parameters specify whether we return the current position PnL or the accumulated PnL throughout the lifetime of the obligation (even before it was closed).

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/obligations/:obligationPubkey/pnl/?positionMode=current_obligation
```

Example response:

```json
{
  "usd": "25.21",
  "sol": "1.0"
}
```

#### Get PnL per user

You can specify pnl mode with query param `positionMode` with one of these values: {`user_all_time`, `user_all_current_positions`}. By default, pnl mode is set to `user_all_current_positions` position

These parameters specify whether we return the current position PnL or the accumulated PnL throughout the lifetime of the user's obligation (even before they were closed).

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/users/:userPubkey/pnl/?positionMode=user_all_current_positions
```

#### Get interest fees earned per obligation

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/obligations/:obligationPubkey/interest-fees/
```

Query params:

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- start: start date (inclusive), e.g. `2023-05-01T00:55:00.000Z`
- end: end date (exclusive), e.g. `2023-05-01T00:55:00.000Z`
- frequency: frequency of the snapshots, e.g. `"hour" (default) | "day"`
- positionMode: position mode, e.g. `"obligation_all_time" | "current_obligation (default)"`

Example response:

```json
{
  "totalFeesEarnedObligation": {
    "5QvxCCLUnmxgoAF8kuUsyPjKd6RV2XPGivsqq773TzjK": {
      "ts": 1695200400000,
      "solFees": "0.06313669856513711976206400769604439360579",
      "usdFees": "1.269225847264109025064706147202735158871",
      "nativeFees": "1.261001541373388353921537172595541265214"
    }
  },
  "feesObligation": {
    "5QvxCCLUnmxgoAF8kuUsyPjKd6RV2XPGivsqq773TzjK": [
      {
        "USDH": { "ts": 1695117600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" },
        "STSOL": { "ts": 1695117600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" },
        "UXD": { "ts": 1695117600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" },
        "USDC": { "ts": 1695117600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" },
        "MSOL": { "ts": 1695117600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" },
        "SOL": { "ts": 1695117600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" },
        "JITOSOL": { "ts": 1695117600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" },
        "kUXDUSDCOrca": { "ts": 1695117600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" }
      },
      {
        "USDH": {
          "ts": 1695200400000,
          "solFees": "0.009127930668530089532008568492476558595633",
          "usdFees": "0.1835061678794232293068324983297403016265",
          "nativeFees": "0.1840858688969978660085239011713265619735"
        },
        "STSOL": { "ts": 1695200400000, "solFees": "0", "usdFees": "0", "nativeFees": "0" },
        "UXD": { "ts": 1695200400000, "solFees": "0", "usdFees": "0", "nativeFees": "0" },
        "USDC": {
          "ts": 1695200400000,
          "solFees": "0.05354646057380662740920466718071105954604",
          "usdFees": "1.076487775841993410013844914006083239768",
          "nativeFees": "1.07645428734911397907795579880118253498"
        },
        "MSOL": {
          "ts": 1695200400000,
          "solFees": "0.0000000002729669118503462954743437138234390588713",
          "usdFees": "0.00000000548767445443401462031404881729929319511",
          "nativeFees": "0.0000000002409057587721427566924936757882938254622"
        },
        "SOL": {
          "ts": 1695200400000,
          "solFees": "0.00004161517214718125325688557874458751446175",
          "usdFees": "0.0008366234411376726934711156672714631160833",
          "nativeFees": "0.00004161517214718125325688557874458751446176"
        },
        "JITOSOL": {
          "ts": 1695200400000,
          "solFees": "0.00001637958277767565995674794591594066633801",
          "usdFees": "0.0003292919913773938484626639982754417744366",
          "nativeFees": "0.00001545746301911876686717846158351857411716"
        },
        "kUXDUSDCOrca": { "ts": 1695200400000, "solFees": "0", "usdFees": "0", "nativeFees": "0" }
      }
    ]
  }
}
```

#### Get interest fees earned per user

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/users/:userPubkey/interest-fees/
```

Query params:

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- start: start date (inclusive), e.g. `2023-05-01T00:55:00.000Z`
- end: end date (exclusive), e.g. `2023-05-01T00:55:00.000Z`
- frequency: frequency of the snapshots, e.g. `"hour" (default) | "day"`
- positionMode: position mode, e.g. `"user_all_time" | "user_all_current_positions (default)"`

Example response:

```json
{
  "totalFeesEarnedObligation": {
    "MX68wMfQRkQA13SrKnSX47wwjiV2TDBak7NMb4Qfg5g": {
      "ts": 1695200400000,
      "solFees": "0.000000001697888184902321902172852546261584381586",
      "usdFees": "0.0000000341340184992168245010582859455671467486",
      "nativeFees": "0.000000001602302340966625236782497000842808190647"
    },
    "598TqTp5HfFQcNwWhgXyxE5zFYgFUYnRJSAADdfVXm2N": {
      "ts": 1695200400000,
      "solFees": "0.00001252855751062609254654040779155507638912",
      "usdFees": "0.0002501232156778048715929059083302644728593",
      "nativeFees": "0.00001252855751062609254654040779155507638912"
    }
  },
  "feesObligation": {
    "MX68wMfQRkQA13SrKnSX47wwjiV2TDBak7NMb4Qfg5g": [
      { "JITOSOL": { "ts": 1695117600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695121200000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695124800000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695128400000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695132000000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695135600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695139200000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695142800000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695146400000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695150000000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695153600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695157200000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695160800000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695164400000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695168000000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695171600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695175200000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695178800000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695182400000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695186000000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695189600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695193200000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "JITOSOL": { "ts": 1695196800000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      {
        "JITOSOL": {
          "ts": 1695200400000,
          "solFees": "0.000000001697888184902321902172852546261584381586",
          "usdFees": "0.0000000341340184992168245010582859455671467486",
          "nativeFees": "0.000000001602302340966625236782497000842808190647"
        }
      }
    ],
    "598TqTp5HfFQcNwWhgXyxE5zFYgFUYnRJSAADdfVXm2N": [
      { "SOL": { "ts": 1695117600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695121200000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695124800000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      {
        "SOL": {
          "ts": 1695128400000,
          "solFees": "0.000001799717797858716590338358835351733653603",
          "usdFees": "0.00003596415275114428523179196620477162156107",
          "nativeFees": "0.000001799717797858716590338358835351733653604"
        }
      },
      { "SOL": { "ts": 1695132000000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695135600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695139200000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695142800000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695146400000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695150000000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695153600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695157200000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695160800000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695164400000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      {
        "SOL": {
          "ts": 1695168000000,
          "solFees": "0.000006064101667947088381984925284168149623453",
          "usdFees": "0.0001209912293028143483368695347715114560122",
          "nativeFees": "0.000006064101667947088381984925284168149623456"
        }
      },
      { "SOL": { "ts": 1695171600000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695175200000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695178800000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695182400000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      { "SOL": { "ts": 1695186000000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      {
        "SOL": {
          "ts": 1695189600000,
          "solFees": "0.000003084857420134046011686340063997035617416",
          "usdFees": "0.00006146691096373786754851793467830291215417",
          "nativeFees": "0.000003084857420134046011686340063997035617417"
        }
      },
      {
        "SOL": {
          "ts": 1695193200000,
          "solFees": "0.0000004106794248750007481729916324997687533188",
          "usdFees": "0.000008195525933433248949303575906066025362104",
          "nativeFees": "0.0000004106794248750007481729916324997687533189"
        }
      },
      { "SOL": { "ts": 1695196800000, "solFees": "0", "usdFees": "0", "nativeFees": "0" } },
      {
        "SOL": {
          "ts": 1695200400000,
          "solFees": "0.000001169201199811240814357791975538388741346",
          "usdFees": "0.00002350539672667512152642289676961245776999",
          "nativeFees": "0.000001169201199811240814357791975538388741347"
        }
      }
    ]
  }
}
```

#### Get metrics for market reserves

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/reserves/metrics
```

Query params:

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

Example response:

```json
[
    {
        "reserve": "d4A2prbA2whesmvHaL88BH6Ewn5N4bTSU2Ze8P6Bc4Q",
        "liquidityToken": "SOL",
        "liquidityTokenMint": "So11111111111111111111111111111111111111112",
        "maxLtv": "0.65",
        "borrowApy": "0.05450988511483601",
        "supplyApy": "0.038266801210808055",
        "totalSupply": "3270.370813041054416388850832278062044876",
        "totalBorrow": "2314.442149604473429941074038490263985018",
        "totalBorrowUsd": "132328.2299036357776146495065685745631077",
        "totalSupplyUsd": "186983.4512356222993385157984997158629711"
    },
    {
        "reserve": "G31zKdH2SkDZPhmoQraep5xbTSPyk3VZxAeBdC3nmq5J",
        "liquidityToken": "STEP",
        "liquidityTokenMint": "StepAscQoEioFxxWGnh2sLBDFp9d8rvKz2Yp39iDpyT",
        "maxLtv": "0",
        "borrowApy": "0.00009999925246972907",
        "supplyApy": "0",
        "totalSupply": "14.957139738",
        "totalBorrow": "0",
        "totalBorrowUsd": "0",
        "totalSupplyUsd": "0.5941041438503336806162179231"
    }
]
```

### Trades

#### Get trade history

`POST https://api.hubbleprotocol.io/trades?env=mainnet-beta&source=hellomoon`

Example POST request:

```shell
# obtain the first page of 10 trades
curl --location 'https://api.hubbleprotocol.io/trades?env=mainnet-beta&source=hellomoon' --header 'Content-Type: application/json' --data '{
    "tokenAMint": "So11111111111111111111111111111111111111112",
    "tokenBMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "start": "2023-01-01T00:00Z",
    "end": "2023-02-01T00:00Z"
}'
# use "paginationToken" property in response in the next request to get next 10 trades:
curl --location 'https://api.hubbleprotocol.io/trades?env=mainnet-beta&source=hellomoon' --header 'Content-Type: application/json' --data '{
    "tokenAMint": "So11111111111111111111111111111111111111112",
    "tokenBMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "start": "2023-01-01T00:00Z",
    "end": "2023-02-01T00:00Z",
    "paginationToken": "MzQ3OTQzMQ=="
}' 
```

Query params:
  * env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
  * source: trades fetched from source`"hellomoon" (default)`

Body params:
  * tokenAMint: public key of the first mint, e.g. SOL mint: `"So11111111111111111111111111111111111111112"`
  * tokenBMint: public key of the second mint, e.g. USDC mint: `"EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"`
  * start: start of the date range to fetch trades from, e.g. date ISO string: `"2023-01-01T00:00Z"` or epoch in ms: `1678381747854`
  * end: end of the date range to fetch trades to, e.g. date ISO string: `"2023-01-01T00:00Z"` or epoch in ms: `1678381747854`
  * paginationToken: pagination token to use for retrieving results. 
  If the response contains a `paginationToken` JSON property, 
  you can use that in the next request to fetch more data from the last trade onwards.
  If the property does not exist, you've reached the end.

Please note that the response will contain both "directions" (buys, sells) of the trade.
For example, if you input tokenA/tokenB mints for SOL/USDC in the request body, 
the response will contain trades with (source = SOL, destination = USDC) or (source = USDC, destination = SOL).

Example response: 

```json
{
    "trades": [
        {
            "transactionId": "67enaV7ufBGu5quuzZFwn4B2kqpNiqRunAwXbytt6TCkVkEJzUmNRyCYnDPvPi8yP8aDpHL16oVecqrqa3rvvbv9",
            "sourceAmount": "30.202777503",
            "destinationAmount": "704.59445",
            "tradedOn": "2023-01-16T12:54:49.000Z",
            "aggregator": "Jupiter v3",
            "programId": "whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3uctyCc",
            "sourceMint": "So11111111111111111111111111111111111111112",
            "destinationMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
        },
        {
            "transactionId": "4rZRGF8P667EjprBZomP7MiwHhUDwa6Dnuw2i5MvLn8jbVfVdzBg9DcvoHpkuykrRJpf4LTbPMaFiTR13QCgMxPb",
            "sourceAmount": "30.202526264",
            "destinationAmount": "704.766755",
            "tradedOn": "2023-01-16T12:54:49.000Z",
            "aggregator": "Jupiter v3",
            "programId": "whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3uctyCc",
            "sourceMint": "So11111111111111111111111111111111111111112",
            "destinationMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"          
        },
        {
            "transactionId": "3PXDTVaYKoDcMdyyyxGu4pMLbv2ApAx9biapEJv1rjPZdePTdGgqK6LstV8MBiAWtA3zaYJDJ86fT8hveNUN2VEN",
            "sourceAmount": "0.04293291",
            "destinationAmount": "1.001095",
            "tradedOn": "2023-01-16T12:54:42.000Z",
            "aggregator": "Jupiter v3",
            "programId": "whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3uctyCc",
            "sourceMint": "So11111111111111111111111111111111111111112",
            "destinationMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
        }
    ],
    "source": "hellomoon",
    "start": "2023-01-01T00:00:00.000Z",
    "end": "2023-02-01T00:00:00.000Z",
    "paginationToken": "MTQ2MTAzMA=="
}
```

### Simulator

#### Get simulator pool history processed data

```http request
// GET https://api.hubbleprotocol.io/simulator/pools/:poolPubkey/history?env={cluster}&start={start}&end={end}
GET https://api.hubbleprotocol.io/simulator/pools/7qbRF6YsyGuLUVs6Y1q64bdVrfe4ZcUUz1JRdoVNUJnm/history?env=mainnet-beta&start=2023-05-01T00%3A55%3A00.000Z&end=2023-05-03T00%3A00%3A00.000Z'
```

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
* start: start date (inclusive), e.g. `2023-05-01T00:55:00.000Z`
* end: end date (exclusive), e.g. `2023-05-01T00:55:00.000Z`

Example request:

https://api.hubbleprotocol.io/simulator/pools/7qbRF6YsyGuLUVs6Y1q64bdVrfe4ZcUUz1JRdoVNUJnm/history?env=mainnet-beta&start=2023-05-01T00%3A55%3A00.000Z&end=2023-05-03T00%3A00%3A00.000Z'

Sample response:

```json
[
  {
    "date": "2023-05-01T00:00:00.000Z",
    "liquidity": "128579835037512.1296149423008288059642304",
    "tokenAPrice": "22.8225",
    "tokenBPrice": "0.999899925",
    "priceAToB": "22.80281142119161469576395863338884312471",
    "priceBToA": "0.04381507266239798735733225736404509709275",
    "sourceAmountAToB": "2254.958720845",
    "sourceAmountBToA": "127153.433494",
    "destinationAmountAToB": "51419.398474",
    "destinationAmountBToA": "5571.236927813"
  },
  {
    "date": "2023-05-01T01:00:00.000Z",
    "liquidity": "118590788019032.7258026104877490267918479",
    "tokenAPrice": "22.18812871",
    "tokenBPrice": "0.999922935",
    "priceAToB": "22.35004452461559675442332082035821547892",
    "priceBToA": "0.04489632573185525843575517883550659203986",
    "sourceAmountAToB": "25052.494904219",
    "sourceAmountBToA": "288330.780462",
    "destinationAmountAToB": "559924.376562",
    "destinationAmountBToA": "12944.992638142"
  }
]
```

### Leaderboard

#### Get strategy leaderboard

```http request
GET https://api.hubbleprotocol.io/strategies/leaderboard?env={cluster}&period={24h/7d/30d/90d/180d/1y}
```

Example requests:
- https://api.hubbleprotocol.io/strategies/leaderboard?env=mainnet-beta
- https://api.hubbleprotocol.io/strategies/leaderboard?env=mainnet-beta&period=24h

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
* period: leaderboard time period `"24h" | "7d" (default) | "30d" | "90d" | "180d" | "1y"`

Example response:
```json
{
  "period": "24h",
  "strategies": [
    {
      "strategy": "AepjvYK4QfGhV3UjSRkZviR2AJAkLGtqdyKJFCf9kpz9",
      "apy": "0.615191615489133766348795004255205034784",
      "pnl": "0.001035403769",
      "volume": "15763.8054565617",
      "fees": "35.6656098455"
    },
    {
      "strategy": "5QgwaBQzzMAHdxpaVUgb4KrpXELgNTaEYXycUvNvRxr6",
      "apy": "0.03826440591163673741072800936266385589",
      "pnl": "0.000726162458",
      "volume": "90454.9846990331",
      "fees": "8.1861761153"
    }
  ]
}
```

Leaderboard response is always ordered by profit and loss (PnL) descending.

APY and PNL properties are in decimal form, to convert to percentage multiply it by 100.
Volume and fees are both in USD.

#### Get user leaderboard

```http request
GET https://api.hubbleprotocol.io/users/leaderboard?env={cluster}&period={24h/7d/30d/90d/180d/1y/all-time}
```

Example requests:
- https://api.hubbleprotocol.io/users/leaderboard?env=mainnet-beta
- https://api.hubbleprotocol.io/users/leaderboard?env=mainnet-beta&period=24h

Query params:
* env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
* period: leaderboard time period `"24h" | "7d" (default) | "30d" | "90d" | "180d" | "1y" | "all-time"`

Example response:
```json
{
  "period": "24h",
  "users": [
    {
      "user": "7QnXf4d1YQ6k8oTzCLXjEiikGFm3KRZgmFJmry9vZxdW",
      "totalReturns": {
        "sol": "200",
        "usd": "2000"
      },
      "pnl": {
        "sol": "0.6896551724137931034482758620689655172416",
        "usd": "0.606060606060606060606060606060606060606"
      },
      "totalPositionsValue": {
        "sol": "350",
        "usd": "3500"
      }
    },
    {
      "user": "5HDpYuGtrmdRe49r48BoasoZ66Fsz74xShc4DQYDhNii",
      "totalReturns": {
        "sol": "200",
        "usd": "2000"
      },
      "pnl": {
        "sol": "0.6896551724137931034482758620689655172416",
        "usd": "0.606060606060606060606060606060606060606"
      },
      "totalPositionsValue": {
        "sol": "350",
        "usd": "3500"
      }
    }
  ]
}
```

Leaderboard response is always ordered by USD profit and loss (PnL) descending.

PNL is in decimal form, to convert to percentage multiply it by 100.
Total positions value and total returns are both in USD.

### Raydium

These endpoints use Raydium's API directly and cache their responses to avoid rate-limits.

#### Get AMM Pools

```http request
GET https://api.hubbleprotocol.io/raydium/ammPools
```

Example request: https://api.hubbleprotocol.io/raydium/ammPools

#### Get pool liquidity distribution

```http request
GET https://api.hubbleprotocol.io/raydium/positionLine/:poolPubkey
```

Example request: https://api.hubbleprotocol.io/raydium/positionLine/2QdhepnKRTLjjSqPL1PtKNwqrUkoLee5Gqs8bvZhRdMv
