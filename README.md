# Rana Family AI – v8 Provider Manager + Family Tree

This version includes a dynamic AI provider manager.

## New in v8
- Per API key: provider, model, and priority.
- Gemini: **Load models** reads the models available for that specific key and filters for `generateContent`.
- `AUTO` model selection: automatically chooses a usable model from the key's model list.
- If a configured model returns 404, the system tries `AUTO` for the same key and then the next provider.
- Existing API keys remain server-side when providers are edited in the Admin Panel; the browser receives only a masked preview.
- Each API key is treated as a separate credential. Multiple keys can be entered in one provider field (one per line/commas); on 401/403/404/408/409/429/5xx/timeout the router skips that key and continues with the next key/provider.
- OpenAI-compatible providers can use their `/models` endpoint; otherwise the model can be entered manually.
- Weather and time continue to use direct live routes.

## Gemini model availability
The Gemini API provides a `models.list` endpoint. Rana filters models that support `generateContent`. Availability can differ by project/key, so the Admin Panel shows the current list for the entered key.

## Deploy via GitHub + Netlify

This app is **not a GitHub Pages app**: it uses Netlify Functions and Netlify Blobs for authentication, AI routing and stored data. Use GitHub as the source repository and Netlify as the runtime.

### Recommended workflow: GitHub + Netlify continuous deployment

1. Create or use a GitHub repository and push this project to it.
2. In Netlify, choose **Add new project → Import an existing project → GitHub**.
3. Select the repository. Netlify will read `netlify.toml` automatically.
4. Keep `public` as the publish directory and `netlify/functions` as the functions directory (both are already configured).
5. Add secrets in Netlify under **Project configuration → Environment variables**. Do not commit secrets to GitHub.
6. Set `main` as the production branch. Netlify will deploy `main` to the live production site.

### Free Deploy Preview workflow

Use a GitHub branch + Pull Request for changes you want to test before production:

```text
feature/my-change
       ↓
GitHub Pull Request → Netlify Deploy Preview
       ↓
review/test
       ↓
merge into main → Production
```

Netlify automatically creates a Deploy Preview for a Pull Request against the production branch. Deploy Previews are separate from production and, on Netlify's current credit-based plans, the Deploy Preview deployment itself uses **0 credits**; production deploys use 15 credits each.

### Important: do not use GitHub Actions to deploy

This repository intentionally does **not** contain a `.github/workflows/netlify.yml` deployment workflow. Netlify's native Git integration should perform the deployment. This avoids having two deployment systems fighting each other and keeps Pull Request Deploy Previews working through Netlify's built-in workflow.

### Environment variables for previews

If a Deploy Preview needs the same API keys/secrets as production, configure them in Netlify for the appropriate deploy contexts. You can also give Deploy Previews different values when you want a safe test environment. Never put real API keys, `SESSION_SECRET`, or `.env` files in GitHub.

### Local development

```bash
npm install
npm run dev
```

The Netlify CLI can emulate the site's functions locally.

## Live services

### OpenRouteService
Set `openrouteservice_api_key` in Admin → AI settings. Route questions such as `route from Amsterdam to Utrecht` use OpenRouteService for geocoding and route distance/time.

### Hugging Face
Set `huggingface_api_key` in Admin → AI settings. The key is automatically exposed as a `huggingface` AI provider using the Hugging Face Inference Providers OpenAI-compatible router. `huggingface_model` defaults to `deepseek-ai/DeepSeek-V3-0324:fastest`.

## AI task router
The chat uses one automatic router for different tasks:
- **chat** → general conversational model
- **reasoning** → reasoning/thinking model when available
- **code** → code/coder model when available
- **vision** → multimodal model for images/attachments
- **document** → writing task + automatic files
- **chart** → model creates a structured chart specification, then Rana creates SVG + CSV
- **spreadsheet** → model creates table data, then Rana creates XLSX and/or CSV
- **image** → specialized image model

The router uses each configured model's capabilities plus quality/speed/cost heuristics. Existing provider rules continue to work; missing capabilities are inferred from provider/model names. For example, a Gemini image model can be configured as a separate provider rule with model `gemini-3.1-flash-image` and will automatically be used for image tasks.

## Files in chat
The chat can store generated **DOCX, PDF, HTML, Markdown, TXT, XLSX, CSV, and SVG** files in Netlify Blobs. Each generated file appears as a link in the same assistant message and is accessible only to the user who created it.

## Better live search
Current/search questions can activate a separate live-search layer. If `BRAVE_SEARCH_API_KEY` and/or `TAVILY_API_KEY` is configured as a Netlify environment variable, real live results are collected and combined. If external search keys are missing, the router can use a configured Gemini provider with Google Search grounding as a fallback. Time and weather are handled directly by realtime services rather than by a language model.

### Multiple API keys
You can configure around 30 keys (or more). Each key is handled independently. A provider entry can contain multiple keys separated by a newline, comma, or semicolon. On 429/401/403/404/timeout the router moves to the next key and then to other providers. A quota warning appears only when all actually tested key routes return 429.

## Document Engine 2.0
Generated documents use a document-specific visual profile. DOCX and PDF support professional covers, hierarchy, headers/footers, page numbers, tables, and structured sections. HTML uses a responsive document layout. XLSX uses formatted headers, column widths, frozen headers, and filters. Markdown/TXT remain portable plain formats.

Supported document profiles include professional reports, academic/school documents, proposals/project plans, business documents, correspondence, CV/resume, meeting minutes/agendas, and manuals/guides.

## Voice conversation / microphone
The chat includes built-in voice features:
- 🎙️ **Microphone**: speaks text through the browser and sends it automatically.
- 📞 **Voice conversation**: listens again after each answer and reads the AI response aloud.
- Uses the browser Web Speech API; the browser asks for microphone permission.
- For best support, use a recent Chrome or Edge browser.

## Voice conversation v2
- **Microphone toggle**: 🎙️ on / 🔇 off, independent of the session.
- **Stop session**: ⏹️ immediately ends listening and text-to-speech.
- **Interrupt**: when Rana is speaking, the user can start speaking; her audio stops immediately and the new input is processed.
- After an answer, the session automatically listens again while the microphone remains enabled.

## ElevenLabs: multiple API keys with rotation
In Admin → AI settings → **ElevenLabs API keys**, enter **one key per line** (commas, semicolons, or a JSON array also work).
You can optionally also use the environment variable `ELEVENLABS_API_KEYS` (or `ELEVENLABS_API_KEY`); those keys are added as well.

How it works:
- **Round-robin**: each voice request starts with the next key so usage is distributed.
- **Automatic failover** to the next key on quota errors (`quota_exceeded`), invalid key (401), rate limit (429), server error (5xx), or network error.
- **Cooldown**: a key with quota/rate-limit/auth problems is temporarily skipped (quota 1 hour, invalid 24 hours, rate limit 30 seconds). If all keys are in cooldown, they are tried again.
- Errors unrelated to the key (for example, an incorrect voice ID) stop immediately so all keys are not called unnecessarily.
- If all keys fail, the site falls back to the browser voice.
- A single key continues to work as before.

## Spoken language
Next to the microphone button is a **language selector**:
- **🌐 Auto** (default): follows the conversation. The app detects the language of the answer and your latest input and uses it for speech and recognition. The default is now English.
- **Fixed language**: English, Nederlands, Deutsch, Français, Español, Türkçe, العربية, اردو, हिन्दी, Русский, 中文, 日本語, 한국어. The choice is remembered.

The language detector ignores code blocks, links, and emoji when detecting language. The detector remains multilingual even though the interface defaults to English.

## Responsive design
The entire UI (chat, login, admin) adapts to the screen. Shared styles are in `public/app.css`; chat-specific layout is in `public/index.html`.

- The composer adapts to the **chat column width**, not the overall screen.
- The sidebar becomes a drawer at ≤900px.
- File chips stay compact and scroll horizontally.
- Landscape phone mode keeps the composer on one row.
- Touch targets are at least 44px.
- iPhone/Android use `100dvh`, safe-area handling, and keyboard-aware layout.
- Admin tabs and tables scroll inside their containers on small screens.
- The long Enter/Shift+Enter placeholder is shown only when there is enough keyboard width.

Test with `node test/responsive.mjs`. It renders the real pages at 12 screen sizes (320px to 2560px, including landscape) and checks for overflow/overlap and sufficiently large touch targets.

## Quick time route
The direct `fastTimeAnswer` route in `chat.mjs` recognizes time/date/day questions in multiple languages and answers from the realtime clock service without a language model. The interface and default responses are English, while multilingual input remains supported.

## Images and vision (fix)
- **Any question that comes with an image is a `vision` task**, whatever the wording (for example "what is the average grade"). Before, only a few trigger words ("analyseer", "bekijk", ...) worked, so most questions were sent to a text-only chat model that then replied "I can't view images".
- **Follow-up questions keep the image.** Attachments are stored with the message (owner-checked, in the `files` store) and re-attached to the next question in the same chat if you do not attach a new one. They are deleted together with the chat.
- **Only vision-capable models answer image questions.** If none is configured, the reply says so instead of giving a misleading answer. Gemini (including model `auto`), GPT-4o/4.1/5, Claude, Llama 4, Pixtral, Qwen-VL, Gemma 3 and others are recognised.
- "analyseer deze afbeelding" is no longer mistaken for an image-*generation* request when an image is attached.
- Test: `node test/vision-attachments.mjs`.

## InternLM (Intern-S1) provider
Admin → AI settings → Provider **InternLM (Intern-S1)**. Paste the token created at https://internlm.intern-ai.org.cn/api/tokens as the API key.
- Base URL (built in, OpenAI-compatible): `https://chat.intern-ai.org.cn/api/v1`
- Model: `intern-latest` (or e.g. `intern-s1`, `intern-s1-pro`; enter manually — do not rely on AUTO/Load models).
- Rate limit by InternLM: 30 requests/minute per user by default.
- **Images:** models `intern-s1`, `intern-s1-pro`, `intern-s1-mini`, `intern-s2-preview`, `intern-latest` and `internvl*` are treated as vision-capable, so image questions are routed to them. Images are sent in the OpenAI `image_url` format (base64 data URL).
- **Deep thinking:** for `intern-s*` models Rana sends `thinking_mode: false` for normal chat/vision/document tasks (and `true` only for reasoning/chart tasks) to stay within the 15s provider timeout.
- **PDFs:** the InternLM chat API cannot open PDFs. When a PDF is attached, Gemini providers are tried first; other models are told the PDF is unreadable instead of guessing. Text files (txt, csv, md, json, html, xml) are read by every provider.

## Fix: saving providers no longer drops keys
Adding/reordering/pausing a provider could silently remove other keys (rows with several keys in one field, rows saved without an id, and paused rows). Saving now keeps every key: multi-key rows are split into one row per key (`<id>-key1`, `-key2`, …) with the correct key recovered server-side.

## Fix: AI Provider Manager showed an empty list and overwrote existing providers
The Admin page read the provider list from `S.providers`, but the server sends it as `S.stats.providers`. The manager therefore always started with an empty draft, and **Save providers** replaced all stored providers with only the newly added one. The Admin page now reads `S.stats.providers`, so existing providers are shown, kept and extended.

## Install-app button
In the chat the "Install app" button now lives in the sidebar footer (next to Log out) instead of floating over the send button. Login/admin keep the small floating button.

## AUTO model for InternLM + clearer AI errors
- Model `AUTO` on an InternLM provider no longer fails when InternLM has no usable `/models` list: it falls back to `intern-latest`.
- When no provider can answer, the message now lists every distinct failure (provider, HTTP status, reason) instead of only the first one.

## InternLM with AUTO is vision-capable; failing keys go last
- An InternLM provider set to `AUTO` is now treated as image-capable (before, image questions skipped it and only tried Gemini). For image questions AUTO uses `intern-s1`, otherwise `intern-latest`.
- Routes that just failed with an invalid key (30 min) or quota/429 (2 min) are tried last on the next messages instead of first (best effort, per running function instance).

## "Failed to fetch" / server time limit
The AI router uses an internal time budget (`AI_TIME_BUDGET_MS`, default 8500 ms) so provider calls remain responsive. If you change that value, keep it below the effective Netlify function execution limit for your plan and workload. InternLM with `AUTO` no longer makes a `/models` request first (one call instead of two).

## Which model answered?
The small line under each reply (`Task · provider · model`) now shows the provider and model that **actually answered** (before it showed the best-ranked guess, which for AUTO just said "auto"). For image questions you should see e.g. `Task: vision · internlm · model: intern-s1`.

## InternLM + images: speed
- Photos/screenshots are now shrunk in the browser (max 1280px, JPEG) before upload, which makes uploads and the AI's image processing faster. Originals up to 30 MB are accepted; the 4 MB total limit applies to the shrunk size.
- The `chat` function uses Netlify's normal function execution settings; `netlify.toml` no longer overrides its timeout.
- If InternLM is still too slow for images within your time limit, set the provider's model explicitly (e.g. `intern-s1-mini`, a smaller and usually faster model) instead of AUTO (`intern-s1` for images).

## Test image reading (Admin → AI settings)
Choose a provider/key and click **Test image reading**. A tiny red PNG is sent through the same request path as a real chat and the result shows: whether the image was read (answer "red"), the model used, the time, or the exact HTTP error. Use it to tell "this model cannot read images" apart from "a real photo is just too slow".
