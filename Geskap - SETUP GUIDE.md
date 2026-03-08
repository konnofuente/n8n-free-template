# Geskap - Create Content Social Media — Setup Guide

## Workflows Created

### 1. `Geskap - Create Content Social Media.json` (Main workflow)
**Flow:** Schedule → AI Content (Gemini) → Image (Nano Banana) → Telegram Preview → Approve/Reject → Post to 3 platforms

### 2. `Geskap - Telegram Approval Handler.json` (Helper workflow)
**Flow:** Telegram callback button click → Resume main workflow → Confirm action

---

## Step-by-Step Setup

### Step 1: Import Both Workflows into n8n
1. Go to https://n8n.srv842116.hstgr.cloud
2. Import `Geskap - Create Content Social Media.json`
3. Import `Geskap - Telegram Approval Handler.json`

### Step 2: Get Your Telegram Chat ID
1. Send any message to your Telegram bot
2. Open: `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`
3. Find `"chat": { "id": 123456789 }` — that number is your chat ID
4. Replace **ALL** `REPLACE_WITH_YOUR_TELEGRAM_CHAT_ID` in the main workflow (4 places):
   - Send Telegram Preview
   - Notify Posted
   - Notify Rejected
   - Notify Error

### Step 3: Set Up Meta (Facebook + Instagram) API
1. Go to https://developers.facebook.com/
2. Create an app (type: Business)
3. Add "Facebook Login" and "Instagram Graph API" products
4. Generate a **Page Access Token** with these permissions:
   - `pages_manage_posts`
   - `pages_read_engagement`
   - `instagram_basic`
   - `instagram_content_publish`
5. Get your **Facebook Page ID**: Go to your page → About → Page ID
6. Get your **Instagram Business Account ID**:
   ```
   GET https://graph.facebook.com/v21.0/{PAGE_ID}?fields=instagram_business_account&access_token={TOKEN}
   ```
7. Replace in the workflow:
   - `REPLACE_WITH_YOUR_FACEBOOK_PAGE_ID` → Your Facebook Page ID
   - `REPLACE_WITH_YOUR_FACEBOOK_PAGE_ACCESS_TOKEN` → Your Page Access Token
   - `REPLACE_WITH_YOUR_INSTAGRAM_BUSINESS_ACCOUNT_ID` → Your IG Business Account ID

### Step 4: Set Up TikTok API
1. Go to https://developers.tiktok.com/
2. Create an app → Request "Content Posting API" scope
3. Complete OAuth flow to get an access token with `video.publish` scope
4. Replace `REPLACE_WITH_YOUR_TIKTOK_ACCESS_TOKEN` in the workflow

### Step 5: Activate Both Workflows
1. **First**: Activate `Geskap - Telegram Approval Handler` (it listens for button clicks)
2. **Then**: Activate `Geskap - Create Content Social Media` (it runs on schedule)

---

## How It Works

```
Mon-Fri at 19:00 WAT (18:00 UTC)
        ↓
Content pillar auto-rotates:
  Mon → DOULEURS (pain points des PME)
  Tue → SOLUTIONS (fonctionnalités Geskap)
  Wed → ÉDUCATION (tips business)
  Thu → INSPIRATION (success stories)
  Fri → ENGAGEMENT (questions, sondages)
        ↓
Google Gemini generates 3 posts:
  - TikTok (court, percutant)
  - Instagram (visuel, storytelling)
  - Facebook (détaillé, éducatif)
        ↓
Nano Banana generates branded image
  (dark green #194E2F, African context)
        ↓
Telegram sends you a preview with:
  - The image
  - All 3 captions
  - ✅ Approuver | ❌ Rejeter buttons
        ↓
You click ✅ → Posts to all 3 platforms
You click ❌ → Nothing posted, try again tomorrow
        ↓
Telegram confirms the result
```

---

## Credentials Already Configured (from your existing workflows)
- ✅ Telegram: "Telegram account 4"
- ✅ Google Gemini: "Google Gemini(PaLM) Api account 2"
- ✅ Nano Banana: "N8N_Credentials" (kie.ai API)

## Credentials YOU Need to Add
- ⬜ Telegram Chat ID (see Step 2)
- ⬜ Facebook Page Access Token + Page ID (see Step 3)
- ⬜ Instagram Business Account ID (see Step 3)
- ⬜ TikTok Access Token (see Step 4)

---

## Troubleshooting

**Image not generating?**
→ Check if Nano Banana API key is still valid in N8N_Credentials

**Telegram buttons not working?**
→ Make sure "Geskap - Telegram Approval Handler" is ACTIVE

**Posts failing on Instagram?**
→ Instagram requires image to be publicly accessible URL. If Nano Banana returns a private URL, you may need to re-host it (e.g., via ImgBB)

**Schedule not triggering?**
→ The cron `0 18 * * 1-5` runs Mon-Fri at 18:00 UTC = 19:00 WAT. Make sure your n8n timezone is set to UTC.
