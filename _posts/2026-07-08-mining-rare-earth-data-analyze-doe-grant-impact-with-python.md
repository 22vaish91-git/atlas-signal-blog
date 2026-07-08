---
layout: single
title: "Mining Rare Earth Data: Analyze DoE Grant Impact with Python pandas in 30 Minutes"
date: 2026-07-08
author: "AtlasSignal Desk"
category: "ai_tools"
tags: ["ai_tools", "atlas-signal", "deep-research", "AITools", "Productivity", "MachineLearning"]
description: "You'll build a production-ready pandas pipeline to analyze DoE grant announcements, map rare earth element mentions to stock symbols, and calculate sector impac"
canonical_url: "https://atlassignal.in/posts/mining-rare-earth-data-analyze-doe-grant-impact-with-python/"
og_title: "Mining Rare Earth Data: Analyze DoE Grant Impact with Python pandas in 30 Minutes"
og_description: "You'll build a production-ready pandas pipeline to analyze DoE grant announcements, map rare earth element mentions to stock symbols, and calculate sector impac"
og_url: "https://atlassignal.in/posts/mining-rare-earth-data-analyze-doe-grant-impact-with-python/"
og_image: "https://images.pexels.com/photos/1181359/pexels-photo-1181359.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940"
header:
  overlay_image: https://images.pexels.com/photos/1181359/pexels-photo-1181359.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940
  overlay_filter: 0.75
twitter_card: summary_large_image
toc: true
toc_sticky: true
---

![Mining Rare Earth Data: Analyze DoE Grant Impact with Python pandas in 30 Minutes](https://images.pexels.com/photos/1181359/pexels-photo-1181359.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940)


**Difficulty:** Intermediate | **Category:** Ai Tools

# Mining Rare Earth Data: Analyze DoE Grant Impact with Python pandas in 30 Minutes

Peabody Energy just secured a Department of Energy grant to extract rare earth elements from Wyoming coal mines — a watershed moment linking legacy energy with critical mineral supply chains. Within hours of the July 2026 announcement, energy sector analysts needed to quantify cross-sector exposure: which companies benefit, which rare earths are targeted, and what's the portfolio signal? You'll build a pandas-powered intelligence pipeline to answer these questions from raw grant data in under 30 minutes.

## Prerequisites

- **Python 3.11+** with pip installed (`python --version` to check)
- **pandas 2.2+** and **requests 2.32+** libraries (`pip install pandas requests`)
- **API access**: Free tier accounts for EDGAR SEC API (no key needed) and Alpha Vantage (free key at alphavantage.co)
- **Basic familiarity** with DataFrames and JSON APIs

## Step-by-Step Guide

### Step 1: Set Up Your Analysis Environment

Create a project folder and install dependencies with pinned versions for reproducibility:

```bash
mkdir rare-earth-analysis && cd rare-earth-analysis
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install pandas==2.2.2 requests==2.32.3 matplotlib==3.9.0
```

Create `config.py` to store constants:

```python
# config.py
RARE_EARTHS = [
    'neodymium', 'praseodymium', 'dysprosium', 'terbium',
    'europium', 'yttrium', 'scandium', 'lanthanum'
]
ENERGY_TICKERS = ['BTU', 'ARCH', 'AMR', 'CEIX', 'MP']  # Coal + rare earth miners
DOE_GRANT_KEYWORDS = ['critical minerals', 'rare earth', 'REE', 'coal ash']
```

⚠️ **WARNING**: Alpha Vantage free tier caps at 25 requests/day. Cache results locally to avoid hitting limits during development.

### Step 2: Fetch and Parse DoE Grant Announcements

Pull structured data from DOE's public API (simulated here with a realistic dict — in production, replace with actual API endpoint):

```python
import pandas as pd
import requests
from datetime import datetime

def fetch_doe_grants():
    """Simulate DoE grant API response. Replace with requests.get() in production."""
    grants = [
        {
            'grant_id': 'DE-FOA-0003157',
            'recipient': 'Peabody Energy',
            'award_date': '2026-07-05',
            'amount_usd': 2850000,
            'project_title': 'Rare Earth Element Recovery from Coal Refuse',
            'location': 'Campbell County, WY',
            'keywords': ['coal ash', 'neodymium', 'dysprosium', 'REE extraction']
        },
        {
            'grant_id': 'DE-FOA-0003158',
            'recipient': 'University of Wyoming',
            'award_date': '2026-07-05',
            'amount_usd': 1200000,
            'project_title': 'Scalable Separation Technologies for Critical Minerals',
            'location': 'Laramie, WY',
            'keywords': ['praseodymium', 'separation', 'pilot plant']
        }
    ]
    return pd.DataFrame(grants)

df_grants = fetch_doe_grants()
df_grants['award_date'] = pd.to_datetime(df_grants['award_date'])
print(df_grants[['recipient', 'amount_usd', 'award_date']])
```

**Output:**
```
           recipient  amount_usd award_date
0  Peabody Energy     2850000 2026-07-05
1  University of Wyoming 1200000 2026-07-05
```

### Step 3: Extract Rare Earth Element Mentions

Use pandas' string methods to flag which rare earths appear in each grant:

```python
from config import RARE_EARTHS

def extract_ree_mentions(df):
    """Create binary columns for each rare earth element."""
    df_ree = df.copy()
    keywords_str = df_ree['keywords'].apply(lambda x: ' '.join(x).lower())
    
    for element in RARE_EARTHS:
        df_ree[f'mentions_{element}'] = keywords_str.str.contains(element, regex=False)
    
    df_ree['total_ree_count'] = df_ree[[f'mentions_{e}' for e in RARE_EARTHS]].sum(axis=1)
    return df_ree

df_analyzed = extract_ree_mentions(df_grants)
print(df_analyzed[['recipient', 'total_ree_count', 'mentions_neodymium', 'mentions_dysprosium']])
```

**Gotcha:** Keyword lists stored as Python lists need `.apply(lambda x: ' '.join(x))` before using `.str.contains()`. Direct string operations on list columns fail silently.

### Step 4: Enrich with Stock Market Data

Cross-reference grant recipients with publicly traded companies:

```python
import time

def map_recipient_to_ticker(recipient_name):
    """Hardcoded mapping for demo. Use fuzzy matching in production."""
    mapping = {
        'Peabody Energy': 'BTU',
        'Arch Resources': 'ARCH',
        'MP Materials': 'MP'
    }
    return mapping.get(recipient_name, None)

df_analyzed['ticker'] = df_analyzed['recipient'].apply(map_recipient_to_ticker)

def fetch_stock_price(ticker):
    """Get latest close price from Alpha Vantage. Cache aggressively."""
    if ticker is None:
        return None
    # Simulated response — replace with actual API call:
    # url = f"https://www.alphavantage.co/query?function=GLOBAL_QUOTE&symbol={ticker}&apikey=YOUR_KEY"
    mock_prices = {'BTU': 24.73, 'ARCH': 118.45, 'MP': 19.82}
    time.sleep(0.1)  # Rate limit politeness
    return mock_prices.get(ticker)

df_analyzed['stock_price_usd'] = df_analyzed['ticker'].apply(fetch_stock_price)
```

### Step 5: Calculate Impact Scores

Quantify potential market impact with a weighted formula:

```python
def calculate_impact_score(row):
    """
    Impact = (Grant Amount / 1M) * REE Diversity * Stock Liquidity Proxy
    Higher score = stronger signal for sector analysts
    """
    if pd.isna(row['stock_price_usd']):
        return 0
    
    grant_factor = row['amount_usd'] / 1_000_000
    ree_diversity = row['total_ree_count']
    liquidity_proxy = min(row['stock_price_usd'] / 10, 10)  # Cap at 10x
    
    return grant_factor * ree_diversity * liquidity_proxy

df_analyzed['impact_score'] = df_analyzed.apply(calculate_impact_score, axis=1)
df_sorted = df_analyzed.sort_values('impact_score', ascending=False)

print(df_sorted[['recipient', 'ticker', 'amount_usd', 'total_ree_count', 'impact_score']])
```

⚠️ **WARNING**: `.apply()` with complex functions can be 10-100x slower than vectorized operations. For datasets >10K rows, refactor into column-wise calculations.

### Step 6: Export Results for Downstream Tools

Save analysis in multiple formats for different consumers:

```python
# CSV for Excel users
df_sorted.to_csv('doe_grants_analysis.csv', index=False)

# JSON for API consumers
df_sorted.to_json('doe_grants_analysis.json', orient='records', date_format='iso')

# Parquet for data warehouses (requires pyarrow: pip install pyarrow)
df_sorted.to_parquet('doe_grants_analysis.parquet', compression='snappy')

print(f"✓ Exported {len(df_sorted)} grant records across 3 formats")
```

**Pro Tip**: Parquet files are 60-80% smaller than CSV for numerical data and preserve dtypes perfectly — critical when piping into SQL databases or Spark clusters.

### Step 7: Visualize Geographic Clustering

Identify which states are becoming rare earth hubs:

```python
import matplotlib.pyplot as plt

df_analyzed['state'] = df_analyzed['location'].str.extract(r', ([A-Z]{2})$')
state_totals = df_analyzed.groupby('state')['amount_usd'].sum().sort_values(ascending=False)

plt.figure(figsize=(10, 6))
state_totals.plot(kind='barh', color='#2C5F8D')
plt.xlabel('Total Grant Amount (USD)')
plt.title('DoE Rare Earth Grants by State (July 2026)')
plt.tight_layout()
plt.savefig('grants_by_state.png', dpi=150)
print("✓ Saved grants_by_state.png")
```

## Practical Example: Complete End-to-End Pipeline

Here's a production-ready script combining all steps with error handling:

```python
import pandas as pd
from datetime import datetime, timedelta

def run_grant_analysis(days_back=30):
    """
    Complete pipeline: fetch → parse → enrich → score → export
    """
    # Step 1: Load data
    df = fetch_doe_grants()  # From Step 2
    
    # Step 2: Filter recent grants
    cutoff_date = datetime.now() - timedelta(days=days_back)
    df = df[df['award_date'] >= cutoff_date]
    
    if len(df) == 0:
        print(f"⚠️  No grants found in last {days_back} days")
        return None
    
    # Step 3-5: Transform
    df = extract_ree_mentions(df)
    df['ticker'] = df['recipient'].apply(map_recipient_to_ticker)
    df['stock_price_usd'] = df['ticker'].apply(fetch_stock_price)
    df['impact_score'] = df.apply(calculate_impact_score, axis=1)
    
    # Step 6: Export
    output_file = f"doe_ree_analysis_{datetime.now():%Y%m%d}.csv"
    df.sort_values('impact_score', ascending=False).to_csv(output_file, index=False)
    
    print(f"✓ Analyzed {len(df)} grants")
    print(f"✓ Top recipient: {df.iloc[0]['recipient']} (score: {df.iloc[0]['impact_score']:.2f})")
    print(f"✓ Results saved: {output_file}")
    return df

# Run it
results = run_grant_analysis(days_back=15)
```

## Debugging Common Errors

**Error:** `KeyError: 'keywords'`  
**Cause:** Column name mismatch between mock data and production API response  
**Fix:** Print `df.columns` immediately after loading data. Use `df.rename(columns={'old': 'new'})` to standardize.

**Error:** `TypeError: unhashable type: 'list'` during `.groupby()`  
**Cause:** Grouping on a column that contains lists (like `keywords`)  
**Fix:** Convert list columns to strings first: `df['keywords_str'] = df['keywords'].astype(str)`

**Error:** Alpha Vantage returns `{'Note': 'API call frequency exceeded'}`  
**Cause:** Free tier rate limit (25 calls/day)  
**Fix:** Implement local caching with `functools.lru_cache` or store results in SQLite between runs.

## Key Takeaways

- **pandas 2.2** handles 100K-row grant datasets in <2 seconds on a laptop with optimized dtypes (use `pd.read_csv(..., dtype={...})`)
- **String vectorization** (`.str.contains()`) is 50x faster than Python loops for keyword extraction across 1000+ records
- **Impact scoring** turns raw announcements into ranked intelligence — adaptable to earnings calls, patent filings, or FDA approvals
- **Multi-format export** (CSV + JSON + Parquet) makes your analysis consumable by Excel users, web dashboards, and ML pipelines simultaneously

## What's Next

Extend this pipeline by streaming live DoE RSS feeds into your DataFrame with `feedparser`, then auto-post high-impact scores to a Slack channel using webhooks — full automation in <50 lines of additional code.

---

**Key Takeaway:** You'll build a production-ready pandas pipeline to analyze DoE grant announcements, map rare earth element mentions to stock symbols, and calculate sector impact scores — skills transferable to any structured data intelligence workflow.

---

*New AI tutorials published daily on [AtlasSignal](https://atlassignal.in). Follow [@AtlasSignalDesk](https://twitter.com/AtlasSignalDesk) for more.*

---

*This report was produced with AI-assisted research and drafting, curated and reviewed under AtlasSignal's [editorial standards](/editorial-standards/). For corrections or feedback, contact [atlassignal.ai@gmail.com](mailto:atlassignal.ai@gmail.com).*


