# Meridian Orders API: Analysis
*Madona A*

## Task 1: What doesn't match?

| Docs say | Data does | Hurts a builder? |
|---|---|---|
| Status is pending/shipped/delivered/cancelled | `orders_page1.json`, ord_1003: status is `"refunded"` | Yes — an undocumented status can slip through filters and get counted as a normal sale |
| Amounts are integers in the smallest unit (cents) | `orders_page2.json`, ord_1006: `44.0, 3.63, 5.99, 53.62` — decimals, in dollars | Yes — summed with cents, this order counts as ~54 cents instead of $53.62 |
| `total` always equals `subtotal + tax + shipping` | `orders_page1.json`, ord_1004: 6200+511+599=7310, but `total` is 6810 | Yes — $500 unexplained gap, no discount field to justify it |
| `customer.email` is always present | `orders_page2.json`, ord_1005: `email: null` | Moderate — breaks receipts or email-based joins |
| Unknown order ID returns 404 | `order_ord_9999.json`: HTTP 200, `{"order": null}` | Moderate — error handling built on status codes never fires |
| List is most recent first | Orders run oldest→newest across both pages | Low — "latest orders" logic is inverted |
| `has_more` reflects whether more data exists | `orders_page1.json`: `has_more: false` while `next_cursor: "cur_8f2a19bd"` — and that cursor returns 2 more real orders | **Severe** |

**Worst: the pagination mismatch.** Every other issue is visible in data you receive — a wrong status, a decimal, a null. This one hides data you never receive. A client following the documented rule ("check `has_more`, stop when false") never requests page 2 and gets no error, no odd value — just a smaller, normal-looking dataset. In this sample it hides 2 of 6 orders and $79.09 of $328.03 gross revenue, and because pagination underlies every list-based feature, the real impact could be larger at scale.

## Task 2: Total revenue: **$225.70**

Sum of `total` for the five orders that represent completed, unrefunded sales.

Judgment calls made:
- **Excluded ord_1003** (refunded) — the customer got their money back, so it isn't revenue.
- **Followed the pagination cursor into page 2** despite `has_more: false`. Stopping at page 1 alone gives $146.61; including both pages gives $225.70 — a $79.09 difference from this one decision.
- **Used $6810 for ord_1004** (the `total` field, described in the docs as "amount charged") rather than $7310 (sum of parts), since `total` is the field most likely to reflect the actual customer charge.
- **Converted ord_1006 from dollars to cents** ($53.62 → 5362) to match every other order's unit.

Can't tell from this data: whether $6810 or $7310 is the *true* amount for ord_1004 — would need order history to know if $500 was a discount or an error. Also unclear whether six orders represent Priya's full reporting period.

## Task 3a: Reply to Priya

**Subject:** Update on your revenue reconciliation issue (TICKET-4502)

Dear Priya,

Thanks for flagging this — I looked into the gap and found three causes:

1. **A refunded order is being counted as a sale.** It shows a status our documentation doesn't recognize, so it's easy to miscount.
2. **One order was recorded in dollars instead of our standard unit**, understating it by roughly 100x if not converted.
3. **Some orders were getting missed** — our system sometimes signals "no more data" when more exists, so anyone following the documented process misses real orders.

Correcting for these, I get **$225.70** on the data I reviewed. To confirm this closes your gap, could you share your total, the dashboard's figure, and the date range you're reporting on? I've flagged the missing-orders issue to engineering as the most likely source of ongoing silent gaps.

Best, Madona A

## Task 3b: Bug report

**Title:** `GET /v1/orders` returns `has_more: false` while `next_cursor` still points to more data
**Severity:** High. Silent failure — no error, no odd value — so any client following the documented rule loses data with no indication anything is wrong. Hides 2 of 6 orders (33%) and $79.09 of $328.03 gross revenue in this sample.

**Steps to reproduce:**
1. `GET /v1/orders` → 200. Returns 4 orders, `has_more: false`, `next_cursor: "cur_8f2a19bd"`.
2. `GET /v1/orders?starting_after=cur_8f2a19bd` → 200. Returns 2 more orders, `has_more: false`, `next_cursor: null`.

**Expected:** `has_more` should be `true` whenever `next_cursor` is non-null — that's the documented signal for whether to fetch another page.

**Actual:** Step 1 returns `has_more: false` with a non-null cursor; following it anyway (step 2) proves the cursor was valid, since it returns real additional orders.

**Notes:** Only one instance is available, so it's unclear if this is constant or conditional (e.g., only near the end of a list). Worth checking the logic that sets `has_more`. Related but separate: undocumented `refunded` status, dollar/cents mismatch on ord_1006, `total` mismatch on ord_1004 — filed separately as each has a distinct root cause.
