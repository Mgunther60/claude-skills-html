---
name: html5-display-ads
description: Build production-ready animated HTML5 display ads (banners) for Google Ads / Google Display Network and other ad servers, one ZIP per size, with clickTag click tracking, from a video, Figma frames, PNG exports or a brief. Use this whenever someone asks for HTML5 ads, animated banners, display ads, GDN or Google Ads creative, ad resizes (300x250, 728x90, 160x600, 320x50 and so on), a banner set "based on this video/Figma", clickTag, or "make each size a separate zip", even if they only say "banners" or name sizes. Also use it to fix, resize, re-crop or re-brand ads built this way.
---

# HTML5 display ads

Builds a set of animated HTML5 banners, one self-contained ZIP per size, that pass Google Ads' HTML5 checks.
The scripts in `scripts/` do the mechanical work (build, QA, preview, image prep); your job is getting the
**sources** and **layouts** right, which is where nearly all the back-and-forth comes from.

Read `references/google-ads-html5.md` before the first build (limits, clickTag, sizes, upload steps).
Read `references/sources-and-figma.md` as soon as the request involves Figma, a video or exported frames.
Read `references/spec-format.md` when writing layout specs. `references/example-closet-world-bf2026.md`
is a complete worked example (21 sizes) to copy patterns from.

If `scripts/` and `references/` aren't next to this file (for example only SKILL.md was uploaded), follow the
same workflow by hand: write each ad's `index.html` with the same structure (ad.size meta, `var clickTag` in the
head, a whole-ad `window.open(window.clickTag)` exit, CSS-class timeline, `#end` hook, 1px border), zip each size
with `index.html` at the root, and run the same checks in a headless browser.

If the work is for a brand with its own design-system skill (for example Closet World's cw-foundations
and cw-social), read that skill too: it decides logo, colour, type and copy rules. This skill decides the
ad format.

## Workflow

### 1. Intake (ask only what you can't infer)
Get: the sizes (if the user says "all sizes" or names none, use the standard 20-size set below); the offer and headline copy; the CTA wording; the click-through URL (a default is fine,
because Google Ads replaces clickTag with the final URL); and the **source of truth for the design**: a video
cut, Figma frames, PNG exports, or nothing (then you design from the brief and the brand rules).
Check uploads yourself: a file the user mentions may not have arrived. Look in `/mnt/user-data/uploads`.

**Standard set (20 sizes).** Square: 200x200, 240x400, 250x250, 250x360, 300x250, 336x280, 580x400.
Skyscraper: 120x600, 160x600, 300x600, 300x1050. Leaderboard: 468x60, 728x90, 930x180, 970x90, 970x250,
980x120. Mobile: 300x50, 320x50, 320x100. Keep this list in a variable and build from it, so no size is
dropped between rounds.

### 2. Sources: get the real artwork, at enough resolution
The quality of every ad depends on this step. In order of preference:
1. **Official files** the user uploads (logo, lockups, photos). Ask for them by **layer name** and export scale.
2. **Figma** via the connector: `get_metadata` gives frame sizes and layer coordinates; `get_screenshot` with
   `enableBase64Response` lets you *see* a frame. Downloads from figma.com are often blocked in the sandbox
   (curl exit 6), so you usually cannot save Figma pixels. Ask the user to export specific layers instead.
3. **Video frames** (`layers.py frames`), with artwork lifted off flat backgrounds (`layers.py key`) and
   products cut out (`layers.py cutout --region`). These are sharp (1080 px) but tightly framed.
4. **1x frame exports** from the user: exact framing, but soft. Combine them with a sharper source
   (`layers.py find` locates a crop inside a sharper image; `layers.py blend` merges a sharp render into a soft export).

Always say plainly where each asset came from and what it costs (for example "logo lifted from the video, so it's
softer; send the official file to fix"). Users notice soft logos first.

### 3. Layout per size
- **When a Figma frame or exported frame exists for a size, match it exactly.** Measure positions from the PNG
  (colour runs for panels, alpha or non-background bounding boxes for art); don't eyeball. Check the PNG's
  real pixel size: frames get named wrong (a "120x600" exported at 160 wide, a "970x90" drawn at 960).
  Build at the Google size, put any extra pixels somewhere neutral (a solid panel), and say so.
- **When a size has no frame, derive it from the closest approved size** in the same family (see
  `references/spec-format.md` for family patterns), tell the user it is your adaptation, and offer to adjust.
- Users often send frames per scene (frame 1, 2, 3). Treat them as the per-scene truth for photos, crops and
  any element that moves (for example a logo that sits lower when the tagline appears).
- Flipped Figma layers report confusing x positions. Measure from the screenshot or export instead.

### 4. Prepare assets
- Images at **2x** the CSS size (photos as JPG, quality 60-78), artwork as palette PNGs
  (`quantize`, 16-64 colours). Logos at **3x** (they are what people look at; `layers.py logo` does this).
- Products that straddle a two-colour background: flatten each scene onto the ad's own background
  **full-width** at a whole-pixel split row, so there is no visible seam where an image edge meets CSS.
- Keep each ZIP **at or under 150,000 bytes**; trim JPG quality or drop large photos to 1.5x if needed.

### 5. Build
Write one JSON spec per size (`references/spec-format.md`), then:
```
python scripts/build_ad.py specs/300x250.json --assets prepared/300x250 --out build
```
It writes `build/300x250/index.html` plus assets and `build/<prefix>_300x250.zip` (files at the ZIP root),
and prints bytes and file count. The template already has: `<meta name="ad.size">`, `var clickTag` in the head,
a whole-ad exit `window.open(window.clickTag)`, the shared motion timeline, a 1px border, reduced-motion
support and an `#end` URL hook that jumps to the end frame.

The motion (keep it identical across a set unless asked): headline in (0.2 s), offer pops (0.9 s), scene 2
(2.4 s), scene 3 with headline-to-tagline swap (4.8 to 5.1 s), end card slides up with offer, logo and CTA
(7.6 to 8.7 s), CTA shine; plays twice, then rests on the end card at about 21 s (under Google's 30 s).

### 6. QA (every size, every time)
```
python scripts/qa_ads.py build/300x250 build/728x90 --zips build/*.zip --timing
```
It checks the ad.size meta, clickTag declaration and exit, CSS brace balance, page and console errors, external
requests, broken images, elements spilling outside the ad, end card showing, CTA text clipping, the click
opening the clickTag URL, the resting state after two plays, ZIP root, bytes and file count. Fix and re-run
until everything passes. Then look at the result: render each scene state (classes `still s1 s2`,
`still s1 s2 c2`, `still s1 s2 c2 c3 s3`, and the end) and compare it side by side with the reference frames.
A pixel diff by region (left zone, photo, offer) quickly shows what's off.

### 7. Preview
```
python scripts/record_preview.py build/300x250 build/728x90 --out outputs/preview_name --zoom 2
```
This writes a true-speed MP4 and GIF (it retimes the capture with an on-page clock). Check four frames
(about 1.5, 3.4, 6.0 and 9.5 s) before presenting: headline and offer, scene 2, tagline, end card.

### 8. Deliver
- **Check coverage first:** compare the delivered ZIP names against the requested size list and build anything
  missing before replying (a 250x250 once went missing for several rounds). Re-run QA on the whole set after any
  change to a shared asset (logo, offer, template).
- One ZIP per size, named `<Brand>_<Campaign>_<WxH>.zip`, copied to `/mnt/user-data/outputs/` and presented
  with the preview GIF first. If you rebuild a ZIP, confirm the delivered file's contents changed (`cmp` a file
  inside it), because timestamps can look stale.
- For a full set, also deliver an all-sizes package (README, an overview image of every size, and the ZIPs).
- In the reply: what's in each file, the motion in one short sequence, sizes in KB, which layouts follow the
  Figma frames and which are your adaptation, and any compromise (soft source, changed CTA wording, 960 to 970).
- First-time users: explain that the HTML file is inside the ZIP, that they upload the ZIP itself, and that
  unzipping and double-clicking `index.html` previews it (the images must stay next to it).

## Gotchas learned the hard way
- **Class names collide with state classes.** The timeline adds `c2`/`c3` to `#ad`; never give elements
  those class names (an image class `.c2{opacity:0}` once hid the whole ad). The template uses `sc2`/`sc3`.
- **Brace typos kill the rules after them.** A stray `}` silently disabled the end card. The builder and QA
  both check brace balance.
- **Video encodings shift colours slightly; Figma is exact.** Take brand colours from Figma (`get_design_context`
  on a solid layer) or the brand skill, not from video pixels. Note any conflict between the creative's reds
  and the brand standard instead of silently "fixing" it.
- **Headless capture runs 10-35% slow and unevenly.** Never trust a recording's raw timing; use `record_preview.py`.
- **Text at 50 px tall.** "Schedule a free consultation" doesn't fit a 300x50 end card; use a short CTA
  ("Book now") there and keep the full wording where it fits. Tell the user.
- **Google counts 150 KB.** Stay at or under 150,000 bytes so either reading of the limit passes.
- **CTA text crowding.** A label slightly wider than the space inside the padding stays visible but sits off-centre.
  `qa_ads.py` warns about this; reduce the button padding (14 to 10 px) or the font size.
- **Shared-asset changes touch every ZIP.** When a logo or lockup is replaced, rebuild every size, re-run QA,
  replace or delete stale bundles (for example an old all-sizes ZIP), and confirm each delivered file changed.
- **Ask once, precisely.** When you need files, list the exact layers, frames and export scale in a table, plus the
  alternative (an org owner can allowlist `www.figma.com`). Don't ask again for something already answered.
