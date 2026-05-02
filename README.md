# Customer Cohort Analytics

**Status:** Candidate Project  
**Market Size:** $14–17B (product analytics) + $2–3B (customer success platforms) at 15–22% CAGR  
**Last Updated:** 2026-05-02

## Overview

An open-source, AI-native customer cohort analytics platform combining behavioral cohort analysis with integrated ML-based churn prediction and explainability. Today's market splits into two:

1. **Product analytics tools** (PostHog, Amplitude, Mixpanel): Great cohort tracking but no ML prediction
2. **Customer success platforms** (Gainsight, ChurnZero): ML predictions but $60K–$140K/yr and vendor lock-in

This project bridges the gap with:

- **Behavioral cohort builder + ML churn prediction** in one platform
- **Automated cohort discovery:** AI surfaces statistically significant segments correlated with retention—no manual hypothesis required
- **XAI-powered churn scoring:** SHAP-explained feature importance per user in plain English ("this user is at risk because login frequency dropped 60%")
- **Proactive retention narratives:** Weekly summaries pushed to Slack when a cohort's health diverges from baseline
- **Open-source + affordable:** Free self-hosted option; managed SaaS tier for teams wanting support

## The Market Gap

The churn reduction opportunity is massive: **reducing churn by 5% can increase profits by 25–95%.** Yet the market is fragmented:

- **Product analytics (PostHog, Amplitude, Mixpanel):** $0–$5K/month; excellent cohort tracking but no predictive ML
- **Customer success (Gainsight, ChurnZero, Totango):** $60K–$140K/year TCO; ML predictions but enterprise pricing, B2B-centric, high implementation overhead
- **Predictive ML (Pecan AI):** $5K–$50K/month; strong churn models but no cohort visualization, batch-only, requires data warehouse

**The gap:** No open-source tool combines behavioral cohort analysis with integrated, explainable ML-based churn prediction and natural-language interface. Teams want:
- Free entry point (eliminates $60K/yr minimum contracts)
- Cohort analysis + churn prediction in one place (not two separate tools)
- Plain-English explanations of why a customer is at risk
- Self-hosting for data sovereignty

## Core Features

### MVP (Must-Have)
- Behavioral cohort builder with GUI: define cohorts by event sequences, properties, and time windows without requiring SQL
- Day-N retention curves and funnel analysis filterable by any user cohort
- JavaScript (web) and mobile (iOS/Android) SDK for event collection using Segment `.track()` / `.identify()` event schema
- **ML-based churn risk scoring:** Per-user probability score trained on customer's own behavioral data, updated on rolling basis
- **XAI feature attribution per prediction:** Surface top 3–5 behavioral drivers for each at-risk user in plain English
- Dashboard creation and Slack/email delivery for scheduled cohort health reports

### Should-Have (v1.1)
- **Automated cohort discovery:** AI surfaces statistically significant behavioral segments correlated with retention or churn without analysts defining hypothesis first
- **Natural-language cohort definition:** Conversational interface for describing cohort filters ("users who invited a teammate in their first week but haven't logged in for 21 days")
- **Account-level (B2B) aggregation:** Roll up individual user behavioral signals to company account health score for B2B SaaS use cases
- **Intervention recommendation engine:** Given a flagged at-risk cohort, suggest which intervention type has historically been most effective for similar cohorts

### Nice-to-Have (Backlog)
- **Proactive cohort health summaries:** Weekly LLM-generated narratives pushed to Slack/email when cohort's retention trajectory diverges from baseline
- Session replay integration: link funnel drop-off events to session recordings for qualitative investigation
- Survival analysis with continuous model updating: replace static churn scoring with survival model that updates as new behavioral data arrives
- GDPR/CCPA self-service controls: built-in right-to-erasure request processing and consent audit trail for regulated deployments

## AI-Native Opportunities

1. **Automated cohort discovery instead of manual definition**
   - Today, analysts manually define cohorts ("users who signed up in Q1, used Feature X within 7 days, and converted to paid")
   - An AI-native platform can automatically surface statistically significant behavioral cohorts that correlate with retention
   - Example: "users who invited a teammate in their first session retain at 3x the rate"—no analyst hypothesis required

2. **Natural-language cohort querying**
   - Current cohort builders require multi-step UI workflows
   - An LLM interface allows analysts to define cohorts conversationally: "show me users who upgraded from free in the last 90 days but haven't logged in for 30 days, broken down by acquisition channel"
   - Reduces time to insight from hours to seconds; democratizes access to non-SQL-literate team members

3. **Integrated churn prediction with explainability**
   - Existing tools (Gainsight, ChurnZero) provide health scores based on heuristic rules; Pecan and Amplitude provide ML but as black boxes
   - An AI-native open-source platform could offer XAI-powered churn prediction (SHAP-explained feature importance)
   - Automatically updating survival models as customer behavior evolves—giving CS teams actionable explanations ("account is at risk because support tickets are up 40% and product logins are down 60% vs. their cohort average")

4. **Proactive retention narrative generation**
   - CS teams must currently monitor dashboards manually
   - An AI agent could push weekly cohort health summaries, flag when a previously healthy cohort shows early warning signs, and suggest specific intervention playbooks based on what worked for similar at-risk cohorts historically

5. **Open-source differentiation**
   - PostHog ($0–$5K/mo) offers cohort tracking but no ML prediction
   - Gainsight/ChurnZero ($60K–$140K/yr) offer ML but at enterprise pricing with proprietary lock-in
   - A free, self-hostable, AI-native platform with integrated churn prediction and NL querying would serve the underserved middle: growth-stage startups and mid-market SaaS

## Competitive Landscape

| Tool | Type | Cohorts | ML Churn | XAI | NL Query | Cost | OSS |
|------|------|---------|----------|-----|----------|------|-----|
| **PostHog** | OSS + SaaS | ✓ | ❌ | ❌ | ❌ | Free self-hosted | ✓ (MIT) |
| **Amplitude** | Commercial | ✓ | ✓ | ❌ | ✓ (AI copilot) | $995+/mo | ❌ |
| **Mixpanel** | Commercial | ✓ | ❌ | ❌ | ✓ (Spark, 2025) | $336–$5K+/yr | ❌ |
| **Gainsight** | Commercial | ✓ | ✓ | ❌ | ❌ | $90K–$140K/yr | ❌ |
| **ChurnZero** | Commercial | ✓ | ✓ | ❌ | ❌ | $60K–$99K/yr | ❌ |
| **Pecan AI** | Commercial | ❌ | ✓ | ✓ (aggregate-level) | ❌ | Quote-based | ❌ |
| **This Project** | OSS + SaaS | ✓ | ✓ | ✓ (per-user) | ✓ | Free self-hosted | ✓ (MIT) |

## Technical Design Considerations

- **Event ingestion:** Segment-compatible SDK (`.track()`, `.identify()`, `.page()`); alternative RudderStack / mParticle connectors
- **Cohort engine:** SQL-based query builder translating UI/NL definitions into efficient queries against event warehouse
- **ML pipeline:** scikit-learn or XGBoost for churn modeling; SHAP for explainability; automated feature engineering (rolling windows, lag features)
- **Feature store:** Optional integration with Databricks Feature Store or Tecton for feature management at scale
- **Real-time updates:** Event streaming via Kafka for near-real-time cohort and churn score updates (optional)
- **NL layer:** Claude API for cohort definition interpretation; survival model explanation generation
- **Deployment:** Docker Compose for self-hosted; managed SaaS with team collaboration (similar to PostHog Cloud model)

## Market Validation

- **Market size:** SaaS companies improving retention rates by 15% on average vs. those doing no cohort analysis
- **Buyer pain:** Correct cohort pattern identification reduces unnecessary retention spending by 64% while improving intervention effectiveness by 2.3x
- **Benchmark:** Median SaaS NRR is 101% (barely flat); top performers (churn < 5%/mo) achieve 111%+ NRR

- **Customer personas:**
  - Product managers at B2C / B2B2C SaaS tracking feature adoption, onboarding funnel drop-off
  - Customer success ops teams at B2B SaaS managing renewal risk and at-risk account identification
  - Growth and marketing teams running acquisition channel cohort comparisons
  - CFOs tracking NRR and LTV:CAC by cohort as core financial metrics

## Why Build This

1. **Market timing:** Churn reduction ROI is proven (5% reduction → 25–95% profit increase); demand is high
2. **Vendor gap:** No open-source tool offers cohort analysis + ML prediction + explainability together
3. **Technology maturity:** Survival models and SHAP-based ML explainability are well-established; open-source implementations exist
4. **Commercial opportunity:** Build on free tier; offer managed SaaS for teams wanting support and integrations
5. **Platform leverage:** Use existing SDKs (Segment API) and ML libraries (XGBoost, SHAP); focus on UI and integration

## Success Metrics

- **Adoption:** 500+ GitHub stars within 12 months; featured as PostHog + Amplitude + Gainsight alternative
- **Commercial:** Win 20+ mid-market customers with managed SaaS tier ($2K–$10K/mo)
- **Churn improvement:** Demonstrate 10%+ churn reduction for pilot customers (case studies)
- **Community:** Active issue triage; 3+ contributors beyond core team

---

**Resources:**
- [Amplitude Churn Prediction Documentation](https://amplitude.com/blog/churn-prediction)
- [SHAP (SHapley Additive exPlanations)](https://github.com/slundberg/shap)
- [PostHog Cohorts Documentation](https://posthog.com/docs/features/cohorts)
- [Segment Spec (Event Schema Standard)](https://segment.com/docs/connections/spec/)
- [Survival Analysis Overview](https://lifelines.readthedocs.io/)
