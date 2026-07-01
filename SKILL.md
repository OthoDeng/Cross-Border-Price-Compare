---
name: Cross-Border Price Compare
description: >
  Compare Walmart product prices between China and a foreign country
  (default Canada, walmart.ca). Use this skill whenever the user wants to check
  cross-border price differences, decide whether it's worth bringing an item when
  traveling abroad, compare China vs foreign Walmart prices, or mentions "比价",
  "沃尔玛比价", "带什么东西", "出国带", "价格比较", "walmart price comparison",
  "is X cheaper in China". Triggers on any travel-packing or price-comparison
  request involving Walmart or cross-border shopping.
---

# Walmart Cross-Border Price Compare

Compare the same (or similar) product across Walmart China and Walmart in a
foreign country. Compute the price ratio to decide whether you should pack it
when traveling — if the foreign price is significantly higher, it's worth the
luggage space.

## Workflow

### 1. Clarify Parameters

If the user hasn't provided them, ask briefly:

- **Product name** (required)
- **Foreign country** (default: Canada, using walmart.ca)
- **Threshold ratio** (default: 2.0 — foreign price ≥ 2× China price triggers
  a "worth bringing" recommendation)

Don't over-interrogate; only ask what's missing.

### 2. Search China Prices via HaiNa Shopping Assistant

Use the **HaiNa-Shopping-Assistant** skill's search engine to find the product
price in the Chinese market. The HaiNa assistant uses the 值得买 (Zhidemai) API
for product and content search across all Chinese e-commerce platforms (京东,
天猫, 淘宝, etc.), providing a more comprehensive price picture than
Walmart China's limited web presence.

**API Key Setup:**

Before running any search, set the environment variables:

```bash
export ZHIDEMAI_CONTENT_SEARCH="{YOUR_API}"
export ZHIDEMAI_PRODUCT_SEARCH="{YOUR_API}"
```

**Search Method:**

Use the "优惠好价" (best price) intent mode — it searches product listings
directly and returns prices. Run the HaiNa search script from the
HaiNa-Shopping-Assistant skill directory:

```bash
cd "{PATH_TO_SKILL}/Haina-Shopping-Assistant/scripts" && \
python search_work_main.py \
  --user_query "<product name in Chinese> 价格" \
  --intent "优惠好价" \
  --product_query '["<product name in Chinese>"]' \
  --product_size 5
```

The script prints the search result JSON to stdout. Parse it to extract:
- Product name
- Price (in CNY)
- Purchase link / platform

If the script fails (API error, network issue), fall back to using
`mcp__exa__web_search_exa` to search "京东 <product name> 价格" for
approximate pricing.

### 3. Search Foreign Prices via Walmart.ca

Use `mcp__exa__web_search_exa` to search the foreign Walmart site:

```
site:walmart.ca <product name in English>
```

Then use `mcp__exa__web_fetch_exa` on the top 1–2 product page URLs to extract:
- Product name/title
- Price (numeric value and currency)
- Whether the product is in stock

Run steps 2 and 3 in **parallel** — they are independent.

### 4. Get Exchange Rate

Search for the current exchange rate:

```
<foreign currency> to CNY exchange rate today
```

Use `mcp__exa__web_search_exa`. Prefer XE.com or Wise.com results.

### 5. Compute and Report

Convert both prices to CNY for a fair comparison, then compute:

```
ratio = foreign_price_in_cny / china_price_in_cny
```

**Edge cases:**

| Situation | Ratio | Meaning |
|-----------|-------|---------|
| Found in both markets | `foreign / china` | Normal comparison |
| Only available in China | `∞` (infinity) | Item is unavailable abroad |
| Only available in foreign market | `0` | Item only sold overseas |

**Recommendation logic:**

- `ratio ≥ threshold` → **Worth bringing.** The foreign price is strikingly higher.
- `ratio < threshold` but > 1 → Somewhat more expensive, but not dramatic.
- `ratio ≤ 1` → Cheaper or same abroad — not worth packing.
- `ratio = ∞` → **Strongly worth bringing.** Can't buy it there at all.
- `ratio = 0` → No need to bring; it's only sold overseas.

## Output Format

Present results in a clear table:

```
## Price Comparison: <Product Name>

| Market  | Product Found        | Local Price | Price in CNY |
|---------|----------------------|-------------|--------------|
| China   | <title>              | ¥XX.XX      | ¥XX.XX       |
| Canada  | <title>              | $XX.XX      | ¥XX.XX       |

**Exchange rate:** 1 CAD = X.XX CNY
**Price ratio (CA/CN):** X.XX
**Threshold:** 2.0

**Verdict:** <recommendation text>
```

If only one market has the product, show "Not found" for the other and state
the special ratio case.

## Tips

- When a price range is shown (e.g. "$12.99 – $18.99"), use the lowest
  available price and note the range.
- The HaiNa API searches across all major Chinese platforms — use the best
  available price for comparison, but note which platform it came from.
- If the HaiNa product search returns no results, try content search mode
  ("商品推荐") for broader price discovery before falling back to Exa.
- If walmart.ca doesn't have the exact product, note a comparable substitute
  and its price.
