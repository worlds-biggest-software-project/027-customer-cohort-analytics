# Customer Cohort Analytics — Feature & Functionality Survey

> Candidate #27 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Amplitude | Commercial SaaS | Proprietary / Starter free; Growth $995/month; Enterprise custom | https://amplitude.com |
| Mixpanel | Commercial SaaS | Proprietary / Free up to 20M events/month; Growth from ~$28/month | https://mixpanel.com |
| PostHog | Open source + commercial SaaS | MIT (OSS) / commercial cloud | https://posthog.com |
| Heap (Contentsquare) | Commercial SaaS | Proprietary / enterprise quote | https://heap.io |
| Gainsight | Commercial SaaS | Proprietary / ~$90K–$140K/yr TCO | https://www.gainsight.com |
| ChurnZero | Commercial SaaS | Proprietary / ~$60K–$99K/yr TCO | https://churnzero.com |
| Totango | Commercial SaaS | Proprietary / ~$50K/yr + 20% setup | https://www.totango.com |
| Pecan AI | Commercial SaaS | Proprietary / quote-based mid-market | https://www.pecan.ai |
| Kissmetrics | Commercial SaaS | Proprietary / from $299/month | https://www.kissmetrics.io |
| Countly | Open source + commercial SaaS | AGPL-3.0 (community) / commercial enterprise | https://countly.com |

---

## Feature Analysis by Solution

### Amplitude

**Core features**
- Behavioral cohort builder: defines user groups based on any combination of event sequences, properties, and time windows
- Predictive Cohorts (Amplitude Predictions): transformer-based ML model using hundreds of behavioral signals to assign probabilistic scores for user likelihood to retain, convert, or churn within 7, 30, 60, or 90-day windows
- Predictive models auto-populate dynamic cohorts that update as customer behavior evolves; faster than batch-processing systems
- Automated discovery engine: scans datasets to surface correlation scores, identifying which specific behaviors correlate with retention (e.g., "users who invited a teammate in their first session retain at 3x the rate")
- Funnel analysis, retention analysis (day-N cohort curves), path analysis, and session replay
- Conversational AI co-pilot (2025): non-technical analysts query data in plain English using NLP
- Pre-built integrations with Braze, Iterable, Facebook Ads, Google Ads, LaunchDarkly, Appcues, and Amazon S3 for activation of predictive cohort lists

**Differentiating features**
- The most mature ML-based predictive cohort product in the market: transformer model over hundreds of signals versus the 3–5 manually configured signals used in traditional rule-based cohorts
- Amplitude AI's automated discovery of non-obvious retention drivers surfaces insights analysts did not know to look for
- Amplitude Engage: connects predictive cohort lists directly to marketing/activation tools, closing the loop between prediction and intervention

**UX patterns**
- Chart-type workflow: users select an analysis type (funnel, retention, cohort, path), add event filters, then optionally break down by cohort
- Cohort builder is a multi-step filter UI: event + property + time window logic
- NL co-pilot accessible as a side panel for less-technical team members
- Dashboard sharing and scheduled digest emails for stakeholder distribution

**Integration points**
- SDKs for web, iOS, Android, React Native, Flutter, and server-side event tracking
- Segment, mParticle, RudderStack as upstream CDPs
- Pre-built destination integrations: Braze, Iterable, HubSpot, Salesforce, Facebook Ads, Google Ads
- Data warehouse export: BigQuery, Snowflake, Redshift, S3
- Amplitude REST API for programmatic cohort creation and export

**Known gaps**
- Free tier limited to 50K monthly tracked users (MTUs); growth-stage B2C products hit the limit quickly
- Predictive Cohorts and Predictions features are in the Growth/Enterprise tier ($995/month minimum); not accessible to early-stage startups
- Churn predictions are probabilistic scores, not SHAP-explained feature importance; the model is not transparent about why a specific user is flagged
- No self-hosted or open-source option; all data must be sent to Amplitude's cloud
- Strong product analytics, but less suited to B2B SaaS account-level (company) health scoring than Gainsight

**Licence / IP notes**
- Fully proprietary SaaS; Amplitude went public in 2021 (AMPL). No open-source components. No patent concerns identified in public sources, though transformer-based ML prediction techniques are subject to broad industry patenting.

---

### Mixpanel

**Core features**
- Event-based behavioral cohort builder: groups users by any sequence of events and properties; cohorts available across all chart types
- Retention analysis: configurable day-N retention tables and charts (D1, D7, D30, custom intervals)
- Funnel analysis with conversion rates, time-to-convert, and cohort-based funnel comparisons
- Anomaly Detection and Root Cause Analysis: AI-powered flagging of metric changes with automated drill-down to contributing segments
- Spark (AI query builder, 2025): NL query interface for constructing analytics without using the filter UI
- AI Summaries on session replays; AI-powered Magic Playlists for session replay curation
- MCP Server (2025): connects Mixpanel directly to Claude, ChatGPT, and Cursor for conversational analytics from an LLM interface
- Metric Trees: causal metric decomposition for understanding which upstream metrics drive a KPI change (shipped June 2025)
- Pure event-based pricing (February 2025): simplified pricing model by event volume, removing per-user seat limits

**Differentiating features**
- Anomaly Detection with Root Cause Analysis is the most developed AI-assisted triage feature in the product analytics category as of 2025: flags drops ("signup conversion dropped 18% yesterday") and automatically surfaces the contributing segments
- MCP Server integration means Mixpanel data can be queried conversationally via Claude or ChatGPT without building a custom integration
- Event-based pricing with a generous free tier (20M events/month) makes it accessible to high-volume B2C products at a lower cost than Amplitude

**UX patterns**
- Chart type selector is the primary navigation: funnels, retention, flows, insights (trends), user profiles
- Cohort builder available as an inline filter across all chart types
- AI anomaly alerts surfaced in the monitoring feed and via email/Slack

**Integration points**
- SDKs: web JavaScript, iOS, Android, React Native, Flutter, Python, Ruby, Java, Go
- Segment, mParticle, RudderStack CDP integration
- Data warehouse export: BigQuery, Snowflake, Redshift
- Webhook and REST API for cohort export and event streaming
- MCP Server for Claude/ChatGPT/Cursor LLM integration

**Known gaps**
- No native ML-based churn prediction; cohort analysis is behavioral/rule-based only
- Anomaly Detection does not provide a predictive risk score; it flags what has already changed, not what is likely to change
- No built-in customer success workflow (playbooks, health scores); purpose-built for product analytics teams, not CS ops
- Complex multi-touch attribution models are limited compared to dedicated attribution tools
- No self-hosted option; cloud-only

**Licence / IP notes**
- Fully proprietary SaaS. No open-source components. No patent concerns identified in public sources.

---

### PostHog

**Core features**
- Open-source all-in-one product analytics: funnels, retention analysis, trends, paths, user cohorts, session replay, feature flags, A/B testing, error tracking, surveys, and CDP in one platform
- Dynamic user cohorts: group users by any behavioral property or event sequence; cohorts available across all analysis types
- Retention and stickiness charts with configurable time windows; cohort-by-cohort retention curve comparison
- SQL-level direct access for advanced queries: analysts can write SQL against the PostHog event data model
- Session replay linked to cohort and funnel events: click through from a funnel drop-off to watch the session replays of users who dropped off at that exact step
- Free cloud tier: 1M events and 5K session recordings per month at no cost
- Self-hostable: Docker Compose or Kubernetes deployment; community build supports full self-hosting

**Differentiating features**
- The only open-source tool in this category that combines cohort analysis, session replay, feature flags, and A/B testing in a single platform — the breadth that otherwise requires four separate commercial tools
- MIT licence on the core product: maximally permissive; organisations can self-host, modify, and deploy without source-disclosure obligations
- Direct SQL access against the event store differentiates it from all other tools reviewed, which require using the platform's own query UI

**UX patterns**
- Insight creation wizard: select chart type, add events/filters, optionally segment by cohort
- People tab: individual user profiles with event history, cohort memberships, and linked session replays
- Feature flag and experiment results tied to cohort analysis: measure retention impact of feature rollouts by cohort

**Integration points**
- SDKs: JavaScript (web), React, React Native, iOS, Android, Python, Go, Ruby, PHP, Java
- Segment, RudderStack, and mParticle as upstream CDPs (bidirectional)
- Warehouse export: BigQuery, Snowflake, Redshift, S3 (PostHog data warehouse feature)
- REST API and Zapier/Make.com integrations for cohort export and event ingestion
- Slack alerts for cohort-based metric changes

**Known gaps**
- No predictive ML churn model; cohort analysis is behavioral/rule-based only
- Self-hosted community build misses some cloud features; full feature parity requires PostHog Cloud
- No customer success workflow features (health scores, playbooks, renewal tracking) — positioned for product analytics, not CS ops
- Not designed for B2B account-level (company) analysis out of the box, though group analytics partially addresses this
- PostHog's AI features (Max AI assistant) are cloud-only and less mature than Amplitude's or Mixpanel's AI capabilities

**Licence / IP notes**
- Core product: MIT licence — maximally permissive. No copyleft; no patent concerns identified. Organisations can build on, modify, or white-label PostHog without any source-disclosure obligations. Some premium cloud features are under a separate commercial licence.

---

### Heap (by Contentsquare)

**Core features**
- Retroactive event capture: automatically records every user action (click, submit, change, page view) without requiring manual instrumentation; allows retroactive cohort analysis on events that were not explicitly tracked at setup time
- Cohort builder: group users by acquisition date, first action, or any behavioral property; compare retention curves across cohorts
- Funnel analysis with retroactive cohort segmentation
- Integrated session replay: quantitative funnel/cohort analysis linked directly to qualitative session recordings of users who experienced a specific funnel step or cohort behaviour
- Contentsquare integration (post-acquisition, December 2023): combined platform for experience analytics (heatmaps, session replay, journey mapping) and product behavioral analytics

**Differentiating features**
- Retroactive event capture is unique in this category: teams can define new cohorts based on past user behaviours that were never explicitly tracked as custom events
- Combined Contentsquare + Heap platform provides the broadest experience analytics scope: from quantitative cohort/funnel data to qualitative session replay and heatmaps in one product

**UX patterns**
- Define events visually using the Heap Visual Labeler (point-and-click on any UI element to tag it as an event) — no code required
- Retroactive event definition propagates historically, so cohorts based on newly defined events show data from before the event was defined

**Integration points**
- JavaScript snippet for web; iOS and Android SDKs
- Salesforce, HubSpot, Zendesk, and Marketo connectors for CRM/marketing integration
- Warehouse export: Snowflake, BigQuery, Redshift
- Segment integration (bidirectional)

**Known gaps**
- Retroactive capture generates very large event volumes; data storage and query performance can be challenging at scale
- Contentsquare + Heap integration is still maturing (as of 2025–2026): workflows span two products that do not yet feel fully unified
- No predictive ML churn model
- Pricing is enterprise-level; not accessible to early-stage startups
- No open-source option

**Licence / IP notes**
- Fully proprietary. No open-source components. No patent concerns identified in public sources.

---

### Gainsight

**Core features**
- Customer health scoring: configurable multi-signal model combining product usage, CRM data, support ticket volume, survey results (NPS/CSAT), contract value, and custom signals into a single health score per account
- Cohort segmentation: group accounts by industry, tier, health score range, lifecycle stage, or any custom property for targeted playbook execution
- Playbooks: automated or CSM-triggered workflow sequences (emails, tasks, calls) activated by health score thresholds or lifecycle stage transitions
- NRR forecasting and renewal risk modelling at the account portfolio level
- C360 (Customer 360): unified view of all customer touchpoints, usage data, and CS activities in one account record
- Spaces (launched 2025): shared customer portal where clients and CSMs collaborate on success plans and mutual action items
- Machine learning-based health score prediction using historical behavioral and interaction data
- Recognised as Leader in 2025 Gartner Magic Quadrant for Customer Success Management Platforms (two consecutive years)

**Differentiating features**
- The most comprehensive enterprise CS platform: built for CS as a revenue discipline, not just a support function; multi-signal health scoring tied to real account structure (subsidiaries, business units)
- Early expansion signal detection and renewal forecasting at portfolio scale; designed for enterprise CS teams managing hundreds of complex accounts
- SAP acquisition (2023) provides deep ERP data integration for enterprise customers

**UX patterns**
- CSM workspace: individual account cockpit view; health score, open tasks, timeline of customer interactions
- Portfolio dashboard: risk and health heatmaps across the entire account base
- Playbooks are defined by CS ops, executed (manually or automatically) by CSMs in the cockpit

**Integration points**
- Salesforce CRM (deepest integration), HubSpot, Microsoft Dynamics
- Product usage data via API, Segment, or warehouse connection
- Zendesk, Jira, ServiceNow for support ticket ingestion
- SCIM and SSO for enterprise identity management
- REST API for custom integrations

**Known gaps**
- Very high TCO ($90K–$140K/yr Year 1 including implementation); accessible only to well-funded B2B SaaS companies
- Implementation typically takes 6–12 months; complex setup and ongoing admin requirements
- Health score configuration requires dedicated CS ops expertise; out-of-the-box scores are generic and need heavy tuning for each customer
- ML health score prediction, while present, is not explainable (SHAP-style feature attribution is not surfaced to users)
- Product analytics capabilities are secondary to CS workflow; not suitable as a replacement for Amplitude/Mixpanel for product teams

**Licence / IP notes**
- Fully proprietary. SAP-owned since 2023. No open-source components. No patent concerns identified in public sources.

---

### ChurnZero

**Core features**
- Health scoring with customisable signal weighting (product usage, email engagement, support tickets, NPS, and custom data)
- Cohort segmentation for targeted CS plays and campaign targeting
- Automated playbooks (ChurnZero Journeys): trigger-based sequences activated by health score changes, lifecycle events, or manual CS manager initiation
- In-app messaging and guided walkthroughs for customer engagement directly within the product
- CRM integration: Salesforce, HubSpot, and Dynamics for account data synchronisation
- Real-time activity feed per account; usage heatmaps and feature adoption tracking
- ChurnZero AI (2025): AI-assisted playbook recommendations and health score anomaly alerts
- Faster deployment than Gainsight: typical 4–8 weeks vs. 6–12 months

**Differentiating features**
- In-app messaging and walkthroughs embedded natively within the CS platform — not requiring a separate tool like Intercom or Pendo
- Mid-market sweet spot: strong feature set at lower TCO than Gainsight; faster time to value
- ChurnZero AI playbook recommendations reduce the cognitive load on CSMs managing large account books

**UX patterns**
- Account card view with health score, recent activity, and open tasks
- Segment builder for cohort definition by multiple account properties
- Plays/Journeys dashboard showing active automation sequences and their account coverage

**Integration points**
- Salesforce, HubSpot, Dynamics 365 CRM
- Product usage data via API or Segment
- Zendesk and Intercom for support data
- SSO and SCIM for enterprise provisioning

**Known gaps**
- Health score ML is heuristic-weighted rather than true statistical modelling; limited explainability
- Less sophisticated multi-signal health modelling than Gainsight for large enterprise accounts with complex structures
- In-app messaging competes with (and may conflict with) existing product in-app tools; requires coordination with product teams
- No open-source option; vendor lock-in for account data and playbook configurations

**Licence / IP notes**
- Fully proprietary SaaS. No open-source components. No patent concerns identified in public sources.

---

### Totango

**Core features**
- Modular "SuccessBLOC" architecture: pre-built playbook and automation modules for specific CS use cases (onboarding, adoption, renewal, expansion) that can be activated independently
- Customer health scoring with segment-level and account-level scoring
- Customer journey mapping: visual timeline of lifecycle stages and touchpoints with automated transitions between stages
- Touchpoint tracking: logs all customer interactions (emails, calls, in-app activity) in a unified account timeline
- Totango AI (2025): segment intelligence and recommendation features
- Premium setup cost: typically 20% of annual contract value; implementation overhead is notable

**Differentiating features**
- Modular SuccessBLOC approach allows teams to start with one CS use case and expand incrementally; lower initial complexity than Gainsight's full-platform deployment
- Journey mapping with automated lifecycle stage transitions is more visual and process-oriented than Gainsight or ChurnZero

**UX patterns**
- SuccessBLOC dashboard shows active modules and their health status
- Account profile view with journey stage, health score, and touchpoint timeline
- Self-service activation of new SuccessBLOCs reduces dependency on vendor professional services

**Integration points**
- Salesforce, HubSpot CRM
- Product usage data via API or Segment
- Zendesk, Jira for support ticket ingestion
- Email integration for touchpoint logging

**Known gaps**
- High setup cost (20% of ACV) makes total cost of ownership among the highest in the mid-market CS category
- Smaller customer community and ecosystem compared to Gainsight and ChurnZero
- AI features are less mature than competitors; SuccessBLOC intelligence is rule-based rather than ML-driven
- No product analytics capabilities; must integrate with Amplitude/Mixpanel/PostHog for product usage data

**Licence / IP notes**
- Fully proprietary SaaS. No open-source components. No patent concerns identified in public sources.

---

### Pecan AI

**Core features**
- No-code predictive ML platform: analysts and marketers build churn prediction models without data science expertise via a guided co-pilot interface
- GenAI-driven natural language guidance through the model-building process (conversational AI introduced 2026)
- Churn prediction outputs: probability scores per customer with configurable prediction horizons
- Prediction-level explainability: transparent dashboards show the key drivers behind every individual prediction (top contributing features per customer)
- Rapid model deployment: predictive models reaching production up to 32x faster than traditional ML approaches
- Achieves approximately 12% average churn reduction across deployments; models trained on customer's own data
- Integrates with the customer's existing data warehouse; does not require a new data collection layer

**Differentiating features**
- The only purpose-built, no-code predictive analytics platform in this segment that combines conversational model building with prediction-level feature explainability
- Gartner 2025 Cool Vendor in Cross-Functional Supply Chain Technology (signals broader ML deployment capability)
- Warehouse-native: trains models on data already in the customer's Snowflake, BigQuery, or Redshift instance without requiring data migration

**UX patterns**
- Wizard-driven model setup: select prediction target (churn, LTV, conversion), connect data, guided feature selection, review model performance, deploy
- Explainability dashboard: per-customer and aggregate feature attribution charts
- Prediction output: scored lists exportable to CRM, marketing platforms, or CS tools for activation

**Integration points**
- Data source: Snowflake, BigQuery, Redshift, and other SQL warehouses
- Prediction output: Salesforce, HubSpot, Segment, Braze, and other activation platforms via API or warehouse write-back
- REST API for programmatic model management and score retrieval

**Known gaps**
- Not a real-time analytics or session-level event tracking platform; designed for batch prediction over warehouse data
- No cohort visualisation or funnel analysis; focused purely on predictive scoring, not exploratory cohort analytics
- Quote-based pricing with no self-serve tier; requires a sales engagement
- No open-source version; vendor lock-in on the model training pipeline and output format

**Licence / IP notes**
- Fully proprietary SaaS. Raised $66M Series C (2023). No open-source components. No patent concerns identified in public sources; the explainability techniques used (SHAP) are derived from published research and are not patent-blocked.

---

### Countly

**Core features**
- Open-source product analytics with self-hosted deployment (AGPL-3.0 Community Edition)
- Cohort retention analysis with configurable time periods
- Conversion funnels with drop-off tracking
- Individual user tracking with complete event history
- User segmentation by behavioral properties and event sequences
- Push notification system (mobile): built-in, no third-party push service required
- Crash reporting, remote configuration, and surveys in the same platform
- Strong mobile analytics roots: SDKs for iOS, Android, React Native, Flutter, and desktop platforms
- Infrastructure requirements: minimum 4 vCPU / 8 GB RAM; MongoDB 6+

**Differentiating features**
- Only fully self-hostable open-source product analytics platform with cohort analysis, crash reporting, and push notifications in one product — strong appeal for privacy-regulated industries (GDPR, HIPAA) that cannot send data to US-hosted SaaS
- Built-in push notification system removes a common external dependency for mobile-first product teams

**UX patterns**
- Dashboard-first: pre-built report templates for retention, funnels, active users, revenue
- Explorer view for ad hoc cohort definition
- Admin panel for SDK configuration, plugin management, and data governance

**Integration points**
- SDKs: iOS, Android, macOS, Windows, React Native, Flutter, Unity, and 20+ others
- REST API for event ingestion and report data export
- Webhooks for alerting on metric thresholds
- Enterprise edition: Salesforce and Zendesk connectors

**Known gaps**
- No ML-based churn prediction or predictive cohort scoring
- Less advanced AI features than commercial alternatives; AI capabilities in the community edition are minimal
- Smaller developer community and slower feature cadence than PostHog
- MongoDB requirement creates an additional infrastructure dependency vs. PostgreSQL-native tools
- AGPL-3.0 licence imposes copyleft obligations on modifications; commercial deployments integrating modified Countly must release source changes

**Licence / IP notes**
- Community Edition: AGPL-3.0. Modifications to Countly itself must be open-sourced if distributed. SaaS products that use an unmodified Countly as a backend analytics service do not trigger the AGPL obligation (AGPL's "use over a network" clause applies only to modifications). Commercial Enterprise edition available under a proprietary licence. No patent concerns identified.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Behavioral cohort builder: define user groups by event sequences, properties, and time windows with a GUI (no SQL required)
- Day-N retention curves: visualise what percentage of a cohort returns on Day 1, 7, 14, 30, and custom intervals
- Funnel analysis with cohort-based filtering: measure conversion rates for specific user groups
- Dashboard creation and scheduled reporting: share cohort analysis results with non-analyst stakeholders
- SDK-based event tracking: JavaScript (web) and mobile (iOS, Android) event collection
- Role-based access control: restrict data access by team or user role

### Differentiating Features
- ML-based predictive churn scoring: probabilistic per-user risk scores using behavioral signals (Amplitude, Pecan); not offered by open-source tools
- SHAP-explained prediction drivers: per-user feature attribution showing why a specific customer is at risk (currently absent from all tools reviewed except Pecan, which offers aggregate-level attribution)
- Automated discovery of non-obvious retention drivers: AI surfaces behavioral correlates of retention without requiring analysts to define hypotheses first (Amplitude's automated discovery)
- Natural-language cohort querying: define cohorts via conversational prompts rather than multi-step filter UI (Amplitude AI co-pilot, Mixpanel Spark; open-source tools lack this)
- Account-level (B2B) health scoring: aggregate individual user behavioral signals to a company account score (Gainsight, ChurnZero; absent from product analytics tools)

### Underserved Areas / Opportunities
- Open-source predictive cohort ML: no widely adopted open-source tool combines behavioral cohort analysis with integrated ML-based churn prediction; the gap between PostHog (OSS, no ML) and Pecan (ML, no cohorts) is entirely unserved
- XAI-powered churn prediction at the individual level: existing ML tools (Amplitude Predictions, Pecan) score users but do not surface SHAP-level per-user explanation to operations teams in actionable language
- Proactive retention narrative: no current tool automatically pushes a natural-language cohort health summary to CS or product teams when a cohort begins showing early warning patterns
- Natural-language cohort definition: all tools reviewed require multi-step UI workflows or SQL to define cohorts; conversational definition remains nascent even in the most AI-advanced commercial tools
- Open-source + affordable mid-market churn prediction: enterprise tools (Gainsight, ChurnZero) are $60K–$140K/yr; product analytics tools (Amplitude, Mixpanel) provide some ML but at $1K–$5K/month; a free self-hostable option with integrated ML churn prediction does not exist

### AI-Augmentation Candidates
- Automated cohort discovery: statistically significant behavioral segments correlated with retention or churn, surfaced without requiring analysts to hypothesise what to look for
- NL cohort builder: conversational definition of cohorts ("show me users who upgraded from free in the last 90 days but haven't logged in for 30 days, broken down by acquisition channel")
- SHAP-explained survival models: continuously updating churn prediction with feature attribution surfaced as plain-English explanations for CS team action
- Proactive cohort health narratives: weekly LLM-generated summaries pushed to Slack/email when a cohort's retention trajectory diverges from its historical baseline
- Intervention recommendation: given a flagged at-risk cohort, suggest which playbooks or interventions have historically worked for similar cohorts

---

## Legal & IP Summary

Open-source tools in this category carry two distinct licence regimes. PostHog (MIT) is maximally permissive: any downstream product can incorporate, modify, and deploy it with no source-disclosure obligations. Countly's Community Edition (AGPL-3.0) requires that modifications to Countly itself be open-sourced if distributed; unmodified use as a backend service does not trigger AGPL. All commercial platforms (Amplitude, Mixpanel, Heap, Gainsight, ChurnZero, Totango, Pecan AI, Kissmetrics) are fully proprietary with no disclosed patent concerns in public sources beyond standard software IP portfolios. The explainability techniques central to the AI-native opportunity — SHAP (SHapley Additive Explanations), LIME, and survival analysis — are derived from published academic research and are not patent-blocked for new implementations. Behavioral cohort analytics products inevitably handle personal data subject to GDPR (EU Regulation 2016/679) and CCPA: consent management, data minimisation, right-to-erasure workflow, and cross-border transfer controls are legal requirements for any new product in this space, not optional features. ISO/IEC 27001-level information security is required by enterprise buyers.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Behavioral cohort builder with GUI: define cohorts by event sequences, properties, and time windows without requiring SQL
- Day-N retention curves and funnel analysis filterable by any user cohort
- JavaScript (web) and mobile (iOS/Android) SDK for event collection using the Segment `.track()` / `.identify()` event schema
- ML-based churn risk scoring: per-user probability score trained on the customer's own behavioral data, updated on a rolling basis
- XAI feature attribution per prediction: surface the top 3–5 behavioral drivers for each at-risk user in plain English ("This user is at risk because their session frequency dropped 60% vs. their cohort average")
- Dashboard creation and Slack/email delivery for scheduled cohort health reports

**Should-have (v1.1)**
- Automated cohort discovery: AI surfaces statistically significant behavioral segments correlated with retention or churn without requiring analysts to define the hypothesis first
- Natural-language cohort definition: conversational interface for describing cohort filters ("users who invited a teammate in their first week but haven't logged in for 21 days")
- Account-level (B2B) aggregation: roll up individual user behavioral signals to a company account health score for B2B SaaS use cases
- Intervention recommendation engine: given a flagged at-risk cohort, suggest which intervention type has historically been most effective for similar cohorts

**Nice-to-have (backlog)**
- Proactive cohort health summaries: weekly LLM-generated narratives pushed to Slack/email when a cohort's retention trajectory diverges from its historical baseline
- Session replay integration: link funnel drop-off events to session recordings for qualitative investigation
- Survival analysis with continuous model updating: replacing static churn scoring with a survival model that updates as new behavioral data arrives
- GDPR/CCPA self-service controls: built-in right-to-erasure request processing and consent audit trail for regulated deployments
