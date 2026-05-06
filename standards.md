# Standards & API Reference

> Project: Conversion Rate Optimization · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

- **ISO 9241-210:2019 — Human-centred design for interactive systems** — https://www.iso.org/standard/77520.html — Defines the human-centred design process underpinning UX experimentation and conversion analysis.
- **ISO/IEC 25010:2011 — Systems and software Quality Requirements and Evaluation (SQuaRE)** — https://www.iso.org/standard/35733.html — Quality model used to assess usability metrics (effectiveness, efficiency, satisfaction) that correlate with conversion outcomes.
- **ISO/IEC 27001:2022 — Information security management systems** — https://www.iso.org/standard/27001 — Required posture for SaaS CRO platforms handling visitor session data.
- **ISO/IEC 29100:2024 — Privacy framework** — https://www.iso.org/standard/85938.html — Privacy principles applicable to session recording, heatmap, and behavioural analytics data.
- **ISO/IEC 20889:2018 — Privacy-enhancing data de-identification techniques** — https://www.iso.org/standard/69373.html — Guidance for anonymising session replay and click data before analysis.
- **ISO 20252:2019 — Market, opinion and social research** — https://www.iso.org/standard/73671.html — Quality framework for survey/voice-of-customer modules embedded in CRO tools.

### W3C & IETF Standards

- **W3C Navigation Timing Level 2** — https://www.w3.org/TR/navigation-timing-2/ — Browser API providing page-load metrics that feed CRO performance analysis.
- **W3C Resource Timing Level 3** — https://www.w3.org/TR/resource-timing-3/ — Per-resource latency data informing site-speed conversion correlation.
- **W3C Largest Contentful Paint** — https://www.w3.org/TR/largest-contentful-paint/ — Core Web Vitals metric strongly correlated with conversion.
- **W3C Element Timing API** — https://www.w3.org/TR/element-timing/ — Granular element render timing for above-the-fold optimisation.
- **W3C Long Tasks API** — https://www.w3.org/TR/longtasks-1/ — Detects main-thread jank that depresses conversion.
- **W3C Tracking Preference Expression (DNT)** — https://www.w3.org/TR/tracking-dnt/ — Honours visitor opt-outs from behavioural tracking.
- **W3C Web Analytics Working Group / Privacy CG** — https://www.w3.org/community/privacycg/ — Forum for privacy-respecting analytics primitives (e.g., Private Aggregation, Attribution Reporting).
- **RFC 6265 — HTTP State Management Mechanism (Cookies)** — https://www.rfc-editor.org/rfc/rfc6265 — Cookie semantics underlying visitor identity for experiments.
- **RFC 7234 / RFC 9111 — HTTP Caching** — https://www.rfc-editor.org/rfc/rfc9111 — Cache directives critical for serving experiment variants without flicker.
- **RFC 8446 — TLS 1.3** — https://www.rfc-editor.org/rfc/rfc8446 — Required transport security for tracking beacons.
- **RFC 9110 — HTTP Semantics** — https://www.rfc-editor.org/rfc/rfc9110 — Underlying semantics for experiment APIs and beacon endpoints.

### Data Model & API Specifications

- **OpenAPI Specification 3.1** — https://spec.openapis.org/oas/v3.1.0 — Standard for describing REST APIs exposed by CRO platforms.
- **AsyncAPI 3.0** — https://www.asyncapi.com/docs/reference/specification/v3.0.0 — Useful for streaming clickstream/event channels.
- **JSON Schema 2020-12** — https://json-schema.org/specification — Validates experiment, variant, and event payloads.
- **GraphQL (October 2021 spec)** — https://spec.graphql.org/October2021/ — Used by Optimizely Data Platform and others for flexible reporting queries.
- **CloudEvents 1.0 (CNCF)** — https://github.com/cloudevents/spec — Standard envelope for emitting conversion/event data across systems.
- **OpenTelemetry — Semantic Conventions for HTTP & Browser** — https://opentelemetry.io/docs/specs/semconv/ — Useful for correlating experiments with performance traces.
- **Segment Spec (Track/Identify/Page)** — https://segment.com/docs/connections/spec/ — De-facto event taxonomy for marketing/CRO tooling.
- **OpenFeature Specification** — https://openfeature.dev/specification/ — Vendor-neutral feature-flag/experimentation API standard (CNCF incubating).
- **W3C Decentralized Identifiers (DID)** — https://www.w3.org/TR/did-core/ — Emerging standard for privacy-preserving visitor identity.
- **IAB Tech Lab — Open Measurement SDK (OM SDK)** — https://iabtechlab.com/standards/open-measurement-sdk/ — Standard for viewability/measurement that complements CRO instrumentation.
- **IAB Global Privacy Platform (GPP)** — https://iabtechlab.com/gpp/ — Unified consent string format applicable to CRO data collection.

### Security & Authentication Standards

- **OAuth 2.0 (RFC 6749) and OAuth 2.1 draft** — https://datatracker.ietf.org/doc/html/rfc6749 / https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/ — Standard delegated auth for platform APIs and integrations.
- **OpenID Connect Core 1.0** — https://openid.net/specs/openid-connect-core-1_0.html — SSO for CRO admin consoles.
- **RFC 7519 — JSON Web Token (JWT)** — https://www.rfc-editor.org/rfc/rfc7519 — Common bearer token for API access and signed visitor assignments.
- **OWASP ASVS 4.0** — https://owasp.org/www-project-application-security-verification-standard/ — Application security baseline for SaaS CRO products.
- **OWASP Top 10 (2021)** — https://owasp.org/www-project-top-ten/ — Risk checklist for the experimentation web layer.
- **NIST SP 800-53 Rev. 5** — https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final — Security and privacy controls relevant to enterprise CRO deployments.
- **NIST Privacy Framework 1.0** — https://www.nist.gov/privacy-framework — Privacy risk-management aligned with session-data collection.
- **GDPR (EU 2016/679)** — https://gdpr-info.eu/ — Lawful basis, consent, and DSR requirements for behavioural tracking.
- **CCPA / CPRA** — https://oag.ca.gov/privacy/ccpa — California disclosure and opt-out obligations affecting CRO scripts.
- **ePrivacy Directive (2002/58/EC, "Cookie Law")** — https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32002L0058 — Consent requirements for cookies and similar trackers.
- **SOC 2 (AICPA Trust Services Criteria)** — https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2 — Common enterprise procurement requirement for CRO SaaS.
- **WCAG 2.2** — https://www.w3.org/TR/WCAG22/ — Accessibility baseline; CRO experiments must not regress accessibility.

### MCP Server Specifications

- **Model Context Protocol — Specification** — https://modelcontextprotocol.io/specification — Vendor-neutral protocol for exposing tools/resources to LLM clients; relevant for AI-native CRO assistants connecting to analytics, experiment platforms, and CMS.
- **MCP Reference Servers** — https://github.com/modelcontextprotocol/servers — Examples (Postgres, GitHub, fetch) usable as patterns for an "experiments" or "analytics" MCP server.
- **MCP TypeScript SDK** — https://github.com/modelcontextprotocol/typescript-sdk — SDK for implementing CRO-side MCP integrations.
- **MCP Python SDK** — https://github.com/modelcontextprotocol/python-sdk — SDK for Python-based CRO/analytics MCP servers.

## Similar Products — Developer Documentation & APIs

### Optimizely (Web Experimentation & Feature Experimentation)
- **Description:** Enterprise experimentation platform with web, full-stack, and feature-flag testing.
- **API Documentation:** https://docs.developers.optimizely.com/web-experimentation/reference and https://docs.developers.optimizely.com/feature-experimentation/reference
- **SDKs/Libraries:** JavaScript, Python, Java, Ruby, PHP, C#, Go, Swift, Android, React — https://docs.developers.optimizely.com/feature-experimentation/docs/sdk-reference-guides
- **Developer Guide:** https://docs.developers.optimizely.com/
- **Standards:** REST/JSON, OpenAPI, GraphQL (Optimizely Data Platform)
- **Authentication:** OAuth 2.0, Personal Access Tokens

### VWO (Visual Website Optimizer)
- **Description:** A/B testing, heatmaps, funnels, and personalization platform.
- **API Documentation:** https://developers.vwo.com/reference
- **SDKs/Libraries:** Server-side SDKs in Node.js, Java, Python, Ruby, PHP, .NET, Go — https://developers.vwo.com/v2.0/docs/fme-sdks
- **Developer Guide:** https://developers.vwo.com/docs
- **Standards:** REST/JSON
- **Authentication:** API Token (Bearer)

### Hotjar (Contentsquare)
- **Description:** Heatmaps, session recordings, surveys, and feedback widgets.
- **API Documentation:** https://help.hotjar.com/hc/en-us/categories/115001317847-Hotjar-Tracking-Code-and-API
- **SDKs/Libraries:** JavaScript tracking snippet; Identify API; Events API
- **Developer Guide:** https://help.hotjar.com/hc/en-us/articles/115011867948
- **Standards:** REST/JSON, JavaScript snippet
- **Authentication:** Site ID + API key

### FullStory
- **Description:** Digital experience intelligence with session replay, funnels, and AI insights.
- **API Documentation:** https://developer.fullstory.com/server/v2/introduction/
- **SDKs/Libraries:** Browser, iOS, Android, React Native, server SDKs
- **Developer Guide:** https://developer.fullstory.com/
- **Standards:** REST/JSON, OpenAPI 3
- **Authentication:** API Key (Bearer)

### Crazy Egg
- **Description:** Heatmaps, scrollmaps, recordings, and lightweight A/B testing.
- **API Documentation:** https://help.crazyegg.com/article/57-api-documentation
- **SDKs/Libraries:** JavaScript snippet; segment integration
- **Developer Guide:** https://help.crazyegg.com/category/179-developers
- **Standards:** REST/JSON
- **Authentication:** API Key

### Convert.com
- **Description:** Privacy-first A/B and multivariate testing.
- **API Documentation:** https://api.convert.com/doc/serving/
- **SDKs/Libraries:** JavaScript, Node.js, PHP server-side SDKs — https://github.com/Convertcom
- **Developer Guide:** https://www.convert.com/developer/
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** HMAC signed requests, API Key

### AB Tasty
- **Description:** Experimentation and personalization with AI-assisted prioritization.
- **API Documentation:** https://developers.abtasty.com/
- **SDKs/Libraries:** JavaScript, Node.js, Python, Java, PHP, Go, .NET, Swift, Android (Flagship SDKs)
- **Developer Guide:** https://docs.developers.flagship.io/
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0, API Key

### Kameleoon
- **Description:** A/B testing and personalization with predictive segments and server-side experiments.
- **API Documentation:** https://developers.kameleoon.com/
- **SDKs/Libraries:** JavaScript, Java, Node.js, PHP, Python, Ruby, Go, .NET, Swift, Kotlin
- **Developer Guide:** https://developers.kameleoon.com/docs/sdk-reference-guides
- **Standards:** REST/JSON, server-sent events for streaming flag updates
- **Authentication:** OAuth 2.0, Client Credentials

### Mouseflow
- **Description:** Session replay, heatmaps, funnels, and form analytics.
- **API Documentation:** https://api.mouseflow.com/
- **SDKs/Libraries:** JavaScript snippet; REST API
- **Developer Guide:** https://docs.mouseflow.com/en/articles/4014097-mouseflow-api
- **Standards:** REST/JSON
- **Authentication:** API Key

### LaunchDarkly (adjacent — feature experimentation)
- **Description:** Feature flag and experimentation platform widely used for product CRO.
- **API Documentation:** https://apidocs.launchdarkly.com/
- **SDKs/Libraries:** 25+ SDKs across server, client, and mobile — https://docs.launchdarkly.com/sdk
- **Developer Guide:** https://docs.launchdarkly.com/home/getting-started
- **Standards:** REST/JSON, OpenAPI 3, server-sent events for flag streaming, OpenFeature provider
- **Authentication:** API Access Tokens, OAuth 2.0

### Statsig (adjacent — experimentation analytics)
- **Description:** Feature gating, experimentation, and product analytics platform.
- **API Documentation:** https://docs.statsig.com/server/introduction
- **SDKs/Libraries:** Node, Python, Java, Go, Ruby, PHP, .NET, Erlang, Rust; client SDKs for JS, iOS, Android, React Native
- **Developer Guide:** https://docs.statsig.com/
- **Standards:** REST/JSON, OpenFeature provider
- **Authentication:** Server Secret Key, Client SDK Key

### Segment (adjacent — event collection backbone)
- **Description:** Customer data platform commonly used to pipe events into CRO tools.
- **API Documentation:** https://segment.com/docs/connections/sources/catalog/libraries/server/http-api/
- **SDKs/Libraries:** Analytics.js, iOS, Android, Node, Python, Ruby, Java, Go, .NET, PHP
- **Developer Guide:** https://segment.com/docs/
- **Standards:** REST/JSON, Segment Spec event taxonomy
- **Authentication:** Write Key (Basic Auth)

## Notes

- **OpenFeature** is rapidly becoming the vendor-neutral standard for feature-flag/experimentation APIs and is a strong candidate baseline for an AI-native CRO project's flag/variant interface.
- **Privacy Sandbox APIs** (Attribution Reporting, Private Aggregation, Topics) from Chrome are still evolving and will reshape how CRO measurement works post third-party cookies. Track at https://privacysandbox.google.com/.
- **MCP** is new (2024–2025); no incumbent CRO vendor yet ships official MCP servers, representing a green-field opportunity to expose experiments, heatmaps, and funnels as MCP tools/resources for AI agents.
- **W3C Private Advertising Technology CG** work on attribution may provide future standard primitives for conversion measurement without per-user tracking.
- Several incumbent APIs (Hotjar, Crazy Egg, Mouseflow) are notably thinner than enterprise platforms; an open, OpenAPI-described interface would be a meaningful differentiator.
