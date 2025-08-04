# Marinade staked SOL (mSOL) token analysis
[Full Dashboard](https://joshuatochinwachi.github.io/Marinade-staked-SOL-token-analysis/) | [Data story telling on 𝕏](https://x.com/defi__josh/status/1908469547604025559)

This dashboard and entire analysis was built and developed using [Flipside Crypto](https://flipsidecrypto.xyz) by me. Unfortunately, Flipside’s querying and dashboarding tool has gone dark because of their collaboration with [Snowflake](https://www.snowflake.com/en/).

Flipside also used Snowflake’s SQL dialect as their SQL flavor when they were active, so the SQL code you’ll see in this project—for both querying and dashboarding—is very similar to Snowflake’s syntax.

Using [Python scripts](https://github.com/joshuatochinwachi/Flipside_dashboard_porter), I scraped my Flipside dashboard and visualizations from the Flipside website.

## Queries/Metrics used

### 1. Market cap, Price and Supply

```
WITH token_mints AS (
  SELECT 
    mint,
    SUM(CASE WHEN succeeded = TRUE THEN mint_amount/POWER(10, decimal) ELSE 0 END) as total_minted
  FROM solana.defi.fact_token_mint_actions
  WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
  AND succeeded = TRUE
  GROUP BY 1
),
token_burns AS (
  SELECT 
    mint,
    SUM(CASE WHEN succeeded = TRUE THEN burn_amount/POWER(10, decimal) ELSE 0 END) as total_burned
  FROM solana.defi.fact_token_burn_actions
  WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
  AND succeeded = TRUE
  GROUP BY 1
),
latest_price AS (
  SELECT 
    token_address,
    price as current_price,
    name,
    symbol
  FROM solana.price.ez_prices_hourly
  WHERE token_address = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
  AND hour >= DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY hour DESC
  LIMIT 1
)

SELECT 
  tm.mint as token_address,
  p.name,
  p.symbol,
  (tm.total_minted - COALESCE(tb.total_burned, 0)) as current_supply,
  p.current_price as price_usd,
  (tm.total_minted - COALESCE(tb.total_burned, 0)) * price_usd as market_cap_usd
FROM token_mints tm
LEFT JOIN token_burns tb ON tm.mint = tb.mint
LEFT JOIN latest_price p ON tm.mint = p.token_address;
```

### 2. mSOL/USD Price Movement - Last 365 Days (Candlestick chart)

```
WITH token_mints AS (
  SELECT 
    mint,
    SUM(CASE WHEN succeeded = TRUE THEN mint_amount/POWER(10, decimal) ELSE 0 END) as total_minted
  FROM solana.defi.fact_token_mint_actions
  WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
  AND succeeded = TRUE
  GROUP BY 1
),
token_burns AS (
  SELECT 
    mint,
    SUM(CASE WHEN succeeded = TRUE THEN burn_amount/POWER(10, decimal) ELSE 0 END) as total_burned
  FROM solana.defi.fact_token_burn_actions
  WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
  AND succeeded = TRUE
  GROUP BY 1
),
latest_price AS (
  SELECT 
    token_address,
    price as current_price,
    name,
    symbol
  FROM solana.price.ez_prices_hourly
  WHERE token_address = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
  AND hour >= DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY hour DESC
  LIMIT 1
),
daily_prices AS (
  SELECT 
    DATE_TRUNC('day', hour) as date,
    token_address,
    FIRST_VALUE(price) OVER (PARTITION BY DATE_TRUNC('day', hour), token_address ORDER BY hour) as day_open,
    LAST_VALUE(price) OVER (PARTITION BY DATE_TRUNC('day', hour), token_address ORDER BY hour) as day_close,
    MAX(price) OVER (PARTITION BY DATE_TRUNC('day', hour), token_address) as day_high,
    MIN(price) OVER (PARTITION BY DATE_TRUNC('day', hour), token_address) as day_low
  FROM solana.price.ez_prices_hourly
  WHERE token_address = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
  AND hour >= DATEADD('year', -1, CURRENT_TIMESTAMP())
  QUALIFY ROW_NUMBER() OVER (PARTITION BY DATE_TRUNC('day', hour), token_address ORDER BY hour DESC) = 1
)

SELECT 
  dp.date as "day",
  dp.day_open as "opening price",
  dp.day_close as "closing price",
  dp.day_high as "highest price",
  dp.day_low as "lowest price",
  ((dp.day_close - dp.day_open) / dp.day_open) * 100 as daily_percent_change,
  (dp.day_high - dp.day_low) as daily_price_range
FROM daily_prices dp
ORDER BY 1 DESC;
```

### 3. 24 hours trading volume

```
WITH mSOL_swaps AS (
  SELECT 
    SUM(CASE 
      WHEN swap_from_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So' THEN swap_from_amount_usd
      WHEN swap_to_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So' THEN swap_to_amount_usd
    END) as total_volume_usd,
    COUNT(*) as number_of_swaps
  FROM 
    solana.defi.ez_dex_swaps
  WHERE 
    block_timestamp >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
    AND (
      swap_from_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So' 
      OR swap_to_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    )
)
SELECT 
  'mSOL' as token,
  ROUND(total_volume_usd, 2) as volume_usd_24h,
  number_of_swaps as total_swaps_24h
FROM 
  mSOL_swaps;
```

### 4. Token activity summary over the last 30 days

```
WITH token_metadata AS (
  SELECT 
    token_address,
    symbol,
    name,
    decimals
  FROM solana.price.ez_asset_metadata
  WHERE token_address = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
),
token_activity AS (
  SELECT 
    COUNT(DISTINCT tx_id) as total_transfers,
    COUNT(DISTINCT tx_from) as unique_senders,
    COUNT(DISTINCT tx_to) as unique_receivers,
    SUM(amount) as total_transfer_volume
  FROM solana.core.fact_transfers
  WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
  AND block_timestamp >= DATEADD('day', -30, CURRENT_TIMESTAMP())
),
token_price AS (
  SELECT 
    price as current_price
  FROM solana.price.ez_prices_hourly
  WHERE token_address = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
  AND hour >= DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY hour DESC
  LIMIT 1
),
mint_burn_summary AS (
  SELECT 
    SUM(CASE WHEN event_type = 'mint' THEN mint_amount ELSE 0 END) as total_minted,
    COUNT(DISTINCT CASE WHEN event_type = 'mint' THEN tx_id END) as mint_count,
    SUM(CASE WHEN event_type = 'burn' THEN burn_amount ELSE 0 END) as total_burned,
    COUNT(DISTINCT CASE WHEN event_type = 'burn' THEN tx_id END) as burn_count
  FROM (
    SELECT tx_id, 'mint' as event_type, mint_amount, 0 as burn_amount
    FROM solana.defi.fact_token_mint_actions
    WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    AND block_timestamp >= DATEADD('day', -30, CURRENT_TIMESTAMP())
    UNION ALL
    SELECT tx_id, 'burn' as event_type, 0 as mint_amount, burn_amount
    FROM solana.defi.fact_token_burn_actions
    WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    AND block_timestamp >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  )
)

SELECT 
  m.token_address,
  m.symbol,
  m.name,
  p.current_price,
  a.total_transfers,
  a.unique_senders,
  a.unique_receivers,
  a.total_transfer_volume,
  mb.total_burned,
  mb.burn_count
FROM token_metadata m
LEFT JOIN token_activity a ON 1=1
LEFT JOIN token_price p ON 1=1
LEFT JOIN mint_burn_summary mb ON 1=1
```

### 5. current number of holders

```
WITH latest_balances AS (
    SELECT 
        owner,
        balance,
        ROW_NUMBER() OVER (PARTITION BY owner ORDER BY block_timestamp DESC) as rn
    FROM solana.core.fact_token_balances
    WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
)

SELECT COUNT(DISTINCT owner) as total_holders
FROM latest_balances
WHERE rn = 1 
AND balance > 0;
```

### 6. Holders distribution and balance ranges

```
WITH current_balances AS (
  SELECT 
    owner,
    balance
  FROM solana.core.fact_token_balances
  WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
  AND block_timestamp >= DATEADD(day, -30, CURRENT_TIMESTAMP)
  AND balance > 0
  QUALIFY ROW_NUMBER() OVER (PARTITION BY account_address ORDER BY block_timestamp DESC) = 1
),
balance_ranges AS (
  SELECT 
    CASE 
      WHEN balance <= 1 THEN '0-1 mSOL'
      WHEN balance <= 10 THEN '1-10 mSOL'
      WHEN balance <= 100 THEN '10-100 mSOL'
      WHEN balance <= 1000 THEN '100-1000 mSOL'
      WHEN balance <= 10000 THEN '1000-10000 mSOL'
      ELSE '10000+ mSOL'
    END AS balance_range,
    COUNT(DISTINCT owner) as num_holders,
    SUM(balance) as total_balance
  FROM current_balances
  GROUP BY 1
)

SELECT 
  balance_range as "balance range",
  num_holders as "no. of holders",
  total_balance as "total balance",
  ROUND(100.0 * num_holders / SUM(num_holders) OVER (), 2) as "percentage of holders",
  ROUND(100.0 * total_balance / SUM(total_balance) OVER (), 2) as "percentage of supply"
FROM balance_ranges
ORDER BY 
  CASE balance_range
    WHEN '0-1 mSOL' THEN 1
    WHEN '1-10 mSOL' THEN 2
    WHEN '10-100 mSOL' THEN 3
    WHEN '100-1000 mSOL' THEN 4
    WHEN '1000-10000 mSOL' THEN 5
    WHEN '10000+ mSOL' THEN 6
  END;
```

### 7. Transaction volume and active addresses over the last 30 days

```
WITH daily_metrics AS (
  SELECT 
    DATE_TRUNC('day', block_timestamp) as date,
    COUNT(DISTINCT CASE WHEN tx_from != tx_to THEN tx_from END) + 
    COUNT(DISTINCT CASE WHEN tx_from != tx_to THEN tx_to END) as daily_active_addresses,
    COUNT(DISTINCT tx_id) as number_of_transfers,
    SUM(amount) as total_volume
  FROM solana.core.fact_transfers
  WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    AND block_timestamp >= DATEADD('day', -30, CURRENT_DATE)
    AND tx_from != tx_to -- Exclude self-transfers
  GROUP BY 1
  ORDER BY 1
)

SELECT 
  date as "day",
  daily_active_addresses as "daily active addresses",
  number_of_transfers as "number of transfers",
  total_volume as "total volume",
  AVG(total_volume) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as "seven day avg volume"
FROM daily_metrics
ORDER BY date DESC;
```

### 8. Trading Volume (using DEX swaps) over the last 30 days

```
WITH mSOL_swaps AS (
  SELECT 
    DATE_TRUNC('day', block_timestamp) as date,
    SUM(CASE 
      WHEN swap_from_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So' 
      THEN swap_from_amount_usd 
      ELSE swap_to_amount_usd 
    END) as daily_volume_usd,
    COUNT(*) as number_of_swaps
  FROM solana.defi.ez_dex_swaps
  WHERE (swap_from_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So' 
    OR swap_to_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So')
    AND block_timestamp >= DATEADD('day', -30, CURRENT_DATE)
  GROUP BY 1
  ORDER BY 1
)
SELECT 
  date as "day",
  daily_volume_usd as "daily volume (USD)",
  number_of_swaps as "number of swaps",
  SUM(daily_volume_usd) OVER (ORDER BY date) as "cumulative volume (USD)"
FROM mSOL_swaps;
```

### 9. Unique trading pairs and their liquidity

```
WITH mSOL_pools AS (
  SELECT DISTINCT
    pool_address,
    pool_name,
    token_a_mint,
    token_a_symbol,
    token_b_mint,
    token_b_symbol,
    platform
  FROM solana.defi.ez_liquidity_pool_actions
  WHERE block_timestamp >= DATEADD('day', -30, CURRENT_TIMESTAMP)
    AND (token_a_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    OR token_b_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So')
    AND action_type = 'deposit' -- Looking at deposits to measure liquidity
),

latest_liquidity AS (
  SELECT 
    pool_address,
    SUM(CASE 
      WHEN token_a_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So' THEN token_a_amount_usd
      WHEN token_b_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So' THEN token_b_amount_usd
    END) as msol_liquidity_usd
  FROM solana.defi.ez_liquidity_pool_actions
  WHERE block_timestamp >= DATEADD('day', -30, CURRENT_TIMESTAMP)
    AND action_type = 'deposit'
    AND pool_address IN (SELECT pool_address FROM mSOL_pools)
  GROUP BY 1
)

SELECT 
  mp.platform as "platform",
  mp.pool_name as "pairs",
  mp.pool_address as "addy",
  CASE 
    WHEN mp.token_a_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So' 
    THEN mp.token_b_symbol 
    ELSE mp.token_a_symbol 
  END as paired_token,
  ll.msol_liquidity_usd as "liquidity (USD)"
FROM mSOL_pools mp
LEFT JOIN latest_liquidity ll ON mp.pool_address = ll.pool_address
WHERE ll.msol_liquidity_usd IS NOT NULL
ORDER BY ll.msol_liquidity_usd DESC;
```

### 10. Liquidity pool actions over the last 30 days

```
SELECT 
  DATE_TRUNC('day', block_timestamp) as "day",
  action_type as "action type",
  SUM(CASE 
    WHEN token_a_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So' THEN token_a_amount_usd
    WHEN token_b_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So' THEN token_b_amount_usd
    WHEN token_c_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So' THEN token_c_amount_usd
    WHEN token_d_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So' THEN token_d_amount_usd
  END) as "total value (USD)",
  COUNT(*) as "number of actions",
  platform as "platform"
FROM solana.defi.ez_liquidity_pool_actions
WHERE (token_a_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    OR token_b_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    OR token_c_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    OR token_d_mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So')
  AND block_timestamp >= DATEADD('day', -30, CURRENT_DATE)
GROUP BY 1, 2, 5
ORDER BY 1, 2;
```

### 11. Mint and burn activities over the last 30 days

```
WITH mint_data AS (
    SELECT 
        DATE_TRUNC('day', block_timestamp) as date,
        COUNT(*) as mint_count,
        SUM(mint_amount/POWER(10, decimal)) as total_tokens_minted
    FROM solana.defi.fact_token_mint_actions
    WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
        AND succeeded = true
        AND block_timestamp >= DATEADD(day, -30, CURRENT_DATE)
    GROUP BY 1
),
burn_data AS (
    SELECT 
        DATE_TRUNC('day', block_timestamp) as date,
        COUNT(*) as burn_count,
        SUM(burn_amount/POWER(10, decimal)) as total_tokens_burned
    FROM solana.defi.fact_token_burn_actions
    WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
        AND succeeded = true
        AND block_timestamp >= DATEADD(day, -30, CURRENT_DATE)
    GROUP BY 1
)
SELECT 
    COALESCE(m.date, b.date) as date,
    COALESCE(m.mint_count, 0) as "mint count",
    COALESCE(m.total_tokens_minted, 0) as "total tokens minted",
    COALESCE(b.burn_count, 0) as "burn count",
    COALESCE(b.total_tokens_burned, 0) as "total tokens burned",
    COALESCE(m.total_tokens_minted, 0) - COALESCE(b.total_tokens_burned, 0) as "net token change"
FROM mint_data m
FULL OUTER JOIN burn_data b ON m.date = b.date
ORDER BY date DESC;
```

### 12. Market Value to Realized Value Ratio (MVRV) over the last 30 days

```
WITH current_supply AS (
  SELECT 
    SUM(balance) as total_supply,
    block_timestamp::date as date
  FROM solana.core.fact_token_balances
  WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    AND block_timestamp >= DATEADD('day', -30, CURRENT_DATE)
  GROUP BY date
),
market_value AS (
  SELECT 
    cs.date,
    cs.total_supply * p.price as market_value,
    cs.total_supply
  FROM current_supply cs
  LEFT JOIN solana.price.ez_prices_hourly p
    ON p.token_address = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    AND DATE_TRUNC('day', p.hour) = cs.date
  WHERE p.hour = DATE_TRUNC('hour', p.hour) -- get one price per day
    AND EXTRACT(hour FROM p.hour) = 0 -- use midnight price
),
realized_value AS (
  SELECT 
    DATE_TRUNC('day', t.block_timestamp) as date,
    SUM(t.amount * p.price) as realized_value
  FROM solana.core.fact_transfers t
  LEFT JOIN solana.price.ez_prices_hourly p
    ON p.token_address = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    AND DATE_TRUNC('hour', t.block_timestamp) = p.hour
  WHERE t.mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    AND t.block_timestamp >= DATEADD('day', -30, CURRENT_DATE)
  GROUP BY date
)
SELECT 
  mv.date as "day",
  mv.total_supply as "total supply",
  mv.market_value as "market value",
  rv.realized_value as "realized value",
  mv.market_value / NULLIF(rv.realized_value, 0) as "mvrv ratio"
FROM market_value mv
LEFT JOIN realized_value rv ON mv.date = rv.date
WHERE mv.market_value IS NOT NULL
ORDER BY mv.date DESC;
```

### 13. Network Value to Transactions (NVT) Ratio over the last 30 days

```
WITH daily_transfers AS (
  SELECT 
    DATE_TRUNC('day', t.block_timestamp) as date,
    SUM(ABS(t.amount)) as daily_transfer_volume,
    MAX(p.price) as token_price
  FROM solana.core.fact_transfers t
  LEFT JOIN solana.price.ez_prices_hourly p
    ON DATE_TRUNC('hour', t.block_timestamp) = p.hour
    AND t.mint = p.token_address
  WHERE t.mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    AND t.block_timestamp >= CURRENT_DATE - 30
  GROUP BY 1
),
daily_balances AS (
  SELECT 
    DATE_TRUNC('day', block_timestamp) as date,
    SUM(balance) as total_supply
  FROM solana.core.fact_token_balances
  WHERE mint = 'mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So'
    AND block_timestamp >= CURRENT_DATE - 30
  GROUP BY 1
)

SELECT 
  dt.date as "day",
  dt.token_price as "token price",
  db.total_supply as "total supply",
  dt.daily_transfer_volume as "daily transfer volume",
  (db.total_supply * dt.token_price) as "network value",
  (db.total_supply * dt.token_price) / NULLIF(dt.daily_transfer_volume, 0) as "nvt ratio"
FROM daily_transfers dt
LEFT JOIN daily_balances db ON dt.date = db.date
WHERE dt.date >= CURRENT_DATE - 30
ORDER BY dt.date DESC;
```
