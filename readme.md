# cached-hafas-client

**Pass in a [HAFAS client](https://github.com/public-transport/hafas-client/tree/5), cache data from it.**

[![npm version](https://img.shields.io/npm/v/cached-hafas-client.svg)](https://www.npmjs.com/package/cached-hafas-client)
[![build status](https://api.travis-ci.org/public-transport/cached-hafas-client.svg?branch=master)](https://travis-ci.org/public-transport/cached-hafas-client)
![ISC-licensed](https://img.shields.io/github/license/public-transport/cached-hafas-client.svg)
[![chat on gitter](https://badges.gitter.im/public-transport/Lobby.svg)](https://gitter.im/public-transport/Lobby)
[![support Jannis via GitHub Sponsors](https://img.shields.io/badge/support%20Jannis-donate-fa7664.svg)](https://github.com/sponsors/derhuerst)

*Note:* This package is mainly **intended to prevent expensive and/or frequent API calls to HAFAS**. As a side effect, it *may* reduce local CPU load & latency, but that depends on the specific use case.

`cached-hafas-client`'s core logic is separated from data storage code; You can pick the store implementation that fits your use case best. Right now the following stores are implemented:

store name | built on top of | notes
-----------|-----------------|------
[`cached-hafas-client/stores/redis`](stores/redis.js) | [Redis](https://redis.io/) |
[`cached-hafas-client/stores/sqlite`](stores/sqlite.js) | [SQLite](https://www.sqlite.org/) | TTL not implemented yet


## Installation

```shell
npm install cached-hafas-client
```


## Usage

Because `cached-hafas-client` caches HAFAS responses by "request signature", it is build on the assumption that, aside from the ever-changing transit data underneath, HAFAS works deterministically.

This is why **you must send deterministic queries**; for example, you *must* pass `opt.duration` to [`departures()`](https://github.com/public-transport/hafas-client/blob/5/docs/departures.md)/[`arrivals()`](https://github.com/public-transport/hafas-client/blob/5/docs/arrivals.md) for it to know which "time range" the data returned by HAFAS is for.

```js
// create HAFAS client
const createHafas = require('vbb-hafas')
const hafas = createHafas('my-awesome-program')

// create a store backed by Redis
const {createClient: createRedis} = require('redis')
const createRedisStore = require('cached-hafas-client/stores/redis')
const redis = createRedis()
const store = createRedisStore(redis)

// wrap HAFAS client with cache
const withCache = require('cached-hafas-client')
const cachedHafas = withCache(hafas, store)

const wollinerStr = '900000007105'
const husemannstr = '900000110511'
const when = new Date(Date.now() + 60 * 60 * 1000)

// will fetch fresh data from HAFAS
await cachedHafas.departures(wollinerStr, {duration: 10, when})

// within the "time range" of a recent departures() call,
// so it will use the cached data
await cachedHafas.departures(wollinerStr, {
	duration: 3, when: new Date(+when + 3 * 60 * 1000)
})
```

*Note:* `cached-hafas-client` is only compatible with [`hafas-client@5`](https://github.com/public-transport/hafas-client/tree/5).

## Using a custom TTL

```js
const cachePeriod = 5 * 60 * 1000 // 5 minutes
const cachedHafas = withCache(hafas, store, cachePeriod)
```

## Counting cache hits & misses

```js
cachedHafas.on('hit', (hafasClientMethod, ...args) => {
	console.info('cache hit!', hafasClientMethod, ...args)
})
cachedHafas.on('miss', (hafasClientMethod, ...args) => {
	console.info('cache miss!', hafasClientMethod, ...args)
})
```

## Bypassing the cache

```js
const {CACHED} = require('cached-hafas-client')

// will always fresh data
await cachedHafas.departures(wollinerStr, {[CACHED]: false})
```


## API

```js
createCachedHafas(hafas, storage, opt = {})
```

`hafas` must be a [`hafas-client@5`](https://github.com/public-transport/hafas-client/tree/5)-compatible API client.

`opt` overrides this default configuration:

```js
{
	cachePeriods: {
		departures: 30_1000, arrivals: 30_1000, // 30s
		journeys: 30_1000, // 30s
		refreshJourney: 60_1000, // 1m
		trip: 30_1000, // 30s
		radar: 10_1000, // 10s
		locations: 3_600_1000, // 1h
		stop: 3_600_1000, // 1h
		nearby: 3_600_1000, // 1h
		reachableFrom: 30_1000,
	},
}
```


## Contributing

If you have a question or need support using `cached-hafas-client`, please double-check your code and setup first. If you think you have found a bug or want to propose a feature, use [the issues page](https://github.com/public-transport/cached-hafas-client/issues).
