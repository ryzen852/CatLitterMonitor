# 🐱 Cat Litter Monitor

A self-hosted, AI-powered litter box monitor that runs entirely in a mobile browser — no server, no native app, no subscription.

Point any phone or tablet with a camera at your litter box and get per-cat visit logs, health alerts, and Discord/Telegram notifications — all powered by Claude Vision.

---

## Features

- **Motion-triggered capture** — only calls the API when a cat is actually present, keeping costs minimal
- **Smart 3-shot capture** — entry shot for identification, drop shot timed to the squat moment, presence-check shot to confirm the cat has left
- **Multi-cat support** — add as many cats as you have; each gets a name, breed, reference photo, and colour tag
- **AI identification** — Claude compares live frames against per-cat reference photos and written descriptions; records as "unknown" rather than guessing when confidence is low
- **Waste health analysis** — urine clump size, colour, feces consistency and colour; flags abnormalities automatically
- **Health monitoring** — alerts for unusual visit frequency, abnormally long sessions, no-elimination visits, and extended gaps between defecation records
- **Discord + Telegram notifications** — configurable modes: every visit, alerts only, or daily summary
- **Bilingual UI** — switch between Traditional Chinese and English at any time; notifications follow the same language
- **Zero backend** — runs as a static HTML file; all data stored in browser LocalStorage

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
           Session starts
                │
                ├─ +3s         → Shot 1 (entry)           → Claude API
                ├─ peak drops  → Shot 2 (squat/eliminate)  → Claude API
                │
                ▼  stillness for 45s → AI presence check
                │
                ├─ cat still there → reset timer, continue
                └─ cat gone   → Shot 3 (presence check + waste analysis) → Claude API
                                   aggregate results + health check
                                   save to LocalStorage
                                   notify Discord + Telegram
```

Motion detection runs entirely on-device — the Claude API is only called after a session is confirmed, and at most 3 times per session.

---

## Setup

### 1. Configure

Open the app, go to **Settings** and fill in:

| Field | Where to get it |
|-------|----------------|
| Claude API Key | [console.anthropic.com](https://console.anthropic.com) |
| Discord Webhook URL | Server Settings → Integrations → Webhooks |
| Telegram Bot Token | [@BotFather](https://t.me/BotFather) on Telegram |
| Telegram Chat ID | [@userinfobot](https://t.me/userinfobot) or your group ID |

### 2. Add your cats

Under **Cat Profiles**, tap each cat card to expand it:

- Upload a clear **reference photo** (the single biggest accuracy improvement)
- Select **breed** from the dropdown
- Write a **description** of unique features — markings, size, ear shape, anything distinguishing

Tap **＋ Add Cat** to add more.

### 3. Start monitoring

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

---

## Cost Estimate

| Model | Per week | Notes |
|-------|----------|-------|
| `claude-haiku-3-5` | ~$0.28 USD | Recommended |
| `claude-sonnet-4-5` | ~$1.10 USD | More accurate identification |

Assumes 2 cats, ~7 litter box visits per day combined, 3 API calls per session, with ~1.3× for reference photo token overhead. Actual cost depends on number of cats and visit frequency.

---

## Data & Privacy

- All logs are stored in **browser LocalStorage** on the device — nothing leaves your network except the camera frames sent to the Claude API for analysis
- Reference photos are resized to 512px max before storage
- Logs can be exported as JSON from the Settings tab

---

## Tech Stack

- Vanilla HTML/CSS/JS — single file, no build step, no dependencies
- [Claude Vision API](https://docs.anthropic.com/en/docs/vision) for cat identification, activity classification, and waste health analysis
- Canvas API for local motion detection
- Screen Wake Lock API to keep the display on
- Discord Webhooks + Telegram Bot API for notifications

---

## Limitations

- **Enclosed litter boxes** will significantly reduce accuracy — the camera can't see inside
- **Very similar cats** (same breed, same colour, no distinguishing marks) may frequently return "unknown" even with reference photos
- **Lighting** — camera feed quality at night depends on ambient light; a small IR night-light near the litter box helps
- **LocalStorage cap** — browsers typically allow 5–10 MB; logs are capped at 500 entries automatically

---

## License

MIT
