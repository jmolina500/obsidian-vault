# GovOps Agent
## Government RFP Monitoring for Maximo/Asset Management Opportunities

**Concept:** A specialized Hermes Agent that monitors government contracting opportunities (SAM.gov, GSA eBuy) for IBM Maximo and Enterprise Asset Management (EAM) related contracts.  
**Status:** Planning Phase  
**Created:** 2026-05-09  
**Delivery:** Email alerts + Discord/Google Chat notifications

---

## Executive Summary

Government agencies spend billions annually on asset management systems, CMMS implementations, and maintenance services. Finding the right RFPs requires constant monitoring of multiple platforms. This agent automates that discovery process, filtering for high-relevance opportunities in the Maximo/EAM space and delivering curated alerts.

**Target Opportunity Value:** Federal agencies alone spend $600B+ annually on contracts. Maximo/EAM consulting contracts typically range from $100K to $50M+.

---

## Data Sources

### Primary: SAM.gov (System for Award Management)
- **API:** https://sam.gov/api/prod/opportunities/v1/search
- **Coverage:** All federal contracts > $25K
- **Access:** Free API key required
- **Update Frequency:** Real-time as agencies post
- **Relevant NAICS Codes:**
  - 541511 - Custom Computer Programming Services
  - 541512 - Computer Systems Design Services
  - 541519 - Other Computer Related Services
  - 334118 - Computer Terminal Manufacturing
  - 541330 - Engineering Services
  - 561210 - Facilities Support Services
  - 541611 - Administrative Management Consulting

### Secondary: GSA eBuy
- **What:** GSA's RFQ platform for pre-approved vendors
- **API:** Limited, may require scraping or GSA API access
- **Coverage:** Federal agencies using GSA Schedules
- **Focus:** Task orders under GSA Schedule 70 (IT), 874 (Facilities)
- **Access:** Requires GSA Schedule holder status for full access

### Tertiary: State & Local (Future)
- **Examples:**
  - Puerto Rico PR.gov procurement portal
  - State of Florida MyFloridaMarketPlace
  - California Cal eProcure
  - Texas SmartBuy
- **Approach:** State-specific scrapers or API integrations

---

## Technical Architecture

### Option A: CLI Tool + Cron (MVP)

```
┌─────────────────────────────────────────────────────────────┐
│                    GOVOPS AGENT                             │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │ SAM.gov API  │    │  eBuy Parser │    │  AI Filter   │  │
│  │   Client     │───▶│   (Scrape)   │───▶│   Engine     │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                                        │          │
│         └────────────────────────────────────────┘          │
│                            │                                │
│                   ┌────────▼────────┐                       │
│                   │  Opportunity DB   │                       │
│                   │   (SQLite/JSON)   │                       │
│                   └────────┬────────┘                       │
│                            │                                │
│         ┌──────────────────┼──────────────────┐            │
│         ▼                  ▼                  ▼            │
│  ┌────────────┐   ┌──────────────┐   ┌──────────────┐     │
│  │   Email    │   │   Discord    │   │ Google Chat  │     │
│  │  (SMTP)    │   │  (Webhook)   │   │  (Webhook)   │     │
│  └────────────┘   └──────────────┘   └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

**CLI Commands:**
```bash
# Search once
govops search --source=sam.gov --naics=541511 --keywords="maximo,asset management,cmms"

# Configure alerts
govops config set alert.email=you@company.com
govops config set alert.discord_webhook=https://discord.com/api/webhooks/...
govops config set filter.keywords="maximo,eam,asset management,ibm tririga,cmms"
govops config set filter.min_value=100000

# Run monitoring (for cron)
govops monitor --since=yesterday

# Generate report
govops report --days=7 --format=html
```

### Option B: MCP Server (Agent-Native)

Expose as Hermes MCP tools:

```yaml
# tools/search_opportunities
{
  "source": "sam.gov",
  "keywords": ["maximo", "asset management"],
  "naics_codes": ["541511", "541512"],
  "posted_since": "2026-05-01",
  "min_value": 100000
}

# tools/get_opportunity_details
{
  "opportunity_id": "ABC-123-DEF",
  "include_attachments": true
}

# tools/analyze_fit
{
  "opportunity_id": "ABC-123-DEF",
  "company_capabilities": "IBM Maximo implementation, asset management consulting, CMMS migration"
}
```

**Recommended:** Start with Option A (CLI) for immediate utility, add Option B (MCP) for AI agent integration later.

---

## Core Features

### 1. Multi-Source Aggregation
- SAM.gov API integration
- GSA eBuy parsing (HTML scraping if needed)
- State/local procurement portals (future)
- Unified opportunity format

### 2. Intelligent Filtering

**Keyword Matching:**
```python
HIGH_PRIORITY_KEYWORDS = [
    "ibm maximo",
    "maximo implementation", 
    "asset management system",
    "cmms implementation",
    "enterprise asset management",
    "eam consulting",
    "maximo upgrade",
    "maximo migration",
    "ibm tririga",
    "facilities management system"
]

MEDIUM_PRIORITY_KEYWORDS = [
    "asset management",
    "maintenance management",
    "facilities management",
    "infrastructure management",
    "work order system",
    "preventive maintenance"
]
```

**AI Scoring:**
- Relevance score (0-100) based on description analysis
- Win probability estimate
- Effort estimation (small/medium/large)
- Deadline proximity alert

### 3. Alert System

**Email Alerts:**
- Daily digest of new opportunities
- Immediate alerts for high-value (>$1M) matches
- Weekly summary with stats
- Full opportunity details with links

**Discord/Google Chat:**
- Real-time webhook notifications
- Rich embeds with key details
- Action buttons (View in SAM.gov, Save, Ignore)

### 4. Opportunity Database

```sql
-- SQLite schema
CREATE TABLE opportunities (
    id TEXT PRIMARY KEY,
    source TEXT,  -- sam.gov, ebuy, state_portal
    title TEXT,
    agency TEXT,
    naics_code TEXT,
    value_min INTEGER,
    value_max INTEGER,
    posted_date DATE,
    response_deadline DATE,
    description TEXT,
    url TEXT,
    keywords_matched TEXT,  -- JSON array
    relevance_score INTEGER,
    status TEXT,  -- new, viewed, interested, submitted, won, lost, ignored
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

### 5. Analysis & Reporting

- Pipeline view (opportunities by stage)
- Agency breakdown (which agencies post most Maximo work)
- Historical pricing (average contract values)
- Win/loss tracking
- Competitor analysis (who's winning these contracts)

---

## Implementation Phases

### Phase 1: MVP (2-3 weeks)
**Goal:** Basic SAM.gov monitoring with email alerts

**Deliverables:**
- [ ] SAM.gov API client
- [ ] Keyword filtering engine
- [ ] SQLite opportunity store
- [ ] Email alert sender
- [ ] Daily cron job setup
- [ ] Simple CLI interface

**Success Criteria:**
- Captures 90%+ of Maximo-related SAM.gov opportunities
- Email alerts deliver within 1 hour of posting
- False positive rate < 20%

### Phase 2: Intelligence (3-4 weeks)
**Goal:** AI-powered relevance scoring and multi-channel delivery

**Deliverables:**
- [ ] Claude AI integration for opportunity analysis
- [ ] Relevance scoring (0-100)
- [ ] Win probability estimation
- [ ] Discord webhook integration
- [ ] Google Chat webhook integration
- [ ] Rich opportunity summaries

**Success Criteria:**
- AI correctly identifies Maximo/EAM opportunities 95%+ of time
- Users can configure alert preferences
- Multi-channel delivery working

### Phase 3: Scale (4-6 weeks)
**Goal:** GSA eBuy, state/local, and analytics

**Deliverables:**
- [ ] GSA eBuy scraper/parser
- [ ] State procurement portal integrations (PR, FL, TX, CA)
- [ ] Web dashboard (optional)
- [ ] Analytics and reporting
- [ ] Competitor tracking
- [ ] API for external integrations

**Success Criteria:**
- 3+ data sources integrated
- Historical analysis available
- Export to CRM/bidding tools

### Phase 4: Automation (Future)
**Goal:** Proposal assistance (beyond research)

**Potential Features:**
- [ ] Auto-download RFP attachments
- [ ] Extract key requirements
- [ ] Draft proposal outlines
- [ ] Compliance checklist generation
- [ ] Past performance matching

---

## Technical Stack

| Component | Technology |
|-----------|-----------|
| **CLI Framework** | Python (Typer) |
| **API Client** | httpx + tenacity (retries) |
| **Database** | SQLite (local) / PostgreSQL (shared) |
| **AI Analysis** | Claude API or local LLM |
| **Email** | SMTP (SendGrid, AWS SES, or direct) |
| **Webhooks** | Discord.py or simple HTTP |
| **Scheduling** | Cron (Linux) / Task Scheduler (Windows) |
| **Config** | YAML or TOML |
| **Logging** | structlog |

---

## Configuration Example

```yaml
# govops.yaml
sources:
  sam_gov:
    enabled: true
    api_key: ${SAM_GOV_API_KEY}
    poll_interval: 3600  # seconds
    
  gsa_ebuy:
    enabled: false  # Phase 2
    username: ${GSA_USERNAME}
    password: ${GSA_PASSWORD}
    
  state_pr:
    enabled: false  # Phase 3
    url: https://www.pr.gov/procurement

filtering:
  keywords:
    high_priority:
      - "ibm maximo"
      - "maximo implementation"
      - "asset management system"
      - "cmms implementation"
    medium_priority:
      - "asset management"
      - "facilities management"
      
  naics_codes:
    - "541511"
    - "541512"
    - "541519"
    
  min_contract_value: 100000  # $100K
  max_contract_value: 50000000  # $50M
  
  agencies_include:  # empty = all
    - "Department of Defense"
    - "Department of Energy"
    - "GSA"
    
  agencies_exclude:
    - "Department of Agriculture"  # unless doing facilities

alerting:
  email:
    enabled: true
    smtp_host: smtp.gmail.com
    smtp_port: 587
    username: ${EMAIL_USER}
    password: ${EMAIL_PASS}
    to: 
      - you@company.com
      - partner@company.com
    digest_schedule: "0 8 * * *"  # Daily at 8am
    immediate_threshold: 1000000  # Alert immediately if >$1M
    
  discord:
    enabled: true
    webhook_url: ${DISCORD_WEBHOOK_URL}
    channel: "#gov-opportunities"
    
  google_chat:
    enabled: false
    webhook_url: ${GOOGLE_CHAT_WEBHOOK_URL}

scoring:
  min_relevance_score: 60  # Only alert if score >= 60
  ai_analysis_enabled: true
  
storage:
  db_path: ~/.govops/opportunities.db
  retention_days: 365
```

---

## Business Model Options

### Option 1: Internal Tool (Free)
- Build for your own use
- Capture Maximo government contracts before competitors
- Estimate value: $500K-5M+ in additional contract wins

### Option 2: Consulting Firm License
- Sell to Maximo implementation partners
- $500-2000/month per firm
- 10 firms = $60K-240K/year

### Option 3: Per-User SaaS
- Web dashboard + alerts
- $99-499/month per user
- Target: BD/sales people at government contractors

### Option 4: Pay-Per-Lead
- Charge only for qualified opportunities delivered
- $50-200 per high-quality lead
- Performance-based, lower friction

---

## Competitive Analysis

| Competitor | Price | Coverage | Maximo Focus | AI | Notes |
|------------|-------|----------|--------------|-----|-------|
| **GovWin IQ** | $$$$ | Fed/State/Local | No | Limited | Expensive, generalist |
| **SAM.gov Alerts** | Free | Fed only | No | No | Manual, noisy |
| **FedBizOpps** | Free | Fed only | No | No | Deprecated, moved to SAM |
| **Onvia** | $$$ | Fed/State/Local | No | No | General government contracts |
| **Deltek GovWin** | $$$$ | Full | No | Some | Enterprise sales tool |
| **BidNet** | $$ | State/Local | No | No | Focus on construction |
| **Our Solution** | $-$$ | Fed (→State) | **Yes** | **Yes** | **Niche focus + AI** |

**Differentiation:**
- Only tool specifically tuned for Maximo/EAM opportunities
- AI-powered relevance (not just keyword matching)
- Multi-channel delivery (email + chat)
- Open/transparent (can self-host)
- Affordable vs enterprise tools

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| SAM.gov API changes | Medium | High | Abstract API layer, version handling |
| Too many false positives | Medium | Medium | AI scoring, user feedback loop |
| Missing opportunities | Low | High | Multiple keyword strategies, manual audit |
| GSA eBuy blocks scraping | Medium | Medium | Use official APIs if available, rate limiting |
| Competition from GovWin | Low | Medium | Niche focus, price advantage |
| Data retention/compliance | Low | Medium | Clear data policies, opt-in only |

---

## Success Metrics

**Technical:**
- Uptime: 99.5%+
- Opportunity capture rate: 95%+
- Alert delivery latency: <1 hour
- False positive rate: <20%

**Business:**
- Opportunities captured per month
- Contract value of captured opportunities
- Win rate on alerted opportunities
- Time saved vs manual monitoring

**User:**
- Daily/weekly active users
- Alert open rate
- Opportunity click-through rate
- User satisfaction (survey)

---

## Next Steps

### Immediate (This Week)
1. [ ] Get SAM.gov API key (free at https://sam.gov/api)
2. [ ] Review reference repo: https://github.com/mvanhorn/cli-printing-press.git
3. [ ] Validate SAM.gov has sufficient Maximo opportunities
4. [ ] Decide: CLI-first or MCP-first architecture

### Short Term (Next 2 Weeks)
5. [ ] Build SAM.gov API client prototype
6. [ ] Create opportunity data model
7. [ ] Implement basic keyword filtering
8. [ ] Set up SQLite storage
9. [ ] Build email alert sender

### Medium Term (Next Month)
10. [ ] Integrate Claude AI for relevance scoring
11. [ ] Add Discord webhook integration
12. [ ] Deploy to VPS for 24/7 monitoring
13. [ ] Test with real opportunities for 2 weeks
14. [ ] Iterate based on false positive/negative rate

---

## Related Resources

- SAM.gov API Docs: https://open.gsa.gov/api/sam-opp-api/
- GSA eBuy: https://www.gsa.gov/buy-through-us/purchasing-programs/gsa-ebuy
- NAICS Code Lookup: https://www.naics.com/search/
- Reference CLI Tool: https://github.com/mvanhorn/cli-printing-press.git
- Maximo Government Contracts: https://sam.gov/search/?index=opp&keywords=maximo

---

## Open Questions

1. Should we focus on federal only initially, or include state/local from day 1?
2. Do you have existing SAM.gov or GSA eBuy access/credentials?
3. What's the minimum contract value worth alerting on? ($50K? $100K? $500K?)
4. Preferred hosting: Your VPS, or cloud service (AWS/GCP)?
5. Who else should receive alerts? (Partners, sales team)

---

*This is a high-value niche with clear differentiation. The technical risk is low (well-documented APIs), and the potential ROI is significant given typical Maximo contract sizes.*
