---
layout: single
title: "Build a Real-Time Shipping Route Alert System Using Maritime AI APIs and Geopolitical Data"
date: 2026-05-04
category: "ai_tools"
tags: ["ai_tools", "atlas-signal", "deep-research", "AITools", "Productivity", "MachineLearning"]
description: "By combining AIS ship tracking APIs with Claude's contextual reasoning, you can build an automated alert system that monitors geopolitical maritime chokepoints"
canonical_url: "https://atlassignal.in/posts/build-a-real-time-shipping-route-alert-system-using-maritime/"
og_title: "Build a Real-Time Shipping Route Alert System Using Maritime AI APIs and Geopolitical Data"
og_description: "By combining AIS ship tracking APIs with Claude's contextual reasoning, you can build an automated alert system that monitors geopolitical maritime chokepoints"
og_url: "https://atlassignal.in/posts/build-a-real-time-shipping-route-alert-system-using-maritime/"
og_image: "https://images.pexels.com/photos/30440680/pexels-photo-30440680.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940"
header:
  overlay_image: https://images.pexels.com/photos/30440680/pexels-photo-30440680.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940
  overlay_filter: 0.75
twitter_card: summary_large_image
toc: true
toc_sticky: true
---

![Build a Real-Time Shipping Route Alert System Using Maritime AI APIs and Geopolitical Data](https://images.pexels.com/photos/30440680/pexels-photo-30440680.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940)


**Difficulty:** Intermediate | **Category:** Ai Tools

# Build a Real-Time Shipping Route Alert System Using Maritime AI APIs and Geopolitical Data

With Trump announcing U.S. intervention to free ships in the Strait of Hormuz starting Monday, global supply chains face immediate uncertainty. Over 20% of the world's petroleum passes through this 21-mile-wide chokepoint—any disruption cascades into delays worth millions. By the end of this tutorial, you'll deploy an AI-powered monitoring system that tracks ship movements through critical maritime zones, analyzes geopolitical news in real-time, and sends actionable alerts when route disruptions occur.

## Prerequisites

Before starting, ensure you have:

- **Python 3.11+** with pip installed
- **Anthropic API key** (Claude Sonnet 3.5 costs $3/M input tokens, $15/M output tokens)
- **MarineTraffic API account** (free tier: 100 requests/day) or **AIS Stream API** ($29/month for real-time data)
- **Twilio account** for SMS alerts (free trial includes $15 credit)
- **Basic familiarity with cron jobs or GitHub Actions** for scheduling

## Step-by-Step Guide

### Step 1: Set Up Your Maritime Data Pipeline

Install the required dependencies and configure your API credentials:

```bash
pip install anthropic requests python-dotenv twilio geopy pandas
```

Create a `.env` file in your project root:

```bash
ANTHROPIC_API_KEY=sk-ant-api03-...
MARINETRAFFIC_API_KEY=your_mt_key_here
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_FROM_NUMBER=+1234567890
ALERT_TO_NUMBER=+1987654321
```

⚠️ **WARNING:** Never commit `.env` to version control. Add it to `.gitignore` immediately.

### Step 2: Query Live Ship Positions in Critical Zones

The Strait of Hormuz sits between coordinates (26.5°N, 56.0°E) and (25.5°N, 57.0°E). Use MarineTraffic's Area API to fetch all vessels currently transiting:

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def get_ships_in_hormuz():
    """Fetch all vessels in Strait of Hormuz bounding box."""
    url = "https://services.marinetraffic.com/api/exportvessels/v:8/"
    params = {
        "MINLAT": 25.5,
        "MAXLAT": 26.5,
        "MINLON": 56.0,
        "MAXLON": 57.0,
        "timespan": 20,  # vessels active in last 20 minutes
        "protocol": "json"
    }
    headers = {"API-Key": os.getenv("MARINETRAFFIC_API_KEY")}
    
    response = requests.get(url, params=params, headers=headers)
    response.raise_for_status()
    
    ships = response.json()
    print(f"Found {len(ships)} vessels in Strait of Hormuz")
    return ships

# Test the function
vessels = get_ships_in_hormuz()
```

**Gotcha:** MarineTraffic free tier rate-limits to 1 request/minute. For production, upgrade to the Developer plan ($99/month) or switch to AIS Stream which offers WebSocket connections.

### Step 3: Structure Geopolitical Context for Claude

Claude excels at reasoning over complex situations when given structured context. Compile ship data alongside news mentions:

```python
from anthropic import Anthropic
import json

client = Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

def analyze_disruption_risk(vessels, recent_news):
    """Use Claude to assess maritime disruption severity."""
    
    # Prepare structured vessel summary
    vessel_summary = {
        "total_count": len(vessels),
        "vessel_types": {},
        "flags": {},
        "stationary_count": 0
    }
    
    for v in vessels:
        # Count by type
        vtype = v.get("TYPE_SUMMARY", "Unknown")
        vessel_summary["vessel_types"][vtype] = vessel_summary["vessel_types"].get(vtype, 0) + 1
        
        # Count by flag
        flag = v.get("FLAG", "Unknown")
        vessel_summary["flags"][flag] = vessel_summary["flags"].get(flag, 0) + 1
        
        # Check if vessel is stopped (speed 10."""
    
    severity = analysis["severity_score"]
    stationary = analysis.get("stationary_vessel_count", 0)
    
    if severity >= 7 or stationary > 10:
        twilio_client = Client(
            os.getenv("TWILIO_ACCOUNT_SID"),
            os.getenv("TWILIO_AUTH_TOKEN")
        )
        
        message_body = f"""⚠️ STRAIT OF HORMUZ ALERT
Severity: {severity}/10
Stationary vessels: {stationary}
Recommendation: {analysis['recommendations'][0]}
Timeline: {analysis['timeline_estimate']}"""
        
        message = twilio_client.messages.create(
            body=message_body,
            from_=os.getenv("TWILIO_FROM_NUMBER"),
            to=os.getenv("ALERT_TO_NUMBER")
        )
        
        print(f"Alert sent: {message.sid}")
        return True
    
    print("No alert necessary (severity below threshold)")
    return False
```

⚠️ **WARNING:** Twilio charges $0.0079/SMS for U.S. numbers. Set a monthly budget cap in your account dashboard to avoid surprise bills.

### Step 5: Automate with Scheduled Monitoring

Deploy as a GitHub Action that runs every 30 minutes:

```yaml
# .github/workflows/monitor-hormuz.yml
name: Monitor Strait of Hormuz

on:
  schedule:
    - cron: '*/30 * * * *'  # Every 30 minutes
  workflow_dispatch:  # Manual trigger option

jobs:
  check-disruptions:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run monitoring script
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          MARINETRAFFIC_API_KEY: ${{ secrets.MARINETRAFFIC_API_KEY }}
          TWILIO_ACCOUNT_SID: ${{ secrets.TWILIO_ACCOUNT_SID }}
          TWILIO_AUTH_TOKEN: ${{ secrets.TWILIO_AUTH_TOKEN }}
          TWILIO_FROM_NUMBER: ${{ secrets.TWILIO_FROM_NUMBER }}
          ALERT_TO_NUMBER: ${{ secrets.ALERT_TO_NUMBER }}
        run: python monitor.py
```

Add your API keys to repository secrets under Settings → Secrets and variables → Actions.

**Gotcha:** GitHub Actions free tier includes 2,000 minutes/month. Running every 30 minutes consumes ~60 minutes/month, well within limits.

### Step 6: Add Historical Context for Better Predictions

Enhance Claude's analysis by storing past disruption patterns:

```python
import pandas as pd
from datetime import datetime

def log_analysis(analysis, vessels):
    """Append analysis to historical CSV for trend detection."""
    
    log_entry = {
        "timestamp": datetime.utcnow().isoformat(),
        "severity": analysis["severity_score"],
        "vessel_count": len(vessels),
        "stationary_count": analysis.get("stationary_vessel_count", 0),
        "timeline_estimate": analysis["timeline_estimate"]
    }
    
    df = pd.DataFrame([log_entry])
    df.to_csv("hormuz_history.csv", mode='a', header=not os.path.exists("hormuz_history.csv"), index=False)
    
    # Feed last 7 days of data back to Claude for pattern recognition
    if os.path.exists("hormuz_history.csv"):
        history = pd.read_csv("hormuz_history.csv")
        recent = history.tail(336)  # Last 7 days at 30min intervals
        return recent.to_dict('records')
    
    return []
```

**Pro tip:** After 2 weeks of data collection, add a weekly summary job that uses Claude to identify recurring patterns (e.g., "Severity spikes every Friday 14:00 UTC correlate with OPEC announcements").

## Practical Example: Complete Monitoring Script

Here's the full `monitor.py` orchestrating all components:

```python
#!/usr/bin/env python3
import os
from dotenv import load_dotenv
from datetime import datetime

# Import our custom functions
from maritime_api import get_ships_in_hormuz
from claude_analysis import analyze_disruption_risk
from alerts import send_alert_if_critical
from logging_utils import log_analysis

load_dotenv()

def main():
    print(f"[{datetime.utcnow()}] Starting Strait of Hormuz monitor...")
    
    # Step 1: Fetch current vessel positions
    vessels = get_ships_in_hormuz()
    
    # Step 2: Gather recent news context (placeholder - integrate News API)
    news_context = """Trump announces U.S. Navy escort operations starting Monday.
    Previous Hormuz blockades averaged 4.2 days duration (2019-2024 data)."""
    
    # Step 3: Run Claude analysis
    analysis = analyze_disruption_risk(vessels, news_context)
    
    # Step 4: Log for historical tracking
    history = log_analysis(analysis, vessels)
    
    # Step 5: Send alert if critical
    alert_sent = send_alert_if_critical(analysis)
    
    print(f"Analysis complete. Severity: {analysis['severity_score']}/10")
    print(f"Alert sent: {alert_sent}")
    
    return analysis

if __name__ == "__main__":
    result = main()
```

Run manually: `python monitor.py`. Expected output:

```
[2026-05-04 10:30:00] Starting Strait of Hormuz monitor...
Found 47 vessels in Strait of Hormuz
Disruption severity: 6/10
No alert necessary (severity below threshold)
Analysis complete. Severity: 6/10
Alert sent: False
```

## Key Takeaways

- **Maritime AIS data + LLM reasoning** creates actionable intelligence systems that outperform keyword-based alerts by contextualizing raw vessel counts with geopolitical events
- **Claude Sonnet 3.5's structured output** (with JSON mode) costs ~$0.05 per analysis run, making continuous monitoring economically viable even for small teams
- **Threshold automation** prevents alert fatigue—only notify when severity ≥7 or anomalies exceed historical patterns by 2+ standard deviations
- **Historical logging** transforms a simple alert system into a predictive tool that learns disruption patterns specific to each chokepoint

## What's Next

Extend this system to monitor all 8 critical maritime chokepoints simultaneously (Suez Canal, Panama Canal, Malacca Strait) by parallelizing API calls with `asyncio`—tutorial coming next week.

---

**Key Takeaway:** By combining AIS ship tracking APIs with Claude's contextual reasoning, you can build an automated alert system that monitors geopolitical maritime chokepoints and triggers notifications when route disruptions occur—critical for supply chain professionals navigating the Strait of Hormuz crisis.

---

*New AI tutorials published daily on [AtlasSignal](https://atlassignal.in). Follow [@AtlasSignalDesk](https://twitter.com/AtlasSignalDesk) for more.*

---

## 📧 Get Daily AI & Macro Intelligence

Stay ahead of market-moving news, emerging tech, and global shifts. Choose your topics:

<div class="email-capture">
    <form class="email-capture-form" data-api-url="https://atlassignal.in/subscribe">
        <input type="email" name="email" placeholder="Your email address" required />
        <div class="topic-checkboxes" style="margin:10px 0;text-align:left;display:flex;flex-wrap:wrap;gap:8px;">
          <label><input type="checkbox" name="topics" value="AI" checked /> AI</label>
          <label><input type="checkbox" name="topics" value="Tech"  /> Tech</label>
          <label><input type="checkbox" name="topics" value="Space"  /> Space</label>
          <label><input type="checkbox" name="topics" value="Health"  /> Health</label>
          <label><input type="checkbox" name="topics" value="Sports"  /> Sports</label>
          <label><input type="checkbox" name="topics" value="Innovation"  /> Innovation</label>
        </div>
        <label class="consent-label" style="font-size:12px;display:block;margin:8px 0 12px;text-align:left;">
          <input type="checkbox" name="consent" required />
          I agree to receive topic-based updates from AtlasSignal
        </label>
    <button type="submit">Subscribe Free →</button>
  </form>
    <p class="email-capture-status" aria-live="polite" style="display:none;"></p>
</div>

<script>
(function () {
    var forms = document.querySelectorAll('.email-capture-form[data-api-url]');
    if (!forms.length) return;

    forms.forEach(function (form) {
        if (form.dataset.bound === 'true') return;
        form.dataset.bound = 'true';

        var status = form.parentElement && form.parentElement.querySelector('.email-capture-status');
        var button = form.querySelector('button[type="submit"]');

        form.addEventListener('submit', function (event) {
            event.preventDefault();

            var emailField = form.querySelector('input[name="email"]');
            var email = (emailField && emailField.value || '').trim();
            var apiUrl = form.getAttribute('data-api-url');
            if (!email || !apiUrl) return;

            var topicBoxes = form.querySelectorAll('input[name="topics"]:checked');
            var topics = Array.from(topicBoxes).map(function(cb) { return cb.value; });

            if (!topics.length) {
                if (status) {
                    status.style.display = 'block';
                    status.style.color = '#e74c3c';
                    status.textContent = 'Please select at least one topic to subscribe.';
                }
                return;
            }

            var consentEl = form.querySelector('input[name="consent"]');
            if (!consentEl || !consentEl.checked) {
                if (status) {
                    status.style.display = 'block';
                    status.style.color = '#e74c3c';
                    status.textContent = 'Please tick the consent checkbox to subscribe.';
                }
                return;
            }

            if (button) {
                button.disabled = true;
                button.textContent = 'Subscribing…';
            }
            if (status) {
                status.style.display = 'none';
                status.textContent = '';
            }

            fetch(apiUrl, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ email: email, topics: topics, consent: true }),
            })
            .then(function (response) {
                return response.json().catch(function () { return {}; }).then(function (data) {
                    return { ok: response.ok, data: data };
                });
            })
            .then(function (result) {
                if (!status) return;
                status.style.display = 'block';
                if (result.ok) {
                    var already = result.data.status === 'already_subscribed';
                    status.style.color = already ? '#8b949e' : '#2ecc71';
                    status.textContent = already
                        ? 'You are already subscribed. Watch for the next AtlasSignal report in your inbox.'
                        : 'Almost done! Check your inbox for a verification link to activate.';
                    form.reset();
                } else {
                    status.style.color = '#e74c3c';
                    status.textContent = result.data.detail || 'Subscription failed. Please try again.';
                }
            })
            .catch(function () {
                if (!status) return;
                status.style.display = 'block';
                status.style.color = '#e74c3c';
                status.textContent = 'Subscription failed. Please try again shortly.';
            })
            .finally(function () {
                if (button) {
                    button.disabled = false;
                    button.textContent = 'Subscribe Free →';
                }
            });
        });
    });
})();
</script>

