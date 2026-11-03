# Afterlight — Development Plan

**Document status:** Proposed execution plan  
**Version:** 1.0  
**Product status:** UI prototype exists; production application has not started  
**Primary goal:** Turn the Afterlight prototype into a reliable, installable, offline-capable short-fiction product with privacy-conscious personalisation.

> **Product promise:** A whole new world, in 15 minutes.

This document is the implementation roadmap and the source of truth for sequencing the work. Build the phases in order. A phase is complete only when its acceptance criteria pass; do not mark a phase complete merely because its screens or files exist.

---

## 0. Product and engineering principles

1. **The story is the product.** Every technical decision should make it easier to discover, read, finish, save, or return to a good story.
2. **Offline reading is a core capability.** The reader must be designed around a local-first data access layer, not retrofitted after the online product is finished.
3. **Cloud is authoritative; local storage is a resilient working copy.** Supabase is the cloud source of truth. IndexedDB stores downloaded content, local preferences, progress, and a queue of changes waiting to sync.
4. **Personalisation must be useful and explainable.** Learn from relevant reading signals and direct preferences. Do not collect data just because it is technically possible.
5. **No manipulative retention.** Do not optimise for endless time in the app, punitive streaks, fake urgency, or mid-story interruptions.
6. **Security is enforced by the database.** Use Clerk for identity and Supabase Row Level Security (RLS) for data access. Never trust frontend checks as authorisation.
7. **Ship a small, polished catalogue first.** A few genuinely good stories are more valuable than a large collection of repetitive stories.
8. **Keep the first architecture boring.** Prefer the simplest system that supports the product. Add queues, model-based ranking, billing, and complex infrastructure only when evidence justifies them.

## 1. Target architecture

### Selected stack

| Layer | Choice | Responsibility |
| --- | --- | --- |
| Web application | React + TypeScript | UI, application logic, reusable components |
| Build and development | Vite | Fast local development and production bundling |
| Routing | React Router | Screen and URL navigation |
| Server-state/cache | TanStack Query | Fetch lifecycle, request caching, invalidation and retries |
| Authentication | Clerk React SDK | Sign-up, sign-in, sessions and account management |
| Cloud database and APIs | Supabase PostgreSQL | Story catalogue, user library, progress, preferences and events |
| Media/content storage | Supabase Storage | Story artwork and published content assets |
| Local database | Dexie.js over IndexedDB | Downloaded stories, local library state and offline change queue |
| PWA | vite-plugin-pwa / Workbox | Web app manifest, service worker and static asset caching |
| Validation | Zod | Runtime validation at network, storage and form boundaries |
| Hosting | Vercel | HTTPS hosting and deployment from Git |

Use Supabase Edge Functions or another trusted server-side runtime only for work that needs privileged credentials, webhooks, scheduled processing, or protected model-provider keys. The Supabase service-role key must never be shipped to the browser.

### High-level architecture

\`\`\`text
Reader
  |
  v
React + TypeScript PWA
  |
  +-- UI / feature modules
  |
  +-- Application services
        |
        +-- Story repository ------> IndexedDB (Dexie)
        |          |
        |          +---------------> Supabase (when online)
        |
        +-- Reading / library service
        |          |
        |          +---------------> Local-first writes + sync outbox
        |
        +-- Identity service ------> Clerk
        |
        +-- Analytics service -----> Event validation / consent
        |                              |
        |                              +--> Supabase events
        |
        +-- Recommendation service -> Catalogue candidates + reader signals
                                       |
                                       +--> Ranked story IDs
\`\`\`

The UI must call application services such as \`getStory()\`, \`saveProgress()\`, and \`getLibrary()\`. Components must not independently decide whether data comes from Supabase or IndexedDB.

### Repository direction

Keep the existing \`prototype/\` as a visual reference until the production UI has parity. Build the production application in a clearly named directory (recommended: \`app/\`) or move the prototype only after agreeing on the new structure. Do not casually overwrite the prototype before its useful design decisions have been carried forward.

---

# Phase 1 — Product specification and technical foundation

**Objective:** Remove ambiguity before implementation. Convert the product foundation into testable behaviour and make the key architectural decisions explicit.

### Work

- Review \`README.md\`, \`docs/PRODUCT_FOUNDATION.md\`, and the existing UI prototype.
- Confirm the first supported audience, initial genres, story length range, content standards, and reading contexts.
- Define the MVP user journey: first visit → onboarding → discovery → preview → read → completion → feedback → next recommendation → library.
- Define account states: signed out, online signed in, offline returning reader, expired session, and signed out on a shared device.
- Define what must work offline and what must clearly require a connection.
- Confirm React + TypeScript + Vite, Clerk, Supabase, Dexie, and PWA as the initial stack.
- Define coding conventions, TypeScript strictness, environment-variable naming, error handling, accessibility expectations, and test strategy.
- Create an architecture decision record for any decision that would be expensive to reverse.
- Define a small initial catalogue and an editorial quality checklist. Do not build mass story generation yet.

### Deliverables

- MVP scope and explicit non-goals.
- User-flow and screen requirements.
- Technology decision record.
- Initial content and moderation standards.
- Acceptance criteria for the online and offline journeys.

### Exit criteria

- Every MVP screen and primary user action has a written requirement.
- Offline behaviour is defined for each user journey.
- No unresolved architecture decision blocks Phase 2.
- The team can explain what is deliberately excluded from the first release.

---

# Phase 2 — Application foundation, design system and PWA shell

**Objective:** Create the production application and a fast, accessible foundation before connecting real user data.

### Work

- Initialise the Vite + React + TypeScript application.
- Enable strict TypeScript and linting/formatting checks.
- Add React Router and a clear route structure.
- Add TanStack Query for server-state handling.
- Add Zod for runtime validation of external data.
- Establish environment variable handling and safe example environment files. Commit only placeholders, never secrets.
- Create the shared design tokens from the prototype: charcoal/warm-black surfaces, muted amber accents, warm cream reading surface, serif story typography, and readable sans-serif interface typography.
- Build shared components: buttons, inputs, navigation, story cards, badges, dialogs, loading states, empty states, error states, and accessible feedback controls.
- Implement responsive layouts for narrow mobile screens, tablets, and desktop.
- Add the PWA manifest, icons, install metadata, and a minimal service worker.
- Cache the application shell and versioned static assets. Do not broadly cache authenticated API responses.
- Provide a deliberate update flow when a new service worker version is available; do not unexpectedly interrupt a reading session.

### Deliverables

- Production app boots locally and builds for production.
- Shared design system and responsive layout.
- Route skeleton for Home, Discover, Story Preview, Reader, Completion, Library, and Settings.
- Installable PWA shell with an offline fallback screen.
- Lint, type-check, and build scripts.

### Exit criteria

- The app installs on supported test devices over HTTPS.
- Main routes render without console errors.
- The layout is usable at mobile and desktop widths.
- With the network disabled after the first successful load, the app shell opens and explains which features are unavailable.
- No private key or privileged database credential appears in the client bundle.

---

# Phase 3 — Supabase data model, migrations and security

**Objective:** Design a clean cloud schema before application features begin depending on it.

### Initial data model

Create versioned SQL migrations for the following conceptual tables. Exact columns and indexes should be documented in the migrations.

| Table | Purpose | Important notes |
| --- | --- | --- |
| \`app_users\` | App-side user record keyed by Clerk user ID | Store only fields the product needs |
| \`stories\` | Published story catalogue | Stable ID, title, synopsis, status, reading-time estimate, content version |
| \`story_revisions\` | Immutable or versioned published story content | Lets downloads identify the exact content version |
| \`genres\` / \`story_genres\` | Controlled genre taxonomy | Avoid arbitrary, unreviewed tags |
| \`series\` / \`series_stories\` | Connected story worlds and ordering | Optional for standalone stories |
| \`story_assets\` | Cover and supporting asset metadata | Store public/private access rules explicitly |
| \`user_preferences\` | Declared genres, moods and recommendation controls | Editable and deletable by the reader |
| \`user_library\` | Saved, completed and in-progress membership | Unique user/story pair |
| \`reading_progress\` | Current location and progress state | Version/timestamp strategy must be defined |
| \`bookmarks\` | Reader-created bookmarks | User-owned |
| \`reading_events\` | Allowlisted behavioural events | Retention and deletion rules required |
| \`recommendation_feedback\` | Helpful/not-helpful, hide and rating feedback | Keep separate from raw impressions |
| \`recommendation_impressions\` | What was recommended, where and under which version | Needed to evaluate recommendations |

Do not create every table in one unreviewed migration. Add only fields that have a concrete use in the MVP.

### Work

- Configure Supabase project and local development workflow.
- Store schema changes as repeatable, ordered SQL migrations in the repository.
- Add primary keys, foreign keys, uniqueness constraints, indexes, timestamps, and sensible delete behaviour.
- Enable RLS on every user-owned table.
- Use Clerk's native Supabase third-party authentication integration. Pass the Clerk session token through the Supabase client. Do not start a new implementation with the deprecated Clerk JWT-template integration.
- Write policies using the verified Clerk subject claim, for example comparing \`auth.jwt()->>'sub'\` with the row's \`user_id\`.
- Ensure public catalogue reads expose only published, approved stories.
- Ensure users can read and modify only their own private records.
- Keep admin/editor publishing operations separate from normal reader permissions.
- Add seed data for a tiny development catalogue, never real user data.
- Add automated RLS tests with at least two separate test identities.

### Deliverables

- Supabase project configuration and migrations.
- ER diagram or schema overview.
- RLS policies for every user-owned table.
- Typed data contracts shared by application services.
- Local seed catalogue and security tests.

### Exit criteria

- A signed-in user can access only their own library, bookmarks, preferences, and progress.
- A second test user cannot read or change the first user's records, even by guessing IDs.
- Anonymous access cannot write private data.
- Unpublished stories are not exposed through reader-facing queries.
- Migrations can build a clean development database from scratch.

---

# Phase 4 — Clerk authentication and account lifecycle

**Objective:** Make identity reliable online and safe when the reader returns offline.

### Work

- Create the Clerk application and configure the React SDK.
- Build sign-up, sign-in, sign-out, and account-management routes.
- Connect Clerk to Supabase through its native third-party auth integration.
- Create one central Supabase client that obtains the current Clerk session token safely.
- Define the user bootstrap flow: authenticated Clerk identity → app user row → initial preferences → library initialisation.
- Handle loading, expired session, revoked session, network failure, and account deletion states.
- Partition all local IndexedDB records by Clerk user ID. Never allow one signed-in account to see another account's local library.
- Permit a previously authenticated reader to open already-downloaded content offline. Explain that account changes, new downloads, cloud actions, and fresh authentication require connectivity.
- On sign-out, stop sync immediately and clear or securely isolate the active account's private local data according to the documented product policy. Never leave one account's library visible to the next account on a shared device.
- Add an account deletion workflow that removes or schedules removal of the user's personal data and local cache according to the published policy.

### Deliverables

- Working Clerk screens and protected routes.
- Native Clerk–Supabase token integration.
- User bootstrap and account lifecycle service.
- Account-partitioned local storage.
- Sign-out and account deletion behaviour.

### Exit criteria

- New and returning users can authenticate and reach the app.
- Supabase RLS recognises the Clerk identity.
- Signing out prevents further private cloud writes.
- Offline users can access only the local content previously made available to that same account.
- Switching accounts never leaks the previous account's local records.

---

# Phase 5 — Story catalogue and publishing workflow

**Objective:** Make story content trustworthy, structured, versioned and ready for both online delivery and offline downloads.

### Work

- Define the story content format. Prefer structured metadata plus safe, renderable content blocks rather than arbitrary executable HTML.
- Define required metadata: stable story ID, title, hook/synopsis, genre, mood, reading-time estimate, cover, series/order, publication status, content notes, language, author/editorial attribution, and content version.
- Establish editorial states such as draft, review, approved, published, and archived.
- Build a minimal internal publishing workflow. It may initially use protected admin tools or reviewed seed/import scripts; a full CMS is not required for the MVP.
- Validate every published story with Zod and content-quality checks.
- Store story text and assets in the appropriate Supabase tables/storage locations with explicit access policies.
- Add a small curated catalogue that is representative of the target genres.
- Create a content-version strategy so an offline download can identify which exact version it contains.
- Define how removed or updated stories behave on devices that already downloaded an older version.
- Add appropriate age guidance and content notices where required by the story.

### Deliverables

- Story schema and content renderer contract.
- Editorial publish/review checklist.
- Protected content import/publishing workflow.
- Curated development catalogue with covers and metadata.
- Content version and asset delivery strategy.

### Exit criteria

- A story cannot become visible to readers until it passes validation and approval.
- Published content renders consistently on mobile and desktop.
- The same story ID maps to a known content version.
- A content update does not silently corrupt an existing offline download.
- Catalogue queries return only eligible, published stories.

---

# Phase 6 — Core reading product and library

**Objective:** Implement the complete product loop using real data and a repository abstraction that can later serve both online and offline sources.

### Work

- Implement Home with a featured story, continue reading, curated categories, new releases, and a small personalised section.
- Implement Discover with controlled genre/mood filters and clear loading/empty/error states.
- Implement Story Preview with hook, artwork, genre, reading-time estimate, content notices, and Start Reading.
- Implement the Reader as the most carefully tested screen:
  - comfortable type scale, line length, spacing and contrast;
  - reader-controlled font size and theme;
  - subtle progress indicator;
  - bookmark action;
  - reliable resume;
  - keyboard and screen-reader support where applicable;
  - no ad or recommendation interruption inside a story.
- Implement Completion with a calm ending state, feedback, save action, and one relevant next-story suggestion.
- Implement Library shelves for Continue Reading, Saved, Completed, and Downloads.
- Build application services such as \`getStory\`, \`getDiscoverFeed\`, \`getLibrary\`, \`saveProgress\`, \`toggleBookmark\`, and \`completeStory\`.
- Keep UI components independent from Supabase and Dexie implementation details.
- Save reading progress at sensible checkpoints and on lifecycle transitions. Avoid sending a request for every scroll event.
- Add basic accessibility and performance checks.

### Deliverables

- Functional Home, Discover, Story Preview, Reader, Completion, and Library.
- Repository/service layer for stories and reader state.
- Progress, bookmarks, and completion persistence.
- Error, empty, loading, and retry states.
- End-to-end online reading journey.

### Exit criteria

- A new reader can discover, preview, start, leave, resume, finish, rate, and save a story.
- Progress survives reloads and a normal browser restart.
- A reader can find their saved and completed stories in the library.
- Reader preferences persist.
- The UI never assumes that a network request succeeded before confirming or reconciling its result.

---

# Phase 7 — Offline-first storage, downloads and synchronisation

**Objective:** Make Afterlight genuinely useful without an internet connection, not merely installable.

### Offline feature contract

| Capability | Online | Offline |
| --- | --- | --- |
| Open app shell and navigation | Yes | Yes, after first successful app load |
| Read downloaded stories | Yes | Yes |
| Browse downloaded library | Yes | Yes |
| Save local progress and bookmarks | Yes | Yes, queued for sync |
| Change local reading preferences | Yes | Yes, queued for sync |
| Download a new story | Yes | No |
| Browse the live catalogue | Yes | No; show cached/ downloaded catalogue only |
| AI/model-backed recommendations | Yes | No; show a clearly labelled local fallback |
| New sign-in or account switch | Yes | No |
| Sync across devices | Yes | No |

### Work

- Implement Dexie database schema and versioned migrations.
- Store downloaded story content, required metadata, cover assets, content version, download timestamp, and download status.
- Build explicit Download and Remove Download controls with progress, cancellation, retry, and storage information.
- Keep the service worker responsible for the application shell and static assets; use IndexedDB for structured story records and reader state.
- Implement a local-first read path: check local cache/download first, use Supabase when online, then update the local copy where appropriate.
- Implement a local-first write path for progress, bookmarks, and preferences.
- Add a durable sync outbox with stable event IDs, idempotent server writes, retry/backoff, and visible sync status.
- Trigger sync on app launch, network restoration, and return to foreground. Do not rely exclusively on Background Sync because browser support and execution are not guaranteed.
- Define conflict rules:
  - progress: reconcile using explicit version/timestamp rules and preserve a valid resumable position;
  - bookmarks: idempotent upsert/delete operations;
  - preferences: versioned last-write strategy with clear rules;
  - feedback/events: immutable, deduplicated by event ID.
- Handle partial downloads, expired/removed content, low storage, quota errors, stale app shells, and failed sync.
- Add a manual “Retry sync” action and clear states such as Saved on this device, Waiting to sync, and Synced.
- Do not cache authentication tokens or private API responses in the service worker cache.

### Deliverables

- Dexie schema and migration path.
- Download manager and local story repository.
- Offline progress/bookmark/preferences support.
- Idempotent sync queue and conflict-resolution rules.
- Offline status indicators and recovery UI.

### Exit criteria

- After a story is downloaded, it remains readable in airplane mode after closing and reopening the app.
- Progress and bookmarks made offline survive a browser restart.
- When connectivity returns, queued changes sync without duplicate records.
- A failed sync does not delete locally saved work.
- Switching accounts cannot expose another account's downloads or progress.
- The app handles storage exhaustion and incomplete downloads gracefully.
- Offline behaviour is tested on target Android browsers and at least one desktop browser; iOS Safari/PWA behaviour is separately verified before being advertised as supported.

---

# Phase 8 — Privacy-conscious analytics and reader-interest signals

**Objective:** Collect enough high-quality first-party signals to understand reader preferences and improve recommendations, without turning Afterlight into invasive surveillance.

## Important rule: do not collect literally everything

More data is not automatically better. Excessive tracking creates privacy risk, noisy datasets, higher storage cost, and false conclusions. Collect only documented events that support a clear product or recommendation decision. Do not collect keystrokes, unrelated device activity, contacts, precise location, cross-site browsing, or hidden fingerprinting data.

Make personalisation understandable and controllable. Explain what reading activity is used for, provide a way to pause or reset personalised recommendations, and support account data export/deletion. Avoid inferring sensitive traits such as health, religion, sexuality, political beliefs, or other sensitive personal characteristics from reading choices. Do not use behavioural profiles to target advertising.

### Event taxonomy

Use a controlled event vocabulary rather than arbitrary event names. Initial event types:

| Event | Meaning | Recommendation value |
| --- | --- | --- |
| \`story_impression\` | A story card was actually shown in a visible feed position | Exposure denominator |
| \`story_opened\` | Reader opened the story preview | Initial interest |
| \`story_started\` | Reader began reading | Stronger interest |
| \`reading_progress\` | A coarse progress checkpoint was reached | Engagement and likely completion |
| \`story_completed\` | Reader reached the end | Strong positive signal, not proof of quality |
| \`story_rated\` | Reader submitted an optional rating | Direct quality signal |
| \`story_saved\` / \`story_unsaved\` | Reader saved or removed a story | Explicit preference |
| \`story_hidden\` | Reader asked not to see a story again | Negative preference |
| \`recommendation_feedback\` | Reader marked a recommendation helpful/not helpful | Direct recommendation feedback |
| \`genre_followed\` / \`genre_unfollowed\` | Reader changed a declared preference | Explicit preference |
| \`story_downloaded\` / \`story_removed_download\` | Reader downloaded or removed offline content | Utility preference |

For each event, define an allowlisted schema containing only the fields required for analysis, such as event ID, Clerk user ID, event type, story ID, content version, timestamp, feed surface/position, and recommendation version/request ID when applicable. Validate it at runtime. Use a random event ID so retries can be deduplicated.

Do not store raw search text by default. Search text can contain personal information. Start with query category or selected filters; revisit raw-query collection only if a clear need, disclosure, and retention policy are approved.

### Work

- Write a data dictionary: event name, trigger, properties, purpose, owner, and retention period.
- Separate direct preferences (genre choices, followed topics, hidden stories) from inferred preference scores.
- Build one analytics service so UI components do not write ad hoc database events.
- Record a story impression only when the card has genuinely been displayed, not merely fetched.
- Batch coarse reading progress checkpoints; do not emit a network event on every scroll.
- Record recommendation exposure with position and algorithm version so click-through can be interpreted correctly.
- Build a reader-interest profile from allowed signals: genres, moods, themes, story format, series interest, and explicit feedback.
- Use recency decay so a reader's interests can evolve. Avoid permanently labelling a reader based on one click.
- Treat a completion as a weak-to-medium signal unless supported by rating, save, or repeat behaviour. A reader may finish a story and dislike it.
- Make the personalisation controls visible in Settings: pause personalisation, edit declared interests, clear inferred interests, and clear reading-history-derived recommendations where feasible.
- Add retention and deletion jobs. Proposed starting point for review: keep raw behavioural events for a limited period (for example, 90 days), then retain only useful aggregated statistics. Final periods must be confirmed against product needs and applicable legal requirements.
- Document privacy notice requirements and obtain a qualified legal/privacy review before public launch, particularly for age-appropriate use and applicable jurisdictions.

### Deliverables

- Event dictionary and analytics service.
- Validated event ingestion path and deduplication.
- Reader-interest feature model with explicit vs inferred fields.
- Personalisation settings and data deletion/export design.
- Retention schedule and privacy documentation.

### Exit criteria

- Every collected event has a documented product purpose.
- No unapproved free-text or sensitive personal data is captured.
- A reader can understand and control personalisation.
- Event ingestion is idempotent and does not alter the reading experience.
- Analytics dashboards distinguish exposure, opening, starting, completion, rating, and hiding.
- User deletion/reset workflows cover both cloud records and the applicable local data.

---

# Phase 9 — Recommendation engine and next-story prediction

**Objective:** Recommend the next story from the actual catalogue using useful reader signals, with a simple baseline before introducing machine learning.

### Recommendation system design

\`\`\`text
Published story catalogue
        |
        v
Candidate generation
        |
        v
Eligibility and exclusion filters
        |
        v
Ranking using reader signals
        |
        v
Diversity / exploration check
        |
        v
Small ranked list of story IDs
        |
        v
Reader sees recommendation
        |
        v
Impression + open + completion + feedback events
        |
        +---------------------> Evaluation and improvement
\`\`\`

### Work

- Start with a deterministic baseline. Do not begin with a black-box model.
- Build candidate generation from published, available stories and relevant connected series.
- Exclude stories the reader has hidden, already completed where repetition is not intended, or cannot access under the current product rules.
- Rank using a documented combination of:
  - declared genres and moods;
  - story opens and starts;
  - completion and optional ratings;
  - saves, follows and explicit feedback;
  - recency and freshness;
  - series/connected-world continuity;
  - negative signals such as hide/not-helpful.
- Weight signals by strength and recency. Direct preference and explicit negative feedback should carry more meaning than a single impression.
- Add diversity so recommendations do not collapse into one genre or repeat near-identical stories.
- Include controlled exploration so readers can discover something outside their strongest known preferences.
- Handle cold start with onboarding choices, editorial picks, and broad popular stories; do not pretend a new account has a detailed taste profile.
- Return a small list of valid story IDs plus an internal reason code or explanation category. The UI should never receive an arbitrary story generated by a model that does not exist in the catalogue.
- Keep ranking and model-provider keys server-side if an external model is later introduced.
- If a language model is used later, send only the minimum non-sensitive preference summary and catalogue metadata needed. Validate every returned story ID against eligible catalogue candidates.
- Evaluate recommendations using completed stories, positive ratings, saves, useful feedback, hide/not-helpful rates, repeat satisfaction and voluntary return. CTR alone is insufficient; raw time spent is not the target.
- Keep an editorial override for featured stories, new releases, and quality-sensitive placements.
- Store the recommendation version and impression context to make experiments reproducible.

### Model progression

1. **Version 0 — Editorial:** manually curated collections and basic genre filters.
2. **Version 1 — Rules/weights:** transparent ranking from declared preferences and first-party events.
3. **Version 2 — Tested statistical ranking:** only after there is enough high-quality labelled data and a sound evaluation set.
4. **Version 3 — Optional model-assisted ranking/explanations:** only if it measurably improves reader satisfaction and remains constrained to the approved catalogue.

Do not train or tune on a tiny, biased dataset. Split evaluation by reader or time where appropriate to reduce leakage. Keep a stable baseline so a more complex model can be compared against it.

### Deliverables

- Candidate generator and eligibility rules.
- Explainable baseline ranking service.
- Cold-start and exploration behaviour.
- Recommendation impression/feedback integration.
- Offline evaluation dataset and experiment plan.
- Documented path for a future statistical/AI ranking model.

### Exit criteria

- A reader with no history receives useful non-personalised recommendations.
- A returning reader gets recommendations influenced by declared and observed interests.
- Hidden/not-helpful stories are suppressed appropriately.
- The system never recommends an unpublished or ineligible story.
- Recommendation quality is evaluated beyond click-through rate.
- Ranking can be rolled back to a known baseline without changing the reader's library or content.

---

# Phase 10 — Quality, beta launch, operations and monetisation readiness

**Objective:** Prove that the product is reliable, safe, understandable, and worth returning to before scaling acquisition or monetisation.

### Work: quality and security

- Add unit tests for validation, ranking, interest-score updates, conflict rules, and repositories.
- Add component tests for critical UI states.
- Add integration tests for Clerk → Supabase → RLS.
- Add end-to-end tests for registration, discovery, reading, completion, library, downloads, offline reading, and reconnection.
- Test two-account isolation on the same browser/device.
- Test network loss during downloads and sync, duplicate retries, browser restarts, storage quota failures, and service-worker updates.
- Test mobile performance, accessibility, keyboard support, contrast, reduced motion, and screen-reader labels.
- Review content safety, age guidance, moderation, story quality, copyright provenance, and reporting/removal procedures.
- Review Supabase policies, storage permissions, secrets, rate limits, abuse controls, backup/restore and account deletion.
- Add monitoring for application errors, failed sync, failed downloads, API errors and recommendation failures. Avoid logging personal reading content or secrets.
- Verify production environment variables and deployment configuration. Keep preview and production data separated.

### Work: closed beta and product validation

- Invite a small, diverse group of readers after the privacy and safety review.
- Observe whether readers can find a story quickly, understand its premise, read comfortably, resume later, and find their library.
- Collect voluntary feedback about story quality and recommendation relevance.
- Review story completion together with ratings and qualitative comments.
- Investigate failures and confusion before increasing catalogue size or traffic.
- Keep a changelog and record product decisions learned from beta usage.

### Work: monetisation readiness (not a launch blocker for the reading MVP)

- Validate a single clear premium proposition before implementing multiple tiers.
- Design subscription entitlements and access checks server-side.
- Select an appropriate billing provider only after pricing, platform, and legal requirements are reviewed.
- If advertising is introduced, keep placements clearly labelled and outside active reading. Do not use sensitive behavioural profiles for ad targeting.
- Do not make offline reading, progress saving, or basic privacy controls artificially unreliable to force an upgrade.
- Add purchase/restore/cancel flows and entitlement tests before accepting payments.

### Deliverables

- Automated test suite and release checklist.
- Security, privacy, content and accessibility review.
- Monitoring and backup/restore procedures.
- Closed-beta feedback report and prioritised fixes.
- Production deployment with documented rollback.
- Separate monetisation decision record.

### Exit criteria

- Critical user journeys pass on the supported browser/device matrix.
- No known critical security or account-isolation defects remain.
- Offline downloads and later sync pass repeatable tests.
- Readers understand what is saved locally and what is used for personalisation.
- The product can be monitored, restored, and rolled back.
- Beta evidence supports continuing the product direction.
- Monetisation is enabled only after its own value, trust and operational checks pass.

---

## Cross-phase quality gates

These requirements apply throughout all ten phases.

### Security

- Never commit API keys, service-role keys, Clerk secrets, tokens, or real user exports.
- Never expose privileged credentials in browser code.
- Enable and test RLS for every user-owned table.
- Validate all external and persisted data at runtime.
- Treat content, model output, and client-supplied IDs as untrusted input.
- Use least-privilege storage and server-side operations.

### Offline reliability

- IndexedDB is the structured local store; the service worker is not the primary database.
- Service-worker cache rules must be explicit and must not cache private authenticated responses by default.
- Every offline mutation must be durable before the UI reports it as saved.
- Sync must be idempotent and recover from interruption.
- The app must explain offline limitations instead of showing broken controls.

### Personalisation and data governance

- Collect only first-party signals with documented purposes.
- Separate explicit preferences from inferred interests.
- Never infer or target sensitive personal characteristics.
- Do not use reading profiles for behavioural advertising.
- Provide pause/reset/delete controls and a clear explanation of personalisation.
- Define data retention before collecting production events.
- Review age-appropriate design and applicable privacy obligations before public launch.

### Product quality

- Do not optimise solely for time spent, scroll depth, or recommendation clicks.
- Track completed stories and satisfaction together.
- Give readers meaningful control over preferences and notifications.
- Do not interrupt active reading with ads or aggressive recommendation prompts.
- Maintain editorial review for all published stories.

---

## Proposed MVP definition

The first public MVP is complete when a reader can:

1. Sign up or sign in.
2. Discover a small curated set of short stories.
3. Preview and read a story in a comfortable reader.
4. Resume reading after closing the app.
5. Save stories, bookmark, and find completed stories in their library.
6. Download a story and read it with no internet connection.
7. Save progress and preferences offline and sync them when connectivity returns.
8. Receive useful, explainable recommendations based on direct preferences and relevant first-party reading signals.
9. Understand and control personalisation and their stored data.

Everything else is secondary until this loop works reliably.

## Documentation references

Use the official documentation when implementing; verify the current SDK/API instructions rather than copying old snippets blindly.

- [Vite Guide](https://vite.dev/guide/)
- [Clerk — Integrate Supabase](https://clerk.com/docs/guides/development/integrations/databases/supabase)
- [Supabase — Clerk third-party auth](https://supabase.com/docs/guides/auth/third-party/clerk)
- [Supabase — Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Vite PWA documentation](https://vite-pwa-org.netlify.app/)
- [Dexie.js documentation](https://dexie.org/)
- [MDN — Offline and background operation](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Offline_and_background_operation)
- [MDN — Background Synchronization API support](https://developer.mozilla.org/en-US/docs/Web/API/Background_Synchronization_API)

---

## Execution rules for the implementation agent

1. Treat this file as the sequencing source of truth.
2. Work on one phase at a time. Do not silently skip a phase or declare a phase complete without checking its exit criteria.
3. Before coding a phase, inspect the current repository and identify what already exists. Reuse working code where it is sound.
4. Before modifying schema, inspect the current migrations and data model. Make migrations forward-only and reviewable.
5. Make small, logically grouped commits. Keep README and relevant technical docs current.
6. Run lint, type-check, tests, and production build after meaningful changes. Add tests alongside the feature, not at the end.
7. Never invent environment variables, credentials, successful integrations, or test results. Document what needs dashboard configuration.
8. If a third-party service requires a human dashboard action, create exact setup instructions and continue with independent work. Do not pretend the integration is live.
9. Record blockers, assumptions, and decisions in the relevant documentation.
10. At the end of each phase, report: completed work, files changed, tests run and results, remaining risks, and the next phase. Update the phase status below only when its exit criteria pass.

## Phase status

| Phase | Status |
| --- | --- |
| 1. Product specification and technical foundation | In progress |
| 2. Application foundation, design system and PWA shell | Not started |
| 3. Supabase data model, migrations and security | Not started |
| 4. Clerk authentication and account lifecycle | Not started |
| 5. Story catalogue and publishing workflow | Not started |
| 6. Core reading product and library | Not started |
| 7. Offline-first storage, downloads and synchronisation | Not started |
| 8. Privacy-conscious analytics and reader-interest signals | Not started |
| 9. Recommendation engine and next-story prediction | Not started |
| 10. Quality, beta launch, operations and monetisation readiness | Not started |
