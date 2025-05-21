# 🛰 Hubble Public API

Hubble Public API is a TypeScript API (using Express) that serves public data of the Hubble Protocol.

## Table of contents

- [Development](#development)
  - [Database](#database)
  - [Local API Setup](#local-api-setup)
  - [Tests](#tests)
  - [Deployment](#deployment)
- [Usage](#usage)
  - [Metrics](#metrics)
  - [Metrics History](#metrics-history)
  - [Config](#config)
  - [IDL](#idl)
  - [Circulating Supply Value (HBB)](#circulating-supply-value-hbb)
  - [Circulating Supply (HBB)](#circulating-supply-hbb)
  - [API Version](#api-version)
  - [Maintenance mode](#maintenance-mode)
  - [Borrowing version](#borrowing-version)
  - [Loans](#loans)
  - [Staking](#staking)
  - [Stability](#stability)
  - [Strategies (Kamino)](#strategies-kamino)
  - [Whirlpools (Kamino)](#whirlpools-kamino)
  - [Prices](#prices)
  - [Borrowing Market State](#borrowing-market-state)
  - [Transactions](#transactions)
  - [Kamino Lending](#kamino-lending)
  - [Trades](#trades)
  - [Simulator](#simulator)
  - [Leaderboard](#leaderboard)
  - [Points](#points)
  - [Airdrop](#airdrop)
  - [Tokens](#tokens)
  - [KVaults](#kvaults)
  - [Limo](#limo)
  - [Slot](#slot)
  - [Data](#data)
  - [KSwap](#kswap)
  - [Referrals](#referrals)

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

You will need to use [yarn](https://www.yarnpkg.com/) to install the dependencies.

We are using private nodes without rate limit for fetching mainnet-beta and devnet chain data. You will have to add an `.env` file in the root of this repository with the correct environment variables inside. Please take a look at the example `.env.example` file:

```shell
cd hubble-public-api
cp .env.example .env
# edit .env with actual endpoints with your favorite editor
# nano .env
# code .env
# ...
```

We also use Redis for caching API responses from PostgreSQL database. You can use docker-compose to run an instance of the Redis cluster locally.

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

# a few examples below

# list all keys in all cluster nodes:
for port in {7001..7006}; do echo "Keys from localhost:$port:"; redis-cli -c -h localhost -p $port --scan; done
# get specific redis key contents:
for port in {7001..7006}; do value=$(redis-cli -c -h localhost -p $port GET strategies-leaderboard-mainnet-beta); if [ "$value" ]; then echo "$value"; break; fi; done
# delete specific redis key:
for port in {7001..7006}; redis-cli -c -h localhost -p $port DEL strategies-leaderboard-mainnet-beta; done
# check when a specific redis key will expire:
for port in {7001..7006}; do redis-cli -c -h localhost -p $port TTL staking-yields-v2; done
# clear entire cache in the cluster - delete all redis keys (useful when testing locally, not enabled in production)
for port in {7001..7006}; do redis-cli -c -h localhost -p $port FLUSHALL; done
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

History endpoint will only return the historical data of the current year by default. **Warning:** the maximum allowed period for the history endpoint is 1 year per request.

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

#### Get circulating supply value of HBB (number of HBB issued \* HBB price).

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

Please note: This route is not exposed to the public and requires basic authentication. Please use the route above to get monthly data instead.

```http request
GET https://api.hubbleprotocol.io/staking/lido/eligible-loans?env=devnet&start=2022-06-01&end=2022-07-01
```

#### Get staking yields (DEPRECATED)

:warning: **DEPRECATED, PLEASE USE /v2/staking-yields INSTEAD!** :warning:

```http request
GET https://api.hubbleprotocol.io/staking-yields
```

Example request: https://api.hubbleprotocol.io/staking-yields

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

#### Get staking yields v2

```http request
GET https://api.hubbleprotocol.io/v2/staking-yields
```

Example request: https://api.kamino.finance/v2/staking-yields

Example response:

```json
[
  {
    "apy": "0.252619403785070711510296827413476389122",
    "tokenMint": "mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So"
  },
  {
    "apy": "0.263020843260597671599388280100460408894",
    "tokenMint": "J1toso1uCk3RLmjorhTtrVwY9HJ7X8V9yYac6Y7kGCPn"
  },
  {
    "apy": "0.24426365077443429533690036384035569496",
    "tokenMint": "bSo13r4TkiE4KumL71LsHTPpL2euBYLFx6h9HP3piy1"
  },
  {
    "apy": "0.2727867352767135230363060182415164271",
    "tokenMint": "jupSoLaHXQiZZTSfEWMTRRgpnyFm8f6sZdosWBjx93v"
  },
  {
    "apy": "0.282542463074495351542281213938801414377",
    "tokenMint": "he1iusmfkpAdwvxLNGV8Y1iSbj4rUy6yMhEA3fotn9A"
  }
]
```

#### Get median staking yields v2

```http request
GET https://api.kamino.finance/v2/staking-yields/median
```

Example request: https://api.kamino.finance/v2/staking-yields/median

Example response:

```json
[
  {
    "apy": "0.252619403785070711510296827413476389122",
    "tokenMint": "mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So"
  },
  {
    "apy": "0.263020843260597671599388280100460408894",
    "tokenMint": "J1toso1uCk3RLmjorhTtrVwY9HJ7X8V9yYac6Y7kGCPn"
  },
  {
    "apy": "0.24426365077443429533690036384035569496",
    "tokenMint": "bSo13r4TkiE4KumL71LsHTPpL2euBYLFx6h9HP3piy1"
  },
  {
    "apy": "0.2727867352767135230363060182415164271",
    "tokenMint": "jupSoLaHXQiZZTSfEWMTRRgpnyFm8f6sZdosWBjx93v"
  },
  {
    "apy": "0.282542463074495351542281213938801414377",
    "tokenMint": "he1iusmfkpAdwvxLNGV8Y1iSbj4rUy6yMhEA3fotn9A"
  }
]
```

#### Get mean staking yields v2

```http request
GET https://api.kamino.finance/v2/staking-yields/mean
```

Example request: https://api.hubbleprotocol.io/v2/staking-yields/mean

Example response:

```json
[
  {
    "apy": "0.252619403785070711510296827413476389122",
    "tokenMint": "mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So"
  },
  {
    "apy": "0.263020843260597671599388280100460408894",
    "tokenMint": "J1toso1uCk3RLmjorhTtrVwY9HJ7X8V9yYac6Y7kGCPn"
  },
  {
    "apy": "0.24426365077443429533690036384035569496",
    "tokenMint": "bSo13r4TkiE4KumL71LsHTPpL2euBYLFx6h9HP3piy1"
  },
  {
    "apy": "0.2727867352767135230363060182415164271",
    "tokenMint": "jupSoLaHXQiZZTSfEWMTRRgpnyFm8f6sZdosWBjx93v"
  },
  {
    "apy": "0.282542463074495351542281213938801414377",
    "tokenMint": "he1iusmfkpAdwvxLNGV8Y1iSbj4rUy6yMhEA3fotn9A"
  }
]
```

#### Get epochs

```http request
GET https://api.hubbleprotocol.io/epochs
```

Example request: https://api.hubbleprotocol.io/epochs

Example response:

```json
[
  {
    "epoch": 627,
    "first_slot": 270864000,
    "last_slot": 271295999,
    "start_block_time": "2024-06-09T18:21:01.000Z",
    "end_block_time": "2024-06-11T23:21:40.000Z"
  },
  {
    "epoch": 626,
    "first_slot": 270432000,
    "last_slot": 270863999,
    "start_block_time": "2024-06-07T14:10:07.000Z",
    "end_block_time": "2024-06-09T18:21:00.000Z"
  }
]
```

#### Get staking rate history

```http request
GET https://api.hubbleprotocol.io/staking-rates/tokens/:mint/history?start={start timestamp}&end={end timestamp}&interpolate={true/false}
```

Example request: https://api.hubbleprotocol.io/staking-rates/tokens/mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So/history?start=2024-01-01T00:00Z&end=2024-02-01T01:00Z&interpolate=false

Example response (first item in array is epoch timestamp, second is stake rate):

```json
[
  [1704067200000, "0.8636310820345713"],
  [1704067500000, "0.8636310820345713"],
  [1704067800000, "0.8636310820345713"]
]
```

#### Get staking yield history

```http request
GET https://api.hubbleprotocol.io/staking-yields/tokens/:mint/history?start={start timestamp}&end={end timestamp}
```

Example request: https://api.hubbleprotocol.io/staking-yields/tokens/mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So/history?start=2024-01-01T00:00Z&end=2024-02-01T01:00Z

Example response:

```json
[
  {
    "apy": "0.0685815525901495360734829436051179078079",
    "epoch": 587,
    "startBlockTime": "2024-03-11T21:37:21.000Z",
    "endBlockTime": "2024-03-14T00:25:05.000Z"
  },
  {
    "apy": "0.0772467014510651101771957924044147574663",
    "epoch": 586,
    "startBlockTime": "2024-03-09T19:55:31.000Z",
    "endBlockTime": "2024-03-11T21:37:20.000Z"
  }
]
```

#### Get median staking yield history

```http request
GET https://api.hubbleprotocol.io/staking-yields/tokens/:mint/history/median
```

Example request: https://api.hubbleprotocol.io/staking-yields/tokens/mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So/history

Example response:

```json
[
  {
    "startEpoch": 502,
    "startBlockTime": "2023-09-11T05:22:50.000Z",
    "endBlockTime": "2023-10-05T01:18:02.000Z",
    "endEpoch": 512,
    "apy": 0.06882309627010348
  },
  {
    "startEpoch": 503,
    "startBlockTime": "2023-09-13T10:14:26.000Z",
    "endBlockTime": "2023-10-07T03:15:54.000Z",
    "endEpoch": 513,
    "apy": 0.06900975295240162
  }
]
```

#### Get yield history for specific source

```http request
GET https://api.hubbleprotocol.io/yields/:source/history
```

Example request, get JLP pool yield history: https://api.hubbleprotocol.io/yields/5BUwFW4nRbftYTDMbgxykoFWqWHPzahFSNAaaaJtVKsq/history

Example response (please note APR/APY is in decimal format, not in percentage, multiply by 100 to get percentage):

```json
[
  {
    "createdOn": "2024-01-01T00:00:07.569Z",
    "apr": "5.16226661065975",
    "apy": "1.72282489033179"
  },
  {
    "createdOn": "2024-01-01T01:00:07.569Z",
    "apr": "5.53223063067687",
    "apy": "9.201613187495"
  }
]
```

#### Get USDs rewards yield

:warning: **DEPRECATED, THIS ENDPOINT IS NO LONGER AVAILABLE** :warning:

Wrapper over Allez API

```http request
GET https://api.hubbleprotocol.io/yields/usds-rewards
```

Example request:

- https://api.kamino.finance/yields/usds-rewards

Example response:

```json
{
  "timestamp_hour": "2024-11-28 14:00:00",
  "usds_balance_total": 58012704.78112555,
  "usds_flagged_addresses": 452091.5947593333,
  "usds_total_supply": 58464796.37588488,
  "bridged_usds_total": 159993178.337911,
  "effective_bridged_usds_total": 155287132.57078904,
  "net_usds_supply_in_kamino_total": 35401432.65644375,
  "effective_usds_balance_total": 34337023.08437288,
  "hourly_rewards_per_unit": 0.000017335168915938846,
  "per_unit_rewards_apr": 0.1518560797036242,
  "per_unit_rewards_apy": 0.163991170143378,
  "accrued_rewards_total": 129166.66666666612
}
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
GET https://api.hubbleprotocol.io/v2/strategies/:strategyPubkey/history?env={cluster}&start={start}&end={end}&frequency={frequency}&includePerfFee={true/false}
```

Query params:

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- start: start date (inclusive), e.g. `2023-05-01T00:00:00.000Z` (optional, default since beginning of strategy)
- end: end date (exclusive), e.g. `2023-05-02T00:00:00.000Z` (optional, default now)
- frequency: frequency of the snapshots, e.g. `"hour" (default) | "day"`
- includePerfFee: add perfFee field to the response (optional, default false)

Example requests:

- https://api.hubbleprotocol.io/v2/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/history?env=mainnet-beta
- https://api.hubbleprotocol.io/v2/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/history?env=mainnet-beta&start=2023-01-01&end=2023-02-01&frequency=hour
- https://api.hubbleprotocol.io/v2/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/history?env=mainnet-beta&start=2023-01-01&end=2023-02-01&frequency=day

Example response:

```js
[
  {
    timestamp: '2023-06-06T00:00:00.000Z',
    feesCollectedCumulativeA: '4528.851182',
    feesCollectedCumulativeB: '4388.399699',
    rewardsCollectedCumulative0: '6816.973791',
    rewardsCollectedCumulative1: '869.925191',
    rewardsCollectedCumulative2: '0',
    kaminoRewardsIssuedCumulative0: '0',
    kaminoRewardsIssuedCumulative1: '0',
    kaminoRewardsIssuedCumulative2: '0',
    sharePrice: '1.009166844496339887986660123337354133388',
    sharesIssued: '2000370.229713',
    tokenAAmounts: '1941835.079462',
    tokenBAmounts: '83285.638712',
    tokenAPrice: '0.9966972458',
    tokenBPrice: '0.99999998',
    reward0Price: '0.042612354',
    reward1Price: '0.6421835081',
    reward2Price: '0.99999998',
    kaminoReward0Price: '0',
    kaminoReward1Price: '0',
    kaminoReward2Price: '0',
    totalValueLocked: '2018707.312543886771519599999999999999998983602957644',
    solPrice: '0',
    profitAndLoss: '0',
  },
];
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

Fetch strategy shareholder fees and rewards for only the latest position. This endpoint takes a look at when the user's position was last closed (fully withdrawn) and opened again and only calculates the fees and rewards after that time.

For all-time fees and rewards of a strategy shareholder use endpoint mentioned in the previous section (`Get fees and rewards earned for a strategy shareholder` above).

```http request
// GET https://api.hubbleprotocol.io/v2/strategies/:strategyPubkey/shareholders/:shareholderPubkey/fees-and-rewards/latest-position?env={cluster}
GET https://api.hubbleprotocol.io/v2/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L/fees-and-rewards/latest-position?env=mainnet-beta
```

Example request:

https://api.hubbleprotocol.io/v2/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L/fees-and-rewards/latest-position?env=mainnet-beta

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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- period: time period for fees and rewards `"24h" (default) | "7d" | "30d"`
- status: strategy status `"LIVE" (default) | "STAGING" | "SHADOW" | "IGNORED" | "DEPRECATED"`, status query param can also be used multiple times, for example:
  - https://api.hubbleprotocol.io/strategies/fees-and-rewards?status=LIVE&status=STAGING&status=SHADOW

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
// GET https://api.hubbleprotocol.io/v2/strategies/:strategyPubkey/shareholders/:shareholderPubkey/pnl?env={cluster}
GET https://api.hubbleprotocol.io/v2/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L/pnl?env=mainnet-beta
```

Example request:

https://api.hubbleprotocol.io/v2/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L/pnl?env=mainnet-beta

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

Returns hourly timeseries if user's latest position is under 15 days old, otherwise it returns daily timeseries data. This is an optimization for frontend charts.

```http request
// GET https://api.hubbleprotocol.io/v2/strategies/:strategyPubkey/shareholders/:shareholderPubkey/pnl/history?env={cluster}&start={date}&end={date}
GET https://api.hubbleprotocol.io/v2/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L/pnl/history?env=mainnet-beta
```

Query params:

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- start: start date (inclusive), e.g. `2023-05-01T00:55:00.000Z`
- end: end date (exclusive), e.g. `2023-05-01T00:55:00.000Z`

Example request:

https://api.hubbleprotocol.io/v2/strategies/Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN/shareholders/HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L/pnl/history?env=mainnet-beta

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

Transaction type can be `buy`, `sell` or `mark-to-market`. Buys and sells are actual transactions of the user. Mark to market represents potential position and PnL if they bought/sold at that specific time.

#### Get volume for strategies:

:warning: **DEPRECATED, PLEASE USE /v2/strategies/volume INSTEAD!** :warning:

```http request
GET https://api.hubbleprotocol.io/strategies/volume?env={cluster}&status={strategyStatus}
```

Query params:

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- status: strategy status `"LIVE" (default) | "STAGING" | "SHADOW" | "IGNORED" | "DEPRECATED"`, status query param can also be used multiple times, for example:
  - https://api.hubbleprotocol.io/strategies/volume?status=LIVE&status=STAGING&status=SHADOW

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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

Example request:

https://api.hubbleprotocol.io/strategies/tvl?env=mainnet-beta

Example response:

```json
{
  "tvl": "8360625.990205809513007916645145375207353"
}
```

#### Get all strategies with filters

- The current filters that are supported are:

  - `strategyType` which can be:

    - `NON_PEGGED`: e.g. SOL-BONK
    - `PEGGED`: e.g. BSOL-JitoSOL
    - `STABLE`: e.g. USDH-USDC

  - `strategyCreationStatus` which can be:
    - `IGNORED`
    - `SHADOW`
    - `LIVE`
    - `DEPRECATED`
    - `STAGING`

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

Example request to get total MSOL stored in Kamino: https://api.hubbleprotocol.io/strategies/tokens/mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So/amounts?env=mainnet-beta

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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- period: time period for fees and rewards `"24h" (default) | "7d" | "30d"`
- start: start date (inclusive), e.g. `2023-05-01T00:55:00.000Z`
- end: end date (exclusive), e.g. `2023-05-01T00:55:00.000Z`

You can either use a custom range with start/end query params, or a fixed range with period query param. Fixed range should generally be faster due to caching.

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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- period: time period for fees and rewards `"24h" (default) | "7d" | "30d"`
- start: start date (inclusive), e.g. `2023-05-01T00:00:00.000Z`
- end: end date (exclusive), e.g. `2023-06-01T00:00:00.000Z`

You can either use a custom range with start/end query params, or a fixed range with period query param. Fixed range should generally be faster due to caching.

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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- start: start date (inclusive), e.g. `2023-05-01T00:00:00.000Z` (optional, default since beginning of strategy)
- end: end date (exclusive), e.g. `2023-05-02T00:00:00.000Z` (optional, default now)
- frequency: frequency of the snapshots, e.g. `"hour" (default) | "day"`

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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

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

Example request: https://api.hubbleprotocol.io/prices?env=mainnet-beta&source=scope Example response:

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
- https://api.hubbleprotocol.io/prices?env=mainnet-beta&source=scope&token=So11111111111111111111111111111111111111112 Example response:

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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- token: name (deprecated soon) or mint (**use of mint recommended!**) pubkey, e.g. `"SOL"` or `"So11111111111111111111111111111111111111112"`
- start: start date (inclusive), e.g. `2023-05-01T00:55:00.000Z` (optional, default 1 day ago)
- end: end date (inclusive), e.g. `2023-05-01T00:55:00.000Z` (optional, default now)
- frequency: frequency of the prices, e.g. `"minute" (default) | "hour" | "day"` for price every minute/hour/day for the specified timeseries
- type: price type, e.g. `"spot" (default) | "TWAP"`

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
    '17.35422124', // <-- token price
    1668162327, // <-- seconds since epoch
  ],
  ['16.51238765', 1668172327],
];
```

#### Get token price moving averages:

```http request
GET https://api.hubbleprotocol.io/prices/moving-averages?env={cluster:optional, default mainnet}&token={tokenName:required}&durationInSec={duration:optional, default 3600}
```

Example request: https://api.hubbleprotocol.io/prices/moving-averages?env=mainnet-beta&token=SOL&durationInSec=3600

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

### Transactions

#### Get all kamino transactions v2

Get all instructions for the specified Kamino (liquidity) shareholder.

```http request
// GET https://api.hubbleprotocol.io/v2/shareholders/:shareholderPubkey/transactions?env={cluster}
GET https://api.hubbleprotocol.io/v2/shareholders/7QnXf4d1YQ6k8oTzCLXjEiikGFm3KRZgmFJmry9vZxdW/transactions?env=mainnet-beta
```

Example response:

```json
{
  "transactions": [
    {
      "createdOn": "2024-02-02T12:07:19.000Z",
      "timestamp": "1706875639000",
      "transactionSignature": "DyUW9Rhkis65Y4RAoD6Xj3C2iZQigZn67ZJt5P11Uk8kZxC9Zx8r84bsAKYMgfvzows1LNPDmB3hD5kimHqjufF",
      "transactionName": "deposit",
      "strategy": "HBuYwvq67VKnLyKxPzDjzskyRMk7ps39gwHdvaPGwdmQ",
      "tokenA": "JUP",
      "tokenAMint": "JUPyiwrYJFskUPiHa7hkeR8VUtAeFoSYbKedZNsDvCN",
      "tokenAAmount": "1172.664445",
      "tokenAPrice": "1.4235452",
      "tokenB": "bSOL",
      "tokenBMint": "bSo13r4TkiE4KumL71LsHTPpL2euBYLFx6h9HP3piy1",
      "tokenBAmount": "6.452820291",
      "tokenBPrice": "1.55343",
      "usdValue": "1345.28593",
      "numberOfShares": "5315.114625",
      "sharePrice": "0.1248459",
      "solPrice": "90.1348745894",
      "latestPosition": true
    }
  ]
}
```

### Kamino Lending

#### Get All Kamino Markets config

:warning: **DEPRECATED, PLEASE USE /v2/kamino-market/ INSTEAD!** :warning:

```http request
GET https://api.hubbleprotocol.io/kamino-market?env={cluster}
```

Example: https://api.hubbleprotocol.io/kamino-market?env=mainnet-beta

#### Get Kamino Market config

:warning: **DEPRECATED, PLEASE USE /v2/kamino-market/:marketPubkey INSTEAD!** :warning:

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey?env={cluster}
```

Example: https://api.hubbleprotocol.io/kamino-market/J5ndTP1GJe6ZWzGiZQR2UKJmWWMJojbWxCxZ2yUXwakR?env=mainnet-beta

#### Get All Kamino Markets config V2

```http request
GET https://api.hubbleprotocol.io/kamino-market?programId={programId}
```

Example: https://api.hubbleprotocol.io/kamino-market?programId=KLend2g3cP87fffoy8q1mQqGKjrxjC8boSyAYavgmjD

#### Get Kamino Market config V2

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey?programId={programId}
```

Example: https://api.hubbleprotocol.io/kamino-market/6WVSwDQXrBZeQVnu6hpnsRZhodaJTZBUaC334SiiBKdb?programId=SLendK7ySfcEzyaFqy93gDnD3RtrpXJcnRwb6zFHJSh

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

- env: solana cluster, e.g. "mainnet-beta" (default) | "devnet"
- start: start date (inclusive), e.g. 2023-05-01T00:00:00.000Z (optional, default since beginning of strategy)
- end: end date (exclusive), e.g. 2023-05-02T00:00:00.000Z (optional, default now)
- frequency: frequency of the snapshots, e.g. "hour" (default) | "day"

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

#### Get KLend reserve borrow APY and staking APY history

Get history of klend reserve borrow interest APY and staking APY. This will only return data for reserves that contain a LST token with staking yield.

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/reserves/:reservePubkey/borrow-and-staking-apys/history?env={cluster}&start={date}&end={date}'
```

Median:

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/reserves/:reservePubkey/borrow-and-staking-apys/history/median?env={cluster}&start={date}&end={date}'
```

Query params:

- env: solana cluster, e.g. "mainnet-beta" (default) | "devnet"
- start: start date (inclusive), e.g. 2023-05-01T00:00:00.000Z
- end: end date (exclusive), e.g. 2023-05-02T00:00:00.000Z

Examples:

- https://api.hubbleprotocol.io/kamino-market/7u3HeHxYDLhnCoErrtycNokbQYbWGzLs6JSDqGAv5PfF/reserves/FBSyPnxtHKLBZ4UeeUyAnbtFuAmTHLtso9YtsqRDRWpM/borrow-and-staking-apys/history?env=mainnet-beta&start=2023-10-15T00%3A00Z&end=2023-11-14T00%3A00Z'
- https://api.hubbleprotocol.io/kamino-market/7u3HeHxYDLhnCoErrtycNokbQYbWGzLs6JSDqGAv5PfF/reserves/FBSyPnxtHKLBZ4UeeUyAnbtFuAmTHLtso9YtsqRDRWpM/borrow-and-staking-apys/history/median?env=mainnet-beta&start=2023-10-15T00%3A00Z&end=2023-11-14T00%3A00Z'

```json
[
  {
    "createdOn": "2023-10-17T15:00:06.009Z",
    "borrowInterestApy": "0.027610992938039702",
    "stakingApy": "0.066527842899023711405366628964490454572"
  },
  {
    "createdOn": "2023-10-17T15:05:06.009Z",
    "borrowInterestApy": "0.017610992938039702",
    "stakingApy": "0.078527842899023711405366628964490454572"
  }
]
```

#### Get KLend obligation history

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/obligations/:obligationPubkey/metrics/history?env={cluster}&start={date}&end={date}&frequency={frequency}'
```

Query params:

- env: solana cluster, e.g. "mainnet-beta" (default) | "devnet"
- start: start date (inclusive), e.g. 2023-05-01T00:00:00.000Z (optional, default since beginning of strategy)
- end: end date (exclusive), e.g. 2023-05-02T00:00:00.000Z (optional, default now)
- frequency: frequency of the snapshots, e.g. "hour" (default) | "day"

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

#### Get KLend obligation history V2

```http request
GET https://api.hubbleprotocol.io/v2/kamino-market/:marketPubkey/obligations/:obligationPubkey/metrics/history?env={cluster}&start={date}&end={date}&useStakeRateForObligation={true|false}'
```

Query params:

- env: solana cluster, e.g. "mainnet-beta" (default) | "devnet"
- start: start date (inclusive), e.g. 2023-05-01T00:00:00.000Z (optional, default since beginning of strategy)
- end: end date (exclusive), e.g. 2023-05-02T00:00:00.000Z (optional, default now)
- useStakeRateForObligation: uses stake rate to calc net sol value

Example: https://api.hubbleprotocol.io/v2/kamino-market/9pMFoVgsG2cNiUCSBEE69iWFN7c1bz9gu9TtPeXkAMTs/obligations/63QrAB1okxCc4FpsgcKYHjYTp1ua8ch6mLReyKRdc22o/metrics/history?env=mainnet-beta&start=2023-01-01&end=2023-01-02&frequency=day&useStakeRateForObligation=true

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

#### Get all user obligations

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/users/:userPubkey/obligations?env={cluster}'
```

Query params:

- env: solana cluster, e.g. "mainnet-beta" (default) | "devnet"

Example: https://api.hubbleprotocol.io/kamino-market/7u3HeHxYDLhnCoErrtycNokbQYbWGzLs6JSDqGAv5PfF/v2/users/AcNSmd5CxwLs21TYUmhWt7CW2v159TdYRkvQxb1iBYRj/obligations

#### Get KLend tx history per user (for all obligations)

```http request
GET https://api.hubbleprotocol.io/v2/kamino-market/:marketPubkey/users/:userPubkey/transactions/
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
GET https://api.hubbleprotocol.io/v2/kamino-market/:marketPubkey/obligations/:obligationPubkey/transactions/
```

Query params:

- env: solana cluster, e.g. "mainnet-beta" (default) | "devnet"
- useLogPrices: scans logs for token prices instead of using scope prices

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

#### Bulk get Klend transactions

This endpoint is not open to the public, it is private for analytical purposes only, add authorization headers to the request.

```http request
POST https://api.hubbleprotocol.io/v2/kamino-market/:marketPubkey/transactions?programId={programId}?includeRawJson={true|false, false by default}
```

```json
{
  "start": "2024-01-01T00:00Z",
  "end": "2024-01-02T00:00Z",
  "instruction": "depositReserveLiquidityAndObligationCollateral",
  "paginationToken": ""
}
```

Example cURL request:

```bash
curl --location 'https://api.hubbleprotocol.io/v2/kamino-market/7u3HeHxYDLhnCoErrtycNokbQYbWGzLs6JSDqGAv5PfF/transactions?programId=KLend2g3cP87fffoy8q1mQqGKjrxjC8boSyAYavgmjD' --header 'Content-Type: application/json' --header 'Authorization: Basic ENTER_CREDENTIALS' --data '{
    "start": "2024-01-01",
    "end": "2024-01-02",
    "instruction": "depositReserveLiquidityAndObligationCollateral",
    "paginationToken": ""
}'
```

Example Python request:

```python
import requests
import json

url = "https://api.hubbleprotocol.io/v2/kamino-market/7u3HeHxYDLhnCoErrtycNokbQYbWGzLs6JSDqGAv5PfF/transactions?programId=KLend2g3cP87fffoy8q1mQqGKjrxjC8boSyAYavgmjD"

payload = json.dumps({
  "start": "2024-01-01",
  "end": "2024-01-02",
  "instruction": "depositReserveLiquidityAndObligationCollateral",
  "paginationToken": ""
})
headers = {
  'Content-Type': 'application/json',
  'Authorization': 'Basic ADD-CREDENTIALS-HERE'
}

response = requests.request("POST", url, headers=headers, data=payload)

print(response.text)
```

Example response:

```json
{
  "result": [
    {
      "ixs": [
        {
          "solPrice": "100.123",
          "liquidityToken": "SOL",
          "liquidityUsdValue": "100.123",
          "liquidityTokenMint": "So11111111111111111111111111111111111111112",
          "liquidityTokenPrice": "100.123",
          "liquidityTokenAmount": "1",
          "isLiquidationWithdrawal": false
        }
      ],
      "timestamp": "2024-02-19T00:20:45.000Z",
      "signature": "NysWUiARXCRntcCmbyyRv43aLv2m4rKF39nCTJKsaV59ASu9cDLVpxuMVWn9HujorM81iXHUYtRYw29jNnAHhTD",
      "obligation": "7vbVt9y3dtajhwgbrbSb7RR3SuainvcpLAJYGge9fRco",
      "wallet": "3YFyQwu7rSQmxvSrGawxr3fq4CyFT6NQkYry8jkCSQuk"
    }
  ],
  "paginationToken": "eyJsYXN0SWQiOjIwNzh9"
}
```


#### Get PnL per obligation v2

You can specify pnl mode with query param `positionMode` with one of these values: {`obligation_all_time`, `current_obligation`}. By default, pnl mode is set to `current_obligation` position For xSOL pairs, `useStakeRate` query param can be set to true to calculate the PnL using the stake rate. By default, it is set to false.

These parameters specify whether we return the current position PnL or the accumulated PnL throughout the lifetime of the user's obligation (even before they were closed).

```http request
GET https://api.hubbleprotocol.io/v2/kamino-market/:marketPubkey/obligations/:obligationPubkey/pnl/?positionMode=current_obligation&useStakeRate=true
```

Example response:

```json
{
  "usd": "25.21",
  "sol": "1.0"
}
```


#### Get interest fees earned per obligation

```http request
GET https://api.kamino.finance/v2/kamino-market/:marketPubkey/obligations/:obligationPubkey/interest-fees/
```

Query params:

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- start: start date (inclusive), e.g. `2023-05-01T00:55:00.000Z`. Furthest start date is capped at the obligation creation date.
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

#### Get interest fees paid per obligation

```http request
GET https://api.kamino.finance/v2/kamino-market/:marketPubkey/obligations/:obligationPubkey/interest-paid/
```

Query params:

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- start: start date (inclusive), e.g. `2023-05-01T00:55:00.000Z`. Furthest start date is capped at the obligation creation date.
- end: end date (exclusive), e.g. `2023-05-01T00:55:00.000Z`
- frequency: frequency of the snapshots, e.g. `"hour" (default) | "day"`

Example response:

```json
{
  "totalFeesPaidObligation": {
    "uzARQQZSeLY29ykwZh4XehpEf6XwJcMjcJJwYbgK4Ve": {
      "ts": 1742972400000,
      "solFees": "0",
      "usdFees": "2232.567996820126640786565461905920473664"
    }
  },
  "historicalFeesObligation": {
    "uzARQQZSeLY29ykwZh4XehpEf6XwJcMjcJJwYbgK4Ve": [
      {
        "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v": {
          "ts": 1742572800000,
          "solFees": "0",
          "usdFees": "0",
          "nativeFees": "0"
        }
      },
      {
        "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v": {
          "ts": 1742576400000,
          "solFees": "0",
          "usdFees": "20.09918785299190537182024766849124494505",
          "nativeFees": "20.10066706108093031748231119177184743531"
        }
      },
      {
        "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v": {
          "ts": 1742580000000,
          "solFees": "0",
          "usdFees": "20.09995908102914681452713324418504285405",
          "nativeFees": "20.10089115935220597631825512167503492542"
        }
      },
      {
        "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v": {
          "ts": 1742583600000,
          "solFees": "0",
          "usdFees": "20.10071490277332916712852677133521975735",
          "nativeFees": "20.10111793018782943310866059998024936135"
        }
      },
      {
        "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v": {
          "ts": 1742587200000,
          "solFees": "0",
          "usdFees": "20.10121483273506965773714235577788849993",
          "nativeFees": "20.10136800515926897136670417006366438506"
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

#### Get metrics for leverage/multiply vaults

```http request
GET https://api.hubbleprotocol.io/kamino-market/:marketPubkey/leverage/metrics?env={cluster}
```

Query params:

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`

Example request:

- https://api.hubbleprotocol.io/kamino-market/7u3HeHxYDLhnCoErrtycNokbQYbWGzLs6JSDqGAv5PfF/leverage/metrics?env=mainnet-beta

Example response:

```json
[
  {
    "avgLeverage": "2.757170327410913560319220040749227783417",
    "totalBorrowed": "154395.1202118977930406687953589698645807",
    "totalDeposited": "193096.379305039",
    "totalBorrowedUsd": "16739712.70542562946167548855378444003952",
    "totalObligations": "1836",
    "totalDepositedUsd": "23093580.44995144380237321675219340108242",
    "tvl": "23093580.44995144380237321675219340108242",
    "updatedOn": "2023-12-29T10:29:30.896Z",
    "depositReserve": "H9vmCVd77N1HZa36eBn3UnftYmg4vQzPfm1RxabHAMER",
    "borrowReserve": "d4A2prbA2whesmvHaL88BH6Ewn5N4bTSU2Ze8P6Bc4Q",
    "tag": "Multiply"
  },
  {
    "avgLeverage": "2.604314516472810767833910368870052572746",
    "totalBorrowed": "26112.30734890929199125495560713880090385",
    "totalDeposited": "34303.490774368",
    "totalBorrowedUsd": "2831129.134758960612809682991545473977498",
    "totalObligations": "1168",
    "tvl": "23093580.44995144380237321675219340108242",
    "totalDepositedUsd": "4315255.85162694588194992120445036586528",
    "updatedOn": "2023-12-29T10:29:30.896Z",
    "depositReserve": "FBSyPnxtHKLBZ4UeeUyAnbtFuAmTHLtso9YtsqRDRWpM",
    "borrowReserve": "d4A2prbA2whesmvHaL88BH6Ewn5N4bTSU2Ze8P6Bc4Q",
    "tag": "Leverage"
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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- source: trades fetched from source`"hellomoon" (default)`

Body params:

- tokenAMint: public key of the first mint, e.g. SOL mint: `"So11111111111111111111111111111111111111112"`
- tokenBMint: public key of the second mint, e.g. USDC mint: `"EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"`
- start: start of the date range to fetch trades from, e.g. date ISO string: `"2023-01-01T00:00Z"` or epoch in ms: `1678381747854`
- end: end of the date range to fetch trades to, e.g. date ISO string: `"2023-01-01T00:00Z"` or epoch in ms: `1678381747854`
- paginationToken: pagination token to use for retrieving results. If the response contains a `paginationToken` JSON property, you can use that in the next request to fetch more data from the last trade onwards. If the property does not exist, you've reached the end.

Please note that the response will contain both "directions" (buys, sells) of the trade. For example, if you input tokenA/tokenB mints for SOL/USDC in the request body, the response will contain trades with (source = SOL, destination = USDC) or (source = USDC, destination = SOL).

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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- start: start date (inclusive), e.g. `2023-05-01T00:55:00.000Z`
- end: end date (exclusive), e.g. `2023-05-01T00:55:00.000Z`

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

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- period: leaderboard time period `"24h" | "7d" (default) | "30d" | "90d" | "180d" | "1y"`

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

APY and PNL properties are in decimal form, to convert to percentage multiply it by 100. Volume and fees are both in USD.

#### Get user leaderboard

```http request
GET https://api.hubbleprotocol.io/users/leaderboard?env={cluster}&period={24h/7d/30d/90d/180d/1y/all-time}
```

Example requests:

- https://api.hubbleprotocol.io/users/leaderboard?env=mainnet-beta
- https://api.hubbleprotocol.io/users/leaderboard?env=mainnet-beta&period=24h

Query params:

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- period: leaderboard time period `"24h" | "7d" (default) | "30d" | "90d" | "180d" | "1y" | "all-time"`

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

PNL is in decimal form, to convert to percentage multiply it by 100. Total positions value and total returns are both in USD.

### Raydium

:warning: **DEPRECATED, PLEASE USE /v2/raydium INSTEAD!** :warning:

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

### Raydium V2

This is a backwards compatible v2 endpoint that uses the new Raydium API.

These endpoints use Raydium's API directly and cache their responses to avoid rate-limits.

#### Get AMM Pools

```http request
GET https://api.kamino.finance/v2/raydium/ammPools
```

Example request: https://api.kamino.finance/v2/raydium/ammPools

#### Get pool liquidity distribution

```http request
GET https://api.kamino.finance/v2/raydium/positionLine/:poolPubkey
```

Example request: https://api.kamino.finance/v2/raydium/positionLine/2QdhepnKRTLjjSqPL1PtKNwqrUkoLee5Gqs8bvZhRdMv

### Points

#### Get points leaderboard

```http request
GET https://api.hubbleprotocol.io/points/leaderboard?offset={offset}&limit=${limit}&source={source}
```

Example requests:

- get first 20 leaderboard ranks: https://api.hubbleprotocol.io/points/leaderboard?offset=0&limit=20
- get leaderboard ranks between 100-200: https://api.hubbleprotocol.io/points/leaderboard?offset=100&limit=100
- get first 20 leaderboard ranks for season 2: https://api.hubbleprotocol.io/points/leaderboard?offset=0&limit=20&source=Season2

To implement pagination fetch the first page (offset = 0, limit = 20 for example) and then look at the property `totalLeaderboardCount` to calculate the amount of pages you will display.

Example response:

```json
{
  "totalLeaderboardCount": "90129",
  "leaderboard": [
    {
      "leaderboardRank": "1",
      "ownerPubkey": "7tH1k4PsMu3sNUYJxD5ezxhAmYYXyVVZz5c3dbaxcvUV",
      "lastCalculatedStrategies": "2024-01-10T16:00:00.000Z",
      "lastCalculatedBorrowLend": "2024-01-10T15:00:00.000Z",
      "lastCalculatedMultiply": null,
      "lastCalculatedLeverage": null,
      "pointsEarnedStrategies": "122367620435.3735191599063727893014010360341",
      "pointsEarnedBorrowLend": "1267843473.82435611251081561593395418572529",
      "pointsEarnedMultiply": "0",
      "pointsEarnedLeverage": "0",
      "totalPointsEarned": "123635463909.19787527241718840523535522175939",
      "currentPointsPerSecondStrategies": "560.478391755217855861526657236069466256",
      "currentPointsPerSecondBorrowLend": "89.41877886905689864601546998888888888885",
      "currentPointsPerSecondMultiply": "0",
      "currentPointsPerSecondLeverage": "0",
      "totalCurrentPointsPerSecond": "649.89717062427475450754212722495835514485"
    },
    {
      "leaderboardRank": "2",
      "ownerPubkey": "HZYHFagpyCqXuQjrSCN2jWrMHTVHPf9VWP79UGyvo95L",
      "lastCalculatedStrategies": "2024-01-10T16:00:00.000Z",
      "lastCalculatedBorrowLend": null,
      "lastCalculatedMultiply": null,
      "lastCalculatedLeverage": null,
      "pointsEarnedStrategies": "54903470324.3054572407658617493510029034321",
      "pointsEarnedBorrowLend": "0",
      "pointsEarnedMultiply": "0",
      "pointsEarnedLeverage": "0",
      "totalPointsEarned": "54903470324.3054572407658617493510029034321",
      "currentPointsPerSecondStrategies": "6.48961815479786922528593823056190322725",
      "currentPointsPerSecondBorrowLend": "0",
      "currentPointsPerSecondMultiply": "0",
      "currentPointsPerSecondLeverage": "0",
      "totalCurrentPointsPerSecond": "6.48961815479786922528593823056190322725"
    }
  ]
}
```

#### Get points user breakdown

```http request
GET https://api.hubbleprotocol.io/points/users/:userPubkey/breakdown?env={solana cluster}&source={source}
```

Example requests:

- https://api.hubbleprotocol.io/points/users/7QnXf4d1YQ6k8oTzCLXjEiikGFm3KRZgmFJmry9vZxdW/breakdown?env=mainnet-beta&source=Season1
- https://api.hubbleprotocol.io/points/users/7QnXf4d1YQ6k8oTzCLXjEiikGFm3KRZgmFJmry9vZxdW/breakdown?env=mainnet-beta&source=Season2

Example response:

```json
{
  "leaderboardRank": "1271",
  "updatedOnStrategies": "2024-01-10T16:00:00.000Z",
  "updatedOnBorrowLend": "2024-01-10T15:00:00.000Z",
  "updatedOnMultiply": null,
  "updatedOnLeverage": null,
  "pointsEarnedStrategies": "22761790.71117315449263523732644006896670425",
  "pointsEarnedBorrowLend": "10442477.186904250407027339999999999999982264",
  "pointsEarnedMultiply": "201389.4329133435694562408780984881999283067",
  "pointsEarnedLeverage": "0",
  "totalPointsEarned": "33405657.3309907484691188182045385571666148207",
  "currentPointsPerSecondStrategies": "2.966637247762797453716544976431414311005",
  "currentPointsPerSecondBorrowLend": "1.219000935252249471499999999999999999999",
  "currentPointsPerSecondMultiply": "0.1646297171217158133378010918910399701269",
  "currentPointsPerSecondLeverage": "0",
  "totalCurrentPointsPerSecond": "4.3502679001367627385543460683224542811309",
  "avgBoost": "1",
  "strategiesBreakdown": [
    {
      "strategy": "5EfeGn1h7m6Rx9mGEmamDoxMtdhRmUh2N9fYRiDQqteS",
      "lastCalculated": "2023-08-10T12:00:00.000Z",
      "pointsEarned": "3038.752099688225508715680139338092961558",
      "currentPointsPerSecond": "0",
      "boost": "0"
    },
    {
      "strategy": "5QgwaBQzzMAHdxpaVUgb4KrpXELgNTaEYXycUvNvRxr6",
      "lastCalculated": "2024-01-10T16:00:00.000Z",
      "pointsEarned": "2184181.274912947035496797367836501158876",
      "currentPointsPerSecond": "1.673149451222033829527709343420742693505",
      "boost": "1"
    },
    {
      "strategy": "8DRToyNBUTR4MxqkKAP49s9z1JhotQWy3rMQKEw1HHdu",
      "lastCalculated": "2024-01-10T16:00:00.000Z",
      "pointsEarned": "723051.1325021971074900781555286204179783",
      "currentPointsPerSecond": "0.2045374909997805147118103015547306358167",
      "boost": "1"
    }
  ],
  "klendBreakdown": [
    {
      "lastCalculated": "2024-01-10T15:00:00.000Z",
      "pointsEarned": "4196615.002722601454031089999999999999992",
      "currentPointsPerSecond": "1.219000935252249471499999999999999999999",
      "boost": "1",
      "obligationTag": "Vanilla",
      "obligation": "J3UBd95TrkWVb9d6moomMwEoAsHYcrRPprZX2vf3YG75",
      "market": "7u3HeHxYDLhnCoErrtycNokbQYbWGzLs6JSDqGAv5PfF"
    },
    {
      "lastCalculated": "2024-01-10T15:00:00.000Z",
      "pointsEarned": "200704.9350075835765854408408010038433802",
      "currentPointsPerSecond": "0.1646297171217158133378010918910399701269",
      "boost": "1",
      "obligationTag": "Multiply",
      "obligation": "BnNMMQdhytPsQYambAeX8LZCDfcA3Q72Uo7tV5TWvYcR",
      "market": "7u3HeHxYDLhnCoErrtycNokbQYbWGzLs6JSDqGAv5PfF"
    }
  ],
  "boosts": [
    {
      "boost": "0.07",
      "name": "loyalty_bonus"
    }
  ]
}
```

#### Get rules for points

```http request
GET https://api.hubbleprotocol.io/points/rules?env={solana cluster}
```

Example request:

- https://api.hubbleprotocol.io/points/rules?env=mainnet-beta

Example response:

```json
{
  "strategies": {
    "defaultRate": 2,
    "defaultBoost": 2,
    "strategyRates": {
      "Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN": 2
    },
    "strategyBoosts": {
      "Cfuy5T6osdazUeLego5LFycBQebm9PP3H7VNdCndXXEN": 2
    },
    "tokenRates": {
      "So11111111111111111111111111111111111111112": 2
    },
    "tokenBoosts": {
      "So11111111111111111111111111111111111111112": 2
    }
  },
  "lending": [
    {
      "market": "7u3HeHxYDLhnCoErrtycNokbQYbWGzLs6JSDqGAv5PfF",
      "defaultRates": {
        "borrowRate": 2,
        "depositRate": 2
      },
      "defaultBoosts": {
        "borrowBoost": 2,
        "depositBoost": 2
      },
      "tokenRates": {
        "So11111111111111111111111111111111111111112": {
          "depositRate": 1,
          "borrowRate": 3
        }
      },
      "tokenBoosts": {
        "So11111111111111111111111111111111111111112": {
          "depositBoost": 1,
          "borrowBoost": 3
        }
      }
    }
  ]
}
```

#### Get metrics for points

```http request
GET https://api.hubbleprotocol.io/points/metrics?source={source}
```

Example request:

- Fetch default points source (season 1): https://api.hubbleprotocol.io/points/metrics
- Fetch custom points source: https://api.hubbleprotocol.io/points/metrics?source=Season2

Example response:

```json
{
  "totalPointsEarned": "5984999924.493912541433019214995884898855713422508120471697",
  "totalCurrentPointsPerSecond": "3578.637988920129679710443661281444597259705465236233671",
  "currentPointsPerSecondLeverage": "52.040770416817290492365834856421461150509848186708961",
  "currentPointsPerSecondBorrowLend": "2891.05947353323817434578242345116583467662686554546471",
  "currentPointsPerSecondMultiply": "635.53774497007421487229540297385730143256875150406",
  "currentPointsPerSecondStrategies": "12354.58",
  "updatedOnLeverage": "2024-01-10T15:00:00.000Z",
  "updatedOnMultiply": "2024-01-10T15:00:00.000Z",
  "updatedOnStrategies": "2024-01-10T15:00:00.000Z",
  "updatedOnBorrowLend": "2024-01-10T15:00:00.000Z",
  "numberOfUsers": "91309"
}
```

### Airdrop

#### Get user airdrop allocation

```http request
GET https://api.hubbleprotocol.io/airdrop/users/:pubkey/allocations?source={source}
```

Example requests:

- get season 1 allocation: https://api.hubbleprotocol.io/airdrop/users/7QnXf4d1YQ6k8oTzCLXjEiikGFm3KRZgmFJmry9vZxdW/allocations?source=Season1

Example response:

```json
[
  {
    "quantity": "0.38493976366798",
    "name": "main_allocation"
  },
  {
    "quantity": "5.2414259465036",
    "name": "OG_allocation"
  }
]
```

Example empty response when user has 0 airdrop allocations to claim:

```json
[]
```

#### Get airdrop metrics

```http request
GET https://api.hubbleprotocol.io/airdrop/metrics?source={source}
```

Example requests:

- get season 1 airdrop metrics: https://api.hubbleprotocol.io/airdrop/metrics?source=Season1

Example response:

```json
{
  "totalAllocation": "800000000",
  "totalUsers": "251284",
  "claimDate": "2024-04-15T00:00:00.000Z"
}
```

Example response when no metrics are found (404 Not Found):

```json
{
  "metadata": "Airdrop metrics for source not found"
}
```

### Tokens

#### Get solflare token infos

Fetches all token infos from Solflare that are used by Kamino Liquidity/Lending/Farms.

```http request
GET https://api.hubbleprotocol.io/tokens
```

Example request:

- https://api.hubbleprotocol.io/tokens

Example response:

```json
[
  {
    "address": "27G8MtK7VtTcCHkpASjSDdkWWYfoqT6ggEuKidVJidD4",
    "chainId": 101,
    "name": "Jupiter Perpetuals Liquidity Provider Token",
    "symbol": "JLP",
    "verified": true,
    "decimals": 6,
    "holders": null,
    "logoURI": "https://assets.coingecko.com/coins/images/33094/large/jlp.png?1700631386",
    "tags": [],
    "extensions": {
      "coingeckoId": "jupiter-perpetuals-liquidity-provider-token"
    }
  }
]
```

Example empty response when user has 0 airdrop allocations to claim:

```json
[]
```

### KVaults

#### Get all kvault user transactions

Get all instructions for the specified KVault shareholder.

```http request
// GET https://api.kamino.finance/kvaults/shareholders/:shareholderPubkey/transactions?env={cluster}
```

Example request:

- https://api.kamino.finance/kvaults/shareholders/2frn6q2bDmfiGDnaiSFWqoJBm77fMHpGVnUV9yKYa7ts/transactions?env=mainnet-beta

Example response:

```json
[
  {
    "createdOn": "2024-10-30T11:49:03.000Z",
    "instruction": "invest",
    "tokenMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "tokenAmount": "1.531763",
    "usdValue": "1.53167261066537",
    "transaction": "3w1DghgGN8jJ2nhdYMXtjPPqBS3qf7ohcfazWwQH9qfeq53Gh7RNkJZwvezxqUg68PYnscqVz1xtMvZR4j2HboCZ"
  },
  {
    "createdOn": "2024-10-30T08:49:57.000Z",
    "instruction": "deposit",
    "tokenMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "tokenAmount": "1",
    "usdValue": "0.99994099",
    "transaction": "57KvZHPjcRunvw3HHBgCPbDyxaZubyfFhAGr5idkQiEmtQuNu9nJyoyJ7KBdHkZTVxXUPVSuD3vWQqgaMgoepCLC"
  },
  {
    "createdOn": "2024-10-30T08:04:58.000Z",
    "instruction": "withdraw",
    "tokenMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "tokenAmount": "6.09",
    "usdValue": "6.0896406291",
    "transaction": "3YhSNJPqpmXYUw76fr7ZkWSkgXcTkgKLNTxKhb4dMhXmZi69NTP8BTdZ96Z6fvMqK5GdFMbR1nUDbjLg6edNAEYw"
  }
]
```

#### Get kvault metrics

```http request
GET https://api.kamino.finance/kvaults/:vaultPubkey/metrics
```

Example request:

- https://api.kamino.finance/kvaults/6grCt9ChoQEJrR2e118z78PerPjF9GckYHZNnKDMfBKt/metrics

Example response:

```json
{
  "apy7d": "0.12",
  "apy24h": "0.1",
  "apy30d": "0.11",
  "apy90d": "0.15",
  "apy180d": "0.12525",
  "apy365d": "0.12458939831489045",
  "tokenPrice": "0.99951785",
  "solPrice": "176.16243353",
  "tokensAvailable": "5",
  "tokensAvailableUsd": "4.99758925",
  "tokensInvested": "0",
  "tokensInvestedUsd": "0",
  "sharePrice": "0.99951785",
  "tokensPerShare": "1",
  "apy": "0.060121900226019375",
  "numberOfHolders": 1,
  "sharesIssued": "5",
  "cumulativeInterestEarned": "8",
  "cumulativeInterestEarnedUsd": "6",
  "cumulativeInterestEarnedSol": "4",
  "interestEarnedPerSecond": "10",
  "interestEarnedPerSecondUsd": "2",
  "interestEarnedPerSecondSol": "11",
  "cumulativePerformanceFees": "1",
  "cumulativePerformanceFeesUsd": "7",
  "cumulativePerformanceFeesSol": "5",
  "cumulativeManagementFees": "12",
  "cumulativeManagementFeesUsd": "9",
  "cumulativeManagementFeesSol": "3"
}
```

#### Get kvault metrics history

```http request
GET https://api.kamino.finance/kvaults/:vaultPubkey/metrics/history?start={date}&end=${date}
```

Example request:

- https://api.kamino.finance/kvaults/6grCt9ChoQEJrR2e118z78PerPjF9GckYHZNnKDMfBKt/metrics/history?start=2024-10-20T00%3A00Z&end=2024-10-22T00%3A00Z'

Example response:

```json
[
  {
    "timestamp": "2024-10-20T11:30:00.000Z",
    "tvl": "4.99758925",
    "apy": "0.060121900226019375"
  },
  {
    "timestamp": "2024-10-20T11:35:00.000Z",
    "tvl": "5.23758925",
    "apy": "0.070121900226019375"
  }
]
```

#### Get user vault metrics history

Get timeseries of usd/sol/interest/apy... of specific kvault user position.

```http request
GET https://api.kamino.finance/kvaults/:vaultPubkey/users/:userPubkey/metrics/history?start={date}&end=${date}
```

Example request:

- https://api.kamino.finance/kvaults/GJZhNhQHFn3NpLhPgQjeyivNzAn548rGYg4EcuaxeCEf/users/sadmBTQm5HJsyzWHEjV4YwG9CiahZKVDVqAyS4Wx1zH/metrics/history?start=2024-10-20T00%3A00Z&end=2024-10-22T00%3A00Z'

Example response:

```json
[
  {
    "createdOn": "2025-01-14T11:53:54.734Z",
    "sharesAmount": "1",
    "usdAmount": "1.001009996801491595755991446853909208564",
    "solAmount": "0.005352941291953978820943628975408093773161",
    "apy": "0.07502216110843518",
    "cumulativeInterestEarned": "0",
    "cumulativeInterestEarnedUsd": "0",
    "cumulativeInterestEarnedSol": "0",
    "interestEarnedPerSecond": "0",
    "interestEarnedPerSecondUsd": "0",
    "interestEarnedPerSecondSol": "0"
  },
  {
    "createdOn": "2025-01-14T11:54:25.530Z",
    "sharesAmount": "1",
    "usdAmount": "1.001004212610976305808977819266756417541",
    "solAmount": "0.005350380274689194798426453878491338313099",
    "apy": "0.07502220627483847",
    "cumulativeInterestEarned": "0.000000002024844825466749284216827600944849020349",
    "cumulativeInterestEarnedUsd": "0.00000000202470104123569289035015536401690592662",
    "cumulativeInterestEarnedSol": "0.00000000001082205287120021365776673941575854315707",
    "interestEarnedPerSecond": "0.00000000006575025410659661267230153856437081675242",
    "interestEarnedPerSecondUsd": "0.00000000006574558518105250324683567843211736078072",
    "interestEarnedPerSecondSol": "0.0000000000003514109907520526580023154776522543481129"
  }
]
```

#### Get user total metrics history

Get timeseries of total usd/sol/interest/apy of all user kvault positions.

```http request
GET https://api.kamino.finance/kvaults/users/:userPubkey/metrics/history?start={date}&end=${date}
```

Example request:

- https://api.kamino.finance/kvaults/users/sadmBTQm5HJsyzWHEjV4YwG9CiahZKVDVqAyS4Wx1zH/metrics/history?start=2024-10-20T00%3A00Z&end=2024-10-22T00%3A00Z'

Example response:

```json
[
  {
    "createdOn": "2025-01-14T11:53:41.210Z",
    "usdAmount": "3.289742433439577096054433205125205428826",
    "solAmount": "0.01759203021760035613791855718659833263546",
    "weightedApy": "0.04573439694192186076452675960806281533141",
    "cumulativeInterestEarnedUsd": "0",
    "cumulativeInterestEarnedSol": "0",
    "interestEarnedPerSecondUsd": "0",
    "interestEarnedPerSecondSol": "0"
  },
  {
    "createdOn": "2025-01-14T11:54:25.530Z",
    "usdAmount": "3.289908683294689302333866319938107452865",
    "solAmount": "0.01758460384368975125597048731253291655472",
    "weightedApy": "0.04590131297089078620453886143250344888991",
    "cumulativeInterestEarnedUsd": "0.00000007792300015875231989706197228607687055565",
    "cumulativeInterestEarnedSol": "0.0000000004164994290149102036996178990966197185186",
    "interestEarnedPerSecondUsd": "0.000000002530296147511115725938296013257035461894",
    "interestEarnedPerSecondSol": "0.00000000001352446515829686334893879082828573239386"
  }
]
```

#### Get user vault pnl

```http request
GET https://api.kamino.finance/kvaults/:vaultPubkey/users/:userPubkey/pnl
```

Example request:

- https://api.kamino.finance/kvaults/GJZhNhQHFn3NpLhPgQjeyivNzAn548rGYg4EcuaxeCEf/users/sadmBTQm5HJsyzWHEjV4YwG9CiahZKVDVqAyS4Wx1zH/pnl

Example response:

```json
{
  "totalCostBasis": {
    "token": "1.020063426210193217882166040928",
    "sol": "0.025288434916229183793264807637",
    "usd": "6.000060406018646063098648036787"
  },
  "totalPnl": {
    "token": "4.991144525511436482070882061898",
    "sol": "-0.00173901651987720818426570281",
    "usd": "0.011449308342160062665337709055"
  }
}
```

#### Get user vault pnl history

```http request
GET https://api.kamino.finance/kvaults/:vaultPubkey/users/:userPubkey/pnl/history
```

Example request:

- https://api.kamino.finance/kvaults/GJZhNhQHFn3NpLhPgQjeyivNzAn548rGYg4EcuaxeCEf/users/sadmBTQm5HJsyzWHEjV4YwG9CiahZKVDVqAyS4Wx1zH/pnl/history

Example response:

```json
{
  "history": [
    {
      "timestamp": "2025-01-15T10:04:43.000Z",
      "type": "buy",
      "position": "0.999965",
      "quantity": "0.999965",
      "tokenPrice": {
        "token": "0.99999698",
        "sol": "189.10328327",
        "usd": "1"
      },
      "sharePrice": {
        "token": "1.000098429655231150972450076681",
        "sol": "0.005288620017929811336661590793",
        "usd": "1.000095409357973592174374139882"
      },
      "investment": {
        "token": "1.000063426210193217882166040928",
        "sol": "0.005288434916229183793264807637",
        "usd": "1.000060406018646063098648036787"
      },
      "costBasis": {
        "token": "1.000063426210193217882166040928",
        "sol": "0.005288434916229183793264807637",
        "usd": "1.000060406018646063098648036787"
      },
      "realizedPnl": {
        "token": "0",
        "sol": "0",
        "usd": "0"
      },
      "pnl": {
        "token": "0",
        "sol": "0",
        "usd": "0"
      },
      "positionValue": {
        "token": "1.000063426210193217882166040928",
        "sol": "0.005288434916229183793264807637",
        "usd": "1.000060406018646063098648036787"
      }
    },
    {
      "timestamp": "2025-01-14T22:34:04.000Z",
      "type": "sell",
      "position": "5.999965",
      "quantity": "5",
      "tokenPrice": {
        "token": "250",
        "sol": "250",
        "usd": "1"
      },
      "sharePrice": {
        "token": "0.004",
        "sol": "0.004",
        "usd": "1"
      },
      "investment": {
        "token": "1.020063426210193217882166040928",
        "sol": "0.025288434916229183793264807637",
        "usd": "6.000060406018646063098648036787"
      },
      "costBasis": {
        "token": "1.020063426210193217882166040928",
        "sol": "0.025288434916229183793264807637",
        "usd": "6.000060406018646063098648036787"
      },
      "realizedPnl": {
        "token": "0",
        "sol": "0",
        "usd": "0"
      },
      "pnl": {
        "token": "-0.996063566210193217882166040928",
        "sol": "-0.001288574916229183793264807637",
        "usd": "-0.000095406018646063098648036787"
      },
      "positionValue": {
        "token": "0.02399986",
        "sol": "0.02399986",
        "usd": "5.999965"
      }
    },
    {
      "timestamp": 1737712924618,
      "type": "mark-to-market",
      "position": "5.999965",
      "quantity": "5.999965",
      "tokenPrice": {
        "token": "0.99998362",
        "sol": "264.07338454",
        "usd": "1"
      },
      "sharePrice": {
        "token": "1.0019723663536509348902348102",
        "sol": "0.003794233015158408519892261112",
        "usd": "1.001955954046290062087921308153"
      },
      "investment": {
        "token": "1.020063426210193217882166040928",
        "sol": "0.025288434916229183793264807637",
        "usd": "6.000060406018646063098648036787"
      },
      "costBasis": {
        "token": "1.020063426210193217882166040928",
        "sol": "0.025288434916229183793264807637",
        "usd": "6.000060406018646063098648036787"
      },
      "realizedPnl": {
        "token": "0",
        "sol": "0",
        "usd": "0"
      },
      "pnl": {
        "token": "4.991735702878890013676521662053",
        "sol": "-0.002523169623434263218209437191",
        "usd": "0.01164024980070268927670673489"
      },
      "positionValue": {
        "token": "6.011799129089083231558687702982",
        "sol": "0.022765265292794920575055370446",
        "usd": "6.011700655819348752375354771677"
      }
    }
  ],
  "totalPnl": {
    "token": "4.991735702878890013676521662053",
    "sol": "-0.002523169623434263218209437191",
    "usd": "0.01164024980070268927670673489"
  },
  "totalCostBasis": {
    "token": "1.020063426210193217882166040928",
    "sol": "0.025288434916229183793264807637",
    "usd": "6.000060406018646063098648036787"
  }
}
```

#### Vault shares token (Kvault)

You may use the `env` query param for all the methods specified below (`mainnet-beta`[default],`devnet`,`localnet`,`testnet`).

All mints must be valid kVault share token mints.

##### Get kVault token metadata for mint

```http request
// GET https://api.hubbleprotocol.io/kvault-tokens/:mint/metadata
GET https://api.hubbleprotocol.io/kvault-tokens/BabJ4KTDUDqaBRWLFza3Ek3zEcjXaPDmeRGRwusQyLPS/metadata
```

##### Get kVault token image for mint

```http request
// GET https://api.hubbleprotocol.io/kvault-tokens/:mint/metadata/image.svg
GET https://api.hubbleprotocol.io/kvault-tokens/BabJ4KTDUDqaBRWLFza3Ek3zEcjXaPDmeRGRwusQyLPS/metadata/image.svg


### Limo

#### Get all limo user transactions

Get all instructions for the specified Limo user (maker).

```http request
// GET https://api.kamino.finance/limo/makers/:makerPubkey/transactions?env={cluster}&in={inTokenMint}&out={outTokenMint}
```

Query params:

- env: solana cluster, e.g. `"mainnet-beta" (default) | "devnet"`
- in: input token mint of the order, e.g. `So11111111111111111111111111111111111111112`
- out: output token mint of the order, e.g. `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`

Example requests:

- Fetch all txs for all pairs: https://api.kamino.finance/limo/makers/7bfsDRVpujyxzYcNWHnN3ietYb5XEsAidgdc9cintAuj/transactions?env=mainnet-beta
- Fetch txs for SOL/USDC pair: https://api.kamino.finance/limo/makers/CBd9omWgziKgBhmAqrGREDJqsSvM1HrarEMzE89zawMa/transactions?env=mainnet-beta&in=So11111111111111111111111111111111111111112&out=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
- Fetch txs for SOL input token and any output token: https://api.kamino.finance/limo/makers/CBd9omWgziKgBhmAqrGREDJqsSvM1HrarEMzE89zawMa/transactions?env=mainnet-beta&in=So11111111111111111111111111111111111111112

Example response:

```json
[
  {
    "transactionSignature": "3DA1t96dsHaTjvDm9w6Z1dnBL5G21uq21wYK1mYxbDaBfxQnau2yCEkinn5LoaJPW3Ud12iPBQdfj7cuNtXzJ7qQ",
    "ixName": "closeOrderAndClaimTip",
    "order": "Bh5fXLmfh9WNZPfL3sp1RCa7itrV1BXQGoJcqRfkz3Hh",
    "orderStatus": "Cancelled",
    "orderType": "ExactIn",
    "updatedOn": "2024-11-19T19:12:22.000Z",
    "inMint": "So11111111111111111111111111111111111111112",
    "outMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "initialInAmount": "0.01",
    "remainingInAmount": "0.01",
    "expectedOutAmount": "2.41",
    "filledOutAmount": "0",
    "numberOfFills": "0",
    "totalFilled": "0",
    "orderPriceInOut": "0.00414937759336099585",
    "orderPriceOutIn": "241",
    "executionPriceInOut": "0",
    "executionPriceOutIn": "0",
    "surplus": "0",
    "tipAmount": "0",
    "networkFee": "0.001138901",
    "solPrice": "240.123"
  },
  {
    "transactionSignature": "5Szg5BFsF8XCRCCN6NobRzSxBqoKEJXzkFs9L9473GRN21fm9d5zsdHpT4saPDgjuj4gxRW7CrJC3aFgvPDYdEAN",
    "ixName": "createOrder",
    "order": "Bh5fXLmfh9WNZPfL3sp1RCa7itrV1BXQGoJcqRfkz3Hh",
    "orderStatus": "Active",
    "orderType": "ExactIn",
    "updatedOn": "2024-11-19T19:12:22.000Z",
    "inMint": "So11111111111111111111111111111111111111112",
    "outMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "initialInAmount": "0.01",
    "remainingInAmount": "0.01",
    "expectedOutAmount": "2.41",
    "filledOutAmount": "0",
    "numberOfFills": "0",
    "totalFilled": "0",
    "orderPriceInOut": "0.00414937759336099585",
    "orderPriceOutIn": "241",
    "executionPriceInOut": "0",
    "executionPriceOutIn": "0",
    "surplus": "0",
    "tipAmount": "0",
    "networkFee": "0.001125865",
    "solPrice": "240.123"
  }
]
```

#### Get limo maker metrics

```http request
// GET https://api.kamino.finance/limo/makers/:makerPubkey/metrics
```

Example request:

- https://api.kamino.finance/limo/makers/CBd9omWgziKgBhmAqrGREDJqsSvM1HrarEMzE89zawMa/metrics

Example response:

```json
{
  "totalVolumeUsd": "500.123",
  "totalOrders": 3,
  "totalTipsSol": "0.000043296",
  "totalTipsUsd": "1.24585",
  "totalSurplusUsd": "38.385940",
  "favoritePair": {
    "inTokenPubkey": "So11111111111111111111111111111111111111112",
    "outTokenPubkey": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "tradeCount": 6
  }
}
```

#### Get limo maker pair metrics

```http request
// GET https://api.kamino.finance/limo/makers/:makerPubkey/metrics/pair?in={inTokenMint}&out={outTokenMint}
```

Query params:

- in: input token mint of the order, e.g. `So11111111111111111111111111111111111111112`
- out: output token mint of the order, e.g. `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`

Example request:

- https://api.kamino.finance/limo/makers/CBd9omWgziKgBhmAqrGREDJqsSvM1HrarEMzE89zawMa/metrics/pair?in=So11111111111111111111111111111111111111112&out=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v

Example response:

```json
{
  "totalVolumeUsd": "100.123",
  "totalOrders": 2,
  "totalTipsSol": "0.000043296",
  "totalTipsUsd": "0.3248",
  "totalSurplusUsd": "0.1"
}
```

#### Get limo global metrics

Get metrics for all trades

```http request
// GET https://api.kamino.finance/limo/metrics
```

Example request:

- https://api.kamino.finance/limo/metrics

Example response:

```json
{
  "allTime": {
    "totalVolumeUsd": "32179.204554150009005121121",
    "totalSurplus": "0.01794400389018700094",
    "totalSurplusUsd": "4.3134619210239461983798395146",
    "totalTipAmount": "0.002224553",
    "totalTipAmountUsd": "0.5417640091257365",
    "totalTrades": "10"
  },
  "daily": {
    "totalVolumeUsd": "1.598927244",
    "totalSurplus": "0",
    "totalSurplusUsd": "0",
    "totalTipAmount": "0.000730568",
    "totalTipAmountUsd": "0.182642",
    "totalTrades": "1"
  }
}
```

#### Get limo pair metrics

Get metrics for a specific token pair

```http request
// GET https://api.kamino.finance/limo/metrics/pair?in={inTokenMint}&out={outTokenMint}
```

Query params:

- in: input token mint of the order, e.g. `So11111111111111111111111111111111111111112`
- out: output token mint of the order, e.g. `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`

Example request:

- https://api.kamino.finance/limo/metrics/pair?in=So11111111111111111111111111111111111111112&out=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v

Example response:

```json
{
  "allTime": {
    "totalVolumeUsd": "32179.204554150009005121121",
    "totalSurplus": "0.01794400389018700094",
    "totalSurplusUsd": "4.3134619210239461983798395146",
    "totalTipAmount": "0.002224553",
    "totalTipAmountUsd": "0.5417640091257365",
    "totalTrades": "10"
  },
  "daily": {
    "totalVolumeUsd": "1.598927244",
    "totalSurplus": "0",
    "totalSurplusUsd": "0",
    "totalTipAmount": "0.000730568",
    "totalTipAmountUsd": "0.182642",
    "totalTrades": "1"
  }
}
```

#### Get limo weekly leaderboard metrics

```http request
// GET https://api.kamino.finance/limo/leaderboard/weekly/metrics
```

Example request:

- https://api.kamino.finance/limo/leaderboard/weekly/metrics

Example response:

```json
{
  "weeklyVolumeUsd": "1.598927244",
  "weeklyTickets": "0.001598927244",
  "weeklyTipsUsd": "0.182642",
  "weeklySurplusUsd": "0"
}
```

#### Get limo weekly leaderboard

```http request
GET https://api.kamino.finance/limo/weekly/leaderboard?offset={offset}&limit=${limit}
```

Example requests:

- get first 20 leaderboard ranks: https://api.kamino.finance/limo/weekly/leaderboard?offset=0&limit=20
- get leaderboard ranks between 100-200: https://api.kamino.finance/limo/weekly/leaderboard?offset=100&limit=100

To implement pagination fetch the first page (offset = 0, limit = 20 for example) and then look at the property `totalLeaderboardCount` to calculate the amount of pages you will display.

Example response:

```json
{
  "totalLeaderboardCount": 9,
  "leaderboard": [
    {
      "rank": 1,
      "maker": "tsTSFNQsjqKArV5SvtEVePkZ6Crbkf65KhgmniyrFkV",
      "volumeUsd": "23.57484499976425155",
      "tickets": "0.02357484499976425155",
      "tipsUsd": "0",
      "surplusUsd": "0"
    },
    {
      "rank": 2,
      "maker": "4RHjaQ2c6mFVsjyu7nR4o69QTVn8hS6yn4WTspwR2R19",
      "volumeUsd": "1.59892724398401072756",
      "tickets": "0.00159892724398401073",
      "tipsUsd": "0.1754275283464144",
      "surplusUsd": "0"
    },
    {
      "rank": 3,
      "maker": "AoDMpV5zPYP9trk6d52ZU5nqpYRKHtHYmnoABxapDXQR",
      "volumeUsd": "0.09549704199904502958",
      "tickets": "0.00009549704199904502958",
      "tipsUsd": "0.0044768876250952",
      "surplusUsd": "0.000000003151099999968489"
    }
  ]
}
```

#### Get last 500 limo trades

Get last 500 limo trades (fills)

```http request
// GET https://api.kamino.finance/limo/trades?in={inTokenMint:optional}&out={outTokenMint:optional}
```

Query params:

- in (optional): input token mint of the order, e.g. `So11111111111111111111111111111111111111112`
- out (optional): output token mint of the order, e.g. `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`

If no query params are used, it will return trades for all pairs.

Example request:

- Get all trades: https://api.kamino.finance/limo/trades
- Get SOL -> USDC trades: https://api.kamino.finance/limo/trades?in=So11111111111111111111111111111111111111112&out=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v

Example response:

```json
[
  {
    "updatedOn": "2024-11-26T09:30:33.000Z",
    "inMint": "So11111111111111111111111111111111111111112",
    "outMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "sizeUsd": "23.94585489",
    "tipAmountUsd": "0.001",
    "surplusUsd": "0.01",
    "order": "FqWpdGoN1CS21FVQG4weL9ndon9cfzhWDZmrScfALN8x"
  },
  {
    "updatedOn": "2024-11-22T14:44:18.000Z",
    "inMint": "So11111111111111111111111111111111111111112",
    "outMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "sizeUsd": "119.72927445",
    "tipAmountUsd": "0.0021632847358122",
    "surplusUsd": "0.48982",
    "order": "G1VtPA6y7EMWQSgRUv13L9LSTSgFtcZYij8cXNQddXue"
  }
]
```

#### Bulk get Limo transactions

This endpoint is not open to the public, it is private for analytical purposes only, add authorization headers to the request.

```http request
POST https://api.kamino.finance/limo/transactions
```

```json
{
  "start": "2024-01-01T00:00Z",
  "end": "2024-01-02T00:00Z",
  "paginationToken": ""
}
```

Example cURL request:

```bash
curl --location 'https://api.kamino.finance/limo/transactions' --header 'Content-Type: application/json' --header 'Authorization: Basic ENTER_CREDENTIALS' --data '{
    "start": "2024-01-01",
    "end": "2024-01-02",
    "paginationToken": ""
}'
```

Example Python request:

```python
import requests
import json

url = "https://api.kamino.finance/limo/transactions"

payload = json.dumps({
  "start": "2024-01-01",
  "end": "2024-01-02",
  "paginationToken": ""
})
headers = {
  'Content-Type': 'application/json',
  'Authorization': 'Basic ADD-CREDENTIALS-HERE'
}

response = requests.request("POST", url, headers=headers, data=payload)

print(response.text)
```

Example response:

```json
{
  "result": [
    {
      "maker": "DAnimibJrqNQd8NEjEWKwcB8VZzTKT3DWvFvfHjufDF",
      "taker": null,
      "transactionSignature": "3DYGcEUzzzyYXX7sHEM5Ras7CDzdRdEnMdzEjayqZXJMH6a7KGqaV8jw6SN15PAkM2pPBhqGT7N5DRSDXhyqcfUL",
      "ixName": "createOrder",
      "initialInAmount": "0.001",
      "remainingInAmount": "0.001",
      "expectedOutAmount": "0.000004062",
      "cumulativeFilledOutAmount": "0",
      "lastFilledOutAmount": "0",
      "numberOfFills": "0",
      "volumeUsd": "0",
      "fillRatio": "0",
      "orderPriceInOut": "246.1841457410142787",
      "orderPriceOutIn": "0.004062",
      "executionPriceInOut": "0",
      "executionPriceOutIn": "0",
      "surplus": "0",
      "tipAmount": "0",
      "networkFee": "0.000288189",
      "solPrice": "0",
      "inTokenPrice": "0",
      "outTokenPrice": "0",
      "updatedOn": "2024-11-28T12:01:25.000Z",
      "orderStatus": "Active",
      "inMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "outMint": "So11111111111111111111111111111111111111112",
      "orderPubkey": "BvaSkA7RN3XW3ApxxouZtZNCb6yKdBtSir3xncCsCMCV",
      "orderType": "ExactIn"
    },
    {
      "maker": "DAnimibJrqNQd8NEjEWKwcB8VZzTKT3DWvFvfHjufDF",
      "taker": "76qh9UEKv8We4Um5bznSPvn2jkNrckq2hKeNQt8JNNKQ",
      "transactionSignature": "5tPfMkqgZd6QA8sCnR9mmcnFMu9z1qCLwevUmkAszN61QxPMepSdBeEX39pZqKJvhAXtNi2A7SKgQrJiQSdv1WQt",
      "ixName": "takeOrder",
      "initialInAmount": "0.001",
      "remainingInAmount": "0",
      "expectedOutAmount": "0.000004062",
      "cumulativeFilledOutAmount": "0.000004062",
      "lastFilledOutAmount": "0.000004062",
      "numberOfFills": "1",
      "volumeUsd": "0",
      "fillRatio": "1",
      "orderPriceInOut": "246.1841457410142787",
      "orderPriceOutIn": "0.004062",
      "executionPriceInOut": "246.1841457410142787",
      "executionPriceOutIn": "0.004062",
      "surplus": "0",
      "tipAmount": "0.000000047",
      "networkFee": "0.00025",
      "solPrice": "0",
      "inTokenPrice": "0",
      "outTokenPrice": "0",
      "updatedOn": "2024-11-28T13:03:36.000Z",
      "orderStatus": "Filled",
      "inMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "outMint": "So11111111111111111111111111111111111111112",
      "orderPubkey": "HtT3sFFRC1ScVzxLfXURu3sRha8TBwZbuYVrV8EBSn18",
      "orderType": "ExactIn"
    }
  ],
  "paginationToken": "eyJsYXN0SWQiOjEwNjU3fQ=="
}
```

### Slot

#### Get recent slot duration

Fetch median slot duration (in milliseconds) from the last 10 epochs.

```http request
// GET https://api.kamino.finance/slots/duration
```

Example request:

- https://api.kamino.finance/slots/duration

Example response:

```json
{
  "recentSlotDurationInMs": 441
}
```

### Data

#### Get owner net value history

```http request
GET https://api.kamino.finance/owners/:ownerPubkey/net-values/history?start={date}&end=${date}
```

Example request:

- https://api.kamino.finance/owners/DAnimibJrqNQd8NEjEWKwcB8VZzTKT3DWvFvfHjufDF/net-values/history?start=2025-02-25T00:00Z&end=2025-02-26T15:00Z

Example response:

```json
[
  {
    "createdOn": "2025-02-25T20:00:42.414Z",
    "klendUsd": "12106.302965",
    "strategiesUsd": "31.398135",
    "kvaultsUsd": "0.200337",
    "stakingUsd": "3854.553228"
  },
  {
    "createdOn": "2025-02-25T20:06:18.801Z",
    "klendUsd": "12127.114367",
    "strategiesUsd": "31.251037",
    "kvaultsUsd": "0.200329",
    "stakingUsd": "3837.520045"
  },
  {
    "createdOn": "2025-02-26T10:47:33.799Z",
    "klendUsd": "11807.027686",
    "strategiesUsd": "31.500079",
    "kvaultsUsd": "0.20034",
    "stakingUsd": "3893.594326"
  }
]
```

### Get KMNO circulating supply

Get unlocked circulating supply of KMNO:

Example request:
- https://api.kamino.finance/kmno/circulating-supply

Example response:

```
1395205483.5
```

### Get KMNO total supply

Get total supply of KMNO:

Example request:
- https://api.kamino.finance/kmno/total-supply

Example response:

```
9999983187.806662
```

## KSwap

### Get Token by Mint

Retrieves detailed information about a specific token using its mint address.

```http request
GET https://api.kamino.finance/kamino-swap/tokens/:mint/metadata
```

#### Parameters

| Parameter | Type   | Required | Description              |
| --------- | ------ | -------- | ------------------------ |
| mint      | string | Yes      | The token's mint address |

#### Example Request

https://api.kamino.finance/kamino-swap/tokens/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/metadata

#### Example Response

```json
{
  "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "logo_url": "https://raw.githubusercontent.com/solana-labs/token-list/main/assets/mainnet/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/logo.png",
  "name": "USD Coin",
  "symbol": "USDC",
  "decimals": 6,
  "market_cap_usd": "33429747283.29",
  "volume_usd": "1298374659.87",
  "verified": true
}
```

#### Status Codes

| Status Code | Description           |
| ----------- | --------------------- |
| 200         | Success               |
| 404         | Token not found       |
| 500         | Internal server error |

---

### Search Tokens

Searches for tokens by mint address, name, or symbol. Results are ranked by relevance to the search term, then by trading volume.

```http request
GET https://api.kamino.finance/kamino-swap/tokens/search?query={search_term}
```

#### Parameters

| Parameter | Type   | Required | Description                  |
| --------- | ------ | -------- | ---------------------------- |
| query     | string | Yes      | Search term for token lookup |

#### Example Request

https://api.kamino.finance/kamino-swap/tokens/search?query=usd

#### Example Response

```json
[
  {
    "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "logo_url": "https://raw.githubusercontent.com/solana-labs/token-list/main/assets/mainnet/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/logo.png",
    "name": "USD Coin",
    "symbol": "USDC",
    "decimals": 6,
    "market_cap_usd": "9448162946.26864",
    "volume_usd": "2330415223.6923323",
    "verified": true
  },
  {
    "mint": "Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB",
    "logo_url": "https://raw.githubusercontent.com/solana-labs/token-list/main/assets/mainnet/Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB/logo.svg",
    "name": "USDT",
    "symbol": "USDT",
    "decimals": 6,
    "market_cap_usd": "2038600947.9178107",
    "volume_usd": "500332625.71846175",
    "verified": true
  }
]
```

#### Status Codes

| Status Code | Description           |
| ----------- | --------------------- |
| 200         | Success               |
| 500         | Internal server error |

---

### Get Tokens with Limit

Retrieves a list of tokens sorted by trading volume in descending order.

```http request
GET https://api.kamino.finance/kamino-swap/tokens?limit={limit}&offset={offset}
```

#### Parameters

| Parameter | Type   | Required | Description                                                   |
| --------- | ------ | -------- | ------------------------------------------------------------- |
| limit     | number | No       | Maximum number of tokens to return (default: 1000, max: 1000) |
| offset    | number | No       | Number of tokens to skip (default: 0)                         |

#### Example Request

https://api.kamino.finance/kamino-swap/tokens?limit=10&offset=0

#### Example Response

```json
[
  {
    "mint": "So11111111111111111111111111111111111111112",
    "logo_url": "https://raw.githubusercontent.com/solana-labs/token-list/main/assets/mainnet/So11111111111111111111111111111111111111112/logo.png",
    "name": "Wrapped SOL",
    "symbol": "SOL",
    "decimals": 9,
    "market_cap_usd": "29845721638.45",
    "volume_usd": "1837465998.34",
    "verified": true
  },
  {
    "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "logo_url": "https://raw.githubusercontent.com/solana-labs/token-list/main/assets/mainnet/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/logo.png",
    "name": "USD Coin",
    "symbol": "USDC",
    "decimals": 6,
    "market_cap_usd": "33429747283.29",
    "volume_usd": "1298374659.87",
    "verified": true
  }
  /* Additional tokens... */
]
```

#### Status Codes

| Status Code | Description                       |
| ----------- | --------------------------------- |
| 200         | Success                           |
| 400         | Invalid limit or offset parameter |
| 500         | Internal server error             |

---

### Get Verified Tokens

Retrieves a list of verified tokens sorted by trading volume in descending order.

```http request
GET https://api.kamino.finance/kamino-swap/tokens/verified?limit={limit}&offset={offset}
```

#### Parameters

| Parameter | Type   | Required | Description                                                   |
| --------- | ------ | -------- | ------------------------------------------------------------- |
| limit     | number | No       | Maximum number of tokens to return (default: 1000, max: 1000) |
| offset    | number | No       | Number of tokens to skip (default: 0)                         |

#### Example Request

https://api.kamino.finance/kamino-swap/tokens/verified?limit=5

#### Example Response

```json
[
  {
    "mint": "So11111111111111111111111111111111111111112",
    "logo_url": "https://raw.githubusercontent.com/solana-labs/token-list/main/assets/mainnet/So11111111111111111111111111111111111111112/logo.png",
    "name": "Wrapped SOL",
    "symbol": "SOL",
    "decimals": 9,
    "market_cap_usd": "29845721638.45",
    "volume_usd": "1837465998.34",
    "verified": true
  },
  {
    "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "logo_url": "https://raw.githubusercontent.com/solana-labs/token-list/main/assets/mainnet/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/logo.png",
    "name": "USD Coin",
    "symbol": "USDC",
    "decimals": 6,
    "market_cap_usd": "33429747283.29",
    "volume_usd": "1298374659.87",
    "verified": true
  }
  /* Additional verified tokens... */
]
```

#### Status Codes

| Status Code | Description                       |
| ----------- | --------------------------------- |
| 200         | Success                           |
| 400         | Invalid limit or offset parameter |
| 500         | Internal server error             |

---

### Get Batch Tokens

Retrieves a list of tokens given by the user param

```http request
GET https://api.kamino.finance/kamino-swap/tokens/batch?mints=mint1,mint2,mint3
```

#### Parameters

| Parameter | Type   | Required | Description                                                   |
| --------- | ------ | -------- | ------------------------------------------------------------- |
| mints     | string | Yes      | A comma separated list of mints to return (maxLength 100)     |

#### Example Request

https://api.kamino.finance/kamino-swap/tokens/batch?mints=So11111111111111111111111111111111111111112,EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v,DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263

#### Example Response

```json
[
    {
        "mint": "DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263",
        "logoUrl": "https://arweave.net/hQiPZOsRZXGXBJd_82PhVdlM_hACsT_q6wqwf5cSY7I",
        "name": "Bonk",
        "symbol": "Bonk",
        "decimals": 5,
        "marketCapUsd": "975289192.7060107",
        "volumeUsd": "2645961.153572351",
        "verified": true
    },
    {
        "mint": "So11111111111111111111111111111111111111112",
        "logoUrl": "https://raw.githubusercontent.com/solana-labs/token-list/main/assets/mainnet/So11111111111111111111111111111111111111112/logo.png",
        "name": "Wrapped SOL",
        "symbol": "SOL",
        "decimals": 9,
        "marketCapUsd": "64971616518.36973",
        "volumeUsd": "1708739344.09589",
        "verified": true
    },
    {
        "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "logoUrl": "https://raw.githubusercontent.com/solana-labs/token-list/main/assets/mainnet/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v/logo.png",
        "name": "USD Coin",
        "symbol": "USDC",
        "decimals": 6,
        "marketCapUsd": "9915651729.543299",
        "volumeUsd": "679979407.2115899",
        "verified": true
    }
  /* Additional verified tokens... */
]
```

#### Status Codes

| Status Code | Description                       |
| ----------- | --------------------------------- |
| 200         | Success                           |
| 400         | Invalid mints parameters          |
| 500         | Internal server error             |

---

### Get user swap transactions

Fetch all kswap transactions for a user.

```http request
GET https://api.kamino.finance/kamino-swap/users/:wallet/transactions?limit={limit}&offset={offset}
```

#### Parameters

| Parameter | Type   | Required | Description                                                   |
| --------- | ------ | -------- | ------------------------------------------------------------- |
| limit     | number | No       | Maximum number of tokens to return (default: 1000, max: 1000) |
| offset    | number | No       | Number of tokens to skip (default: 0)                         |

#### Example request:

- https://api.kamino.finance/kamino-swap/users/7QnXf4d1YQ6k8oTzCLXjEiikGFm3KRZgmFJmry9vZxdW/transactions?limit=2

#### Example response:

The array of transactions is ordered by createdOn timestamp descending.

```json
[
  {
    "createdOn": "2025-04-03T11:10:23.000Z",
    "transactionSignature": "f228r3iu4cUmGQi7UpjFr8eeKEZkptRDsxMjVYmFu91FiBapW2iKwxturQBsqQRMXixreAXHgnqq8Sg8JbbuRZe",
    "inAmount": "67171.864666",
    "outAmount": "74121.92593",
    "volumeUsd": "74207.03736617967072",
    "inMint": "HzwqbKZw8HxMN6bF2yFZNrht3c2iXXzpKcFu7uBEDKtr",
    "outMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
  },
  {
    "createdOn": "2025-04-03T11:08:56.000Z",
    "transactionSignature": "3nYsPp59ijTTnmzVJzYpsYg2AtGcx9zwWYxb4AXmp4bGtkSMkVAnsmt8FEAmph8gTodB9QZXPAva486tYNhmhqVq",
    "inAmount": "50000",
    "outAmount": "55122.664699",
    "volumeUsd": "55228.579",
    "inMint": "HzwqbKZw8HxMN6bF2yFZNrht3c2iXXzpKcFu7uBEDKtr",
    "outMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
  }
]
```

---

### Get user swap volume

Fetches all-time kswap volume for a user.

```http request
GET https://api.kamino.finance/kamino-swap/users/:wallet/volume
```

#### Example request:

- https://api.kamino.finance/kamino-swap/users/7QnXf4d1YQ6k8oTzCLXjEiikGFm3KRZgmFJmry9vZxdW/volume

#### Example response:

```json
{
  "cumulativeVolumeUsd": "400.123"
}
```

---

### Get user swap savings

Fetches user savings for a specific date range. If no date range is provided, it will return the cumulative savings for the last 30 days.

```http request
GET https://api.kamino.finance/kamino-swap/users/:wallet/savings
```
#### Example request:

- https://api.kamino.finance/kamino-swap/users/7QnXf4d1YQ6k8oTzCLXjEiikGFm3KRZgmFJmry9vZxdW/savings?start=2025-04-15T00%3A00Z&end=2025-04-30T10%3A00Z

#### Example response:

```json
{
  "cumulativeSavingsUsd": "100.34"
}
```

---

### Get user swap volume leaderboard metrics

```http request
GET https://api.kamino.finance/kamino-swap/leaderboard/users/:wallet/metrics
```

#### Example request:

- https://api.kamino.finance/kamino-swap/leaderboard/users/7QnXf4d1YQ6k8oTzCLXjEiikGFm3KRZgmFJmry9vZxdW/metrics

Example response:

#### Example response:

```json
{
  "leaderboardRank": 1,
  "volumeUsd": "400.46",
  "weeklyLeaderboardRank": 2,
  "weeklyVolumeUsd": "300.123"
}
```

---

### Get swap metrics

```http request
GET https://api.kamino.finance/kamino-swap/metrics
```

#### Example request:

- https://api.kamino.finance/kamino-swap/metrics

#### Example response:

```json
{
  "allTimeVolumeUsd": "1000.2458458",
  "weeklyVolumeUsd": "100.54899",
  "userCount": "4465",
  "searchersCount24h": "57"
}
```

---

### Get Swap Volume Leaderboard

```http request
GET https://api.kamino.finance/kamino-swap/leaderboard?offset={offset}&limit={limit}
```

#### Example request:

- Get top 10 users on swap leaderboard: https://api.kamino.finance/kamino-swap/leaderboard?offset=0&limit=10
- Get next 10 users on swap leaderboard: https://api.kamino.finance/kamino-swap/leaderboard?offset=10&limit=10

Example response:

#### Example response:

```json
{
    "totalLeaderboardCount": 2,
    "leaderboard": [
        {
            "wallet": "24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p",
            "leaderboardRank": 1,
            "volumeUsd": "600.1",
            "weeklyLeaderboardRank": 1,
            "weeklyVolumeUsd": "200.3"
        },
        {
            "wallet": "GNt8N6ccG9RGgPxAEzEEb8HpdncR2KroS1LxM8Vyge75",
            "leaderboardRank": 2,
            "volumeUsd": "200",
            "weeklyLeaderboardRank": null,
            "weeklyVolumeUsd": null
        }
    ]
}
```

---

### Bulk get Swap transactions

This endpoint is not open to the public, it is private for analytical purposes only, add authorization headers to the request.

```http request
POST https://api.kamino.finance/kamino-swap/transactions
```

```json
{
  "start": "2024-01-01T00:00Z",
  "end": "2024-01-02T00:00Z",
  "paginationToken": ""
}
```

#### Example cURL request:

```bash
curl --location 'https://api.kamino.finance/kamino-swap/transactions' --header 'Content-Type: application/json' --header 'Authorization: Basic ENTER_CREDENTIALS' --data '{
    "start": "2024-01-01",
    "end": "2024-01-02",
    "paginationToken": ""
}'
```

#### Example Python request:

```python
import requests
import json

url = "https://api.kamino.finance/kamino-swap/transactions"

payload = json.dumps({
  "start": "2024-01-01",
  "end": "2024-01-02",
  "paginationToken": ""
})
headers = {
  'Content-Type': 'application/json',
  'Authorization': 'Basic ADD-CREDENTIALS-HERE'
}

response = requests.request("POST", url, headers=headers, data=payload)

print(response.text)
```

#### Example response:

```json
{
  "result": [
    {
      "transactionSignature": "5NRXmm1bTK45xaJgc16Rxrrf8UvQxbYnR4sdZ5wAcoHQo4Sysy9JYejDeJv3dG79Q2eckLPvxHvfdKPPhmRyxu3g",
      "transactionFee": "0.00002175",
      "owner": "ARygwR6WBPvatQ2bBrSyytsyBgqrRDwxEuANWSSydfXW",
      "inMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "outMint": "So11111111111111111111111111111111111111112",
      "programAccount": "PytERJFhAKuNNuaiXkApLfWzwNwSNDACpigT3LwQfou",
      "updatedOn": "2025-03-20T09:37:43.000Z",
      "inAmount": "10",
      "outAmount": "0.074781088",
      "referrer": null,
      "searcher": "H8FpJ3XvXDoAjBSwEZECteog4hNVJR63G6fWKTPv1R6k"
    },
    {
      "transactionSignature": "2v1KJctTmvam9uCeVNfNC7w1pQtiebqTXr5A3KH9LYSvdqgfBHwkWzvs2jLfJ8om11AMoge1DBMTQswc6PkcNZ1i",
      "transactionFee": "0.000025125",
      "owner": "BkfR7rREgvDbDQ7TtLSLpmsFAiV4TWxAYwzocPXaLQ2f",
      "inMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "outMint": "So11111111111111111111111111111111111111112",
      "programAccount": "PytERJFhAKuNNuaiXkApLfWzwNwSNDACpigT3LwQfou",
      "updatedOn": "2025-03-20T10:31:51.000Z",
      "inAmount": "1",
      "outAmount": "0.007638043",
      "referrer": "H8FpJ3XvXDoAjBSwEZECteog4hNVJR63G6fWKTPv1R6k",
      "searcher": null
    }
  ],
  "paginationToken": "eyJsYXN0SWQiOjEwNjU3fQ=="
}
```
---


## Referrals

The following endpoints allow users to interact with the referral system on Kamino.

### How to authenticate correctly

1. Fetch the referral message that needs to be signed from the API:

```http request
GET https://api.kamino.finance/users/:pubkey/referral-message
```

Example response:

```json
  {
      "message": "By signing this message you are requesting to create or update your referral code on Kamino Swap. Unique message ID: 726bbe0d-95a9-4da1-aadc-5ac350f03b79",
      "session": "77869934-2e4a-4553-ba4c-fde396857fe0",
      "expiresOn": "2025-04-04T14:18:31.405Z"
  }
```

2. Sign the message with the wallet that is creating or updating the referral code. Save the `session` part of the response as well.
3. Add the `signature` and `session` to the request body of the API endpoint you're calling.

Please see https://solana.com/developers/cookbook/wallets/sign-message for more information on how to sign a message with a wallet.

Example code on how to do it in TypeScript:

```typescript
// In Solana Web3.js v1, we can use the TweetNaCl crypto library:
import { Keypair } from '@solana/web3.js';
import nacl from 'tweetnacl';
import nacl_util from 'tweetnacl-util';

async function signMessageWithWalletProvider(provider: any) {
  const message = 'my-referral-code';
  const encodedMessage = new TextEncoder().encode(message);

  // Request the user's public key and sign the message
  const signedMessage = await provider.signMessage(encodedMessage, 'utf8');

  return {
    walletAddress: signedMessage.publicKey.toBase58(),
    signature: bs58.encode(Buffer.from(signedMessage.signature)),
  };
}

function signMessageWithKeypair(keypair: Keypair){
  const message = 'my-referral-code';
  const messageBytes = nacl_util.decodeUTF8(message);

  // pass this signature to the API
  const signature = bs58.encode(nacl.sign.detached(messageBytes, keypair.secretKey));
  return signature;
}
```

### Get User Referral Code

Retrieve the referral code associated with a given wallet.

```http
GET https://api.kamino.finance/users/:wallet/referral-code
```

#### Parameters:

- `wallet` (path parameter, string, required): The wallet public key.

#### Example request:

- https://api.kamino.finance/users/24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p/referral-code

#### Example response:

```json
{
    "code": "my-referral-code9",
    "referrer": "24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p",
    "pda": "BrzGZgme5PyW8PPp7VKSDcs8jKFWL76ik3Wo9Zfq4iyq"
}
```

#### Possible Errors:

- `400 Bad Request`: Invalid wallet address format.
- `404 Not Found`: No referral code found for the given wallet.

---

### Get Referral Code

Retrieve the wallet associated with a given referral code.

```http
GET https://api.kamino.finance/referral-codes/:code
```

#### Parameters:

- `code` (path parameter, string, required): The referral code.

#### Example request:

- https://api.kamino.finance/referral-codes/my-referral-code

#### Example response:

```json
{
    "code": "my-referral-code9",
    "referrer": "24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p",
    "pda": "BrzGZgme5PyW8PPp7VKSDcs8jKFWL76ik3Wo9Zfq4iyq"
}
```

#### Possible Errors:

- `404 Not Found`: No referral code found for the given wallet.

---

### Create a Referral Code

Allows a user to register a new referral code - please see [How to authenticate correctly](#how-to-authenticate-correctly) section to be able to use this endpoint.

```http
POST https://api.kamino.finance/users/:wallet/referral-code
```

#### Request Body:

```json
{
    "wallet": "24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p",
    "code": "my-referral-code9",
    "signature": "4f7W3rm3FLAYdHUHiCYkAorRcL5EuXJd6DrhYDmPDtpqptMRzC6KRJxZdjzZMbhDX2XNVeBm9Tyvb3DYiSji8kwL",
    "session": "77869934-2e4a-4553-ba4c-fde396857fe0"
}
```

#### Example request:

- `POST https://api.kamino.finance/users/24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p/referral-code`

#### Example response:

```json
{
    "code": "my-referral-code9",
    "referrer": "24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p",
    "pda": "BrzGZgme5PyW8PPp7VKSDcs8jKFWL76ik3Wo9Zfq4iyq"
}
```

#### Possible Errors:

- `400 Bad Request`: Invalid request format or missing parameters.
- `403 Not Authorized`: Wallet is not authorized to create this referral code due to invalid signature.
- `409 Conflict`: Referral code already exists.

---

### Update Referral Code

Update an existing referral code. Please see [How to authenticate correctly](#how-to-authenticate-correctly) section to be able to use this endpoint.

```http
PUT https://api.kamino.finance/users/:wallet/referral-code
```

#### Request Body:

```json
{
    "wallet": "24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p",
    "code": "my-new-referral-code",
    "signature": "4f7W3rm3FLAYdHUHiCYkAorRcL5EuXJd6DrhYDmPDtpqptMRzC6KRJxZdjzZMbhDX2XNVeBm9Tyvb3DYiSji8kwL",
    "session": "77869934-2e4a-4553-ba4c-fde396857fe0"
}
```

#### Example request:

- `PUT https://api.kamino.finance/users/24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p/referral-code`

#### Example response:

```json
{
    "code": "my-new-referral-code",
    "referrer": "24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p",
    "pda": "BrzGZgme5PyW8PPp7VKSDcs8jKFWL76ik3Wo9Zfq4iyq"
}
```

#### Possible Errors:

- `400 Bad Request`: Invalid request format or missing parameters.
- `403 Not Authorized`: Wallet is not authorized to create this referral code due to invalid signature. 
- `404 Not Found`: Referral code does not exist.
- `409 Conflict`: New referral code is already in use.

---

### Get Referral Code for a Referred User

Retrieve the referral code used by a specific referred user (who they were referred by).

```http
GET https://api.kamino.finance/users/:wallet/referred-by
```

#### Parameters:

- `wallet` (path parameter, string, required): The wallet public key of the referred user.

#### Example request:

- https://api.kamino.finance/users/24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p/referred-by

#### Example response:

```json
{
    "code": "my-referral-code9",
    "referrer": "24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p",
    "pda": "BrzGZgme5PyW8PPp7VKSDcs8jKFWL76ik3Wo9Zfq4iyq"
}
```

#### Possible Errors:

- `400 Bad Request`: Invalid wallet address format.
- `404 Not Found`: No referral code found for the given referred user.

---

### Get User's Referred Users

Retrieve all the referred users that the user has referred.

```http
GET https://api.kamino.finance/users/:wallet/referred-users
```

#### Parameters:

- `wallet` (path parameter, string, required): The wallet public key of the referred user.

#### Example request:

- https://api.kamino.finance/users/24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p/referred-users

#### Example response:

```json
[
  {
    "user": "Hy4czLTfeqW5ZTW4EocZh488JmJJEJ5hAAKUnPFubLCh",
    "volumeUsd": "123.456",
    "referredOn": "2024-02-05T12:56:23.000Z"
  },
  {
    "user": "Hy4czLTfeqW5ZTW4EocZh488JmJJEJ5hAAKUnPFubLCc",
    "volumeUsd": "0",
    "referredOn": "2024-01-05T12:56:23.000Z"
  },
]
```

#### Possible Errors:

- `400 Bad Request`: Invalid wallet address format.

---

### Get Referral Leaderboard

```http request
GET https://api.kamino.finance/referral/leaderboard?offset={offset}&limit={limit}
```

#### Example request:

- Get top 10 users on referral leaderboard: https://api.kamino.finance/referral/leaderboard?offset=0&limit=10
- Get next 10 users on referral leaderboard: https://api.kamino.finance/referral/leaderboard?offset=10&limit=10

Example response:

#### Example response:

```json
{
    "totalLeaderboardCount": 2,
    "leaderboard": [
        {
            "wallet": "24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p",
            "leaderboardRank": 1,
            "volumeUsd": "600.1",
            "weeklyLeaderboardRank": 1,
            "weeklyVolumeUsd": "200.3"
        },
        {
            "wallet": "GNt8N6ccG9RGgPxAEzEEb8HpdncR2KroS1LxM8Vyge75",
            "leaderboardRank": 2,
            "volumeUsd": "200",
            "weeklyLeaderboardRank": null,
            "weeklyVolumeUsd": null
        }
    ]
}
```

---

### Get Referral metrics

```http request
GET https://api.kamino.finance/referral/metrics
```

#### Example request:

- https://api.kamino.finance/referral/metrics

Example response:

#### Example response:

```json
{
  "allTimeVolumeUsd": "800.2349",
  "weeklyVolumeUsd": "10.484590",
  "userCount": 2
}
```

---

### Get Referral user metrics

```http request
GET https://api.kamino.finance/referral/leaderboard/users/:wallet/metrics
```

#### Example request:

- https://api.kamino.finance/referral/leaderboard/users/24PNhTaNtomHhoy3fTRaMhAFCRj4uHqhZEEoWrKDbR5p/metrics

Example response:

#### Example response:

If weekly leaderboard rank/volume are null, it means the user's referrals have not made any swaps. 

```json
{
  "leaderboardRank": 1,
  "volumeUsd": "600",
  "weeklyLeaderboardRank": null,
  "weeklyVolumeUsd": null,
  "referredUsers": 2
}
```

---
