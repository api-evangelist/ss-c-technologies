---
name: ss-c-technologies-reconcile-positions-and-activity
description: >-
  Pull an end-of-session view out of SS&C Eze EMS xAPI — today's balances, net and broken-down
  positions, and the day's order and execution activity — and reconcile it against the orders you
  submitted, using OrderTag as the join key.
api: SS&C Eze EMS xAPI
generated: '2026-09-13'
method: generated
source: >-
  openapi/ss-c-technologies-eze-ems-xapi-openapi.json,
  grpc/ss-c-technologies-xapi-utilities.proto,
  https://github.com/ezesoft/xapi/blob/master/readme.md
operations:
  - GET /api/v1/authentication/connect
  - GET /api/v1/order/user-accounts
  - GET /api/v1/utility/todays-balances
  - GET /api/v1/utility/todays-net-positions
  - GET /api/v1/utility/todays-breakdown-positions
  - GET /api/v1/utility/todays-activity
  - GET /api/v1/utility/todays-net-positions/subscribe
  - DELETE /api/v1/utility/todays-net-positions/subscribe
  - GET /api/v1/order/get-order-detail-by-date-range
  - GET /api/v1/utility/strategy-list
  - GET /api/v1/utility/strategy-data
  - GET /api/v1/authentication/disconnect
grpc_equivalents:
  - UtilityServices.GetTodaysBalances
  - UtilityServices.GetTodaysNetPositions
  - UtilityServices.GetTodaysBrokenDownPositions
  - UtilityServices.GetTodaysActivity
  - UtilityServices.GetTodaysActivityJson
  - UtilityServices.SubscribeTodaysNetPositions
  - UtilityServices.UnSubscribeTodaysNetPositions
  - SubmitOrderService.GetOrderDetailByDateRange
  - UtilityServices.GetStrategyList
  - UtilityServices.GetStrategyData
---

# Reconcile positions and activity on SS&C Eze EMS xAPI

Everything in this skill is **read-only**. It is the safe half of the API and the right place to
start when evaluating an Eze integration.

One thing to understand before you call anything: the word in every one of these operation names is
**today's**. The API exposes an intraday view — `GetTodaysBalances`, `GetTodaysNetPositions`,
`GetTodaysActivity`. There is no historical positions or balances operation in the published
contract. The only date-ranged operation anywhere is
`GET /api/v1/order/get-order-detail-by-date-range`, and on gRPC it is server-streaming. If you need
history, you snapshot daily and keep it yourself.

Open a session first (see `ss-c-technologies-place-and-track-an-order`, step 1).

## 1. Scope

`GET /api/v1/order/user-accounts` (`SubmitOrderService.GetUserAccounts`) — the accounts the session
can see. Everything below is scoped by them.

## 2. Balances and positions

- `GET /api/v1/utility/todays-balances` (`UtilityServices.GetTodaysBalances`).
- `GET /api/v1/utility/todays-net-positions` (`GetTodaysNetPositions`) — net position per account
  and symbol.
- `GET /api/v1/utility/todays-breakdown-positions` (`GetTodaysBrokenDownPositions`) — the same
  positions decomposed. Use this when a net number needs explaining.

For a live view, `GET /api/v1/utility/todays-net-positions/subscribe`
(`UtilityServices.SubscribeTodaysNetPositions`) opens a server-streaming subscription; `DELETE` on
the same path closes it. As everywhere in this API, REST manages the subscription and gRPC delivers
the stream. Always close it.

## 3. Activity

`GET /api/v1/utility/todays-activity` (`UtilityServices.GetTodaysActivity`, or
`GetTodaysActivityJson` on gRPC for a pandas-friendly JSON body).

**The include flags all default to false.** An unfiltered call gives you nothing. Set what you need:

- `IncludeUserSubmitOrder` — orders your session submitted.
- `IncludeExchangeTradeOrder` — fills from the exchange.

Other include filters exist on `TodaysActivityJsonRequest`; read the message definition in
`grpc/ss-c-technologies-xapi-utilities.proto` rather than guessing at names.

SS&C's own tutorial reads the JSON form straight into a pandas DataFrame and locates an order with
`df[df['OrderTag']=='...']['OrderId']`, which is exactly the reconciliation join described next.

## 4. Reconcile

`OrderTag` is the join key. For each order your system submitted:

1. Look it up in the activity rows by `OrderTag`.
2. Read its `OrderId` from the matched row.
3. Sum the `IncludeExchangeTradeOrder` rows carrying that `OrderId` to get the filled quantity.
4. Compare against your intended quantity.

Rows you cannot match are the finding, and there are two distinct kinds:

- **An order in your records with no activity row** — it never reached the system, or it was
  rejected. Confirm with `GET /api/v1/order/order-detail-by-order-tag` before concluding anything.
- **An activity row with an OrderTag you do not recognise, or no OrderTag at all** — it came from
  another session, a human trader in the Eze EMS front end, or a retry that double-fired. Because
  this API has no idempotency mechanism, a duplicate created by a retried submission looks exactly
  like a second legitimate order. Escalate these to a human; do not net them out automatically.

## 5. Strategies

`GET /api/v1/utility/strategy-list` and `GET /api/v1/utility/strategy-data`
(`UtilityServices.GetStrategyList` / `GetStrategyData`) return the strategies configured for the
session, as `StrategyDataRow`. `POST /api/v1/utility/create-strategy` and
`PUT /api/v1/utility/update-strategy` write; note there is **no delete operation** for a strategy in
the published contract, so a strategy you create cannot be removed through this API.

## 6. Close the session

`GET /api/v1/authentication/disconnect`. Unsubscribe from the positions stream first.

## Reading responses correctly

A transport 200 is not success. Check `ServerAcknowledgement` (and `ServerResponse` where present)
on every body. Errors arrive as `Google.Rpc.Status`; see
`errors/ss-c-technologies-problem-types.yml`.
