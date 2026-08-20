# Faceless FB Auto-Pipeline

Zero-cost, fully automated pipeline that generates 3-5 short "AI-narrated story"
videos a day and posts them to a Facebook Page — no server, no paid APIs.

Runs entirely inside **GitHub Actions** on a daily cron. Nothing to host.

## How it works (one pipeline run = one day's batch)

```
topics.json (rotating prompts)
      │
      ▼
generate_script.py   → Groq API (free tier, Llama 3.x) writes N short scripts
      │
      ▼
generate_voice.py    → Piper TTS (open source, local, no API key) narrates each script
      │
      ▼
fetch_broll.py        → Pexels + Pixabay (free APIs) pull matching stock b-roll
      │
      ▼
assemble_video.py    → faster-whisper generates captions, ffmpeg stitches
                        narration + b-roll + burned-in captions into an MP4
      │
      ▼
post_to_facebook.py  → Graph API uploads each finished video to your Page
      │
      ▼
log.json committed back to the repo (what posted, when, script used)
```

## One-time setup

### 1. Get your free API keys (all no-cost tiers)

| Service | What for | Where to get it |
|---|---|---|
| Groq | Script generation | console.groq.com → API Keys (free) |
| Pexels | Stock b-roll | pexels.com/api (free, instant) |
| Pixabay | Backup b-roll source | pixabay.com/api/docs (free, instant) |
| Meta | Posting to your Page | developers.facebook.com → create an App → add "Pages API" |

### 2. Meta / Facebook setup (the fiddly part)

1. Go to developers.facebook.com → **My Apps** → **Create App** → type: "Other" → "Business"
2. Add the **Pages** product to the app
3. Under **Tools → Graph API Explorer**: select your app, select your Page,
   generate a User Access Token with `pages_manage_posts`, `pages_read_engagement`,
   `pages_show_list` permissions
4. Exchange that short-lived token for a **long-lived Page Access Token**
   (valid ~60 days) — instructions in `scripts/get_long_lived_token.py`
5. **Important:** while your app is in "Development" mode, it can only post to
   Pages you personally admin — which is exactly what you need to start. You do
   **not** need App Review to post to your own Page. App Review is only
   required if you want other people's Pages to use this, or certain higher-volume
   permissions later. Ignore anyone telling you otherwise for a single-page use case.
6. Set the token to auto-refresh: long-lived Page tokens effectively don't
   expire as long as they're used regularly — but set a calendar reminder
   for 45 days out to regenerate just in case.

### 3. Add secrets to your GitHub repo

Repo → Settings → Secrets and variables → Actions → New repository secret:

- `GROQ_API_KEY`
- `PEXELS_API_KEY`
- `PIXABAY_API_KEY`
- `FB_PAGE_ID`
- `FB_PAGE_ACCESS_TOKEN`

### 4. Push this repo to GitHub

```bash
cd faceless-fb-pipeline
git init
git add .
git commit -m "init pipeline"
git branch -M main
git remote add origin https://github.com/<you>/<repo>.git
git push -u origin main
```

That's it. The workflow in `.github/workflows/daily_pipeline.yml` runs on a
cron schedule automatically. You can also trigger it manually from the
Actions tab any time ("Run workflow" button) to test.

## Local testing (recommended before trusting the cron)

```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
export GROQ_API_KEY=...  # etc, or use a .env file with python-dotenv
python scripts/pipeline.py --count 1
```

Check `output/` for the finished MP4 before letting it post live —
`pipeline.py --no-post` will generate videos without publishing them.

## Customizing

- **Topics/niche**: edit `config/topics.json` — it's a rotating list of prompt
  seeds. Add/remove to steer the niche.
- **Voice**: `generate_voice.py` downloads a Piper voice model on first run —
  swap `VOICE_MODEL` at the top of the file for a different accent/tone
  (browse voices at github.com/rhasspy/piper/blob/master/VOICES.md).
- **Posting frequency**: change the cron expression in the workflow YAML, and
  `--count` in the run command, to adjust videos/day.

## Guardrails baked in

- Scripts are generated fresh each time (not copy-pasted source text) to stay
  on the right side of Facebook's originality requirements
- B-roll only comes from Pexels/Pixabay — both explicitly license footage for
  this kind of reuse, no copyright strikes
- `pipeline.py` logs everything to `log.json` so you can see exactly what
  posted and roll back/investigate if a video underperforms or gets flagged
