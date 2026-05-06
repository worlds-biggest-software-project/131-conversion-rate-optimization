# Conversion Rate Optimization — Feature & Functionality Survey

> Candidate #131 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Hotjar | Commercial SaaS | Proprietary | https://www.hotjar.com/ |
| VWO (Visual Website Optimizer) | Commercial SaaS | Proprietary | https://vwo.com/ |
| Optimizely | Commercial SaaS (enterprise) | Proprietary | https://www.optimizely.com/ |
| FullStory | Commercial SaaS | Proprietary | https://www.fullstory.com/ |
| Crazy Egg | Commercial SaaS | Proprietary | https://www.crazyegg.com/ |
| Convert.com | Commercial SaaS | Proprietary | https://www.convert.com/ |
| AB Tasty | Commercial SaaS | Proprietary | https://www.abtasty.com/ |
| Mouseflow | Commercial SaaS | Proprietary | https://mouseflow.com/ |
| Kameleoon | Commercial SaaS | Proprietary | https://www.kameleoon.com/ |
| GrowthBook | Open Source | MIT License (core; enterprise licensing for some features) | https://www.growthbook.io/ |

## Feature Analysis by Solution

### Hotjar

**Core features**
- Heatmaps: visual representation of user interactions (clicks, scrolls, movement)
- Session recordings (Session Replay): anonymized video playback of individual user sessions across multiple pages
- Surveys: custom feedback collection from visitors
- Funnels: conversion funnel visualization and drop-off analysis
- Visitor segmentation for targeted analysis
- Continuous automatic session recording without manual restarts
- Feedback widgets for collecting user opinions in-context
- Integration with popular analytics and marketing tools

**Differentiating features**
- Broad feature set at entry-level pricing ($39/month); lowest barrier to SMB adoption
- Unified dashboard combining heatmaps, recordings, surveys, and funnels in one view
- Rage-click detection to identify friction points automatically
- Automatic session collection (no snapshot-based gaps)
- Part of Contentsquare ecosystem (acquired 2021), enabling broader data integrations

**UX patterns**
- Progressive disclosure: starts with heatmaps and basic insights, advances to session review and survey analysis
- Visual-first approach: defaults to heatmaps and click hotspots before diving into individual sessions
- Guided data exploration with suggested actions based on behavioral patterns

**Integration points**
- Standard analytics platforms (Google Analytics, Mixpanel, etc.)
- CRM and marketing automation integrations
- Webhook support for event-based workflows
- API for custom integrations

**Known gaps**
- Limited A/B testing capabilities; positioning is behavioral analysis, not experimentation-native
- Session data capped on lower plans; limits data volume for high-traffic sites
- No native WYSIWYG editor for test variations; requires third-party tools for test execution
- Limited multivariate testing functionality
- Minimal AI-powered insights compared to newer entrants (FullStory, AB Tasty)

**Licence / IP notes**
- Proprietary SaaS; no licensing conflicts identified

---

### VWO (Visual Website Optimizer)

**Core features**
- A/B testing: split testing with client-side WYSIWYG editor
- Multivariate testing (MVT): simultaneous testing of multiple variables and combinations
- Server-side testing: backend experimentation for deeper changes
- Heatmaps: click, scroll, movement tracking
- Session recordings: visitor video playback
- Bayesian statistics engine: real-time statistical significance calculation
- VWO Copilot: AI-powered hypothesis suggestions and variation generation
- Visitor segmentation and targeting
- Unlimited variations, metrics, and integrations on premium tiers

**Differentiating features**
- Unified testing + analytics stack: single platform for both experimentation and behavioral understanding
- WYSIWYG visual editor: design test variations without code
- AI Copilot: generative assistance for hypothesis creation and test design
- Transparent, straightforward pricing structure ($300–$2,000+/month)
- Real-time Bayesian reporting: see results faster than traditional frequentist statistics

**UX patterns**
- Test-centric workflow: compose hypothesis → design variations → set targets → deploy
- Real-time results dashboard with live confidence intervals
- AI-assisted suggestions appearing as users build tests

**Integration points**
- Tag-based event system for goal tracking
- Third-party data source integrations
- API for programmatic test management
- Webhooks for event-driven triggers

**Known gaps**
- Complex interface for beginners; steeper learning curve than Hotjar despite visual editor
- No native competitive intelligence or benchmarking features
- Limited personalization; testing is A/B-focused, not real-time audience adaptation
- Form analytics not as deep as dedicated tools (e.g., Mouseflow)

**Licence / IP notes**
- Proprietary SaaS; no licensing conflicts identified

---

### Optimizely

**Core features**
- Feature experimentation: feature flags and A/B testing integrated
- Full-stack testing: client-side, server-side, and edge experimentation
- Global Holdouts: control group designation to separate winners from ongoing experimentation effects
- Opal Agent Directory: AI agents for report generation, natural language querying, and data retrieval
- Feature flag management: version control, environment-based status tracking (Draft, Running, Paused)
- Multi-user collaboration and advanced permissions
- Precise flag status visibility across environments (Production, Development, etc.)
- Comprehensive test targeting and segmentation
- Data-driven experimentation program oversight

**Differentiating features**
- Technically deepest platform: designed for large-scale, multi-team experimentation programs
- Global Holdouts: unique ability to maintain a control population untouched by any experimentation
- Opal Agent architecture: conversational AI that understands your experiment portfolio and can generate structured reports
- Native feature flag + experimentation integration: single system for both progressive rollouts and A/B tests
- Environment-aware flag status: clear visibility into deployment across dev, staging, production

**UX patterns**
- Developer-first: emphasis on code-based setup and flag configuration
- Agent-assisted insights: natural language queries return experiment insights and program overviews
- Flag-centric workflow: design flags → run tests → analyze via Opal → rollout winners
- Advanced dashboards: filtered views by flag status, environment, and experiment type

**Integration points**
- Webhook support for automated actions
- API-first design: all functionality accessible programmatically
- SDK support for multiple languages and frameworks
- Experiment data APIs for external analysis
- Flag status APIs for external dashboards

**Known gaps**
- No native heatmaps or session recording; behavioral analysis requires third-party tools
- Enterprise-only positioning; not accessible to SMBs or product-stage teams
- Steep implementation overhead; requires engineering resources to set up and maintain
- Limited campaign personalization; focus is experimentation, not content adaptation
- Complex setup; not suitable for rapid hypothesis testing without engineering support

**Licence / IP notes**
- Proprietary enterprise SaaS; no licensing conflicts identified

---

### FullStory

**Core features**
- Session intelligence: detailed session replay with digital experience (DX) metrics
- StoryAI: suite of generative AI agents for automated analysis
  - Summaries Agent: analyzes cohorts to generate text reports of behavioral patterns
  - Opportunities Agent: links behavior to business outcomes with revenue impact estimates
  - Ask StoryAI: conversational LLM interface (powered by Google Gemini) for contextual questions
- Powerful search: find sessions and users based on complex behavioral predicates
- AI-generated insights: automatically surface findings (e.g., "form validation errors on iOS checkout")
- Engagement and conversion tracking
- Cross-device session stitching
- Privacy compliance (GDPR, CCPA) with ISO 42001 AI governance certification

**Differentiating features**
- StoryAI is the highest-performing new product in FullStory's history; customers action insights 3.8x faster than non-AI users
- Opportunities Agent uniquely estimates revenue impact of fixes before implementation
- Conversational interface (Ask StoryAI) bridges gap between product questions and behavioral data
- Google Gemini integration for advanced NLP understanding
- ISO 42001 certification: demonstrates AI governance rigor
- Powerful search engine: find exact user cohorts matching complex behavioral criteria

**UX patterns**
- AI-first insights: system surfaces findings automatically rather than requiring analyst exploration
- Conversational analysis: ask questions in plain English and get behavioral answers
- Outcome-linked insights: every finding includes revenue or engagement impact
- Search-driven discovery: refine session cohorts via progressive filtering

**Integration points**
- Webhook support for automated actions
- API for session and user data retrieval
- Google Gemini integration (required for AI features)
- Third-party analytics integrations

**Known gaps**
- No A/B testing or experimentation; positioning is behavioral intelligence only
- Pricing opaque and expensive ($2,000–$50,000+/year); not suitable for SMBs
- Requires Google Gemini sub-processing; privacy implications for some organizations
- Heavy integration lift; requires significant implementation effort
- Form analytics not as detailed as dedicated tools (e.g., Mouseflow)

**Licence / IP notes**
- Proprietary SaaS; StoryAI requires Google Gemini as sub-processor, introducing data processing dependency

---

### Crazy Egg

**Core features**
- Click heatmaps: thermal visualization of clickable element popularity
- Scrollmaps: shows where users focus attention (scroll depth) regardless of clicks
- Movement heatmaps: tracks mouse movement patterns
- Confetti reports: list and overlay maps quantifying clicks per element
- Simple A/B testing: basic split-test execution
- Session recordings: visitor video playback
- AI Analysis: explains behavioral trends in plain language and recommends optimizations
- Heatmap data export: JSON format optimized for LLM analysis (ChatGPT, Gemini, Copilot)
- Segmented heatmaps: separate analysis by user segment or cohort

**Differentiating features**
- Original click heatmap innovator; simplest, most visual interface in category
- Affordable entry point ($29/month); lowest cost among feature-complete tools
- AI Analysis explains trends and recommends page changes automatically
- JSON data export enables deep analysis in external LLMs
- Intuitive visual-first design; no learning curve for non-technical users

**UX patterns**
- Visual-first: heatmap colors dominate; numbers and stats secondary
- Click-to-insight: see a hot zone, get AI recommendation immediately
- Export-to-analyze: advanced users download JSON data and analyze in LLMs
- Simplicity: intentionally narrow feature set; not a full-stack platform

**Integration points**
- Standard web integration (JavaScript snippet)
- Limited third-party integrations
- JSON data export for external analysis
- Basic API for data retrieval

**Known gaps**
- Limited A/B testing; test execution requires designer/developer, not self-service
- No multivariate testing; A/B only
- Minimal statistical reporting; lacks confidence intervals or frequentist/Bayesian analysis
- No session intelligence or behavioral segmentation
- Form analytics completely absent
- No conversion funnel tracking

**Licence / IP notes**
- Proprietary SaaS; no licensing conflicts identified

---

### Convert.com

**Core features**
- A/B testing: split URL and client-side A/B tests
- Multivariate testing (MVT): simultaneous testing of multiple variables
- Multi-page tests: experiments spanning multiple pages or visitor journeys
- Full-stack experiments: backend/API-level testing
- Personalization campaigns (Deploys): real-time visitor-level adaptation
- GDPR & ePrivacy compliance: first-party cookies, anonymized data, no personal data storage
- Privacy-first architecture: EU-based servers, respects DNT and GPC
- Data Processing Agreement (DPA): simplified compliance documentation
- In-app compliance guidance: notifications help users stay compliant
- Accurate, real-time reporting

**Differentiating features**
- Privacy-first design: only 1st party cookies, GDPR-native from day one
- Privacy as competitive advantage: compliance built in, not bolted on
- Multivariate testing at mid-market pricing ($199+/month)
- Flexible experiment types: A/B, MVT, multi-page, and full-stack in one platform
- Simplified compliance: Data Processing Agreement requires only electronic signature

**UX patterns**
- Privacy-conscious workflow: every feature includes in-app compliance guidance
- Test variety support: same interface for A/B, MVT, full-stack, and redirect tests
- Privacy transparency: compliance status and data handling clearly communicated

**Integration points**
- Data Processing Agreement API
- Webhook support for event-driven workflows
- Third-party integrations for data export and analysis
- Compliant event tracking API

**Known gaps**
- No heatmaps or session recording; behavior analysis requires third-party tools
- Smaller ecosystem: fewer integrations than VWO or Optimizely
- Limited AI features; no hypothesis generation or automated insights
- No personalization engine beyond basic deploy-based targeting
- Form analytics completely absent

**Licence / IP notes**
- Proprietary SaaS; no licensing conflicts identified

---

### AB Tasty

**Core features**
- A/B and multivariate testing: client-side and server-side experimentation
- Evi: agentic AI for marketing experimentation (launched November 2025)
- Hypothesis Copilot: AI-assisted hypothesis scoring and refinement
  - Evaluates hypothesis statements against data models
  - Assigns quality scores and identifies gaps
  - Suggests edits to strengthen hypothesis clarity
- Report Copilot: AI summarization of experiment results
- Idea generation: AI-powered test backlog creation
- EmotionsAI: visitor emotional and behavioral segmentation
- Personalization engine: deliver unique user journeys based on intent and emotion
- Natural language reporting: AI-powered insights in plain English
- No-code experiment building

**Differentiating features**
- Evi agent: 3,000+ early adopters reported 73% reduction in campaign setup time and 25% increase in campaign output
- Hypothesis Copilot: uniquely scores and refines hypotheses before test execution
- EmotionsAI: emotional segmentation (not just behavioral) for deeper personalization
- Agentic AI approach: Evi acts as autonomous agent, not just suggestion engine
- Integrated personalization: delivery of AI-recommended variants, not just testing

**UX patterns**
- AI-first experimentation: ideas flow through Evi for scoring and execution
- Emotional intelligence: segmentation and personalization based on visitor intent, not just behavior
- Hypothesis quality assurance: Evi ensures hypotheses are sound before test design begins
- Automated insight generation: results translated to natural language recommendations

**Integration points**
- Evi API for programmatic experiment management
- Hypothesis and result data APIs
- Third-party integration support
- Webhook support for automated actions

**Known gaps**
- Limited heatmap or session replay capabilities; behavioral analysis requires third-party tools
- Complex setup; not suitable for beginners without marketing operations support
- Pricing custom/enterprise; not transparent for small teams
- Limited free tier or trial period mentioned
- No native form analytics

**Licence / IP notes**
- Proprietary SaaS; no licensing conflicts identified

---

### Mouseflow

**Core features**
- Session replay: 100% session capture (no sampling) including clicks, scrolls, hesitations
- Seven heatmap types: click, scroll, movement, attention, geo, interactive, and friction (all auto-generated)
- Friction detection: automatic detection of 7+ friction signals (rage clicks, dead clicks, JS errors, speed browsing, etc.)
- Form analytics: field-level drop-off tracking, completion rates, and hesitation time
- Conversion funnels: multi-step funnel visualization and drop-off analysis
- Journey analytics: path and segment analysis
- Feedback surveys: in-context visitor feedback
- Mina AI: natural-language querying of session data to surface relevant recordings and patterns
- All features available on all plans (no feature tiering)

**Differentiating features**
- Seven heatmap types automatically generated (most comprehensive); no manual configuration required
- Only tool analyzed with dedicated field-level form analytics; unique capability for form optimization
- 100% session capture without sampling; no data loss on high-traffic sites
- Mina AI: describe what you're looking for in natural language; AI surfaces matching sessions and friction patterns
- All-inclusive pricing: every plan includes every feature (no feature paywall)

**UX patterns**
- Comprehensive behavior analysis: session replay + all heatmaps + funnel + form analytics in one view
- Automatic friction detection: system flags problem areas without analyst review
- Natural language discovery: ask Mina AI about sessions and friction; get instant results
- All-features-included: no plan downgrade reduces capability

**Integration points**
- Webhook support for friction alerts and session events
- API for session and heatmap data retrieval
- Third-party integrations (analytics, CRM, etc.)
- Mina AI query interface

**Known gaps**
- No A/B testing or experimentation; positioning is behavioral analysis only
- No personalization capabilities
- No conversion prediction or attribution modeling
- Limited statistical analysis compared to dedicated testing platforms
- No competitive benchmarking

**Licence / IP notes**
- Proprietary SaaS; no licensing conflicts identified

---

### Kameleoon

**Core features**
- A/B testing and multivariate testing (MVT): client-side and server-side
- Predictive AI: real-time user intent prediction and behavioral targeting
- PBX (Prompt-Based Experimentation): generate live A/B tests from natural language descriptions
- AI Copilot: AI-assisted hypothesis generation and test scaling
- AI Experiments: AI-driven test variant creation
- Server-side testing: complex backend experiments without caching constraints
- Personalization engine: real-time delivery of AI-recommended variants per segment
- Traffic allocation: fixed splits or multi-armed bandit (dynamic allocation to best performers)
- Visitor segmentation based on predicted intent and behavior

**Differentiating features**
- Predictive AI targeting: segments users by predicted intent before test execution
- PBX (Prompt-Based Experimentation): turn natural language ideas into live tests in minutes
- Server-side flexibility: removes caching constraints that block other server-side testing platforms
- Multi-armed bandit allocation: dynamic traffic reallocation to winning variants in real time
- Full-stack architecture: unified API for client, server, and edge experimentation

**UX patterns**
- AI-first experimentation: describe ideas in natural language; AI handles design and execution
- Predictive segmentation: tests run against AI-identified high-intent cohorts
- Dynamic adaptation: bandit allocation continuously rebalances traffic toward winners
- Minimal friction: prompt-to-test workflow eliminates test design overhead

**Integration points**
- REST API for full-stack integration
- SDK support for multiple languages and frameworks
- Webhook support for event-driven workflows
- Server-side experiment data APIs
- Predictive AI data feeds

**Known gaps**
- No heatmaps or session recording; behavioral analysis requires third-party tools
- Complex setup; requires engineering resources, not self-service for non-technical users
- Limited SMB appeal; enterprise/mid-market positioning
- Predictive algorithm details not fully documented; potential vendor lock-in for personalization strategies
- No built-in form analytics

**Licence / IP notes**
- Proprietary SaaS; predictive algorithms and AI engine appear proprietary; recommend legal review if reimplementation is planned

---

### GrowthBook

**Core features**
- Feature flags: client-side and server-side flag management
- A/B testing and multivariate testing: integrated experimentation platform
- Product analytics: event tracking and funnel analysis
- Visual editor: drag-and-drop interface for test variation design (premium feature)
- Experiment history: full audit trail and version control
- Statistical analysis: frequentist confidence intervals and Bayesian analysis
- Targeting and segmentation: rule-based audience selection
- API-first design: every feature accessible programmatically
- Open-source core: MIT license for self-hosted deployment

**Differentiating features**
- Completely free (core): MIT licensed, unlimited users and experiments for self-hosted
- Full control: on-premises deployment; no vendor lock-in or data transfer to external servers
- Unified platform: flags, A/B testing, analytics, and product data in one system
- Developer-friendly: API-first architecture with comprehensive SDK support
- Cost-effective at scale: no per-user or per-experiment fees

**UX patterns**
- Developer-first workflow: most features accessed via API; UI is supplementary
- Infrastructure ownership: users control entire stack deployment
- Progressive complexity: start with simple flags, advance to multivariate testing
- Privacy-first: all data remains on-premises

**Integration points**
- Comprehensive API for flags, experiments, analytics, and configuration
- SDKs for JavaScript, React, Python, Ruby, Go, Java, and others
- Webhook support for event-driven workflows
- Kafka/streaming integration for high-volume events
- Third-party analytics integrations

**Known gaps**
- Limited heatmap/session replay in core; no native behavioral analysis
- No AI-powered hypothesis generation or automated insights
- No built-in personalization engine; requires custom implementation
- Steeper learning curve than commercial SaaS alternatives
- Self-hosted deployment requires infrastructure expertise and maintenance
- Visual editor (UI-based experimentation) available only on premium tiers

**Licence / IP notes**
- MIT License (core) with GrowthBook Enterprise License for certain features
- MIT is permissive; safe for commercial use and proprietary derivative work
- Enterprise tier has separate licensing; review carefully if planning commercial use

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

These capabilities must be present in any viable CRO platform:

- **A/B testing**: split tests with real-time or batched statistical analysis (95%+ confidence standard)
- **Heatmaps and session recording**: visualization of user behavior (clicks, scrolls, movement) and individual session playback
- **Conversion funnels**: multi-step funnel visualization and drop-off identification
- **Visitor segmentation**: ability to isolate analysis by cohort, traffic source, device, or custom attributes
- **Statistical significance reporting**: confidence intervals or Bayesian credibility intervals to determine test winners
- **Dashboard and alerts**: visual trend monitoring and configurable alerts for anomalies
- **Privacy compliance**: GDPR/CCPA consent handling and data retention controls

### Differentiating Features

Capabilities that some solutions offer and provide competitive advantage:

- **AI-powered hypothesis generation**: automated test idea creation from historical data (Kameleoon, AB Tasty, VWO Copilot). Reduces manual backlog work from weeks to minutes.
- **Multivariate testing (MVT)**: simultaneous testing of multiple variables and combinations (VWO, Convert, Kameleoon, GrowthBook). Enables faster learning for multivariate pages.
- **Server-side and full-stack testing**: backend experimentation for complex scenarios (Optimizely, Convert, Kameleoon, VWO). Removes reliance on client-side scripts.
- **AI agents for insight generation**: automated analysis and reporting (FullStory's Opportunities Agent, AB Tasty's Evi, Mouseflow's Mina AI). Reduces analyst workload 3–8x.
- **Personalization at scale**: real-time variant delivery per visitor segment based on predicted intent (Kameleoon, AB Tasty, Optimizely). Goes beyond A/B testing to dynamic adaptation.
- **Dynamic (bandit) traffic allocation**: real-time reallocation toward winning variants (Kameleoon, some Optimizely configurations). Shortens test duration by 30–50%.
- **Field-level form analytics**: per-field drop-off and hesitation tracking (Mouseflow only). Unique capability for form optimization.
- **Privacy-first design**: GDPR-native architecture with first-party cookies and no personal data storage (Convert.com). Increasingly important for EU/regulated markets.

### Underserved Areas / Opportunities

Gaps that represent genuine opportunities for differentiation:

- **Unified behavioral + experimentation stack**: most platforms specialize (heatmaps OR testing OR personalization) rather than integrating all three seamlessly. Fragmentation forces teams to use 3–5 tools.
- **Attribution modeling**: connecting CRO wins to revenue and customer lifetime value remains manual and disconnected from experimentation. Most tools lack true multi-touch attribution.
- **Offline-to-online conversion tracking**: phone calls, chat, form submissions, and store visits remain siloed from online experiments. End-to-end funnel visibility is rare.
- **Content testing at scale**: most tools test page layouts and buttons; few support testing copy, subject lines, and written content variation systematically.
- **Cross-channel experimentation**: running coordinated A/B tests across email, SMS, push, and web remains manual; no platform unifies channel-specific experimentation.
- **Hypothesis lifecycle management**: capture hypothesis intent, link tests to strategic goals, and track learning ROI across experiments. Most tools treat tests as ad-hoc activities.
- **Qualitative integration**: combining A/B test results with user interviews, support tickets, and qualitative feedback into unified insights. Usually requires manual correlation.
- **Real-time anomaly detection in live experiments**: automated alerting when experiments show early warning signs of failure, novelty effects, or sample ratio mismatches. Current tools require manual monitoring.

### AI-Augmentation Candidates

Features currently implemented with manual/rule-based approaches where AI could provide meaningful value:

- **Hypothesis generation and prioritization**: Replace PIE/ICE scoring frameworks with ML models trained on historical experiment results. Predict which hypotheses will win before testing.
- **Automatic variant creation**: ML-generated page variations based on user behavior patterns and winning elements from similar sites. Current tools require manual design.
- **Personalization without testing**: Use visitor intent prediction and behavioral history to deliver optimal variants immediately, with A/B test confirming AI choice rather than discovering it. Kameleoon moves in this direction.
- **Real-time experiment monitoring**: Continuous ML-based detection of novelty effects, sample ratio mismatches, and early failure indicators. Flag problems automatically rather than relying on analyst vigilance.
- **Conversion prediction per visitor**: ML models predicting conversion probability per individual before they convert, enabling real-time intervention (chatbot, discount offer, etc.).
- **Content optimization**: NLP-driven analysis of copy, subject lines, and messaging to suggest variants likely to improve conversion. Currently manual A/B testing of copy.
- **Multi-armed bandit optimization**: Replace fixed traffic splits with adaptive allocation that learns in real time. Kameleoon and Optimizely offer this; should be table-stakes.
- **Statistical analysis automation**: Automatic determination of statistical significance and confidence intervals without analyst interpretation. Current tools display numbers; AI could decide winners.
- **Qualitative feedback synthesis**: Use NLP and LLMs to analyze survey responses, support tickets, and session rage-click comments, synthesizing actionable insights. Few tools do this.

---

## Legal & IP Summary

All commercial CRO platforms analyzed (Hotjar, VWO, Optimizely, FullStory, Crazy Egg, Convert, AB Tasty, Mouseflow, Kameleoon) are proprietary SaaS systems with standard SaaS licensing. No copyright, patent, or licensing conflicts were identified for their core features. 

GrowthBook's core (MIT License) is permissive and safe for commercial use and proprietary derivative work. Enterprise features of GrowthBook carry separate licensing that should be reviewed if commercial use is planned.

One uncertainty: Kameleoon's predictive AI algorithm and personalization engine are proprietary and not fully documented. If the project plans to reimplement Kameleoon-style predictive segmentation or bandit allocation, recommend independent legal review to assess reimplementation risk and ensure no patent conflicts.

FullStory's use of Google Gemini as a sub-processor for StoryAI features introduces data processing dependencies and should be considered in privacy assessments.

No material identified as subject to known active software patents that would restrict implementation of core CRO features (A/B testing, heatmapping, funnel analysis, basic personalization).

---

## Recommended Feature Scope

Based on the analysis above, here is a prioritised feature scope for the project:

**Must-have (MVP)**

- A/B testing with client-side WYSIWYG editor and real-time statistical significance (95%+ confidence)
- Heatmaps: click, scroll, and movement tracking with visual representation
- Session recording: anonymized visitor video playback with rage-click detection
- Conversion funnels: multi-step funnel visualization and drop-off identification
- Visitor segmentation: cohort isolation by traffic source, device, custom attributes
- Dashboard with alerts and trend monitoring
- GDPR/CCPA consent and data retention controls
- Basic reporting: confidence intervals and winner declaration

**Should-have (v1.1)**

- Multivariate testing (MVT): simultaneous testing of multiple variables and combinations
- Server-side testing: backend/API-level experimentation without client-side script limitations
- AI-powered hypothesis generation: automated test ideas from historical winning patterns
- Natural language querying: ask questions about session data and get automated answers (Mina AI style)
- Form analytics: field-level drop-off tracking and hesitation detection
- Dynamic (bandit) traffic allocation: real-time reallocation toward winning variants
- Personalization engine: real-time variant delivery per visitor segment or predicted intent

**Nice-to-have (backlog)**

- Multi-armed bandit optimization: ML-driven traffic allocation that learns in real time
- AI agents for insight generation: automated cohort summarization and opportunity identification
- Qualitative feedback synthesis: NLP-driven analysis of surveys, tickets, and session comments
- Offline-to-online attribution: connect phone calls, chats, and store visits to online experiments
- Cross-channel experimentation coordination: unified A/B testing across email, SMS, push, and web
- Hypothesis lifecycle tracking: connect tests to strategic goals and measure learning ROI
- Automatic experiment anomaly detection: real-time alerting for novelty effects and sample mismatches
- Content testing framework: systematic copy, subject line, and messaging variation testing
