# 🐱 Cat Litter Monitor

A self-hosted, AI-powered litter box monitor that runs entirely in a mobile browser — no server, no native app, no subscription.

Point any phone or tablet with a camera at your litter box and get per-cat visit logs, health alerts, video clips, and Discord/Telegram notifications — all powered by Claude Vision.

---

## Features

- **Motion-triggered capture** — only calls the API when movement is detected, keeping costs minimal
- **Smart 3-shot analysis** — entry shot for identification, drop shot timed to the squat moment, presence-check shot to confirm the cat has left
- **Video recording** — records every session automatically; clip is sent to Telegram and Discord when the session ends
- **Multi-cat support** — add as many cats as you have; each gets a name, breed, clothing colour, reference photo, and colour tag
- **AI identification** — Claude compares live frames against per-cat reference photos and written descriptions; records as "unknown" rather than guessing when confidence is low
- **Clothing-aware identification** — specify each cat's clothing colours so AI identifies by body shape, face, ears, tail, and limbs rather than coat colour
- **Low-confidence confirmation** — when AI confidence is below 70%, sends a screenshot to Telegram with inline buttons to confirm which cat it was; Discord receives the screenshot with a text prompt
- **Waste health analysis** — urine clump size, colour, feces consistency and colour; flags abnormalities automatically
- **Health monitoring** — alerts for unusual visit frequency, abnormally long sessions, no-elimination visits, and extended gaps between defecation records
- **Discord + Telegram notifications** — configurable modes: every visit, alerts only, or daily summary; video clip sent on every session
- **Bilingual UI** — switch between Traditional Chinese and English at any time; all notifications follow the same language
- **Zero backend** — runs as a static HTML file served via GitHub Pages; all data stored in browser LocalStorage
- **Cloudflare Workers proxy** — routes Claude API calls through a personal Worker to bypass browser CORS restrictions; API key never exposed client-side

---

## How It Works

```
Camera (24/7)
        │
        ▼  every 0.5s, local only
   Motion detection (ROI)
        │
        ├─ no motion → idle
        │
        └─ motion ≥ threshold for 3s
                │
                ▼
         Session starts + recording begins
                │
                ├─ +3s           → Shot 1 (entry)              → Claude API
                ├─ peak drops    → Shot 2 (squat/eliminate)     → Claude API
                │
                │  cat confirmed present → 10s polling begins
                │      every 10s → lightweight presence check   → Claude API
                │      2× no-cat OR 4 min max → stop recording
                │
                ▼  stillness for 45s → full presence check
                │
                ├─ cat still there → reset timer, continue
                └─ 2× no-cat confirmed → finalize session
                        aggregate results + health check
                        save to LocalStorage
                        send video clip to Discord + Telegram
                        if confidence < 70% → send screenshot
                            with TG inline buttons for cat confirmation
```

Motion detection runs entirely on-device. The Claude API is called at most 3 times for identification plus lightweight polling checks during recording.

---

## Setup

### 1. Deploy a Cloudflare Workers Proxy

The Claude API does not allow direct browser requests (CORS). You need a personal proxy:

1. Sign up at [workers.cloudflare.com](https://workers.cloudflare.com)
2. Create a new Worker and paste the following code:

```javascript
const API_KEY = 'sk-ant-your-key-here'; // paste your Claude API key here

export default {
  async fetch(request) {
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'POST, OPTIONS',
          'Access-Control-Allow-Headers': 'Content-Type',
        },
      });
    }
    if (request.method !== 'POST') return new Response('catmon proxy ok', { status: 200 });

    const body = await request.json();
    const { __apiKey, ...anthropicBody } = body;

    const res = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-api-key': API_KEY,
        'anthropic-version': '2023-06-01',
      },
      body: JSON.stringify(anthropicBody),
    });

    const data = await res.json();
    return new Response(JSON.stringify(data), {
      status: res.status,
      headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
    });
  },
};
```

3. Save & Deploy. Note your Worker URL (e.g. `https://catmon-proxy.yourname.workers.dev`)
4. Update the Worker URL in the app's `index.html` if self-hosting, or enter it in Settings

### 2. Configure the App

Open the app, go to **Settings** and fill in:

| Field | Where to get it |
|-------|----------------|
| Claude API Key | [console.anthropic.com](https://console.anthropic.com) |
| Discord Webhook URL | Server Settings → Integrations → Webhooks |
| Telegram Bot Token | [@BotFather](https://t.me/BotFather) on Telegram |
| Telegram Chat ID | [@userinfobot](https://t.me/userinfobot) or your group ID |

### 3. Add your cats

Under **Cat Profiles**, tap each cat card to expand it:

- Upload a clear **reference photo** in JPG or PNG format (HEIF/HEIC from iPhone not supported — go to iPhone Settings → Camera → Formats → Most Compatible first)
- Select **breed** from the dropdown
- Write a **description** of unique features — markings, size, ear shape, tail length, anything distinguishing
- If your cat wears clothing, fill in **Clothing colour** (e.g. `red, blue`) so the AI identifies by body shape rather than colour

Tap **＋ Add Cat** to add more.

### 4. Start monitoring

Go to **Monitor**, tap **Start**, then drag to draw the litter box zone on the camera feed.

Keep the device plugged in and the screen active. Chrome's Screen Wake Lock is requested automatically; if the device still sleeps, disable screen timeout in your system display settings.

---

## Detection Settings

| Setting | Default | Notes |
|---------|---------|-------|
| Motion sensitivity | 15% | Lower = more sensitive. Increase if getting false triggers from shadows/light changes |
| Trigger delay | 3s | Sustained motion required before a session starts |
| Session end delay | 45s | Stillness required before ending. 45s recommended — cats pause while covering |
| No-elimination alert | 90s | Alert if cat is in box this long without urinating or defecating |
| No-visit alert | 24h | Alert if a cat hasn't visited in this many hours |
| Max daily visits | 8 | Alert if exceeded |
| Max session duration | 15 min | Alert if a single session runs this long |

---

## Notification Modes

| Mode | Behaviour |
|------|-----------|
| Every visit | Send on each confirmed litter box visit |
| Everything | Every visit plus sessions where no cat was detected (useful for debugging) |
| Alerts only | Only send when a health alert is triggered |
| Daily summary | One message at 8 AM summarising the previous day per cat |

A video clip is sent on **every session** regardless of notification mode.

---

## Cat Identity Confirmation (Telegram)

When AI confidence is below 70% or the cat is unidentified:

1. A screenshot is sent to Telegram with inline buttons — one per cat name
2. Tap the correct cat name
3. The app updates the log entry automatically and sends a confirmation message
4. If the app was closed when you tapped, it will resume and process the confirmation next time it opens (pending confirmations are stored locally for up to 24 hours)

Discord receives the same screenshot with a text prompt listing cat names, but does not support interactive buttons.

---

## Video Recording

- Recording starts when a session begins
- Clips are sent to Telegram and Discord when the session ends, regardless of whether a cat was detected
- Maximum clip length: **4 minutes** (approx. 15 MB at 500 kbps)
- Format: MP4 on Safari/iOS, WebM on Chrome/Android — both supported by Telegram and Discord
- If the video exceeds platform limits, a JPEG screenshot is sent as fallback

---

## Cost Estimate

| Model | Per week | Notes |
|-------|----------|-------|
| `claude-haiku-4-5` | ~$0.80 USD | Recommended |
| `claude-sonnet-4-6` | ~$2.50 USD | More accurate identification |

Assumes 2 cats, ~7 litter box visits per day combined, 3 identification shots per session plus lightweight polling checks, with reference photo token overhead. Actual cost depends on number of cats, visit frequency, and session length.

---

## Data & Privacy

- All logs are stored in **browser LocalStorage** on the device — nothing leaves your network except camera frames sent to the Claude API for analysis
- Reference photos are resized to 512px max before storage
- Logs and settings can be exported as a full JSON backup from the Settings tab and re-imported later
- Pending Telegram confirmations are stored in LocalStorage and auto-purged after 24 hours

---

## Tech Stack

- Vanilla HTML/CSS/JS — single file, no build step, no dependencies
- [Claude Vision API](https://docs.anthropic.com/en/docs/vision) for cat identification, activity classification, and waste health analysis
- Cloudflare Workers for CORS proxy
- Canvas API for local motion detection
- MediaRecorder API for session video capture
- Screen Wake Lock API to keep the display on
- Discord Webhooks + Telegram Bot API for notifications and interactive confirmations

---

## Limitations

- **Enclosed litter boxes** will significantly reduce accuracy — the camera can't see inside
- **Very similar cats** (same breed, same colour, no distinguishing marks) may frequently return "unknown" even with reference photos
- **Lighting** — camera feed quality at night depends on ambient light; a small IR night-light near the litter box helps significantly
- **HEIF photos** — iPhone reference photos must be taken in JPEG mode (Settings → Camera → Formats → Most Compatible)
- **LocalStorage cap** — browsers typically allow 5–10 MB; logs are capped at 500 entries automatically
- **Page must be open** for real-time Telegram confirmation polling; confirmations are queued in LocalStorage and processed on next open

---

## License

MIT
