# Shipment Protection — Analytics Event Contract

> **This is the current contract.** Both emitting surfaces now live in this repo, so the contract
> lives here alongside them. A copy of the pre-migration version remains at
> `stord-shopify/docs/SHIPMENT_PROTECTION_ANALYTICS_EVENTS.md`, describing the legacy app's surfaces;
> it is intentionally kept in place while a mid-cutover shop can still run them. Where the two
> disagree, this document is authoritative for the Commerce app. The legacy surfaces are also
> described below, so this file covers both sides of a cutover.

---

## 1. Purpose & Scope

This document defines the authoritative event contract for Shipment Protection (SP) analytics events emitted from the Shopify storefront. It is the single source-of-truth that ClickHouse MV implementers must reference to keep event emission and query logic aligned.

**This contract covers:**

- Event names for all SP user interactions
- The shape and type of each `event_data` payload, and which surface sends which fields
- The `context` key and its allowed values, distinguishing cart vs. checkout surface
- The two-pixel delivery model and its double-counting implications
- Full example payloads for each event type

---

## 2. Emitting Surfaces

Shipment Protection renders in two distinct surfaces. The `context` field in every event payload identifies which surface fired the event.

| Surface                 | `context`    | Current implementation                                                                                                                                                         | Legacy implementation (mid-cutover only)                                               |
| ----------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| Cart page / cart drawer | `"cart"`     | `stord-shopify-commerce` — `extensions/stord-commerce-storefront-widget/assets/stord-sp-cart.js`, loaded by the `cart-drawer-add-to-order.liquid` App Embed (`target: "body"`) | `stord-shopify` — `extensions/stord-theme/blocks/cart.liquid` (`target: "head"` embed) |
| Checkout                | `"checkout"` | `stord-shopify-commerce` — `extensions/stord-shipment-protection/` (Preact checkout UI extension, `purchase.checkout.actions.render-before`)                                   | `stord-shopify` — `extensions/stord-shipment-protection/`                              |

### Mutual exclusion between the two cart widgets

The cart surface is programmatically mutually exclusive; the checkout surface is not.

- **Cart — automatic.** The legacy `cart.liquid` is a `target: "head"` embed, so it runs during `<head>` parse and sets `window.__stordLegacySpActive = true`. The commerce widget's assets are `defer`-loaded, so they always observe the flag; `StordSpCartWidget.start()` sees it and transitions to the terminal `YIELDED` state before rendering or fetching anything. The App Embed's inline bootstrap also skips its checkout-button-hiding style tag. Net effect: with both App Embeds enabled the legacy widget owns the surface and the commerce widget emits nothing. Disabling the legacy App Embed leaves the flag unset and the commerce widget takes over.
- **Checkout — manual.** Checkout UI extensions are merchant-placed via the checkout editor, and a checkout extension is sandboxed from the storefront `window`, so no equivalent flag is possible. Whoever runs a cutover must disable the legacy checkout extension in the checkout editor.

**Neither mechanism stops double-_counting_** — see §7. Widget exclusion controls which surface _emits_; double-counting is a property of which _pixels forward_.

---

## 3. Event Catalogue

| Event Name                         | Trigger                   | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ---------------------------------- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `protection_impression`            | Component render          | Fired once when the SP widget first becomes visible.                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `protection_opt_in`                | User selects protection   | Fired when a user enables / checks shipment protection.                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `protection_opt_out`               | User deselects protection | Fired when a user disables / unchecks shipment protection.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `protection_block_removed`         | Block removal             | Fired once when the SP widget transitions from visible to hidden (e.g. cart exceeds max insurable amount). Includes `reason`; see §4.                                                                                                                                                                                                                                                                                                                                                                             |
| `protection_product_lookup_failed` | SP product lookup 404     | Fired when an SP-product lookup does not find the product/variant: an HTTP 404 (cart variant lookup, or the cart's product re-check after a failed add; checkout cost endpoint) or a null product for the fixed handle (checkout default lookup). Fires independently of widget visibility — including cold start — and carries its **own** payload. See §4. **This event pages production on-call** (Datadog `sp_product_404`), so it is deliberately 404-only: transient 5xx / network errors must not fire it. |

Five events, not four. `protection_product_lookup_failed` was added after the original four and is implemented on both surfaces and forwarded by both pixels.

---

## Data flow

```text
┌──────────────────────┐                          ┌────────────────────────────┐
│ Cart page / drawer   │  protection_impression   │  analytics pixel           │
│ stord-sp-cart.js     │  protection_opt_in       │                            │
│ (context: "cart")    │ ───────────────────────▶ │  • adds session_id,        │
│                      │  protection_opt_out      │    shop_domain, app_source │
│ Shopify.analytics    │  protection_block_removed│  • builds event envelope   │
│   .publish()         │  protection_product_     │  • POST to gateway         │
└──────────────────────┘    lookup_failed         │                            │
                                                  │  Two pixels may both be    │
┌──────────────────────┐                          │  subscribed — see §7       │
│ Checkout extension   │  (same five events)      │                            │
│ (context: "checkout")│ ───────────────────────▶ │                            │
│ useApi().analytics   │                          │                            │
│   .publish()         │                          │                            │
└──────────────────────┘                          └────────────────────────────┘
                                                                │
                                                                ▼
                                          POST {gateway}/v1/analytics/events
                                                                │
                                    ┌───────────────────────────┴────────────┐
                                    │ shopify-gateway  routes/analytics.ts   │
                                    │ resolves shop_redirects with           │
                                    │ application = "stord" (ALWAYS — there  │
                                    │ is no "stord-commerce" analytics row)  │
                                    └───────────────────────────┬────────────┘
                                                                ▼
                                    ┌────────────────────────────────────────┐
                                    │ shopify-analytics-api-worker           │
                                    │ • CX_CACHE lookup: shopify_store:{shop}│
                                    │ • hit  → enrich + main topic           │
                                    │ • miss → shopify-analytics-events-dlq  │
                                    └───────────────────────────┬────────────┘
                                                                ▼
                                          Kafka  shopify-analytics-events
                                                                │
                                    ┌───────────────────────────┴────────────┐
                                    ▼                                        ▼
                    ClickPipes (prod/staging)                  kafka-metrics-consumer
                    → commerce.shopify_events_raw              → Datadog metrics
                    → mv_shipment_protection_daily_dashboard      (incl. sp_product_404)
                    → shipment_protection_daily_dashboard_v1
```

**Flow notes:**

1. **Surfaces** pass only `event.data` — the payload in §4. Everything else is added downstream.
2. **Pixel** adds top-level `session_id` + `shop_domain`, injects `app_source` into `event.data`, and wraps the standard custom-event envelope (`id`, `type`, `timestamp`, `clientId`, `seq`, `context`).
3. **Gateway** always looks up `application = "stord"`, even for Commerce-app traffic. A per-shop `path_redirects` entry for `/v1/analytics/events` on that row is the only way to repoint analytics traffic (this is how local dev tunnels it).
4. **Worker** enriches from `CX_CACHE` (written by marketplace_service) and flattens the envelope into the exact column set of `shopify_events_raw`. **A cache miss sends the event to the DLQ, where no MV will ever see it** — check the DLQ first when events "vanish".
5. **ClickHouse** MVs read `event_data` with `JSONExtract*`. In prod/staging the Kafka→ClickHouse hop is **ClickPipes**; locally it is a Kafka engine table + bridge MV (`commerce-service/local-dev/clickhouse/001_analytics_connector.sql`, applied by `clickhouse:local-connector:init`, **not** by `clickhouse:migrate`).

---

## 4. `event_data` Schema

### Base fields — the four fee-bearing events

Sent by `protection_impression`, `protection_opt_in`, `protection_opt_out`, `protection_block_removed`.

| Field                              | Type                       | Allowed values           | Cart | Checkout | Description                                                                                                                                                                                                                      |
| ---------------------------------- | -------------------------- | ------------------------ | :--: | :------: | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `context`                          | `string`                   | `"cart"` \| `"checkout"` |  ✅  |    ✅    | Emitting surface. Read via `JSONExtractString(event_data, 'context')`.                                                                                                                                                           |
| `cart_value`                       | `string \| null`           | Decimal, e.g. `"49.99"`  |  ✅  |    ✅    | Cart subtotal **before** protection cost, in store currency. Cart sends `null` if unknown.                                                                                                                                       |
| `protection_fee`                   | `string`                   | Decimal, e.g. `"1.98"`   |  ✅  |    ✅    | Protection product fee at time of event.                                                                                                                                                                                         |
| `admin_fee`                        | `string`                   | Decimal, e.g. `"0.50"`   |  ✅  |    ✅    | Admin fee at time of event.                                                                                                                                                                                                      |
| `total_fee`                        | `string`                   | Decimal, e.g. `"3.98"`   |  ✅  |    ✅    | `protection_fee + admin_fee`.                                                                                                                                                                                                    |
| `country_code`                     | `string \| null`           | ISO 3166-1, e.g. `US`    |  ✅  |    ✅    | Destination country. **Nullable on both surfaces** — the cart widget deliberately refuses to guess `"US"` (an unknown destination would price the wrong SP tier), and checkout has no country until the buyer enters an address. |
| `pricing_package_id`               | `string \| number \| null` | package id, or `null`    |  ✅  |    ✅    | A/B pricing dimension. **`mv_shipment_protection_daily_dashboard` groups by this** — see §6. `null`/absent rolls up under the `''` sentinel.                                                                                     |
| `consumer_reference`               | `string \| null`           | UUID, or `null`          |  ✅  |    ✅    | Consumer reference for correlation.                                                                                                                                                                                              |
| `has_subscription`                 | `boolean`                  |                          |  ❌  |    ✅    | Whether the cart contains subscription items. **Checkout only.**                                                                                                                                                                 |
| `subscription_frequencies`         | `string[]`                 |                          |  ❌  |    ✅    | Selling-plan frequencies present in the cart. **Checkout only.**                                                                                                                                                                 |
| `protection_selling_plan_attached` | `boolean`                  |                          |  ❌  |    ✅    | Whether the SP line carries a selling plan. **Checkout only**, and only on events that pass it explicitly.                                                                                                                       |
| `app_source`                       | `string`                   | `"stord-commerce"`       |  ➕  |    ➕    | **Added by the pixel, not the surface.** Present on `stord-commerce-analytics-pixel` events; **absent** on legacy-pixel events (the legacy pixel sets no such field). The only discriminator between the two pixels — see §7.    |

The cart/checkout asymmetry above is intentional (the cart widget has no selling-plan context) but means an MV must not assume the subscription fields exist on cart rows: `JSONExtractBool` returns `false` and `JSONExtractString` returns `''` for missing keys.

### Additional field for `protection_block_removed`

| Field    | Type     | Allowed values                                                                           | Description                                                                                                                                                                                                                                                                                                                               |
| -------- | -------- | ---------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `reason` | `string` | `"sp_unavailable"` \| `"variant_not_found"` \| `"display_mode_hidden"` \| `"cost_error"` | Why the block was removed. `sp_unavailable` = cost API returned unavailable (e.g. cart over the max insurable). `variant_not_found` = no variant price matched the returned fee. `display_mode_hidden` = display mode hidden (e.g. `only_in_ship_options` with no matching delivery option). `cost_error` = cost or product query failed. |

Fee fields on this event come from the **last valid cost response** before removal.

### `protection_product_lookup_failed` — own payload

The one deliberate exception to the base schema. It fires independently of widget visibility (including cold start, when no valid cost response ever arrived), so it carries no fee fields.

**Where the cart raises it.** The widget validates the variant _before_ offering protection, on both paths — an unbuyable variant is never presented, because adding it bounces checkout to Shopify's stock-problems page with no way back:

1. **Product missing** — `products/{handle}.js` 404s. `status_code: 404`. The handle is the per-package one when the response carries `pricing_package_id`, else the store-wide fixed handle, and `lookup_type` follows that same choice.
2. **Variant not usable** — the product resolves but doesn't carry the variant the response named, or carries it as not for sale. `status_code: null`, since nothing 404'd.
3. **A failed add (fallback)** — on a definite add failure the widget re-checks the product and raises the event if it no longer resolves. Ahead-of-time validation makes this rare; it remains as a guard for a product removed between page load and click.

The add's own status can't stand in for the check: Shopify reports an unknown variant on `cart/add.js` as **422 "Cannot find variant"**, indistinguishable by status from an out-of-stock 422, and the description is localizable. Re-checking the product keeps `status_code` honest — the status reported is always the product lookup's, never the add's.

| Field          | Type             | Allowed values             | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| -------------- | ---------------- | -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `context`      | `string`         | `"cart"` \| `"checkout"`   | Which surface hit the failed lookup.                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `lookup_type`  | `string`         | `"default"` \| `"variant"` | `default` = the store-wide fixed-handle product (`stord-shipment-protection`); `variant` = a per-package variant lookup.                                                                                                                                                                                                                                                                                                                                    |
| `status_code`  | `number \| null` | `404` \| `null`            | HTTP status of the lookup that failed. **Not a property of `lookup_type`** — either type can report either value. `404` when an HTTP lookup actually 404'd: the cart's `products/{handle}.js` (per-package _or_ fixed handle), or the checkout cost endpoint. `null` when nothing 404'd: the cart's product resolved but can't supply the named variant, the checkout Storefront query returned `product: null`, or the checkout variant isn't purchasable. |
| `currency`     | `string`         | ISO 4217, e.g. `USD`       | Presentment currency at time of failure. The key triage field: a cart `variant` lookup only 404s when the cart's presentment currency has no matching variant, so this identifies which market broke. Falls back to `"USD"` on the cart when Shopify exposes no active currency (analytics-only, never pricing).                                                                                                                                            |
| `country_code` | `string \| null` | ISO 3166-1, e.g. `US`      | Destination/market. `null` on the checkout default lookup before the buyer enters an address, and `null` on the cart when the destination is unknown.                                                                                                                                                                                                                                                                                                       |

> **Alerting note.** Because `404` means "an HTTP lookup 404'd" rather than "a lookup failed", a monitor filtering
> on `status_code = 404` silently misses every `null` case — which is now the majority, covering both surfaces'
> unpurchasable-variant checks and checkout's null-product lookup. Alert on the **event**, optionally split by
> `lookup_type`, not on `status_code`.

Does **not** include `cart_value`, `protection_fee`, `admin_fee`, `total_fee`. Filter by `event_name` before reading those, or they extract to `''`/`0` for these rows.

### Envelope reference

Only `event.data` is supplied by the emitting surface. Everything below is added by the pixel or the envelope.

| Location  | Field         | Added by   | Description                                                                                                                 |
| --------- | ------------- | ---------- | --------------------------------------------------------------------------------------------------------------------------- |
| Top level | `session_id`  | pixel      | From `getOrCreateSessionId`. **Different cookie per pixel** — see §7.                                                       |
| Top level | `shop_domain` | pixel      | Shop permanent domain, e.g. `mystore.myshopify.com`. Use this for the shop; never read it from `event.data`.                |
| Top level | `event`       | pixel      | The event payload below.                                                                                                    |
| `event`   | `id`          | envelope   | Unique id for this event instance.                                                                                          |
| `event`   | `name`        | envelope   | One of the five event names.                                                                                                |
| `event`   | `type`        | envelope   | `"custom"` for all SP events.                                                                                               |
| `event`   | `timestamp`   | envelope   | ISO 8601 UTC.                                                                                                               |
| `event`   | `clientId`    | envelope   | Shopify analytics client id. **Empty (`''`) in practice for custom events published from checkout UI extensions** — see §7. |
| `event`   | `seq`         | envelope   | Sequence number for stream ordering.                                                                                        |
| `event`   | `context`     | envelope   | Shopify analytics context. Always `{}` for custom events. Not the same as `event.data.context`.                             |
| `event`   | `data`        | **caller** | The §4 payload.                                                                                                             |

For ClickHouse, **`event_data`** in this doc means **`event.data`** in the request body.

---

## 5. Example Payloads

Full request body as sent to the gateway. Surfaces supply only `event.data`.

### 5.1 `protection_impression` — checkout

```json
{
  "session_id": "abc123def456",
  "shop_domain": "mystore.myshopify.com",
  "event": {
    "id": "evt_protection_impression_1",
    "name": "protection_impression",
    "type": "custom",
    "timestamp": "2026-03-09T14:31:05.000Z",
    "clientId": "",
    "seq": 1,
    "context": {},
    "data": {
      "context": "checkout",
      "cart_value": "5000.00",
      "protection_fee": "27.50",
      "admin_fee": "2.00",
      "total_fee": "29.50",
      "country_code": "US",
      "has_subscription": false,
      "subscription_frequencies": [],
      "pricing_package_id": null,
      "consumer_reference": "99dbd8a9-2287-4bd9-babe-c73ec3339eac",
      "app_source": "stord-commerce"
    }
  }
}
```

### 5.2 `protection_impression` — cart

Same envelope. Note the absent subscription fields:

```json
{
  "context": "cart",
  "cart_value": "49.99",
  "protection_fee": "1.98",
  "admin_fee": "0.50",
  "total_fee": "2.48",
  "country_code": "US",
  "pricing_package_id": null,
  "consumer_reference": null,
  "app_source": "stord-commerce"
}
```

### 5.3 `protection_opt_in` / `protection_opt_out`

Identical payload to the impression for the same surface; only `event.name` and `event.timestamp` differ.

### 5.4 `protection_block_removed` — checkout

```json
{
  "context": "checkout",
  "cart_value": "2650.00",
  "protection_fee": "26.50",
  "admin_fee": "2.65",
  "total_fee": "29.15",
  "country_code": "US",
  "pricing_package_id": null,
  "consumer_reference": null,
  "protection_selling_plan_attached": false,
  "reason": "sp_unavailable",
  "app_source": "stord-commerce"
}
```

### 5.5 `protection_product_lookup_failed` — cart, variant lookup 404

```json
{
  "context": "cart",
  "lookup_type": "variant",
  "status_code": 404,
  "currency": "CAD",
  "country_code": "CA",
  "app_source": "stord-commerce"
}
```

Checkout default lookup: `{ "context": "checkout", "lookup_type": "default", "status_code": null, "currency": "USD", "country_code": null, "app_source": "stord-commerce" }`.

---

## 6. ClickHouse MV Alignment

Extract with exact key names — any deviation yields silent `null`/`''`/`0`.

```sql
SELECT
    JSONExtractString(event_data, 'context')                          AS context,
    JSONExtractString(event_data, 'cart_value')::Decimal(18, 2)       AS cart_value,
    JSONExtractString(event_data, 'protection_fee')::Decimal(18, 2)   AS protection_fee,
    JSONExtractString(event_data, 'admin_fee')::Decimal(18, 2)        AS admin_fee,
    JSONExtractString(event_data, 'total_fee')::Decimal(18, 2)        AS total_fee,
    JSONExtractString(event_data, 'country_code')                     AS country_code,
    -- A/B pricing dimension. The live MV GROUPs BY this; '' is the untagged sentinel.
    JSONExtractString(event_data, 'pricing_package_id')               AS pricing_package_id,
    -- Which pixel forwarded this row. '' means the legacy pixel (it sets no app_source).
    JSONExtractString(event_data, 'app_source')                       AS app_source,
    -- Only on protection_block_removed; NULL elsewhere.
    JSONExtract(event_data, 'reason', 'Nullable(String)')             AS removal_reason,
    -- Only on protection_product_lookup_failed; NULL elsewhere. Nullable(...) so a
    -- missing/null key yields NULL — plain JSONExtractInt returns 0 and
    -- JSONExtractString returns '' for missing keys, which would misrepresent this
    -- event (status_code can legitimately be null).
    JSONExtract(event_data, 'lookup_type',  'Nullable(String)')       AS lookup_type,
    JSONExtract(event_data, 'status_code',  'Nullable(Int64)')        AS status_code,
    JSONExtract(event_data, 'currency',     'Nullable(String)')       AS currency
FROM shopify_events_raw
WHERE event_name IN (
    'protection_impression',
    'protection_opt_in',
    'protection_opt_out',
    'protection_block_removed',
    'protection_product_lookup_failed'
)
```

**The live MV is `mv_shipment_protection_daily_dashboard`** (latest definition: migration `50_recreate_mv_shipment_protection_daily_dashboard.sql`) → `shipment_protection_daily_dashboard_v1` (`AggregatingMergeTree`). Two things to know before changing it:

- It selects only the three highest-volume events (`protection_impression`, `protection_opt_in`, `protection_opt_out`); `protection_block_removed` and `protection_product_lookup_failed` are **not** in the dashboard MV. The 404 event is consumed on the Datadog path instead (`kafka-metrics-consumer` → `sp_product_404`).
- Its `GROUP BY` tuple must match the target table's `ORDER BY` exactly, `pricing_package_id` included — required for `AggregatingMergeTree` correctness.

**Per-event columns.** `reason` exists only on `protection_block_removed`; `lookup_type`/`status_code`/`currency` only on `protection_product_lookup_failed`. That event does carry `context` and `country_code` but **not** the fee fields.

### Known aggregate caveat: `event_client_id` is empty

`event_client_id` arrives as `''` for these events, so any `uniq(event_client_id)` metric collapses to a single bucket instead of counting visitors. Observed in the live dashboard: a day with 4 impressions across 4 distinct sessions reports **1** visitor. Every `visitors_*` column in `shipment_protection_daily_dashboard_v1` is affected. Prefer `event_session_id` for distinct-count metrics until this is fixed upstream.

---

## 7. The two pixels, and double counting

|                     | legacy `stord-analytics-pixel` (`stord-shopify`)                                                                                                                    | `stord-commerce-analytics-pixel` (`stord-shopify-commerce`)                      |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| session cookie      | `_stord_session_id`                                                                                                                                                 | `_stord_commerce_session_id`                                                     |
| `app_source`        | not set                                                                                                                                                             | `"stord-commerce"`                                                               |
| event-name registry | `CUSTOM_EVENTS` in `src/sendAnalyticsEvent.ts`                                                                                                                      | `extensions/shared/constants/analyticsEvents.ts`                                 |
| lifecycle           | created/deleted per connection by marketplace_service `manage_web_pixel` (`webPixelCreate`/`webPixelDelete`), via `PUT /connections/shopify/:id/web_pixel {active}` | registered by the Commerce app install; `web_pixel_id` stored on `shopify_shops` |

**Both pixels subscribe to all five SP event names.** Shopify delivers a published custom event to every web pixel subscribed to that name, independent of which app published it. So on a shop where both pixels are active, one logical SP interaction produces **two** rows in `shopify_events_raw`.

This is _not_ prevented by the widget mutual exclusion in §2. That exclusion controls which surface **emits**; this is about which pixels **forward**. Nor is it prevented by the commerce pixel's `isStoreAppActive` guard — that cookie-sniff suppresses duplicate **standard Shopify events** only, and is applied _after_ the custom-event subscribe loop.

Per-metric impact on `mv_shipment_protection_daily_dashboard`:

| Metric family                                               | Aggregate                       | Behaviour with both pixels active                                                |
| ----------------------------------------------------------- | ------------------------------- | -------------------------------------------------------------------------------- |
| `impression_count_*`, `opt_in_count_*`, `opt_out_count_*`   | `countIfState`                  | **doubles** — every row counted                                                  |
| `sessions_*`, `sessions_opted_in_*`, `sessions_opted_out_*` | `uniqIfState(event_session_id)` | **doubles** — the two pixels use different session cookies, so ids never collide |
| `visitors_*`                                                | `uniqIfState(event_client_id)`  | already degenerate (empty client id, see §6)                                     |

**Mitigation.** The legacy pixel's lifecycle is independent of the legacy SP surfaces: disabling the legacy cart App Embed or removing the legacy checkout extension does **not** remove the legacy pixel. A shop that keeps the legacy app installed for its other features (EDD widget, portal tracking) keeps forwarding SP events. Any cutover runbook must therefore explicitly deactivate the legacy web pixel (`PUT /connections/shopify/:id/web_pixel` with `active: false`) as its own step. If instead the MV needs to tolerate both pixels, `app_source` is the only discriminator — and it is asymmetric: filter `app_source = 'stord-commerce'` for new-pixel rows and `app_source = ''` for legacy rows, never `app_source = 'stord'`.
