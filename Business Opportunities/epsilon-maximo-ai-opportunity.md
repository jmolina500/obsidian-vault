# Epsilon Maximo AI — Two-Tier Product Opportunity

**Owner:** Jesus Molina, Epsilon LLC (SDVOSB)
**Status:** Strategic concept — for filing under business opportunities
**Date filed:** May 2026

---

## The opportunity in one paragraph

Package Epsilon's existing Maximo MCP server into a two-tier product offering: (1) **BYO-subscription tier** where customers license the MCP and plug it into their own AI client (Claude Desktop, Copilot, etc.), and (2) **air-gapped appliance tier** where Epsilon ships a complete UI + MCP + local LLM runtime that operates entirely inside the customer's network with zero external dependencies. Tier 1 builds market presence and reference customers; Tier 2 is the federal-shaped, high-margin offering where Epsilon's SDVOSB status and Maximo domain expertise create a defensible moat.

---

## The two tiers

### Tier 1 — BYO Subscription

- **What ships:** Maximo MCP + small admin/config UI (which Maximo instance, auth, enabled endpoints)
- **What customer provides:** Their own AI client (Claude Desktop, Copilot, ChatGPT Enterprise, etc.) and the LLM subscription behind it
- **Price point:** ~$50–500/user/month or $5K–25K/year per Maximo environment
- **Sales cycle:** Weeks
- **Decision-maker:** Maximo admin or IT director
- **Purpose:** Volume play, market presence, evidence base for Tier 2 pitches
- **UI investment:** Minimal — chat UI is whatever the customer's AI client provides

### Tier 2 — Air-Gapped Appliance

- **What ships:** Purpose-built Maximo UI + Maximo MCP + local LLM runtime (Ollama / vLLM / llama.cpp) + offline licensing — all bundled, all signed, fully offline-capable
- **What customer provides:** Compatible hardware (workstation/small server with sufficient VRAM)
- **Price point:** $50K–250K+ per deployment, plus implementation services, plus annual support
- **Sales cycle:** Months to a year
- **Decision-maker:** CISO + Maximo executive sponsor + procurement
- **Purpose:** High-value, defensible federal/regulated-commercial offering
- **UI investment:** Significant — this is the product surface

---

## Why this positioning fits Maximo customers

- **Air-gap-curious buyer base.** Utilities, defense, federal facilities, pharma/GMP, oil & gas, transit, large municipalities. A meaningful subset *cannot* send work-order, asset-criticality, or maintenance-procedure data to cloud LLMs — compliance, ITAR, CUI, NERC-CIP, GMP, state secrets.
- **SDVOSB advantage.** Epsilon already has the federal contracting vehicle. Cloud-SaaS Maximo AI competitors literally cannot bid on the contracts Epsilon can.
- **Natural land-and-expand.** Tier 1 customers who like the tool and grow nervous about data residency upgrade to Tier 2. Federal pilots in sandbox → production air-gap.
- **Right lane for Epsilon.** Not competing with OpenAI/Anthropic — enabling them (Tier 1) or replacing them with local models (Tier 2). Epsilon stays the Maximo specialist, not the AI vendor.

---

## What Tier 2 actually requires (engineering honesty)

| Requirement | Detail |
|---|---|
| **Local model runtime** | Ollama / vLLM / llama.cpp; certified against a specific model + quantization (e.g., Llama 3.3 70B Q4 or Qwen 2.5 32B) |
| **Hardware spec** | 24GB VRAM minimum (small models); 48GB+ ideal (70B-class) |
| **UI that doesn't phone home** | No telemetry, no analytics, no CDN fonts, no Sentry. Every dependency auditable for outbound calls. Federal customers will run Wireshark on it. |
| **Bundled offline operation** | No npm install, Docker Hub pulls, or model downloads at runtime. Everything pre-bundled. |
| **Offline licensing** | Keygen.sh offline mode (already in Epsilon's stack) |
| **Update delivery model** | Signed bundle distribution: USB, customer-controlled, or sneakernet-friendly |
| **Tool-calling validation** | Local model must reliably call MCP tools and generate correct OSLC queries. Testable in a weekend against the existing Maximo OSLC mock server. |

---

## Moat assessment

The real defensibility is **packaging + brand**, not secret IP:

- A competitor could write a Maximo MCP in ~3 weeks
- The moat is shipping a *complete, supported, air-gapped Maximo AI appliance* — not a library
- Combined with Epsilon's existing Maximo credibility and SDVOSB contracting vehicle
- This is a "we ship a polished product backed by domain experts" moat, not a "we have proprietary algorithms" moat — but it is a real moat

---

## UI strategy for Tier 2

Three viable bases, in increasing order of investment:

1. **Themed LibreChat or OpenWebUI** — locked down, no-phone-home configuration, Ollama backend native. Working demo in days. Feels like a chat UI, not a Maximo product.
2. **Hybrid: LibreChat backend + custom Maximo workflow pages** — chat for free-form queries, dedicated views for work orders, asset trees, criticality dashboards, PM analysis. Recommended starting point.
3. **Fully purpose-built Next.js Maximo workflow UI** — long-term destination. Lets the product feel like an EAM tool, not a chat app.

Recommended path: ship themed LibreChat for first 1–2 pilots while building the purpose-built UI in parallel, then swap.

**Explicitly not a fit:** Forking general-purpose personal-agent platforms (e.g., thepopebot). Network-dependent infrastructure, agent-job + PR workflows, and code-workspace features all make air-gap deployment harder, not easier. Strip-down cost exceeds greenfield build.

---

## Strategic risks to acknowledge

- **Federal procurement is slow.** 12–18 months from first conversation to PO is normal for software products. Bridge revenue via Tier 1 and existing services.
- **Selling a product is different from selling services.** Different procurement, support model, margin math. Epsilon has run services engagements; this is new muscle.
- **Local model tool-calling quality is the technical gating risk.** If the certified open-weight model can't reliably call the MCP and produce correct OSLC, Tier 2 doesn't exist yet. Must validate before committing to the build.
- **Design partner risk.** Without a friendly federal/regulated-commercial customer designing alongside, the appliance gets built against imagined requirements and rejected by real CISOs for things not anticipated.

---

## Recommended sequence

1. **Weekend test (highest priority).** Run a local model (Qwen 2.5 32B or Llama 3.3 70B via Ollama) against the Maximo OSLC mock server using the existing MCP. Validate tool calling works and OSLC queries come out correct. This is the go/no-go gate for Tier 2.
2. **Next two weeks.** Define Tier 1 crisply: MCP + tiny admin UI, priced, listed, fulfilled via existing Lemon Squeezy + Keygen pipeline. Land one paying customer.
3. **Concurrent.** Identify one federal or security-conscious commercial design partner for Tier 2. Not a sale yet — a design partnership where their security requirements become the specification.
4. **Months 2–4.** Build Tier 2 against design partner's actual requirements: themed LibreChat + MCP + Ollama-served local model + air-gapped install bundle. Add custom Maximo workflow views as the differentiation.
5. **Month 6+.** Productize. Reference deployment for SDVOSB federal pitches.

---

## Why this matters strategically

- Leverages existing Maximo MCP work (already in progress with OSLC mock server)
- Leverages Epsilon's existing SDVOSB contracting position
- Creates a productized revenue line distinct from billable consulting hours
- Air-gapped tier has near-zero direct competition in the Maximo segment
- Tier 1 is low-risk market validation; Tier 2 is the strategic prize
- Both tiers share the same MCP artifact — engineering investment is leveraged