# GigApp (GigShield) — Legal Advisory App for Gig Workers

A React Native app built with Expo (web-first, Android-capable) that provides AI legal assistance, evidence management, and complaint letter generation for gig workers in India.

## Features

1. **💬 AI Legal Assistant** — chat with a legal-AI persona (plain language, mandatory disclaimers); supports photo attachments
2. **📡 Policy Radar** — detects silent policy changes on Zomato/Swiggy/Uber: versioned policy snapshots, chunk-level diffing (BGE embeddings + token fallback), AI classification in EN + Hinglish, verbatim before/after quotes, and a defensible **₹/week impact estimate** (old rule vs new rule applied to your typical week — never a causal claim), with one-tap complaint letter
3. **📄 Document Analyzer** — upload contracts/notices (image/PDF/DOCX/TXT), OCR + on-device RAG Q&A grounded in the document
4. **📂 Evidence Vault** — photo/video/document storage with filters, multi-select, and notes
5. **✍️ Report Generator** — 4-step wizard producing two AI-drafted letter options (formal/direct), editable, copy/PDF export
6. **📚 Know Your Rights** — bilingual (EN/HI) rights library citing Indian labour law
7. **🏢 Platform Support** — Swiggy/Zomato-specific guidance, deep-links into the letter generator
8. **☎️ Emergency SOS** — one-tap government/legal/platform helplines

## Running the app

```bash
cd frontend
npm install        # uses legacy-peer-deps (see frontend/.npmrc)
npm run web        # dev server (unminified — slow by design)
npm test           # jest suite (60 tests)
npm run build      # production web export → frontend/dist
```

Serve the production build with any static server, e.g. `python -m http.server 8077 --directory frontend/dist`.
**For real performance always test the production build** — `npm run web` is a dev bundle with optimizations off.

## Environment configuration (`frontend/.env`)

| Variable | Purpose |
|---|---|
| `EXPO_PUBLIC_SUPABASE_URL` / `EXPO_PUBLIC_SUPABASE_ANON_KEY` | Supabase auth + database + storage |
| `EXPO_PUBLIC_OPENROUTER_KEY` | Direct OpenRouter fallback (client-visible — demo only) |
| `EXPO_PUBLIC_CHAT_PROXY_URL` / `EXPO_PUBLIC_CHAT_PROXY_SECRET` | Preferred: Supabase `chat-proxy` edge function (key stays server-side) |
| `EXPO_PUBLIC_POLICY_SEARCH_URL` | Policy search endpoint (`/gig` questions). Local dev: `http://localhost:3001` via `node backend/policy-search-local.js` |

⚠️ Env vars are read through **static** `process.env.EXPO_PUBLIC_*` references so Expo inlines them at build time. Do not route them through dynamic `env[name]` lookups — those stay `undefined` in production exports.

## Architecture

```
frontend/
├── App.js                  # Navigation stack, auth gate, fixed 390×844 scaled canvas
├── src/
│   ├── screens/            # 11 screens (all lazy-loaded)
│   ├── components/         # Chat UI, DocChat, modals + ui/ primitives
│   ├── config/api.js       # LLM calls: proxy-first, direct OpenRouter fallback
│   ├── config/supabase.js  # Supabase client (web/native storage adapters)
│   ├── policy/             # Document RAG (chunker/embedder/searcher) + policy Q&A client
│   ├── local/              # Persistence: AsyncStorage stores + Supabase CRUD (database.js)
│   ├── services/           # auth, storage (uploads), api (chat persona)
│   ├── utils/              # pdf, ocr, docText, imageResize, base64, alert, sanitize
│   ├── context/            # Settings + theme providers
│   └── data/               # Static rights/platform content (bilingual)
backend/supabase/           # Edge functions (chat-proxy, embed-proxy, policy-search) + SQL migrations
scripts/                    # Policy index builder (npm run policy:index in frontend/)
AllPolicy/                  # Raw policy corpus (per-platform folders) feeding the RAG index
```

**Storage model** — two modes depending on whether a real Supabase session exists:
- *Local/dev mode*: AsyncStorage (`@gigapp_conversations`, `@gigapp_local_evidence`, `@gigapp_doc_rag_session`, `@gigapp_settings`)
- *Signed in*: Supabase tables `evidence_items`, `timelines`, `letters`, `user_contacts`, `app_settings` (RLS by `user_id`) + `evidence-file` storage bucket

Settings are AsyncStorage-first with the Supabase row as a best-effort cloud mirror.

**AI flow** — all LLM traffic goes through `chatComplete()` (proxy-first, 65s timeout, friendly error mapping). Document Q&A: extract text → chunk (≤1500 chars) → embed locally (BGE-small, 384-dim) → hybrid search (0.6 vector / 0.4 keyword) → grounded answer or `NOT_FOUND_IN_DOCUMENT`. Policy Q&A (`/gig` prefix): server-side retrieval via the policy-search edge function.

**Policy Radar flow** (`src/policy/policyDiff.js` + `policyRadar.js` + `local/policyRadarStore.js`) — every uploaded policy saves a versioned snapshot (last 5 per platform, AsyncStorage). New text is normalized (HTML stripped), chunked, and aligned chunk-by-chunk with the previous version: similarity decides *alignment* only; a pair counts as changed iff its normalized text differs (embeddings rate two-number edits at ~0.97 cosine and cannot make this call). Changed pairs go to the AI for classification (payout/deactivation/insurance/other, real vs cosmetic) and impact: the old and new rule are both applied to the worker's editable weekly numbers (`My week`), producing a ₹/week difference explicitly framed as a rule comparison, never an actual-loss claim. Inputs: **Scan Latest Policy** (pick the platform's policy file — txt/PDF/DOCX/HTML; the normal, reliable path) and a repeatable one-tap demo update per platform. Zomato/Swiggy baselines are verbatim corpus excerpts; the Uber baseline is a labelled curated demo seed.

## Known limitations (intentional / deferred)

- **Auth is in TEST_MODE** (login flow intentionally left broken for testing — the Google button signs in a mock user). Do not ship as-is.
- **Supabase project currently down** (see `.env`) — cloud DB/proxy paths are implemented but dormant; the app runs in local mode until it's restored. Apply `backend/supabase/migrations/` when it is.
- **No vault encryption** and **no GPS/timestamp burn-in** on captured photos (metadata only) — same as the original spec gaps.
- Login background video is 4.4MB (compress when ffmpeg is available).
- Hindi content exists for static screens only; AI replies are English.
- The OpenRouter key in `.env`/netlify.toml has been exposed in source — **rotate it** and prefer the chat-proxy when Supabase returns.

## Scripts

| Command (frontend/) | Action |
|---|---|
| `npm run web` / `android` / `ios` | Dev servers |
| `npm run build` | Production web export |
| `npm test` / `npm run test:coverage` | Jest |
| `npm run policy:index` | Rebuild the policy RAG index (deploys to backend edge function) |
| `npm run policy:test` | Smoke-test policy search (needs local server on :3001) |

## License

See LICENSE. This app provides information only, not legal advice.
