# Conversion Rate Optimization

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform that unifies heatmaps, session recordings, and experimentation with automated hypothesis generation.

Conversion Rate Optimization (CRO) is a candidate project to build a single platform that combines behavioural analytics (heatmaps, session replay, funnels) with experimentation (A/B and multivariate testing) and AI-driven hypothesis generation. It is aimed at e-commerce growth managers, digital product owners, performance marketers, and CRO agencies who today stitch together three to five separate tools to do this work.

---

## Why Conversion Rate Optimization?

- Incumbents specialise: Hotjar, Crazy Egg, and Mouseflow focus on behavioural analytics but offer little or no A/B testing; Optimizely and Kameleoon are testing-first with no native heatmaps or session replay. Teams are forced to operate three to five tools to cover the full CRO workflow.
- Pricing creates a steep cliff: entry tools start at $29–$39/month, mid-market platforms run $199–$500/month, and enterprise platforms (Optimizely, FullStory, Kameleoon) are typically $50,000–$200,000/year, leaving a gap for a capable, self-hostable alternative.
- AI features are gated to high tiers: VWO Copilot, AB Tasty's Evi, FullStory's StoryAI, and Kameleoon's predictive AI are concentrated in proprietary enterprise products, often with sub-processor dependencies (e.g. FullStory routing data through Google Gemini).
- The only mature open-source option in this space is GrowthBook, which is feature-flag and experimentation-led and explicitly lacks heatmaps, session replay, AI hypothesis generation, and a built-in personalization engine.
- Privacy-first architectures (first-party cookies, GDPR/CCPA-native, server-side experimentation) are emerging as buyer requirements, but most incumbents added them as bolt-ons rather than designing for them from day one.

---

## Key Features

### Behavioural Analytics

- Click, scroll, and movement heatmaps with visual overlays
- Session recording with anonymized playback and rage-click detection
- Conversion funnels with multi-step visualization and drop-off identification
- Visitor segmentation by traffic source, device, and custom attributes
- Field-level form analytics for drop-off and hesitation tracking

### Experimentation

- A/B testing with a client-side WYSIWYG editor for variation design
- Multivariate testing (MVT) across multiple variables and combinations
- Server-side and full-stack testing for backend and edge experiments
- Real-time statistical significance reporting with confidence intervals (95%+ standard)
- Dynamic (multi-armed bandit) traffic allocation toward winning variants

### AI-Native Capabilities

- Automated hypothesis generation and prioritization from heatmap and session data
- Natural-language querying of session data to surface relevant recordings and patterns
- AI agents for cohort summarization and opportunity identification
- NLP-driven qualitative feedback synthesis across surveys, tickets, and session comments
- Real-time anomaly detection for novelty effects and sample ratio mismatches

### Personalization

- Real-time variant delivery per visitor segment or predicted intent
- Behavioural and intent-based segmentation
- Content and copy testing with systematic variation

### Privacy and Governance

- GDPR and CCPA consent handling with configurable data retention
- First-party cookie architecture and privacy-first defaults
- Dashboards and configurable alerts for trend monitoring

---

## AI-Native Advantage

Most incumbents bolt AI onto a manual workflow; this project treats AI as the workflow. Heatmaps and session signals feed directly into hypothesis generation and prioritization, replacing PIE/ICE scoring with models trained on historical outcomes. Multi-armed bandits dynamically reallocate traffic, shortening test durations by an estimated 30–50% versus fixed splits. LLMs translate qualitative feedback into structured experiment designs with predicted uplift, and live experiments are continuously monitored for anomalies, novelty effects, and sample ratio mismatches that current tools leave to analyst judgment.

---

## Tech Stack & Deployment

The platform is intended to support both self-hosted and cloud deployment, with an API-first architecture and SDKs for client-side, server-side, and edge experimentation. Statistical analysis covers both frequentist confidence intervals and Bayesian methods. Browser instrumentation builds on the W3C Performance Timing API, and consent gating aligns with GDPR and CCPA requirements. Hypothesis structuring follows established CRO frameworks (PIE/ICE prioritization, the LIFT model).

---

## Market Context

The global CRO software market is estimated at approximately $1.8 billion in 2025, with the adjacent A/B testing tools market valued at ~$969 million in 2025 and projected to reach $1.93 billion by 2031. Heatmap and session recording tools account for roughly 30% of installations and are growing at ~22% CAGR. Primary buyers are e-commerce growth managers, digital product and UX teams, performance marketing leads at DTC brands, enterprise digital experience teams, and CRO agencies reselling tooling to clients.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
