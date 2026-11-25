# Walmart API: Supplier or Marketplace, Why Your Token Expires in 15 Minutes, and What the 180-Day Order Cap Does to Your Data (Plans Compared)

Walmart runs two separate developer APIs, and they don't share a domain, a credential set, or an order schema. The supplier Order Management API lives on `api-gateway.walmart.com`; the marketplace Orders API lives on `marketplace.walmartapis.com`. Neither will return a single record unless you're an approved Walmart or Sam's Club supplier, or an active marketplace seller. And even then, order history stops at 180 days.

What follows is the part that actually decides whether your integration works: how the two systems differ, how authentication and tokens behave, the rate limit that trips most first builds, the exact ceiling on order history, and what to do when the official surface doesn't reach the data you need.

> **Quick answer:** If you need your own Walmart purchase orders or seller orders, apply for supplier or seller access. No scraping tool replaces that. If you need public Walmart product pages (title, price, description, reviews, images), ScraperAPI's Walmart product endpoint handles it in one call per product at 5 credits per request.
> 👉 [Compare ScraperAPI's plans and credit allowances](https://www.scraperapi.com/pricing?fp_ref=coupons)

## ScraperAPI plans, if the scraping route is the one you need

Plans differ mainly by credit allowance and concurrency. Prices scale with the credit allowance, and annual billing takes 10% off every self-serve tier; Enterprise is quoted custom. Walmart pages sit in the e-commerce category, so each request costs 5 credits (more on the add-ons further down), which makes the "Walmart requests" column the number worth checking against your crawl size.

| Plan | API credits | Walmart requests at 5 credits each | Concurrent threads | Pay as you go | Start |
| --- | --- | --- | --- | --- | --- |
| Hobby | 100,000 | 20,000 | 20 | Not available | [Get Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 200,000 | 50 | Not available | [Get Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 600,000 | 100 | Not available | [Get Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 1,000,000 | 200 | Available | [Get Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 2,100,000 | 300 | Available | [Get Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 4,300,000 | 500 | Available | [Get Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 4,400,000+ | 500+ | Available | [Get Enterprise](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Three things the table doesn't show. Auto-upgrade (moves you to the next tier when credits run out) is opt-in on Hobby through Professional and unavailable on Advanced and Enterprise. Credits don't roll over on any plan. Geotargeting is US and EU only on Hobby and Startup, country-level from Business up, and analytics history is 30 days on the two cheapest tiers versus unlimited above them.

## The two Walmart APIs, and why mixing them up wastes a week

The supplier Order Management API is for companies that ship to Walmart. Its production base URL is `https://api-gateway.walmart.com` and the sandbox is `https://sandbox.walmartapis.com`. A purchase order there can be destined for a Walmart or Sam's Club facility, a distribution center, or a store or club directly, which is why the payload carries supply-chain status alongside the order status.

The marketplace Orders API is for third-party sellers and returns orders placed in your own seller account. One endpoint does most of the work: `GET https://marketplace.walmartapis.com/v3/orders`.

|  | Supplier (Order Management) | Marketplace (Orders) |
| --- | --- | --- |
| Base URL | `api-gateway.walmart.com` | `marketplace.walmartapis.com` |
| Who can call it | Suppliers to Walmart and Sam's Club | Marketplace sellers |
| Order identifiers | `hostPurchaseOrderId` | `purchaseOrderId`, `customerOrderId` |
| Line items | `lines[]` | `orderLines.orderLine[]` |
| History | Date filters via `createdAfter` / `createdBefore` | Last 180 days only |

The field names are the tell. If a tutorial shows `hostPurchaseOrderId` and `purchaseOrderSupplyChainStatus`, it's describing the supplier side. If it shows `customerOrderId` and `orderLineStatuses`, it's the marketplace side. Code written against one will not parse responses from the other.

## Authentication: 15-minute tokens and four mandatory headers

Both programs use OAuth 2.0. You exchange `clientId` and `clientSecret` at the Create access token endpoint for an access token, and that token is short-lived: marketplace tokens expire after 15 minutes, and the response's `expires_in` field is what you should schedule refreshes against. Cache the token, refresh shortly before it lapses, and never request a fresh one per call.

Every supplier request carries four headers:

- `wm_sec.access_token`, the token from the Token API
- `WM_QOS.CORRELATION_ID`, a unique UUID per request, used for tracing
- `WM_CONSUMER.CHANNEL.TYPE`, the channel identifier you received during onboarding
- `Accept: application/json`

Two details bite people. The correlation ID must be unique per call, not per session; reusing one makes tracing useless and the docs call it out explicitly. And `WM_CONSUMER.CHANNEL.TYPE` is mandatory on supplier order calls but listed as optional on the token request itself, so a token call that succeeds doesn't prove your headers are complete.

Older WFS inventory endpoints still sit on Basic auth (`Authorization: Basic <Base64(ConsumerKey:ConsumerSecret)>` plus `WM_SVC.NAME: Walmart Marketplace`). Those pages are marked legacy. Don't copy that pattern into a new build; the OAuth flow is the current one. You'll also see the token header written as `WM_SEC.ACCESS_TOKEN` on some fulfillment pages and `wm_sec.access_token` on others. HTTP headers are case-insensitive, so that's formatting, not a different mechanism.

## What you can actually pull out

**Supplier purchase orders.** `GET /v3/walmartOrders` returns POs for the authenticated supplier only, and results are sorted newest first by `createDate`. Filters include `supplierId`, `hostPurchaseOrderId`, `type` (the docs use `03` for buyer-generated domestic orders), `purchaseOrderStatus` (`cancelled`, for example), `purchaseOrderSupplyChainStatus` (`received`), `createdAfter` and `createdBefore` in ISO 8601, and the must-arrive-by date pair. Page size tops out at 100, and the response's `purchaseOrdersListMeta` object gives you `totalCount`, `limit`, and a `nextCursor` to pass back for the next page.

A single PO, `GET /v3/walmartOrders/{purchaseOrderId}`, returns the lines with ordered quantity, unit cost, and line status. The docs' sample line reads as 12 EA at USD 5.25, which is the shape to expect: quantity and cost are objects with a unit of measure and a currency, not bare numbers.

**Marketplace orders.** The orders endpoint takes `limit`, `productInfo`, `shipNodeType` (`SellerFulfilled` for seller-fulfilled, `WFSFulfilled` for Walmart Fulfillment Services), `replacementInfo`, and `incentiveInfo`. You get product name, SKU, quantity, line status (`Created`, `Cancelled`, `Shipped`, `Delivered`), estimated delivery and ship dates, carrier and tracking number, and charges with tax broken out as a separate object from the item price.

## Three things that break integrations

### 1. 5,000 requests an hour, then a 429

Supplier order calls are throttled at 5,000 requests per hour. When you exceed it you get `429 Too Many Requests`, and the documented remedy is exponential backoff and retry. The same applies to `500` and `503`. If you're polling for status changes across a large PO book, budget your calls against that hourly ceiling rather than discovering it in production. Marketplace errors arrive with business codes in the body (`RATE_LIMIT_EXCEEDED.GMP_ORDER_API`, `UNAUTHORIZED.GMP_ORDER_API`, and so on), so log the code, not just the HTTP status.

### 2. The 180-day window and the 10,000-order ceiling

This is the constraint most tutorials skip. Marketplace order data is available for the **last 180 days only**, and a single call can fetch at most **10,000 orders**; ask for more and the API returns an error. There is no parameter that reaches further back.

If you need multi-year order history for accounting, cohort analysis, or dispute records, the only reliable approach is your own store: sync incrementally on a schedule shorter than 180 days, persist everything locally, and treat Walmart as the source of truth for the recent window rather than the archive. The `nextCursor` value in the marketplace response even carries the date bounds and paging offsets, which makes resumable syncs straightforward if you store it.

Supplier data is different. That side exposes `createdAfter` and `createdBefore`, so the query window is yours to define, subject to the page size of 100 and the hourly cap.

### 3. Credentials and hosts don't cross over

A marketplace token sent to `api-gateway.walmart.com` fails. Supplier headers on a marketplace call fail. The two systems were built for different audiences and never promised interoperability. Keep separate credential stores, separate token caches, and separate base URLs in your config, and label them clearly, because the failure mode (401 or 403 with a vague body) looks identical to an expired token.

## What the official APIs don't cover

Read the scope carefully: the supplier docs describe purchase orders issued to you, the marketplace docs describe orders in your seller account, and the item and catalog endpoints operate on your own catalog. Catalog search lets you query your own listings by filters and sort attributes such as price, listing status, and rating, with wildcard support. None of that returns a public product page's current price, a competitor's listing, or customer reviews on items you don't sell.

So if your reason for searching "walmart api" was product research, price monitoring, or review collection rather than order management, the official route doesn't have an endpoint for you at all. That's the gap scraping fills.

👉 [See which ScraperAPI credit allowance covers your Walmart volume](https://www.scraperapi.com/pricing?fp_ref=coupons)

## When ScraperAPI is the right tool, and what it costs

ScraperAPI runs a dedicated Walmart product endpoint. The synchronous version takes a `product_id` and returns product details as JSON or CSV; the async version at `async.scraperapi.com/structured/walmart/product` accepts a `productIds` array for batches, plus `tld` (`com` for walmart.com, `ca` for walmart.ca), a `country_code` for geo targeting, and a webhook callback so you don't hold a connection open. Returned fields include product name, price, description, reviews, and images.

Cost is the part to model before you buy. Walmart requests are billed at **5 credits each**, the e-commerce rate that also covers Amazon and eBay. On Hobby's 100,000 credits that's 20,000 product requests. Add-ons stack on top: bot-protection bypass such as Cloudflare or DataDome adds 10 credits, JavaScript rendering adds 10, premium proxies add 10, and combining premium proxies with rendering costs 25. Geo targeting, sessions, device type, output format, and auto parsing are free. Every response returns its actual cost in the `sa-credit-cost` header, and you can price a URL with the Domain Cost Estimator before committing, or cap a single request with `max_cost`.

There's a free starting allowance of 1,000 credits at 5 concurrent connections, which is enough to run a few hundred Walmart product calls and confirm the response shape before paying for anything.

What ScraperAPI won't do: it can't read your seller account, and it doesn't return purchase orders, fulfillment status, or anything sitting behind Walmart's seller login. Teams that need both usually run the official API for their own orders and a scraping plan for public listing data, side by side.

👉 [Start with the free credits and size the plan from there](https://www.scraperapi.com/pricing?fp_ref=coupons)

## FAQ

**How long does a Walmart API access token last?**
Fifteen minutes on the marketplace side. Use the `expires_in` value from the token response to schedule refreshes rather than hardcoding a timer.

**Why am I getting 429 from the supplier order API?**
You've passed 5,000 requests per hour. Back off exponentially and retry; hammering the endpoint extends the throttle. If it happens daily at the same time, your polling interval is too aggressive for your PO volume.

**Can I pull orders older than 180 days?**
Not through the marketplace orders API. The window is fixed at 180 days and a single request caps at 10,000 orders. Persist your data locally on a rolling basis if you need longer history.

**Do I need a Walmart account to use these APIs?**
Yes. The supplier API is scoped to suppliers of Walmart and Sam's Club; the marketplace API is scoped to seller accounts. There is no public tier for general order data.

**How many pages will a large PO list take?**
At 100 per page with a 5,000-per-hour ceiling, you can pull up to 500,000 POs an hour in theory, but leave headroom for retries and other calls sharing the same credential. Follow `nextCursor` until it comes back empty.

## Which path to take

Need your own purchase orders or seller orders, with costs, line status, and tracking? Apply for supplier or marketplace access and build against the OAuth flow with a token cache, a per-request correlation ID, and backoff on 429. Start your local archive early, because the 180-day marketplace window doesn't wait for you to be ready.

Need public Walmart product pages at scale? That's the ScraperAPI route, and the only number that matters is your monthly product count times five credits, plus whatever rendering or proxy add-ons your targets require.

Before you check out, confirm two things: whether your Walmart pages need JavaScript rendering (it doubles the per-request credit cost), and whether your volume fits inside the plan's credit allowance without pay-as-you-go, which only starts at Scaling.

👉 [Compare ScraperAPI plans and pick the credit tier that matches your crawl](https://www.scraperapi.com/pricing?fp_ref=coupons)
