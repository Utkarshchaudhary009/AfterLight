# Afterlight UI prototype

A responsive, browser-based mockup built with plain HTML, CSS, and JavaScript. Open `index.html` directly or serve this folder with:

```bash
python -m http.server 8000
```

Then visit `http://localhost:8000`.

## Screens
- Home / editorial discovery
- Discover with genre filters
- Story preview
- Distraction-free reader with font-size and theme controls
- Completion with story rating and a next-story suggestion
- Personal library with in-progress, saved, and completed shelves

## Recommendation model
The prototype uses a local deterministic placeholder for next-story suggestions. Replace that function with a server-side recommendation endpoint backed by Gemini or another suitable free/low-cost model. Keep API keys on the server, send only reader-approved preference signals, and validate that the model returns a story ID from the published catalogue. No live model is connected in this prototype.

## Not implemented yet
Authentication, persistent storage, a content-generation pipeline, live AI recommendations, analytics, and real subscription/ad integrations.
