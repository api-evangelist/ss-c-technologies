---
name: ss-c-technologies-place-and-track-an-order
description: >-
  Place a single order through SS&C Eze EMS xAPI and follow it to a terminal state, using the
  client-supplied OrderTag as the correlation key. Covers session setup, entitlement discovery,
  submission, status retrieval, amendment and cancellation, and the specific hazards of an API with
  no idempotency mechanism.
api: SS&C Eze EMS xAPI
generated: '2026-09-13'
method: generated
source: >-
  openapi/ss-c-technologies-eze-ems-xapi-openapi.json,
  grpc/ss-c-technologies-xapi-order.proto,
  grpc/ss-c-technologies-xapi-utilities.proto,
  https://github.com/ezesoft/xapi/blob/master/readme.md,
  https://github.com/ezesoft/xapi/blob/master/faq.md
operations:
  - GET /api/v1/authentication/connect
  - GET /api/v1/order/user-accounts
  - GET /api/v1/utility/user-routes
  - POST /api/v1/order
  - GET /api/v1/order/order-detail-by-order-tag
  - GET /api/v1/order/order-detail-by-order-id
  - PUT /api/v1/order
  - DELETE /api/v1/order
  - GET /api/v1/utility/todays-activity
  - GET /api/v1/authentication/disconnect
grpc_equivalents:
  - UtilityServices.Connect
  - SubmitOrderService.GetUserAccounts
  - UtilityServices.GetUserRoutes
  - SubmitOrderService.SubmitSingleOrder
  - SubmitOrderService.GetOrderDetailByOrderTag
  - SubmitOrderService.GetOrderDetailByOrderId
  - SubmitOrderService.ChangeSingleOrder
  - SubmitOrderService.CancelSingleOrder
  - UtilityServices.GetTodaysActivityJson
  - UtilityServices.Disconnect
---

# Place and track an order on SS&C Eze EMS xAPI

**This skill sends real orders to a real execution management system.** Nothing in this API is a
simulation. There is no dry-run mode and no idempotency key. Do not run any step below against a
production session without an explicit human instruction naming the symbol, side, quantity, account
and route.

The spec declares **no `operationId` on any of its 73 operations**, so every step below is named by
method and path, and by the gRPC RPC it projects. Those are the real handles.

## Before you start

- You need a server, port, user, domain, locale, and at least one route and account. SS&C provisions
  all of them; none is discoverable without a session.
- The REST surface carries every parameter, including `Password` and `UserToken`, in the **query
  string**. Treat request URLs as secrets: do not log them, do not put them in an error message.
- Three login attempts inside 60 seconds lock the account for three minutes, **whether or not they
  succeeded**. Never retry `connect` in a tight loop; back off before the third attempt.

## 1. Open a session

`GET /api/v1/authentication/connect` (`UtilityServices.Connect`) with `UserName`, `Domain`,
`Password`, `Locale`.

Read `UserToken` from `ConnectResponse` and carry it on every subsequent call. Also read:

- `Response` — SS&C's own tutorial checks `Response == 'success'`. A transport 200 is not success.
- `DPermStatusCode` — entitlement status. If the user is not entitled, orders will fail later, not here.
- `MFASecurityCode` / `MFAtotpPreference` / `MFAtotpValue` — if MFA is required, complete it at
  `GET /api/v1/authentication/multi-factor-authentication` before doing anything else.

If the domain is SRP-enabled and policy requires it, use the SRP pair instead:
`GET /api/v1/authentication/start-login-srp` then `GET /api/v1/authentication/complete-login-srp`.

## 2. Discover what you are allowed to trade

Never guess an account or a route. Both are provisioned values and a wrong one is a rejected or,
worse, a misrouted order.

- `GET /api/v1/order/user-accounts` (`SubmitOrderService.GetUserAccounts`) — the tradeable accounts.
- `GET /api/v1/utility/user-routes` (`UtilityServices.GetUserRoutes`) — the destinations.
- `GET /api/v1/utility/user-route-props` — per-route properties, if the route needs extra fields.
- `GET /api/v1/user/user-permissions` (`UserServices.GetUserPermissions`) — entitlements.

If the instruction gives an ISIN, SEDOL, RIC, CUSIP or Bloomberg identifier rather than an Eze
symbol, resolve it first with `GET /api/v1/security/symbol-from-alternate-symbology`
(`MarketDataService.GetSymbolFromAlternateSymbology`). Do not construct an Eze symbol yourself.

## 3. Generate an OrderTag before you submit

`OrderTag` is the only correlation key you control. SS&C's FAQ is explicit: if you do not set one,
you have to match orders back to your system by symbol and quantity, which is ambiguous the moment
two similar orders exist.

Generate a unique, non-reused value (a UUID) and record it **before** the submit call, alongside the
intended symbol, side, quantity, account and route. This local record is the only thing that lets you
answer "did my order go through?" after a timeout.

## 4. Submit

`POST /api/v1/order` (`SubmitOrderService.SubmitSingleOrder`). Parameters are query parameters:

`Symbol`, `Side`, `Quantity`, `Route`, `Account`, `OrderTag`, and optionally `TicketId`, `Staged`,
`ClaimRequire`, `GoodFrom`, `ExpirationDate`, `Price`, `StopPrice`, `UserMessage`, `ReturnResult`,
`ReturnResultTimeOutInMs`, plus `TimeInForce.Expiration` and `PriceType.PriceType`.

- `PriceType.PriceType` takes `Market`, `Limit`, `StopMarket`, `StopLimit` or `Other`. `Price` is
  required for `Limit` and `StopLimit`; `StopPrice` for the stop types.
- `TimeInForce.Expiration` takes `DAY`, `GTC`, `GTX`, `CLO`, `OPG`, `IOC`, `GTD` or `OTHER`. `GTD`
  needs `ExpirationDate`.
- `Staged: true` creates the order without releasing it. If the human wants a review step before the
  order goes live, this is the only mechanism the API offers.
- `ReturnResult: true` with `ReturnResultTimeOutInMs` makes the call block for an acknowledgement
  rather than returning immediately.

Read `SubmitSingleOrderResponse`: `ServerResponse`, `Acknowledgement` (`ServerAcknowledgement`) and
`OrderDetails`. Capture the `OrderId`.

### If the submit call times out or errors — STOP

There is no idempotency key on this operation. **Do not resubmit.** A retry can place a second real
order. Instead:

1. Wait, then call `GET /api/v1/order/order-detail-by-order-tag`
   (`SubmitOrderService.GetOrderDetailByOrderTag`) with the OrderTag you recorded in step 3.
2. If it returns an order, the first submission landed. Use its `OrderId`.
3. If it returns nothing, and only then, consider resubmitting with the **same** OrderTag so the next
   lookup stays unambiguous.
4. If you cannot determine which, escalate to a human. Do not guess.

## 5. Track to a terminal state

- `GET /api/v1/order/order-detail-by-order-id` — status of one order.
- `GET /api/v1/order/order-detail-by-order-tag` — the same, keyed on your tag.
- `GET /api/v1/utility/todays-activity` (`UtilityServices.GetTodaysActivity` /
  `GetTodaysActivityJson`) — the day's activity, narrowed by include flags. Set
  `IncludeUserSubmitOrder` for submissions and `IncludeExchangeTradeOrder` for fills; all include
  flags default to false, so an unfiltered call returns nothing useful.
- `GET /api/v1/order/get-order-detail-by-date-range` — a **server-streaming** RPC on gRPC. Over REST
  you get the request/response projection only.

For live updates prefer gRPC `SubmitOrderService.SubscribeOrderInfo` (or `SubscribeOrderInfoJson`).
The REST path `GET /api/v1/order/subscribe` manages the subscription but cannot deliver the stream.
Run streaming RPCs on a dedicated thread — SS&C's FAQ notes the iterator blocks.

## 6. Amend or cancel

- Amend: `PUT /api/v1/order` (`SubmitOrderService.ChangeSingleOrder`).
- Cancel: `DELETE /api/v1/order` (`SubmitOrderService.CancelSingleOrder`).

Both work **only while the order is still working**. SS&C publishes no window and no guarantee, so
check order status first and treat a cancel as a request, not a certainty. After an amendment,
`OriginalOrderId` on the resulting order points back at its predecessor.

Operations with **no reversal at all** in the published contract — never fire these speculatively:
`POST /api/v1/order/book-trade`, `POST /api/v1/order/trade-report`,
`POST /api/v1/order/allocation-order`.

## 7. Close the session

`GET /api/v1/authentication/disconnect` (`UtilityServices.Disconnect`). Subscriptions are
session-scoped and die with it.

## Errors

Every operation declares one error response: a `default` carrying `Google.Rpc.Status`
(`code`, `message`, `details[]`). `code` is a gRPC canonical status code (3 INVALID_ARGUMENT,
7 PERMISSION_DENIED, 9 FAILED_PRECONDITION, 16 UNAUTHENTICATED and so on). No 4xx or 5xx status is
declared anywhere, and `details[]` holds `google.protobuf.Any` payloads whose concrete types the spec
does not name. Log the whole envelope; you cannot parse it confidently in advance. See
`errors/ss-c-technologies-problem-types.yml`.
