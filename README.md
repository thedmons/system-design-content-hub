# System Design: FinTech Consumer Content Hub
### AEM Component Architecture — MVP & Future State

---

| | |
|---|---|
| **Product** | Consumer Content Hub (fintech.com/stories) |
| **Document Type** | System Design |
| **Version** | 1.0 |
| **Status** | Draft |
| **Date** | Q1 2022 |
| **Platform** | Adobe Experience Manager (AEM) |
| **Author** | Digital Product |
| **Related PRD** | [PRD: FinTech Consumer Content Hub](https://github.com/thedmons/prd-content-hub) |

---

## Table of Contents

1. [Purpose & Scope](#1-purpose--scope)
2. [System Context](#2-system-context)
3. [Architecture Overview](#3-architecture-overview)
4. [AEM Component Model — MVP](#4-aem-component-model--mvp)
5. [Content Model & Taxonomy](#5-content-model--taxonomy)
6. [Authoring Architecture](#6-authoring-architecture)
7. [Content Delivery & Rendering](#7-content-delivery--rendering)
8. [Search Integration](#8-search-integration)
9. [Analytics & Tagging Architecture](#9-analytics--tagging-architecture)
10. [Future State Architecture](#10-future-state-architecture)
11. [Key Design Decisions & Tradeoffs](#11-key-design-decisions--tradeoffs)
12. [Glossary](#12-glossary)

---

## 1. Purpose & Scope

This document describes the AEM component architecture for the FinTech Consumer Content Hub — a dedicated editorial destination at fintech.com/stories delivering financial guidance, product education, and money-life content to consumers.

It covers the MVP system design in full detail, and describes the architectural evolution required to support the post-MVP roadmap phases (Engagement, Personalization, and Delight). It is intended to serve as the technical reference for the AEM squad, product owner, and any engineers working on integrations downstream.

**What this document covers:**
- AEM component architecture and page template structure
- Content model and taxonomy design
- Authoring workflow and publishing pipeline
- Content delivery, rendering, and performance patterns
- Search and analytics integration points
- Future state architectural changes and their implications

**What this document does not cover:**
- Infrastructure provisioning or AEM environment configuration
- CI/CD pipeline setup
- Brand design system specifics

---

## 2. System Context

Before the content hub, the company's financial content lived on a standalone WordPress instance — siloed from the primary AEM platform, disconnected from product pages, and invisible to the broader digital ecosystem. Authors had no shared component library, no taxonomy, and no mechanism to surface related content or drive consumers toward product consideration.

The system design goal is not just a migration. It is a re-architecture that treats the content hub as a connected, scalable platform — one that can evolve from a static editorial destination into a personalized, data-informed content experience.

### 2.1 Current State (Pre-Migration)

```
┌─────────────────────────────────────────────────────────┐
│                    CURRENT STATE                        │
│                                                         │
│  ┌──────────────┐        ┌───────────────────────────┐  │
│  │  WordPress   │        │   AEM (fintech.com)       │  │
│  │  (blog)      │   ✗    │   Product pages           │  │
│  │              │        │   Account portal          │  │
│  │  Standalone  │        │   Marketing pages         │  │
│  │  No taxonomy │        │                           │  │
│  │  No related  │        │   No connection to        │  │
│  │  content     │        │   blog content            │  │
│  └──────────────┘        └───────────────────────────┘  │
│                                                         │
│  Adobe Analytics  ✗  No content event tagging           │
│  Site Search      ✗  WordPress content not indexed      │
│  DAM              ✗  Images stored in WordPress only    │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Target State (MVP)

```
┌────────────────────────────────────────────────────────────────┐
│                    TARGET STATE — MVP                          │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐    │
│  │               AEM (fintech.com)                        │    │
│  │                                                        │    │
│  │  ┌─────────────────┐    ┌──────────────────────────┐  │    │
│  │  │  Content Hub     │    │  Product Pages           │  │    │
│  │  │  /stories        │◄──►│  /mortgage               │  │    │
│  │  │                  │    │  /savings                │  │    │
│  │  │  Hub Homepage    │    │  /auto                   │  │    │
│  │  │  Article Pages   │    │                          │  │    │
│  │  │  Category Pages  │    └──────────────────────────┘  │    │
│  │  └────────┬─────────┘                                  │    │
│  │           │                                            │    │
│  │  ┌────────▼─────────────────────────────────────┐     │    │
│  │  │              AEM DAM                         │     │    │
│  │  │   Article images · Author photos · Assets    │     │    │
│  │  └──────────────────────────────────────────────┘     │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────┐   ┌──────────────────────────────────┐  │
│  │  Adobe Analytics │   │  AEM Search Index                │  │
│  │  Content events  │   │  Articles surfaced site-wide     │  │
│  │  CTA tracking    │   └──────────────────────────────────┘  │
│  └──────────────────┘                                         │
└────────────────────────────────────────────────────────────────┘
```

---

## 3. Architecture Overview

The content hub is built entirely within AEM as a first-class section of fintech.com — not a subdomain, not a separate application. This is a deliberate architectural decision: by living in the same AEM instance as product pages and marketing content, the hub gains access to shared components, the DAM, site search, and the same publish pipeline that serves the rest of the site.

### 3.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AUTHOR TIER                                  │
│                                                                     │
│   AEM Author Instance                                               │
│   ┌──────────────────────────────────────────────────────────┐     │
│   │  Touch UI Authoring                                       │     │
│   │  Component dialogs · Page properties · DAM asset picker  │     │
│   │  Workflow: Draft → Review → Approved → Published         │     │
│   └────────────────────────┬─────────────────────────────────┘     │
│                            │  Replication                          │
└────────────────────────────┼────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│                        PUBLISH TIER                                 │
│                                                                     │
│   AEM Publish Instance(s)                                           │
│   ┌──────────────────────────────────────────────────────────┐     │
│   │  Sling URL resolution                                     │     │
│   │  Component rendering (HTL/Sightly)                        │     │
│   │  Dispatcher cache layer                                   │     │
│   └────────────────────────┬─────────────────────────────────┘     │
│                            │                                        │
└────────────────────────────┼────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│                        DELIVERY TIER                                │
│                                                                     │
│   CDN / Dispatcher                                                  │
│   ┌──────────────────────────────────────────────────────────┐     │
│   │  Static asset caching (images, CSS, JS)                   │     │
│   │  HTML page caching                                        │     │
│   │  301 redirect rules (WordPress → AEM URLs)                │     │
│   └────────────────────────┬─────────────────────────────────┘     │
│                            │                                        │
└────────────────────────────┼────────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   End Users      │
                    │  Browser / App  │
                    └─────────────────┘
```

### 3.2 Page Hierarchy in AEM JCR

The content hub lives under a dedicated node in the AEM JCR (Java Content Repository), structured to mirror the URL taxonomy:

```
/content/fintech/
  └── stories/                          ← Hub root (hub-homepage template)
        ├── home-buying/                ← Category page (category-page template)
        │     ├── how-to-qualify-for-a-mortgage    ← Article (article-page template)
        │     └── first-time-homebuyer-guide       ← Article
        ├── saving/                     ← Category page
        │     └── how-to-build-an-emergency-fund   ← Article
        ├── investing/
        └── auto/
```

This structure maps directly to clean URLs (`fintech.com/stories/home-buying/[article-slug]`) without additional URL mapping configuration.

---

## 4. AEM Component Model — MVP

The MVP requires four page templates and a library of reusable components. All components are built using HTL (HTML Template Language) with backing Sling Models in Java.

### 4.1 Page Templates

| Template | Path | Purpose | Key Regions |
|---|---|---|---|
| `hub-homepage` | `/apps/fintech/templates/hub-homepage` | Root landing page at /stories | Hero, Topic Nav, Article Grid, Editors Picks |
| `category-page` | `/apps/fintech/templates/category-page` | Topic landing (e.g., /stories/saving) | Category Header, Article Listing, Sub-topic Filters |
| `article-page` | `/apps/fintech/templates/article-page` | Individual article | Article Header, Body, Related Articles, Product CTA |
| `author-profile` | `/apps/fintech/templates/author-profile` | Author bio page | Author bio, Article listing by author |

### 4.2 Component Library

Components are organized into three categories: structural (layout), editorial (content), and shared (reusable across site).

#### Structural Components

| Component | Reference | Description |
|---|---|---|
| Hub Header | `fintech/components/structure/hub-header` | Global nav + topic navigation bar specific to /stories |
| Hub Footer | `fintech/components/structure/hub-footer` | Footer with content hub links |
| Article Layout | `fintech/components/structure/article-layout` | Two-column layout: main content + sidebar |
| Category Layout | `fintech/components/structure/category-layout` | Filtered grid layout for topic pages |

#### Editorial Components

| Component | Reference | Description | Priority |
|---|---|---|---|
| Hero Article | `fintech/components/editorial/hero-article` | Featured article with large image, headline, summary, CTA | P0 |
| Article Card | `fintech/components/editorial/article-card` | Thumbnail, headline, topic tag, read time — used in grids and modules | P0 |
| Article Body | `fintech/components/editorial/article-body` | RTE with H2–H4, callout blocks, inline images, pull quotes | P0 |
| Article Header | `fintech/components/editorial/article-header` | Headline, subheadline, hero image, publish date, read time | P0 |
| Author Byline | `fintech/components/editorial/author-byline` | Author name, photo, title — references Author Profile page | P0 |
| Topic Tag | `fintech/components/editorial/topic-tag` | Linked tag rendering taxonomy value — links to category page | P0 |
| Related Articles | `fintech/components/editorial/related-articles` | 3–4 article cards surfaced by tag-matching logic | P0 |
| Table of Contents | `fintech/components/editorial/table-of-contents` | Auto-generated anchor nav from H2 headings in article body | P1 |
| Article Card Grid | `fintech/components/editorial/article-card-grid` | Paginated/load-more grid of article cards | P0 |
| Editors Picks | `fintech/components/editorial/editors-picks` | Manually curated secondary article module | P1 |
| Category Header | `fintech/components/editorial/category-header` | Topic name, description, hero image for category pages | P0 |
| Product CTA | `fintech/components/editorial/product-cta` | End-of-article or inline product callout with configurable headline, body, link | P1 |
| Social Share | `fintech/components/editorial/social-share` | Share buttons: Facebook, X, LinkedIn, copy link | P1 |

#### Shared / Utility Components

| Component | Reference | Description |
|---|---|---|
| SEO Metadata | `fintech/components/shared/seo-metadata` | Renders `<title>`, meta description, canonical, Open Graph, Article schema |
| Breadcrumb | `fintech/components/shared/breadcrumb` | Auto-generated from JCR path |
| Search Bar | `fintech/components/shared/search-bar` | Site search input, submits to AEM search results page |

### 4.3 Component Interaction Diagram

```
article-page template
│
├── seo-metadata          (page <head>)
├── hub-header            (global nav + topic nav)
│
├── article-layout        (main + sidebar regions)
│     ├── [MAIN]
│     │     ├── article-header      (headline, hero, date, read time)
│     │     ├── author-byline       (photo, name, title)
│     │     ├── topic-tag           (linked taxonomy tags)
│     │     ├── table-of-contents   (auto from H2s in body)
│     │     ├── article-body        (RTE — full content)
│     │     ├── product-cta         (inline or end-of-article)
│     │     ├── social-share        (share buttons)
│     │     └── related-articles    (3–4 tag-matched cards)
│     │
│     └── [SIDEBAR]
│           └── product-cta         (sticky sidebar variant, P2)
│
└── hub-footer
```

---

## 5. Content Model & Taxonomy

### 5.1 Content Model

Article pages are stored as JCR nodes under `/content/fintech/stories/`. Each article's properties map directly to the authoring dialog fields and drive both rendering and related content logic.

```
jcr:content/
  ├── headline                (String)
  ├── subheadline             (String, optional)
  ├── heroImage               (String — DAM path)
  ├── heroImageAlt            (String)
  ├── authorRef               (String — path to author-profile page)
  ├── publishDate             (Date)
  ├── lastUpdatedDate         (Date, optional)
  ├── readTime                (Long — minutes)
  ├── primaryTopic            (String — taxonomy path)
  ├── secondaryTags           (String[] — taxonomy paths)
  ├── productAffiliation      (String — taxonomy path, optional)
  ├── bodyContent             (String — HTML from RTE)
  ├── metaTitle               (String, optional override)
  ├── metaDescription         (String)
  ├── canonicalUrl            (String, optional override)
  ├── productCtaHeadline      (String, optional)
  ├── productCtaBody          (String, optional)
  ├── productCtaLink          (String, optional)
  └── relatedArticlesOverride (String[] — paths, optional, max 4)
```

### 5.2 Taxonomy Structure

Taxonomy is managed as AEM Tags (`/content/cq:tags/fintech/`) and referenced by path in article properties. This enables filtering, related content matching, and category page queries.

```
/content/cq:tags/fintech/
  ├── topics/
  │     ├── home-buying
  │     │     ├── mortgages
  │     │     ├── first-time-buyers
  │     │     └── refinancing
  │     ├── saving
  │     │     ├── emergency-fund
  │     │     └── high-yield-savings
  │     ├── investing
  │     ├── auto
  │     └── credit
  │
  └── products/
        ├── mortgage
        ├── hysa
        ├── auto-loan
        └── personal-loan
```

### 5.3 Related Content Logic

The Related Articles component uses a tag-matching query against the JCR to surface relevant articles without requiring an ML recommendation engine at MVP:

```
Query logic (pseudo-code):
1. Get current article's primaryTopic + secondaryTags
2. Query: SELECT articles WHERE primaryTopic = [current]
          OR secondaryTags CONTAINS [any current tag]
          AND path != [current article]
          ORDER BY publishDate DESC
          LIMIT 8 candidates
3. Deduplicate, score by tag overlap
4. Return top 3–4 results
5. If relatedArticlesOverride is set → use those paths directly (skip query)
```

---

## 6. Authoring Architecture

### 6.1 Authoring Workflow

AEM Workflow is configured to enforce a review gate before articles can be published. This prevents content from going live without editorial approval.

```
DRAFT
  │
  │  Author saves article in Touch UI
  ▼
IN REVIEW
  │
  │  Author submits for review (triggers workflow)
  │  Notification sent to Editor / Marketing Owner
  ▼
APPROVED  ──────────────────────────────────────────────────►  REJECTED
  │                                                              │
  │  Editor approves                                            │  Feedback added as
  ▼                                                              │  comment, returns to DRAFT
SCHEDULED / PUBLISHED
  │
  │  Author sets publish date or activates immediately
  ▼
LIVE (replicated to Publish tier)
```

### 6.2 Author Experience Design Principles

The authoring experience is designed around three principles derived from the PRD's author persona requirements:

**1. Guardrails over freedom** — Required fields (headline, hero image, meta description, primary topic) are enforced in the dialog. Authors cannot publish without them. Alt text is required on every image asset selection.

**2. Preview parity** — The Touch UI preview renders the published template, not a simplified editor view. Authors see exactly what readers will see before activating.

**3. No-code publishing** — All article creation, editing, and publishing is achievable without raising an IT ticket. The only engineering dependency is new component creation or template changes.

### 6.3 Component Dialog Design

Each editorial component exposes a Touch UI dialog with clearly labelled fields. Tabs are used to separate content fields from SEO/metadata fields, keeping the authoring surface uncluttered.

```
Article Page Dialog
├── Tab: Content
│     ├── Headline (required)
│     ├── Subheadline
│     ├── Hero Image (DAM picker, required)
│     ├── Hero Image Alt Text (required)
│     ├── Author (page reference picker)
│     ├── Publish Date (date picker)
│     └── Read Time (number, auto or manual)
│
├── Tab: Taxonomy
│     ├── Primary Topic (tag browser, single, required)
│     ├── Secondary Tags (tag browser, multi)
│     └── Product Affiliation (tag browser, single)
│
├── Tab: SEO
│     ├── Meta Title (text, 60 char limit)
│     ├── Meta Description (text, 155 char limit, required)
│     └── Canonical URL (text, optional)
│
└── Tab: Product CTA
      ├── CTA Headline
      ├── CTA Body
      └── CTA Link (path picker)
```

---

## 7. Content Delivery & Rendering

### 7.1 Dispatcher Cache Strategy

AEM Dispatcher acts as the caching and reverse proxy layer between publish instances and the CDN. The content hub has a specific caching strategy that balances freshness with performance targets (LCP < 2.5s, page load < 3s on 4G).

| Content Type | Cache TTL | Invalidation Trigger |
|---|---|---|
| Article pages | 1 hour | On AEM replication (publish/unpublish) |
| Category pages | 15 minutes | On any article publish in that category |
| Hub homepage | 15 minutes | On any article publish or editorial curation change |
| Author profile pages | 24 hours | On author profile edit |
| DAM images | 7 days | On DAM asset update |
| CSS / JS bundles | 30 days | On build deploy (cache-busted by filename hash) |

### 7.2 Image Delivery

All article images are stored in AEM DAM and served through an adaptive image component that generates responsive `srcset` output:

```html
<img
  src="/content/dam/fintech/articles/hero-image.jpg"
  srcset="
    /content/dam/fintech/articles/hero-image.jpg.320.jpg 320w,
    /content/dam/fintech/articles/hero-image.jpg.768.jpg 768w,
    /content/dam/fintech/articles/hero-image.jpg.1200.jpg 1200w
  "
  sizes="(max-width: 768px) 100vw, 1200px"
  alt="[author-configured alt text]"
  loading="lazy"
/>
```

Renditions are pre-generated at publish time by DAM workflow. Hero images on article headers are eager-loaded (no `loading="lazy"`) to protect LCP scores.

### 7.3 URL Structure & 301 Redirects

Clean URLs are resolved by Sling URL mapping. WordPress legacy URLs are redirected at the Dispatcher layer via a redirect map file — not in application code — keeping redirect logic centralized and deployable without code changes.

```
# Dispatcher redirect map (excerpt)
/blog/2021/how-to-qualify-mortgage  →  /stories/home-buying/how-to-qualify-for-a-mortgage
/blog/2020/emergency-fund-guide     →  /stories/saving/how-to-build-an-emergency-fund
```

---

## 8. Search Integration

Articles are indexed by AEM's Lucene-based search (Oak Index) and surfaced through the site's existing search results page. No external search platform is required for MVP.

### 8.1 Search Index Configuration

A custom Oak index definition ensures article-specific fields are indexed and available for full-text and filtered queries:

```
Index: /oak:index/fintech-articles
  ├── type: lucene
  ├── async: true
  ├── includedPaths: /content/fintech/stories
  └── indexedFields:
        ├── headline          (analyzed, stored)
        ├── subheadline       (analyzed)
        ├── bodyContent       (analyzed)
        ├── primaryTopic      (keyword, stored)
        ├── secondaryTags     (keyword, stored, multi-value)
        ├── productAffiliation (keyword, stored)
        ├── publishDate       (date, stored)
        └── metaDescription   (analyzed)
```

### 8.2 Search Result Rendering

When a user searches for a term that matches an article, the existing site search results component renders a content hub result card using the stored `headline`, `metaDescription`, and `publishDate` fields from the index — no additional page render required.

---

## 9. Analytics & Tagging Architecture

All content hub tracking is implemented via Adobe Analytics using the company's standard tag management layer. Tags are defined in the Tagging Spec (separate deliverable) and validated by the Tagging team before go-live.

### 9.1 Page-Level Tags

Every content hub page fires the standard page view beacon with content-hub-specific eVars and props:

| Variable | Type | Value | Purpose |
|---|---|---|---|
| eVar1 | Page section | `content-hub` | Segment all content hub traffic |
| eVar2 | Article topic | `home-buying` | Content performance by topic |
| eVar3 | Product affiliation | `mortgage` | Content-to-product attribution |
| eVar4 | Article slug | `how-to-qualify-for-a-mortgage` | Article-level performance |
| prop1 | Read time bucket | `short / medium / long` | Engagement segmentation |

### 9.2 Interaction Events

| Event | Trigger | Variables Fired |
|---|---|---|
| `content.scroll_depth` | User scrolls 25%, 50%, 75%, 100% of article | eVar4, scroll depth % |
| `content.cta_click` | User clicks Product CTA | eVar3, eVar4, CTA position (inline/end) |
| `content.related_click` | User clicks Related Article | eVar4, clicked article slug |
| `content.social_share` | User clicks Social Share button | eVar4, platform |
| `content.topic_filter` | User selects topic filter on hub homepage | eVar2 |
| `content.search_from_hub` | User submits search from hub search bar | search term |

### 9.3 Tagging Implementation Pattern

Tags are implemented as data attributes on components, read by the tag management layer (TMS) on page load and interaction. This keeps tracking logic decoupled from AEM component code:

```html
<!-- Article page body tag — set at render time by SEO Metadata component -->
<body
  data-page-section="content-hub"
  data-article-topic="home-buying"
  data-article-slug="how-to-qualify-for-a-mortgage"
  data-product-affiliation="mortgage"
>

<!-- Product CTA — fires content.cta_click on click -->
<div class="product-cta"
     data-track-event="content.cta_click"
     data-cta-position="end-of-article"
     data-product="mortgage">
  ...
</div>
```

---

## 10. Future State Architecture

The MVP establishes the foundation. The post-MVP roadmap introduces three phases of architectural evolution, each building on the last.

### 10.1 Phase 2 — Engagement

The Engagement phase adds distribution and measurement capabilities that require new integrations but no changes to the core AEM component model.

**New Components Required:**
- `email-signup` — Integrates with email platform API (e.g., Salesforce Marketing Cloud) via a lightweight form submission service. Captures email + topic preferences.
- `content-series-nav` — Links articles in an ordered editorial series. Series metadata stored as a new JCR node type: `/content/fintech/stories/series/[series-slug]`.

**New Integrations Required:**

```
AEM Publish
    │
    ├──► Email Platform API          (email signup + content digest trigger)
    │
    └──► Analytics Dashboard         (content performance data, pulled from
                                      Adobe Analytics reporting API into
                                      an internal PM dashboard — not in AEM)
```

**Architectural note:** The content performance dashboard is a read-only reporting tool, not a component. It consumes the Adobe Analytics API and is built outside AEM (e.g., Tableau, Looker, or an internal web app). No AEM changes required beyond ensuring eVar coverage is complete.

### 10.2 Phase 3 — Personalization

Personalization is the most architecturally significant phase. It requires a decision on personalization engine and introduces a new data flow between AEM, a CDP or behavioral data platform, and the content delivery layer.

**Recommended Architecture — Rule-Based Personalization (AEM Targeting):**

For the first iteration, use AEM's built-in targeting and segmentation capabilities before introducing an external personalization engine. This keeps the stack consolidated and avoids CDP integration complexity.

```
User visits article
      │
      ▼
AEM Publish resolves page
      │
      ├── Read ContextHub segment (anonymous behavioral data stored client-side)
      │     └── Segments: topic_affinity, visit_count, product_page_visits
      │
      ▼
Related Articles component queries segment-aware variant
      │
      └── Returns topic-affinity-weighted articles
          instead of pure recency-based results
```

**Recommended Architecture — CDP-Driven Personalization (Phase 3b):**

Once authenticated user data is available (Phase 4), personalization can be driven by a CDP:

```
Authenticated User
      │
      ▼
CDP (e.g., Adobe Experience Platform)
      │  User profile: product holdings, life stage,
      │  content history, propensity scores
      │
      ▼
AEM Content Fragment API or Headless endpoint
      │  Returns personalized content recommendations
      │  as JSON, rendered client-side
      │
      ▼
Related Articles component (client-side hydration)
```

**New Components Required:**
- `topic-preferences` — UI for users to follow/unfollow topics. Stored in ContextHub (anonymous) or CDP profile (authenticated).
- `sentiment-reaction` — Lightweight thumbs up/topic feedback widget. Writes to analytics event stream; consumed by personalization logic in later iterations.

### 10.3 Phase 4 — Delight

The Delight phase introduces the highest complexity changes: user authentication, visual story formats, and in-house interactive tools.

**Visual Stories:**

Visual stories (short-form, mobile-optimized, social-linked content) require a new AEM page template and a new component set distinct from the long-form article pattern:

```
New template: visual-story-page
  ├── story-slide (full-bleed image + text overlay, swipeable)
  ├── story-progress-bar (auto-advance or tap-to-advance)
  └── story-cta (end card with product link)

URL pattern: fintech.com/stories/visual/[story-slug]
Social deep link: og:url resolves to visual story template
```

**User Authentication:**

Authentication requires coordination with the company's identity platform (SSO/OAuth). AEM is not the identity provider — it is a consumer of the auth token:

```
User logs in via SSO
      │
      ▼
Identity Provider issues JWT
      │
      ▼
AEM reads JWT from cookie/header
      │
      ├── Unlocks authenticated ContextHub profile
      ├── Enables saved articles (stored in user profile service)
      └── Passes user ID to CDP for full personalization
```

**In-House Tools & Quizzes:**

Interactive tools (mortgage calculators, savings goal quizzes) are built as AEM components that encapsulate client-side logic. They do not require a backend service for basic calculations:

```
New components:
  ├── calculator-tool      (client-side JS, result renders inline)
  ├── quiz-module          (multi-step, result maps to product/article)
  └── assessment-widget    (scoring logic → recommended content path)
```

### 10.4 Architecture Evolution Summary

| Phase | AEM Changes | New Integrations | Complexity |
|---|---|---|---|
| MVP | 4 templates, 15 components | Adobe Analytics, DAM, Search | Medium |
| Phase 2 — Engagement | 2 new components | Email platform API | Low |
| Phase 3 — Personalization | 2 new components, ContextHub config | CDP (optional), AEM Targeting | High |
| Phase 4 — Delight | 1 new template, 5+ new components | Identity provider / SSO | High |

---

## 11. Key Design Decisions & Tradeoffs

### Decision 1: Single AEM Instance vs. Separate Content Hub Instance

**Decision:** Build the content hub within the existing AEM instance rather than standing up a separate instance.

**Rationale:** Shared DAM, shared component library, shared search index, and shared publish pipeline reduce operational overhead and enable native product page integration. A separate instance would require API bridges for every integration point and add significant DevOps complexity.

**Tradeoff:** The content hub shares infrastructure with the rest of fintech.com. High traffic events on the hub (e.g., viral article) could affect publish performance for other sections if not properly isolated at the Dispatcher/CDN layer. Mitigated by aggressive Dispatcher caching on content hub pages.

---

### Decision 2: Rule-Based Related Content vs. ML Recommendations at MVP

**Decision:** Use tag-matching JCR queries for related content at MVP rather than an ML recommendation engine.

**Rationale:** ML recommendations require behavioral data at scale, a model training pipeline, and a serving infrastructure — all out of scope for MVP. Tag-matching provides deterministic, author-controllable related content with zero additional infrastructure.

**Tradeoff:** Related content quality is limited by taxonomy quality and tag coverage. Articles with sparse tagging will surface weaker recommendations. Mitigated by making the `relatedArticlesOverride` field easy to use in authoring, and by investing in taxonomy governance at launch.

**Future path:** Phase 3 personalization introduces behavioral weighting on top of the same tag-matching query before any ML engine is introduced.

---

### Decision 3: AEM Tags for Taxonomy vs. External Taxonomy Service

**Decision:** Use AEM's native tag system (`/content/cq:tags`) as the taxonomy store rather than an external MDM or taxonomy service.

**Rationale:** AEM Tags are natively integrated with the Touch UI tag browser, page property dialogs, and JCR queries. An external taxonomy service would require a custom integration for every touchpoint.

**Tradeoff:** AEM Tags are not designed for complex hierarchical taxonomies or cross-system sharing. If taxonomy needs to be shared with the mobile app, email platform, or CDP in later phases, a synchronization mechanism will be needed. Acceptable for MVP scope.

---

### Decision 4: Dispatcher Redirect Map vs. Application-Level Redirects

**Decision:** Implement WordPress-to-AEM 301 redirects as a Dispatcher redirect map file rather than in AEM application code.

**Rationale:** Dispatcher-level redirects are resolved before the request reaches AEM, reducing load on publish instances. The redirect map file is a flat text file deployable independently of AEM code, allowing SEO team to manage additions without engineering sprints.

**Tradeoff:** Redirect map changes require a Dispatcher configuration deploy (not an AEM publish). Acceptable given that redirect additions post-launch are expected to be infrequent.

---

## 12. Glossary

| Term | Definition |
|---|---|
| AEM | Adobe Experience Manager — enterprise CMS platform used to build and manage fintech.com |
| AEM DAM | Digital Asset Management — AEM's repository for images, PDFs, and other media |
| AEM Dispatcher | Caching and reverse proxy layer that sits in front of AEM Publish instances |
| ContextHub | AEM's client-side data layer for storing and reading user segment data (anonymous) |
| CDP | Customer Data Platform — a system that unifies customer profile data across channels for personalization |
| HTL | HTML Template Language (also called Sightly) — AEM's server-side templating language for component rendering |
| JCR | Java Content Repository — the underlying data store for all AEM content, structured as a node tree |
| Oak Index | AEM's Lucene-based search index built on Apache Jackrabbit Oak |
| Sling | Apache Sling — the web framework AEM is built on; handles URL resolution and request routing |
| Sling Model | Java annotations-based model layer that maps JCR node properties to Java objects for use in HTL templates |
| Touch UI | AEM's modern authoring interface (as opposed to the legacy Classic UI) |
| eVar | Adobe Analytics custom conversion variable used to capture and report on content-specific dimensions |
