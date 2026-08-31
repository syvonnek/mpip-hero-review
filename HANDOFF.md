# MODERN POST IN PREMIERE — COMPLETE HANDOFF
**Last updated:** 2026-09-01 · **Site is LIVE and current at postinpremiere.com**

---

## 🚨 READ THIS FIRST — THE #1 TRAP

**The live site deploys from `fmctraining/modern-post-in-premiere-preview` — NOT from `syvonnek/adobe-color-preview`.**

`adobe-color-preview` is a decoy mirror with nearly identical commit messages. Pushing there does **nothing** to the live site. This cost hours once already. Verify the remote before every push:

```bash
cd /tmp/mpip-prod && git remote -v   # must show fmctraining/modern-post-in-premiere-preview
```

---

## PATHS & INFRASTRUCTURE

| Thing | Location |
|---|---|
| **Working file (source of truth)** | `~/Downloads/mpip-hero-preview/hero-lab.html` |
| **Local preview** | `http://127.0.0.1:8831/hero-lab.html?chat=1` (python http.server, PID may change) |
| **Production repo (clone)** | `/tmp/mpip-prod` → `fmctraining/modern-post-in-premiere-preview` |
| **Production file in repo** | `premiere-color-mode-v17-cyan-blue-violet-magenta-latest.html` |
| **Public preview board** | `/tmp/mpip-hero-review-new` → `syvonnek/mpip-hero-review` |
| **og:image generator** | `~/Downloads/mpip-hero-preview/_og.html` |
| **Live site** | https://postinpremiere.com |

**⚠️ `/tmp` gets swept by macOS overnight.** Both `/tmp` clones can vanish or have `.git` corrupted. If that happens, re-clone (do NOT try to repair):
```bash
cd /tmp && gh repo clone fmctraining/modern-post-in-premiere-preview mpip-prod
cd /tmp && gh repo clone syvonnek/mpip-hero-review mpip-hero-review-new
```

**Serving stack:** Cloudflare **Worker** (not Pages) named `modern-post-in-premiere`, FMC account, assets-only (`wrangler.jsonc` → `assets.directory: "."`). `_redirects` maps `/` → the v17 file, so **that file IS the homepage**. Push to `main` → auto-builds in ~10–20 min (not instant).

**How to verify a deploy actually landed:** request a file that only exists in the new build. A 404 means the build hasn't run — it is NOT a cache issue.
```bash
until curl -s --max-time 12 "https://postinpremiere.com/?cb=$RANDOM" | grep -q "SOME_NEW_STRING"; do sleep 25; done; echo DEPLOYED
```

---

## ⚙️ THE DEPLOY SCRIPT (copy-paste, use every time)

The lab file has dev-only bits that must be stripped for production. **Never copy `hero-lab.html` straight to prod.**

```bash
cd ~/Downloads/mpip-hero-preview && node -e "
const fs=require('fs');const h=fs.readFileSync('hero-lab.html','utf8');
let bad=0;[...h.matchAll(/<script>([\s\S]*?)<\/script>/g)].forEach((x,i)=>{try{new Function(x[1])}catch(e){bad++;console.log(i,e.message)}});console.log('js errors:',bad)"

python3 - <<'PY'
import io
lab=io.open('/Users/syvonnekozuch/Downloads/mpip-hero-preview/hero-lab.html',encoding='utf-8').read()
# 1. bake chat=1 into defaults
s=lab.replace("var DEF=new URLSearchParams('pos=top&t=1&pr=0&bg=cine&ui=frame2&rail=3&eb=ww&intl=8&cards=5');",
              "var DEF=new URLSearchParams('pos=top&t=1&pr=0&bg=cine&ui=frame2&rail=3&eb=ww&intl=8&cards=5&chat=1');")
# 2. strip the corner version badge
i=s.index('<div style="position:fixed;left:12px;bottom:12px;z-index:9999'); j=s.index('</div>',i)+6
s=s[:i]+s[j:]
# 3. inject GA4 (lab file does NOT have it — this is critical)
ga='''<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-5R1LWZKVPH"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-5R1LWZKVPH');
</script>
'''
s=s.replace('</head>',ga+'\n</head>',1)
io.open('/tmp/mpip-prod/premiere-color-mode-v17-cyan-blue-violet-magenta-latest.html','w',encoding='utf-8').write(s)
print('prod written')
PY

cd /tmp/mpip-prod && git add -A && git commit -q -m "MESSAGE" && git push -q origin main && echo PROD
cp ~/Downloads/mpip-hero-preview/hero-lab.html /tmp/mpip-hero-review-new/ && cd /tmp/mpip-hero-review-new && git add -A && git commit -q -m "MESSAGE" && git push -q origin main && echo BOARD
```

**Before overwriting the prod HTML, diff it first** — the live file can be ahead of the lab file. This caught a would-be regression of the contact hash-routing fix once.

---

## 🎨 CURRENT DESIGN TOKENS (verified in file, 2026-09-01)

**Event rail / international band (`.ben-*`, desktop):**
- Icons: `56px` SVG, borderless, no glow filter
- Titles (`.ben-tx b`): `18px`
- Subtext (`.ben-tx span`): base `14.5px` → **`.ben-chat` override sets 16.5px**, color `#C3CAD9`, weight `500`
- Flags (`.ben-chat .ben-flags`): **`19px`**, letter-spacing `.26em`
- Eyebrow (`.intl8 .ben-eb`): **`19px`, weight `541`, letter-spacing `.1em`**, transparent bg over hero image

**Icon optical compensation (do not "fix" these — they're deliberate):**
- Speaker icon: `transform="translate(12 12) scale(1.16) translate(-11.45 -12)"` — was smaller/off-center in its viewBox
- Replay icon: `transform="translate(12 12) scale(1.09) translate(-12.55 -11.7)"` — **circles must be ~9% larger than boxy shapes to look equal** (same principle as "O" vs "H" in type design). Measured gaps are intentionally unequal (21px vs 23px) so they *appear* equal.

**Backgrounds:** `--bg: #030611`. `#why-attend` uses `var(--bg)` (was hardcoded `#0B0F1B`, a lighter navy that visibly mismatched).

**Section separators:** every section is followed by `<div class="fade-div" aria-hidden="true"></div>`.

---

## 🧩 THE URL PARAMETER SYSTEM

`hero-lab.html` is a variant lab. Defaults apply **only when no `t` or `intl` param is present**:
`pos=top&t=1&pr=0&bg=cine&ui=frame2&rail=3&eb=ww&intl=8&cards=5` (+`chat=1` in prod)

Useful params:
- `?chat=1` — the shipped design (band layout, section 2 cards). **Always use this when previewing.**
- `?norv=1` — disables scroll-reveal animations. **Essential for headless screenshots** (otherwise content is invisible).
- `?ben=4` — adds back the 4th "Join from Anywhere" tile (cut; 3 tiles shipped)
- `?ben=3` / `?mreg=2` / `?copy=b` — other variants

---

## 📸 SCREENSHOT GOTCHAS (learned the hard way)

1. **Headless Chrome floors its layout viewport at ~500px.** Any `--window-size` below that renders at 500px and gets clipped — the screenshot looks broken when the page is fine. **For true 390px mobile, use an iframe harness:**
   ```html
   <!-- /tmp/m390/i.html, served on its own port -->
   <iframe src="http://127.0.0.1:8831/hero-lab.html?chat=1&norv=1" style="border:0;width:390px;height:16000px"></iframe>
   ```
2. **The Browser pane intermittently returns solid black frames.** Not a page bug. Fall back to headless Chrome.
3. **Always add `&norv=1`** to headless captures or reveal animations leave sections blank.
4. **Add a cache-buster (`&cb=$RANDOM`)** — but note a query string defeats the no-param defaults, so keep `chat=1` explicit.
5. **Files >4MB fail to upload** to the user. Slice tall pages into 3–5 JPEGs at quality 72–80.

---

## ✅ WHAT'S DONE AND LIVE

**Content**
- Hero: GLOBAL TRAINING EVENT pill, gradient subtitle, "Live on Zoom" meta, Premiere UI panel
- International band: 3 tiles — Translated Live Captions (35+) / Recordings in 7 Languages + flags / 180-Day Replay
- Section 2: Day 1 + Day 2 cards, each a link to `#day1` / `#day2`
- "Created for editors worldwide": 3 cells + Recording Languages flag row + link to Zoom's language list
- **Meagan Keane fully added**: speaker band (2nd position, after Alexis), Day 2 4:45 Q&A session, portrait in session row + drawer
- All Day 2 speakers assigned per Megan's grid (Luisa ×2 added, Eran ×1 added)
- All 12 grid speaker names link to their speaker bands
- Both Q&A rows credentialed; Meagan's grid title is topic-first ("Adobe's Approach to AI in Premiere")

**SEO** (post-audit, Tyree-approved copy)
- Title: `Modern Post in Premiere | Live Adobe Premiere Training` (54 chars)
- Description: 157 chars, leads with worldwide + 39 captions + 7 dubs + 180-day
- JSON-LD: WebSite + Organization + EducationEvent (4 performers incl. Meagan, `audienceType: "Professional video editors worldwide"`)
- **FAQPage schema deliberately REMOVED** — Google killed FAQ rich results May 2026, and schema without matching visible content violates their guidelines
- GA4 `G-5R1LWZKVPH` · canonical · og/twitter · sitemap `lastmod` current · indexing requested in Search Console

**Performance:** 5.87 MB → 2.29 MB (61% lighter)
- Killed a 2.5MB double-download (HTML hardcoded the desert hero with `fetchpriority=high`, then JS swapped it)
- Lossless WebP (mathematically bit-identical, max pixel diff = 0) + intrinsic dimensions

**Fixed bugs (root causes, not patches)**
- Sticky nav: `overflow-x:hidden` on html/body silently made them scroll containers, killing `position:sticky`. Now `overflow-x:clip`.
- Footer copyright: was positioned by **JS measuring pixels at runtime** with a font-load race — if the webfont loaded slower than a 400ms timer, position locked wrong forever. **JS deleted**; both footer rows now share identical `grid-template-columns` so alignment is deterministic. Copyright has `left:-9px` to match `.fcols`'s own nudge.
- Tablet speaker bands: bio text printed over faces at 768–1000px (stacking only kicked in ≤700px). Now stacks ≤1000px.
- Tablet footer: copyright overlapped Pr/Adobe logos.
- Meagan photo missing on some iPhones: session-row image was a bare `<img src=".webp">` with **no PNG fallback**, and `loading="lazy"` inside an `opacity:0` reveal wrapper (Safari can permanently skip that fetch). Both fixed.

---

## 📋 OPEN ITEMS

1. **Verify Meagan's photo on the affected iPhones** (iPhone Pro / Plus) — fix deployed, awaiting real-device confirmation.
2. **Two Slack messages never sent:**
   - *Team:* ask for postinpremiere.com to be linked from FMC's events page, and ask the Adobe contact for a link from adobe.com. **An Adobe inbound link does more for search authority than any remaining on-page work.**
   - *Tyree:* AEO note — but it's now stale (FAQ schema was removed). Shorter version: "Pulled the FAQ schema — Google's guidelines require a visible match and they killed FAQ rich results anyway. Any other AEO moves you'd want?"
3. **Email campaign** — separate thread. Needs the same Day 2 speaker assignments + Meagan insert.
4. **Parked, low value:** heading restructure (H1 split), Day 1/Day 2 `subEvent` schema, speaker `Person` enrichment.

---

## 🧠 CONFIRMED FACTS (don't re-litigate)

- **Live captions:** Zoom translated captions. **35+** is the safe public number (Zoom's list is 36 two-way + 3 receive-only = 39; automated extraction gave inconsistent totals, so "35+" is what's on the band). "39" appears only in the meta description.
- **Dubbed recordings:** 7 languages only — English, Spanish, French, German, Japanese, Portuguese, Italian. **Chosen at registration, NOT "on request."** Every "on request" string was purged site-wide.
- **Delivery timing (Megan, Slack):** English immediately after sessions; dubbed ~1 week; 180 days counts **from delivery**. ⚠️ **This is deliberately NOT on the site** — Megan said "our goal is a week," which is an internal target, not a commitment. Keep it in the registration confirmation email instead.
- **Sessions are presented in English.**

---

## 💬 WORKING PREFERENCES

- Just say "done" + link. No change rundowns unless asked.
- **Change only what's asked.** Never touch anything adjacent.
- Verify the **rendered** result (screenshots / computed styles), never just the source.
- Check mobile (390) AND large desktop (2560) by default.
- Never auto-open a browser — give a clickable localhost link.
- Publish only on explicit go-ahead.
- Voice-dictated messages: infer intent through typos.
- **Diagnose root causes, don't patch symptoms.** Several bugs here recurred for weeks because they were patched, not understood.
