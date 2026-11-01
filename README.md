# Afterlight

> **A whole new world, in 15 minutes.**

Afterlight is a short-fiction entertainment platform built around original, AI-assisted stories that readers can finish in roughly **10–15 minutes**. It combines effortless story discovery, an immersive reading experience, and a library that remembers what each reader enjoys. The business model is a free, ad-supported experience alongside a paid subscription.

The central product idea is simple: **make every visit feel worth the reader's time, and give them a genuine reason to return.** Retention should come from story quality, relevance, and a satisfying experience—not manipulative interface patterns.

## Try the UI prototype

The first responsive UI mockup is in [prototype](prototype/).

It includes Home, Discover, Story Preview, Reader, Completion, and Library screens. Open [prototype/index.html](prototype/index.html) in a browser, or follow the instructions in [prototype/README.md](prototype/README.md).

The next-story recommendation is currently a local placeholder. The planned model-backed integration is described in the prototype README; no AI provider is connected yet.

## Product foundation

The product direction is organised into seven areas. The detailed notes and decisions are in [docs/PRODUCT_FOUNDATION.md](docs/PRODUCT_FOUNDATION.md).

1. **Core experience** — Help someone find and finish a great story in 10–15 minutes.
2. **Visual identity** — Make Afterlight feel atmospheric, literary, cinematic, and premium.
3. **Core screens** — Design discovery, story preview, reading, completion, and the personal library as one connected journey.
4. **Retention strategy** — Earn repeat visits through personalisation, fresh stories, connected story worlds, and optional daily discovery.
5. **Monetisation** — Support a free ad-funded tier and a premium subscription without breaking immersion.
6. **Product metrics** — Measure whether readers discover, enjoy, finish, and return to stories—not just how long they keep the app open.
7. **Differentiation** — Treat AI as the production engine; the product users experience is a consistently entertaining fiction platform.

## Development roadmap

The implementation roadmap is in [plan.md](plan.md). It breaks development into **10 phases**, from architecture and app foundation through authentication, content, reading, offline sync, privacy-conscious analytics, recommendations, and beta launch.

The roadmap is the sequencing source of truth. Each phase has deliverables and exit criteria; do not mark work complete until those criteria pass.

## Product principles

- **Story first.** A beautiful interface cannot rescue weak stories.
- **Fast discovery.** A reader should be able to choose something interesting in seconds.
- **Immersive reading.** Keep the reading surface calm, accessible, and free from interruptions.
- **Satisfying endings.** Finishing a story should feel rewarding, not like a prompt to keep scrolling.
- **Healthy retention.** No punitive streaks, deceptive notifications, or artificial friction designed to trap attention.
- **Reader control.** Let readers manage recommendations, notifications, reading preferences, and their library.
- **Quality over volume.** More generated stories are not useful if they feel repetitive or unfinished.

## Current status

**Stage:** Product concept / pre-MVP. The ideas in the product foundation are hypotheses to validate with readers, not proven product decisions. The responsive UI prototype is available; the production application is not yet implemented.

## Suggested first validation steps

1. Interview a small group of likely readers about when, why, and where they read short fiction.
2. Test the UI prototype with readers and observe where they hesitate or get confused.
3. Test a small curated collection of stories across a few genres before building large-scale generation.
4. Measure story starts, completions, satisfaction, and voluntary return visits.
5. Test willingness to pay and acceptable ad placements before committing to the monetisation design.

## Documentation

- [10-phase development roadmap](plan.md)
- [Product foundation — all seven strategic areas](docs/PRODUCT_FOUNDATION.md)
- [UI prototype and run instructions](prototype/README.md)
