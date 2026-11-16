# AgriFert AI — Enhanced (Fixed v4)

## ⚠️ What was actually wrong (previous "v3 fix" made it worse)

The old README in this project claimed the chatbot and voice assistant were
"fixed" by making them **call the Anthropic API directly from the browser**.
That is what broke real AI understanding:

- `FarmerChatbot.jsx` called `https://api.anthropic.com/v1/messages` straight
  from React **with no API key attached at all** (browsers can't safely hold
  a secret key, so none was ever sent). Every request returned a 401 and
  silently fell back to a hardcoded `if (message.includes('fertilizer'))`
  style keyword matcher — exactly the fixed-question behavior you said not
  to build.
- `VoiceAssistant.jsx` didn't call any AI or backend endpoint at all. It went
  straight to the same kind of keyword-matching function on every question,
  regardless of language or phrasing.

So the app *looked* AI-powered but was 100% rule-based in practice, while a
real, working, keyed Claude integration already existed server-side
(`server/server.js`) and was simply never being called by the chat/voice UI.

## ✅ What I fixed

1. **Chatbot now goes through the backend** (`POST /api/chat/message`). The
   backend holds `ANTHROPIC_API_KEY` server-side (never in frontend code),
   sends the user's real free-form question to Claude, and keeps per-session
   conversational history in MongoDB so follow-ups have context. The old
   keyword matcher is kept **only** as a last-resort fallback if the
   backend/network call itself fails — it is no longer the primary path.
2. **Voice Assistant now goes through the backend** (`POST /api/voice/process`).
   Speech is transcribed in the browser (Web Speech API), the raw transcript
   is sent to the backend, Claude interprets it in natural language and
   replies in the same language, and the reply is spoken back via
   SpeechSynthesis. I removed the login requirement on this one endpoint
   since the Voice Assistant is exposed on the public Home page without
   sign-in, and the route touches no per-user data.
3. **Model id corrected** — all three server-side Claude calls used
   `claude-sonnet-4-20250514`; pointed them at a current, valid model id
   (`claude-sonnet-5`).
4. **Multer upgraded** from the deprecated/vulnerable 1.x line to 2.x (same
   API used here — `memoryStorage()`, `.single()`, `fileFilter` — so no code
   changes were needed beyond the version bump).
5. Verified the backend starts cleanly, JWT/bcrypt auth logic round-trips
   correctly, and the fertilizer/disease rule engine genuinely varies its
   output by crop, soil parameters, and disease.

## 🚀 Running the app

### Backend
```bash
cd server
npm install
# edit .env: set a real MONGO_URI (Atlas or local mongod)
# optionally set ANTHROPIC_API_KEY for real Claude-powered chat/voice/vision
npm run dev      # nodemon, or `npm start` for plain node
```

### Frontend
```bash
cd client
npm install
npm start
```
Open http://localhost:3000. The client talks to the backend at
`REACT_APP_API_URL` (defaults to `http://localhost:5000/api`).

## 🔑 Environment variables

**server/.env**
- `PORT` — backend port (default 5000)
- `MONGO_URI` — your MongoDB connection string (Atlas or local)
- `JWT_SECRET` — long random string, used to sign auth tokens
- `JWT_EXPIRE` — token lifetime (e.g. `30d`)
- `ANTHROPIC_API_KEY` — enables real Claude-powered chat, voice, vision, and
  recommendation explanations; app runs in rule-based fallback mode if unset
- `NODE_ENV`
- `CLIENT_URL` — allowed CORS origin for your deployed frontend

**client/.env**
- `REACT_APP_API_URL` — backend base URL, e.g. `http://localhost:5000/api`
- `REACT_APP_APP_NAME`

No secrets are hardcoded anywhere in the codebase; both `.env` files ship
with placeholders only.

## 📋 How each feature behaves

| Feature | With `MONGO_URI` + backend running | With `ANTHROPIC_API_KEY` set |
|---|---|---|
| Login / Register | Real accounts in MongoDB, JWT auth | — |
| Login / Register (backend unreachable) | Falls back to a browser-local demo account, for UI testing only | — |
| Chatbot | Persists chat history per session | Real Claude natural-language answers |
| Chatbot (no key / AI call fails) | — | Falls back to a small rule-based reply, clearly a degraded mode |
| Voice Assistant | Works without login | Real Claude natural-language answers, in the selected language |
| Disease Detection | Saves detection history | If an image is uploaded, uses Claude Vision for a real per-image read; otherwise a rule-based disease/crop lookup (see limitations) |
| AI Recommendation | Saves recommendation history | Adds a short Claude-generated practical explanation on top of the rule-based fertilizer pick |
| Dashboard | Aggregates the user's saved history | — |

## 🧪 What I verified in this environment

This sandbox has no network access to MongoDB's download/Atlas servers, so I
could not run a live database end-to-end here. What I did verify:
- `node -c server.js` — no syntax errors.
- Backend loads and registers every route without runtime errors; a bad
  Mongo URI is caught and exits cleanly (`process.exit(1)`) instead of
  crashing unpredictably.
- `npm run build` on the client compiles with no errors after the fixes.
- bcrypt hash/compare and JWT sign/verify round-trip correctly in isolation.
- The rule-based fertilizer and disease engines were exercised directly with
  varied inputs: different crops/soil parameters genuinely produce different
  fertilizers and different diseases, never the same result every time.

You should still do one real end-to-end pass yourself (register → login →
recommendation → chat → voice → disease detection → logout) against a real
MongoDB URI and your own Anthropic key before treating this as production
ready.

## ⚠️ Remaining limitations (please read — not glossed over)

- **Disease detection is not a trained image-classification model.** There is
  no CNN, no PlantVillage-trained model, and no separate Python/Flask ML
  service anywhere in this project — the zip never contained one (the
  `controllers`, `models`, `middleware`, `routes` server folders were empty
  placeholders, and `server.js` is one monolithic file). When an image is
  uploaded, the backend sends it to **Claude's vision capability** for a real
  read of that specific image — that part is genuine per-image analysis. When
  no image is uploaded, or no `ANTHROPIC_API_KEY` is set, it falls back to a
  small hand-written disease table keyed by crop plus a few soil heuristics
  (humidity, rainfall, pH). That fallback deliberately varies by input (see
  verification above) so it never returns the same result for every crop, but
  it is not a substitute for a real trained model. If you need genuine image
  classification, you'd need to train/host a CNN (e.g. on the PlantVillage
  dataset) and add a proper `/predict` endpoint for it — that's out of scope
  for what existed in this zip.
- **Fertilizer recommendation is rule-based**, not a trained regression/
  classification model. It's driven by an N/P/K/pH/soil-type scoring table
  that responds to actual inputs (verified above), enhanced with a real
  Claude-generated explanation when a key is present — but the underlying
  ranking logic itself isn't a trained ML model.
- **Demo/offline auth mode** (browser-local accounts when the backend is
  unreachable) only covers login/register. If the backend is genuinely down,
  every other feature (recommendations, chat, disease detection, dashboard)
  will fail with a network error rather than silently degrading, because
  those require the database and/or the Claude API server-side.
- No per-user rate limit on Claude calls beyond the existing generic
  `express-rate-limit` — for a public deployment add a per-user quota so one
  person can't run up your Anthropic bill.
