---
name: ss-c-technologies-subscribe-to-market-data
description: >-
  Retrieve and stream market data from SS&C Eze EMS xAPI — resolve a security from an ISIN, SEDOL,
  RIC, CUSIP or Bloomberg identifier, pull Level 1 snapshots, bars, tick data, option chains and
  greeks, and manage streaming subscriptions with their matching unsubscribes.
api: SS&C Eze EMS xAPI
generated: '2026-09-13'
method: generated
source: >-
  openapi/ss-c-technologies-eze-ems-xapi-openapi.json,
  grpc/ss-c-technologies-xapi-market-data.proto,
  https://github.com/ezesoft/xapi/blob/master/faq.md
operations:
  - GET /api/v1/authentication/connect
  - GET /api/v1/security/symbol-from-alternate-symbology
  - GET /api/v1/security/symbols-from-company-name
  - GET /api/v1/security/security-data
  - GET /api/v1/security/symbol-reference-data
  - GET /api/v1/security/level1-market-data
  - GET /api/v1/security/intraday-bars
  - GET /api/v1/security/daily-weekly-monthly-bars
  - GET /api/v1/security/tick-data
  - GET /api/v1/security/option-chain-for-underlier
  - GET /api/v1/security/options-and-greek-data
  - POST /api/v1/security/symbols
  - DELETE /api/v1/security/symbols
  - GET /api/v1/security/subscribe-symbols
  - DELETE /api/v1/security/subscribe-symbols
  - GET /api/v1/security/subscribe-symbols-level2
  - DELETE /api/v1/security/subscribe-symbols-level2
  - GET /api/v1/security/subscribe-tick-data
  - DELETE /api/v1/security/subscribe-tick-data
  - GET /api/v1/security/subscribe-intraday-bars
  - DELETE /api/v1/security/subscribe-intraday-bars
grpc_equivalents:
  - MarketDataService.GetSymbolFromAlternateSymbology
  - MarketDataService.GetSymbolsFromCompanyName
  - MarketDataService.GetSecurityData
  - MarketDataService.GetSymbolReferenceData
  - MarketDataService.GetLevel1MarketData
  - MarketDataService.GetIntradayBars
  - MarketDataService.GetDailyWeeklyMonthlyBars
  - MarketDataService.GetTickData
  - MarketDataService.GetOptionChainForUnderlier
  - MarketDataService.GetOptionsAndGreekData
  - MarketDataService.AddSymbols
  - MarketDataService.RemoveSymbols
  - MarketDataService.SubscribeLevel1Ticks
  - MarketDataService.UnSubscribeLevel1Data
  - MarketDataService.SubscribeLevel2Ticks
  - MarketDataService.UnSubscribeLevel2Data
  - MarketDataService.SubscribeTickData
  - MarketDataService.UnSubscribeTickData
  - MarketDataService.SubscribeIntradayBars
  - MarketDataService.UnSubscribeIntradayBars
  - MarketDataService.StreamMarketData
---

# Subscribe to market data on SS&C Eze EMS xAPI

Market data here is read-only and safe to call, with one real cost: **every subscription you open
must be closed.** Subscriptions are session-scoped, entitlement-metered, and there is no published
cap telling you when you have opened too many.

Open a session first — see `ss-c-technologies-place-and-track-an-order`, step 1. Everything below
needs the `UserToken` it returns.

## Resolve the security before anything else

Never construct an Eze symbol from a ticker string. Resolve it:

- From a standard identifier: `GET /api/v1/security/symbol-from-alternate-symbology`
  (`MarketDataService.GetSymbolFromAlternateSymbology`). The `SymbolOption` enum accepts `ISIN`,
  `SEDOL`, `RIC`, `CUSIP` and `BBG`. The request takes either a single `Symbol` or a `Symbols` array
  for bulk; when both are given, the array wins. Bulk responses carry `SuccessCount`,
  `FailureCount`, `TotalCount` and a per-symbol `Results` array — check `FailureCount`, because a
  partial success returns 200.
- From a company name: `GET /api/v1/security/symbols-from-company-name`.
- Then confirm with `GET /api/v1/security/security-data` or
  `GET /api/v1/security/symbol-reference-data`.

For options, `GET /api/v1/security/option-symbol-from-description` and
`GET /api/v1/security/description-from-option-symbol` convert between the two forms.

## Snapshots and history (request/response, no cleanup needed)

- `GET /api/v1/security/level1-market-data` — a Level 1 record. It carries `MIC`, the ISO 10383
  market identifier of the reporting venue; use it rather than inferring the venue.
- `GET /api/v1/security/intraday-bars` — intraday OHLC.
- `GET /api/v1/security/daily-weekly-monthly-bars` — daily, weekly or monthly OHLC.
- `GET /api/v1/security/tick-data` — historical ticks.
- `GET /api/v1/security/option-chain-for-underlier` — the chain for an underlier.
- `GET /api/v1/security/options-and-greek-data` — greeks.

None of these paginate. There is no cursor, page, offset or limit parameter anywhere in this API, so
bound the request with its own domain filters (date range, symbol list, interval) rather than
expecting to page through a large result.

## Watchlists

`POST /api/v1/security/symbols` (`MarketDataService.AddSymbols`) adds symbols to the session's
watchlist; `DELETE /api/v1/security/symbols` (`RemoveSymbols`) removes them. `POST /api/v2/security/symbols`
is the v2 add. This is fully reversible and cheap.

## Streaming subscriptions — open and close in pairs

Every subscribe has exactly one unsubscribe, and the REST projection maps them to `GET` and `DELETE`
on the same path:

| Purpose | Subscribe | Unsubscribe | gRPC |
|---|---|---|---|
| Level 1 quotes | `GET /api/v1/security/subscribe-symbols` | `DELETE` same path | `SubscribeLevel1Ticks` / `UnSubscribeLevel1Data` |
| Level 2 book | `GET /api/v1/security/subscribe-symbols-level2` | `DELETE` same path | `SubscribeLevel2Ticks` / `UnSubscribeLevel2Data` |
| Tick data | `GET /api/v1/security/subscribe-tick-data` | `DELETE` same path | `SubscribeTickData` / `UnSubscribeTickData` |
| Intraday bars | `GET /api/v1/security/subscribe-intraday-bars` | `DELETE` same path | `SubscribeIntradayBars` / `UnSubscribeIntradayBars` |

**The REST surface cannot deliver the stream.** It manages subscription lifecycle only. To receive
the data you must call the gRPC RPC, which is server-streaming (and, for
`MarketDataService.StreamMarketData`, bidirectional). This is the single largest capability gap
between the two transports and it is not stated in the OpenAPI.

On gRPC, SS&C's FAQ is explicit that a streaming call returns a blocking iterator: **run each stream
on its own thread**, or the first subscription will freeze the application.

Always unsubscribe in a `finally` block, and unsubscribe before `Disconnect`, so an abandoned
subscription does not consume an entitlement slot you cannot see.

## Liveness

`UtilityServices.SubscribeHeartBeat` (REST `GET /api/v1/utility/heartbeat-subscribe`) is the
documented way to know your session is alive. Per SS&C's FAQ, a subscriber is notified and asked to
reconnect when the server connection terminates. When that happens, **back off before reconnecting**:
three `connect` attempts inside 60 seconds locks the account for three minutes even when the
credentials are correct, so a naive reconnect loop takes you offline for longer than the outage did.

`GET /health` and `GET /ready` are the only operations in the API that need no session.

## Errors

One shape, `Google.Rpc.Status` — see `errors/ss-c-technologies-problem-types.yml`. A 200 does not
mean success; read `ServerAcknowledgement` on the response body.
